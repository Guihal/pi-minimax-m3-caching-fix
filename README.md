# pi-minimax-m3

A standalone [pi](https://github.com/badlogic/pi-mono) extension that fixes two
issues with the built-in **MiniMax-M3** integration:

1. **Silent over-billing on the Anthropic-compatible endpoint.** M3's
   `/anthropic/v1/messages` endpoint ignores `cache_control` markers, so every
   turn was billed at the full input price ($0.60/Mtok) instead of the
   cache-read price ($0.12/Mtok). M3 *does* support passive/automatic prompt
   caching on its OpenAI-compatible endpoint (`/v1/chat/completions`).
2. **Duplicated thinking in the response.** M3 emits thinking content twice:
   once in `reasoning_content` (consumed by pi as a `thinking` block) and once
   in `content` wrapped in `<think>…</think>` markers (which would otherwise
   appear inside the visible text).

This extension registers two new providers — `minimax-m3` and `minimax-cn-m3`
— that route MiniMax-M3 to the OpenAI-compatible endpoint so passive caching
works, and intercepts the finalized assistant message to strip the
duplicated thinking. It mirrors the upstream fix in
[`pi-mono@b85b91c9`](https://github.com/badlogic/pi-mono/commit/b85b91c9)
("route MiniMax-M3 to openai-completions for passive caching") so users can
get the fix on any pi version without waiting for an upstream release.

## Install

```bash
pi install git:github.com/rwese/pi-minimax-m3-caching-fix
```

For local development from a clone:

```bash
git clone https://github.com/rwese/pi-minimax-m3-caching-fix
pi install ./pi-minimax-m3-caching-fix
```

The extension reuses the env vars you already have for the built-in `minimax`
provider — no new credentials required:

| Provider   | Env var                | Endpoint                         |
| ---------- | ---------------------- | -------------------------------- |
| minimax-m3 | `MINIMAX_API_KEY`      | `https://api.minimax.io/v1`      |
| minimax-cn-m3 | `MINIMAX_CN_API_KEY` | `https://api.minimaxi.com/v1`    |

## Quickstart (for the impatient)

```bash
# 1. Make sure your MiniMax API key is exported
export MINIMAX_API_KEY="sk-..."

# 2. Install the extension
pi install git:github.com/rwese/pi-minimax-m3-caching-fix

# 3. Restart any running pi session, then start one
pi

# 4. Inside pi, switch the model
/model
#   pick:  minimax-m3 / MiniMax-M3 (passive cache)

# 5. Verify caching — look at the footer or session log
#    Turn 1: ~99% cache miss (system prompt being written to cache)
#    Turn 2+: ~99% cache read (system prompt being reused)
```

That's it. No new credentials, no config file, no restart of the upstream
`minimax` provider. Just pick the right model in `/model` and the rest
happens automatically.

## Use

1. Run `pi`.
2. Open the model picker with `/model`.
3. Pick **`minimax-m3 / MiniMax-M3 (passive cache)`** for the global endpoint
   or **`minimax-cn-m3 / MiniMax-M3 (passive cache — CN)`** for the China
   endpoint.
4. Send a prompt. The first turn is a cache miss; subsequent turns of the same
   session show a `CH` (cache hit rate) in the footer as the system prompt
   gets reused.

In the session log, the `usage` object on each assistant message shows the
cache reads. For example, a 3-turn session looks like:

| Turn | input | cacheRead | Hit rate |
| ---- | ----- | --------- | -------- |
| 1    | 8932  | 114       | 1%       |
| 2    | 128   | 8946      | 99%      |
| 3    | 128   | 8946      | 99%      |

## Why a separate provider (not overriding the built-in)

`pi.registerProvider(name, { models })` **replaces** every model registered
for that provider. There are two ways that breaks the built-in integration:

- Override `minimax` with `baseUrl` only — this lumps M2.x onto the
  OpenAI-compatible endpoint too, breaking M2.x.
- Override `minimax` with new `models` — this wipes M2.x from the registry.

So this extension registers new provider names (`minimax-m3`,
`minimax-cn-m3`) that don't collide with `minimax` or `minimax-cn`. Users opt
in by switching the model in `/model`. The built-in `minimax / MiniMax-M3`
model is still listed — **pick the one with "(passive cache)" in the name**.

## Limitations

- **Brief visual flash of duplicated thinking during streaming.** The
  `message_end` hook strips the markers and drops the duplicate thinking
  block at the end of each turn; the saved session log is clean. During live
  streaming, the TUI may briefly show the `<think>…</think>` text before the
  hook replaces the final message.
- **Two `MiniMax-M3` entries in `/model`.** The built-in (broken, billing at
  full input price) and the extension's (fixed, passive cache) both appear.
  Pick the one with `(passive cache)` in the name.
- **Requires both env vars for both providers to show.** pi only lists
  providers that have auth configured. If you only have `MINIMAX_API_KEY`,
  only `minimax-m3` shows up; set `MINIMAX_CN_API_KEY` (even to a dummy
  value) to also see `minimax-cn-m3`.

## How the fix works

The extension does two things:

1. **Routes M3 to `/v1/chat/completions`** by registering the two new
   providers. The model metadata mirrors
   `packages/ai/src/models.generated.ts` from the upstream fix:
   `input: ["text", "image"]`, `reasoning: true`, cost
   `$0.6 / $2.4 / $0.12` per million tokens, 1M-token context window, 512K
   max output.
2. **Strips duplicated thinking via a `message_end` hook.** When an assistant
   message from one of the two new providers ends, the extension:
   - drops any `thinking` content block (already shown in the live TUI), and
   - strips `<think>…</think>` markers (and any inner content) from any
     `text` content block.

   This is the same effect as the upstream `compat.skipThinkingBlock` flag,
   but applied at end-of-stream because the user's installed
   `@earendil-works/pi-ai` (0.79.1) predates that compat field. When a future
   pi-ai release includes `skipThinkingBlock`, the hook can be removed and
   the extension becomes a thin shim that just routes the model.

## Removing the extension (when upstream ships the fix)

When pi-mono ships a release that includes `b85b91c9` (or any release whose
`models.generated.ts` lists `MiniMax-M3` with `api: "openai-completions"`
and `skipThinkingBlock: true`), retire the extension:

```bash
pi remove git:github.com/rwese/pi-minimax-m3-caching-fix
```

The built-in `minimax / MiniMax-M3` model will then route correctly out of
the box.

## License

MIT — see [LICENSE](./LICENSE).

## Development

```bash
npm run check    # tsc --noEmit using the bundled tsconfig.json
```

The `tsconfig.json` configures `--skipLibCheck` and `--moduleResolution
bundler` so the type check is reproducible without depending on transitive
type packages of the user's installed pi.

