# Plan Delta — Thinking-strip via extension hook

The original PLAN.md assumed `compat.skipThinkingBlock` exists on `OpenAICompletionsCompat` in the user's installed `@earendil-works/pi-ai`. **It does not** — the latest published npm version (0.79.1) predates the upstream fix commit `b85b91c9`, which adds this flag.

User decision: implement thinking-stripping in the extension itself, using available extension hooks.

## Updated strategy

1. **Routing fix (unchanged):** register `minimax-m3` and `minimax-cn-m3` providers on `/v1/chat/completions` so passive caching works. This is the main cost-saving benefit.
2. **Thinking-strip (new):** register a `message_end` hook that, for assistant messages from these two providers, replaces the final message with one whose `thinking` blocks are dropped and whose `text` blocks have the <think>…</think> markers (and any inner content) stripped. This mirrors what the upstream `skipThinkingBlock` does at end-of-stream.

## Limitations

- **Streaming display:** the TUI shows the thinking markers during streaming. The cleanup happens at `message_end` and replaces the finalized message. Users will see a brief visual flash of the duplicated thinking.
- **Hook coverage:** `message_end` only fires for messages produced by the agent loop. If a caller uses `streamSimple` directly (not via the agent), the hook is not invoked. This is fine for normal pi use.

## What the extension does NOT do

- Does **not** set `compat.skipThinkingBlock: true` (the field doesn't exist in installed pi-ai 0.79.1).
- Does **not** touch the `reasoning_content` stream — it accepts that a thinking block may be created from it, and drops it at `message_end`.

## Files affected

- `index.ts` — added `message_end` handler that strips thinking.
- `README.md` — documents the dual approach and the visual flash during streaming.
- T2, T7, T8 acceptance criteria updated to match.
