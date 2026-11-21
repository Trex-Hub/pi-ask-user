# pi-ask-user (fork): architecture of the changes

Status: plan, nothing built yet.
Fork of `edlsh/pi-ask-user` (MIT, Enzo Lucchesi), based on v0.17.2. Our remote: `git@github.com:Trex-Hub/pi-ask-user.git`.
Published as `@dev8-npm/pi-ask-user` (npm user scope; install with `pi install npm:@dev8-npm/pi-ask-user`). Keep the original `LICENSE` with Enzo's copyright line and add ours; the README credits upstream.
Companion doc: [`../../pi-live/docs/architecture.md`](../../pi-live/docs/architecture.md). Read §1 there first.

## 1. Problems we're solving

1. **No "Needs You".** When `ask_user` shows a prompt, herdr (and so herdr-web) still shows the agent as working.
   - **Root cause:** pi reports agent status to the terminal with OSC 7501 (`idle | working | blocked | done | error`, in `dist/modes/interactive/program-status-reporter.js`). For `ctx.ui.custom()` it reports `blocked` only if the caller passes `options.blocked` (`types.d.ts:143`: `blocked?: { kind: "permission" | "question" | "auth"; message: string }`). `ask_user` never passes it.
   - The `herdr:blocked` bus event it emits instead is only read by `~/.pi/agent/extensions/herdr-agent-state.ts`, which does nothing unless `HERDR_ENV=1`, and herdr-core never sets that. pi's native `ctx.ui.input()`/`select()` already report blocked, so only the custom UI path is broken.
2. **Can't answer from outside.** The prompt's `complete()` is local to `runCustomPrompt` (`index.ts:2700`). Only a keypress, the abort signal or the timeout can resolve it.
3. **Events can't be tied to a tool call.** `execute(_toolCallId, …)` (`index.ts:3142`) drops the id, so `herdr:blocked`, `ask:answered` and `ask:cancelled` can only be matched by question text.

## 2. Decision: what changed from the original brief

The first brief proposed an `ask:answer` bus listener so *another extension* could answer, behind an `acceptExternalAnswers` setting. We dropped that:

- The tool should be self-sufficient, not depend on some other extension being the answerer.
- Clients (herdr-web today, a phone app tomorrow) shouldn't each need a pi extension. They should all **connect to the session** and pick up what's waiting.

So the split is:

| Concern | Owner |
|---|---|
| Show the prompt, keyboard, validation, result shape | **ask_user** (this fork) |
| "Needs You" | **ask_user**, via pi's own `blocked` option |
| Door to the outside, list of open items, clients, first-answer-wins bookkeeping | **pi-live** (new package) |
| Getting answers from a browser or phone to the door | herdr-web backend / bridge |

`ask_user` talks to pi-live only through `pi.events` (`pi-live:park`). It never opens a socket and never imports pi-live. If pi-live isn't there, `ask_user` behaves exactly as it does today.

## 3. Changes, in order

They're split so the first two can go upstream on their own.

### Change A: report "Needs You" (upstreamable, tiny)

`buildCustomUIOptions` (`index.ts:520`) gets a `blocked` argument and passes it in **both** modes. Today inline returns `undefined`.

```ts
function buildCustomUIOptions(
   displayMode: AskDisplayMode,
   blocked: { kind: "question"; message: string },
   onHandle?: (handle: OverlayHandle) => void,
) {
   switch (displayMode) {
      case "inline":
         return { blocked };
      case "overlay":
         return { overlay: true, overlayOptions: { … }, blocked, ...(onHandle ? { onHandle } : {}) };
      …
   }
}
```

- `message` is the question text, or for a batch the heading or "N questions", cut to about 80 characters.
- pi keeps the status `blocked` while an overlay is hidden with alt+o (the overlay still exists), and it drops it when the custom UI closes. So no extra bookkeeping.
- Older pi versions ignore the unknown option, so `peerDependencies` stays `>=0.74.0`.
- `herdr:blocked` stays as it is for backward compatibility.

**Test:** a fake `ui.custom` records its `options`. Assert that `blocked` is set for inline and overlay, for single and batch.

### Change B: pass the tool call id through (upstreamable)

- `execute(toolCallId, …)` passes the id into the single path, `executeBatch`, `whileBlocked` and `createAskEventEmitter`.
- Add `toolCallId` to `herdr:blocked` (both `active: true` and `false`), `ask:answered` and `ask:cancelled`. Keep every existing field; only add.
- README Events section: document the new field.

**Tests:** each event carries the id. Two calls with the same question text produce events with different ids.

### Change C: park on pi-live (our fork)

This is the core. It's described in §4.

### Change D: parking from the other prompt paths (our fork, phase 2)

These are the RPC fallback (`askViaDialogs` `:2502`, `askBatchViaDialogs` `:2582`) and the options-less question that calls `ctx.ui.input` directly (`:3204`). See §4.6.

### Not doing

- No `ask:answer` listener and no `acceptExternalAnswers` setting (see §2).
- No `ask:opened` bus event in v1. The pi-live item covers it for clients. If upstream wants a bus-level "opened" event later, it's a one-line `emit` next to the park.
- No UI, keybinding or result-shape changes. No refactors.

## 4. Change C in detail

### 4.1 Where the hook goes

`runCustomPrompt<T>(ui, request)` (`index.ts:2700`) already owns everything an outside answer needs: the one-shot guard (`customCompleted`), `complete()`, the abort listener, the timeout timer and the cleanup. So the park lives there. Both the single prompt (`AskComponent`) and the batch prompt (`BatchAskComponent`) go through it.

Extend `CustomPromptRequest<T>` with one optional field:

```ts
interface CustomPromptRequest<T> {
   …existing fields…
   /** Offer this prompt to pi-live clients. Omitted = keyboard only (today's behavior). */
   park?: {
      id: string;                       // toolCallId
      kind: "ask_user.question" | "ask_user.batch";
      title: string;
      view: unknown;                    // §5
      /** Validate a raw client answer against this prompt's params. */
      parse: (raw: unknown) => { ok: true; value: T } | { ok: false; reason: string };
   };
}
```

### 4.2 What `runCustomPrompt` does with it

Inside the custom UI factory, which runs once the UI is actually on screen (the same place `complete` is created), emit the park:

```ts
let ticket: ParkTicket | undefined;
let closedBy: CloseOutcome = { by: "error" };

if (request.park) {
   const park = request.park;
   pi.events.emit("pi-live:park", {
      protocol: 1,
      id: park.id,
      owner: "ask_user",
      kind: park.kind,
      title: park.title,
      view: park.view,
      ...(request.timeout ? { expiresAt: Date.now() + request.timeout } : {}),
      answer: (raw, from) => {
         if (customCompleted) return { status: "invalid", reason: "not open" };
         const parsed = park.parse(raw);
         if (!parsed.ok) return { status: "invalid", reason: parsed.reason };
         closedBy = { by: "client", client: from };
         complete(parsed.value);          // same guarded path as a keypress
         return { status: "delivered" };
      },
      accept: (t) => { ticket = t; },
   });
}
```

- Every other way the prompt ends sets `closedBy` before calling `complete`: keyboard → `{ by: "keyboard" }`, the existing timeout → `"timeout"`, the existing abort → `"abort"`. The `finally` that already releases the abort listener, timer and overlay toggle also calls `ticket?.close(closedBy)`.
- `runCustomPrompt` needs `pi` (or just `pi.events`), so thread it in. Today it only takes `ui`.
- `emit` is synchronous (pi's bus is a Node `EventEmitter`). If `ticket` is still `undefined` right after it, pi-live isn't loaded or is disabled, and nothing more happens. This is the "keyboard only" fallback, and it costs nothing.

### 4.3 Why this gives the right race behavior

- `customCompleted` is checked first in `answer()`, so a client answer after a keypress, timeout or abort returns `invalid "not open"`, and pi-live has already marked the id closed for everyone else.
- `complete()` sets `customCompleted` synchronously, so a keypress after a client answer is ignored by the existing guard.
- Everything runs on one thread. Two answers can't interleave between the check and `complete()`.

### 4.4 What the user sees

`complete(value)` closes the custom UI, and the tool returns exactly what a keyboard answer returns. That means the same `content` text (`User answered: Green`), the same `details` (`AskToolDetails` / `AskBatchDetails`) and the same `renderResult` line (e.g. "✓ Green"). `ask:answered` fires as usual.

This works the same in inline and overlay mode, and while the overlay is hidden with alt+o. The toggle only hides it; `complete()` still tears it down.

v1 deliberately does **not** record who answered in `details`. A tool result that's identical to a keyboard answer means the agent and the transcript can't tell, and no renderer changes are needed. pi-live broadcasts `item_closed { by: "client", client }` for UIs that care.

### 4.5 Validation (`parse`)

`parse` turns a raw client answer into exactly the `AskUIResult` / batch result the keyboard UI would produce, or rejects it with a short reason. It reuses the existing normalizers (`createSelectionResponse` `:284`, `createFreeformResponse` `:279`) so trimming and empty handling match the keyboard.

**Single question**: the answer is an `AskResponse`:

```ts
type AskResponse =
   | { kind: "selection"; selections: string[]; comment?: string }
   | { kind: "freeform"; text: string };
```

Rules, checked in this order (the first failure becomes the reason):

1. It must be an object with `kind` either `"selection"` or `"freeform"`.
2. For `selection`:
   - `selections` is a non-empty array of strings,
   - each one equals an offered option's `title` (exact match after trim),
   - there are no duplicates,
   - if `allowMultiple` is false, there's exactly 1.
3. For `selection` with a `comment`: allowed only if the resolved `allowComment` is true (the same value the UI uses, from params plus settings). It must be a string, and an empty one is dropped.
4. For `freeform`: allowed only if `allowFreeform`, and `text` must be non-empty after trim.
5. Unknown extra fields are ignored.

**Batch**: the answer is `{ answers: BatchAnswer[] }`:

```ts
type BatchAnswer = { status: "answered"; response: AskResponse } | { status: "skipped" };
```

1. `answers.length === questions.length`.
2. Each `answered` entry passes the single-question rules against its own question's `options`, `allowMultiple` and `allowFreeform`, plus the batch's `allowComment`.
3. `skipped` is allowed for any entry, because the keyboard review page allows skipping. At least one entry must be `answered`, matching what the review page lets you submit. Verify that against `BatchAskComponent` while implementing, and copy the keyboard's rule exactly.

Reasons are short and readable, for example `option "Purple" not offered`, `only one selection allowed`, `freeform not allowed`, `comment not allowed`, `expected 3 answers, got 2`. pi-live passes them to the client unchanged.

**Parity test:** for each answer type, drive the keyboard UI and a parked answer to the same choice, and assert `deepEqual` on the whole tool result (`content` and `details`).

### 4.6 Other prompt paths (phase 2)

Neither the RPC fallback (`ctx.ui.custom()` returns `undefined` → `askViaDialogs` / `askBatchViaDialogs`) nor the options-less question (`ctx.ui.input` at `:3204`) has a `complete()` we own. Both accept an `AbortSignal`. Plan:

1. Make an `AbortController` linked to the tool's `signal`, and pass its signal to the dialogs.
2. Park with the same `view` and `parse`. On a valid answer, store the parsed value and `abort()` the linked controller.
3. When the dialog returns (now `undefined`/null because of the abort), return the stored answer if there is one; otherwise treat it as cancelled, as today.
4. An abort of the *outer* signal still means cancel. Check `signal.aborted` (the outer one) to tell the two apart.

**Risk:** in RPC mode the dialog is shown by the RPC client (`extension_ui_request`). Aborting resolves it inside pi, but the client may not close its dialog. Test with pi's RPC mode before relying on it. If the client keeps a stale dialog, document parking as **interactive-only** for RPC and leave that path keyboard/dialog-only.

The options-less `ctx.ui.input` path is in-process TUI, so the abort trick should be clean there. It's worth doing in phase 2. Alternatively, route it through `runCustomPrompt` with a small input component, but that's a UI change, so prefer the abort trick.

## 5. The view (what clients render)

`view` is plain JSON and the contract with every client (herdr-web, CLI, phone). It's versioned through `kind`. Breaking changes need a new kind (e.g. `ask_user.question.v2`); adding fields is fine.

**`ask_user.question`**

```jsonc
{
  "question": "Which color?",
  "context": "We need a theme for…",          // omitted if none
  "options": [ { "title": "Green", "description": "Calm" }, { "title": "Blue" } ],
  "allowMultiple": false,
  "allowFreeform": true,
  "allowComment": false,
  "answer": { "kind": "selection | freeform" } // hint only; see the AskResponse shape in §4.5
}
```

`options` may be empty (the options-less question): clients show a text box and send `freeform`.

**`ask_user.batch`**

```jsonc
{
  "heading": "Setup",                          // omitted if none
  "allowComment": false,
  "questions": [
    { "question": "…", "context": "…", "options": [ … ], "allowMultiple": false, "allowFreeform": true }
  ]
}
```

`title` (the pi-live item field) is the question text, or for a batch the heading or `"N questions"`.

**Privacy:** the view includes `context` and option descriptions always. That's different from `emitFullEvents` (issue #51), which controls what goes to *every installed extension* on the bus. The view goes only to pi-live's user-only socket, and clients can't render a question without it. Document this in the README.

## 6. Packaging: shipping pi-live inside ask_user

Installing our `ask_user` should give you pi-live without a second step. pi supports this (`docs/packages.md`, "Declare dependencies").

```jsonc
// package.json (fork)
{
  "name": "@dev8-npm/pi-ask-user",
  "publishConfig": { "access": "public" },   // scoped packages publish private by default
  "dependencies": { "@dev8-npm/pi-live": "^1.0.0" },
  "bundledDependencies": ["@dev8-npm/pi-live"],   // "must be included in the published tarball"
  "pi": {
    "extensions": ["./index.ts", "./node_modules/@dev8-npm/pi-live/index.ts"],
    "skills": ["./skills"]
  }
}
```

- **Loading twice is safe.** If someone also installs `pi-live` directly, or another package bundles it, pi loads separate copies ("installed packages load with separate module roots"). pi-live's `globalThis` guard means only the first one opens a door (pi-live doc §4).
- **No import, only the bus.** A direct import could reach a different copy from the one that won the guard.
- **Fallback stays.** If pi-live is turned off (`PI_LIVE_ENABLED=0`), fails to open the socket or is removed, the park isn't accepted and `ask_user` is keyboard-only. In normal use you never see this path, but it keeps the tool self-sufficient.
- **Upstream PR:** changes A and B go upstream without any of this. Change C could go upstream too, as an optional "park on pi-live if present" with no hard dependency. That's the author's call. Our fork carries the bundling regardless.

## 7. Settings

No new `ask_user` setting in v1. The `pi-live` setting `enabled` is the single switch. If upstream wants a separate switch, add `parkOnLive` (env `PI_ASK_USER_PARK_ON_LIVE` plus `ask-user.json`, read like `emitFullEvents` via `loadAskUserSettings` in `settings.ts`), defaulting to on, since pi-live already requires an explicit install.

## 8. Repo layout and workflow

```
/Users/dev/Desktop/devendras-work/pi-extension/
├── pi-live/         # new package (companion doc)
└── pi-ask-user/     # the fork; move or re-clone from /Users/dev/Desktop/devendras-work/pi-ask-user
```

- Branches:
  - `needs-you` (change A, its own upstream PR),
  - `tool-call-id` (change B, its own upstream PR),
  - `pi-live-park` (change C, our fork),
  - `pi-live-park-dialogs` (change D).
- Local commits only. No pushing without approval.
- Run `bun install` first. `bun test` currently fails with `Cannot find module '@earendil-works/pi-tui'` until deps are installed. The whole existing suite (`index.test.ts`) must stay green, along with `typecheck`, `check:host`, `check:docs` and `check:package`.
- Match the repo style: 3-space indent in `index.ts`, the existing naming and the comment voice. No refactors beyond threading `pi`/`toolCallId` and the `park` field.

## 9. Tests (bun test, in `index.test.ts` style)

Use a fake pi with a real `EventEmitter`-backed `events`, and a fake pi-live: a `pi-live:park` handler that calls `accept` and keeps the request so the test can call `answer()` directly. pi-live's own socket tests live in that package.

- **A, Needs You:** `blocked` is passed for inline/overlay and single/batch, with the right message.
- **B, ids:** `toolCallId` is on `herdr:blocked` (both states), `ask:answered` and `ask:cancelled`. Same text in two calls gives different ids.
- **Park:** single and batch prompts park with the right `id`, `kind`, `title`, `view` and `expiresAt`. With no pi-live (no handler, or a handler that doesn't `accept`), behavior and results are unchanged from today.
- **Answer types:** single choice, multiple choice, freeform, choice plus comment, and batch with a skip. Each resolves the prompt, and the tool result `deepEqual`s the keyboard result.
- **Rejections:**
  - option not offered,
  - two selections when single,
  - freeform when not allowed,
  - comment when not allowed,
  - wrong batch size,
  - a malformed object.

  Each returns `invalid` with a reason, and the prompt stays open and can still be answered by keyboard.
- **No double resolve:**
  - client answer then timeout,
  - abort then client answer (→ `not open`),
  - keypress then client answer (→ `not open`),
  - client answer then keypress (ignored).
- **Close outcomes:** `ticket.close` is called exactly once, with `keyboard`, `client`, `timeout`, `abort` or `error` as appropriate.
- **Hidden overlay:** an answer while the overlay is toggled off still resolves, and the UI is torn down.
- **Phase 2:** an answer during the options-less input and during the RPC dialogs; the outer abort is still a cancel.

## 10. Manual check

Use path loads only. Don't install into `~/.pi` or change the existing setup:

```bash
cd /Users/dev/Desktop/devendras-work/pi-extension
(cd pi-live && bun install) && (cd pi-ask-user && bun install)
PI_LIVE_DIR=/tmp/pi-live-dev pi -e ./pi-live -e ./pi-ask-user
```

Note: `-e ./pi-ask-user` doesn't install its dependencies, so load pi-live explicitly with `-e`. The guard makes a duplicate load harmless.

Then confirm:

1. Ask the agent to call `ask_user`. herdr shows **Needs You** (inline and overlay, and still while the overlay is hidden).
2. `bun pi-live/bin/pi-live.ts watch <session>` shows `item_opened` with the view.
3. `pi-live.ts answer <session> <id> '{"kind":"selection","selections":["Green"]}'` gives `delivered`. The terminal shows "✓ Green", and the agent continues with "Green".
4. An invalid answer gives `invalid`, and the prompt stays open. Answering it with the keyboard still works, and `watch` shows `item_closed by keyboard`.
5. With `PI_LIVE_ENABLED=0`, there's no socket and everything works as before, including Needs You.
6. A batch, answered from the CLI with one skip.

## 11. Build order

1. Change A, with tests and a manual Needs You check in herdr. This fixes problem 1 on its own and is worth shipping first.
2. Change B, with tests.
3. pi-live core (companion doc, steps 1 to 4).
4. Change C, with tests and the manual check.
5. herdr-web client: render the views, send answers.
6. Change D (phase 2), after testing RPC client behavior.
7. PR descriptions for A and B upstream. Pitching C upstream is optional.

## 12. Open questions

- Whether upstream wants C at all, or only A and B. Our fork works either way.
- The batch "at least one answered" rule must be copied from `BatchAskComponent`, not assumed (§4.5).
- Whether herdr-web should show "answered from <client>" in the terminal. v1: no (§4.4).
- Header chip, "chat about this" and ranking (ideas from Claude's AskUserQuestion) are out of scope here, but the view format leaves room for them as new fields.
