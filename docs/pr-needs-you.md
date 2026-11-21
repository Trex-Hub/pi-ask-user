# feat: report pi's "blocked on user" status while ask_user waits (Needs You)

Branch: `needs-you` (one commit on top of upstream `main` 098a678)

## Problem

pi reports when a session is waiting on the user through its terminal program status (OSC 7501). Terminals and multiplexers read it; herdr shows such a session as **Needs You**. pi's own `input()`/`select()` dialogs set this status, but `ctx.ui.custom()` only does so when the caller passes `blocked`, and `ask_user` didn't. So while an `ask_user` question was open, herdr showed the agent as working instead of waiting on you. That was true inline and in the overlay, for a single question and for a batch. The prompt that most needs the user was the one place the status was missing.

## Change

`buildCustomUIOptions()` now always passes `blocked: { kind: "question", message }` to `ctx.ui.custom()`:

- In inline mode it returns `{ blocked }` instead of `undefined`.
- In overlay mode it adds `blocked` to the overlay options.
- `message` is the question for a single question, and the heading or `"N questions"` for a batch. Whitespace and newlines are collapsed to one line of at most 80 characters, with `…` at the end if it was cut.
- Hiding the overlay (`alt+o`) doesn't change it: the prompt is still open, so the session is still blocked.
- Older pi versions ignore the unknown option, so nothing changes for them.

The RPC/headless fallback and the options-less `input()` already report blocked through pi's own dialogs, so they're untouched.

## Tests

`bun test`: 188 pass (181 existing + 7 new). `bun run check` is green.

- New `describe("Needs You (blocked program status)")`:
  - For inline and for overlay: a single question, a batch heading, and a batch without a heading (`"2 questions"`) each get the expected `blocked`.
  - A long multi-line question becomes one line of 80 characters ending in `…`.
- Five existing tests asserted that inline mode passes `undefined` options. They now check `options.overlay` (or the whole `{ blocked }` object), which keeps their intent.

## Example

```ts
ctx.ui.custom(factory, {
   overlay: true,
   overlayOptions: { anchor: "center", width: ASK_OVERLAY_WIDTH, minWidth: ASK_OVERLAY_MIN_WIDTH, maxHeight: "85%", margin: 1 },
   blocked: { kind: "question", message: "Which database should we use?" },
   onHandle,
});
```

Ask a question in pi inside herdr: the pane shows **Needs You** until you answer or cancel, even while the overlay is hidden with `alt+o`.
