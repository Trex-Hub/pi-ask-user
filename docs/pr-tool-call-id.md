# feat: add toolCallId to herdr:blocked, ask:answered and ask:cancelled

Branch: `tool-call-id` (one commit on top of upstream `main` 098a678, independent of `needs-you`)

## Problem

Listeners on `herdr:blocked`, `ask:answered` and `ask:cancelled` can't tell which `ask_user` call an event belongs to. The default (redacted) payloads carry only the question text, so two calls asking the same question, say "Continue?" twice in one session, produce identical events. A listener that tracks open prompts, like a status bar or a remote UI, can't match "blocked" to "answered", and gets confused when calls overlap.

## Change

`execute()` already receives the tool call's id. It's now threaded through `executeBatch()`, `whileBlocked()` and `createAskEventEmitter()`, and added as `toolCallId` to:

- `herdr:blocked`: both `{ active: true, label, toolCallId }` and `{ active: false, toolCallId }`
- `ask:answered` and `ask:cancelled`, in redacted and full mode (`PI_ASK_USER_EMIT_FULL_EVENTS`)
- every question of a batch, which carries the batch call's id next to its `batch: { index, total }`

The change only adds fields. No existing field is renamed, removed or changed, so current listeners keep working. The README's Events section documents the field.

## Tests

`bun test`: 186 pass (181 existing + 5 new). `bun run check` is green, including `check:host`, whose RPC smoke test now expects `toolCallId: "smoke-rpc"`.

- New `describe("tool call id on events")`:
  - A single answer: `herdr:blocked` (true and false) and `ask:answered` carry the id.
  - A cancel with full events on: `ask:cancelled` carries the id.
  - The options-less `input()` path carries the id on all three events.
  - A batch, answered and cancelled: every event carries its call's id.
  - **Two overlapping calls with the same question text** (`"Color?"`) get distinct ids. The interleaved `herdr:blocked` events and the `ask:answered`/`ask:cancelled` events match the right call.
- Existing tests that asserted exact payloads now include `toolCallId`.

## Example

```ts
pi.events.on("herdr:blocked", (e) => console.log(e));
// { active: true, label: "Waiting for user response", toolCallId: "call_1" }
// { active: false, toolCallId: "call_1" }

pi.events.on("ask:answered", (e) => console.log(e));
// { question: "Color?", response: { kind: "selection" }, toolCallId: "call_1" }
```
