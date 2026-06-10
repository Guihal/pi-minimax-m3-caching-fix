# Changelog

All notable changes to `pi-minimax-m3-caching-fix` are documented here.
Format follows [Keep a Changelog](https://keepachangelog.com/).

## [Unreleased]

### Added

- `package.json` `files` whitelist and `.npmignore` so `npm publish` ships
  only `index.ts`, `README.md`, `LICENSE`, `CHANGELOG.md`.
- `prepublishOnly` quality gate that runs `npm run check` (tsc) before
  every publish.
- `repository`, `homepage`, `bugs`, and explicit `author` fields.
- README documents the `pi install npm:pi-minimax-m3-caching-fix` flow
  alongside the existing git install.
- `peerDependencies` on `@earendil-works/pi-coding-agent@0.79.1` and
  `@earendil-works/pi-ai@0.79.1` to document the runtime contract.
- `devDependencies` on the same two packages so `npm run check` (and
  `prepublishOnly`) can resolve types without ad-hoc symlinks.
- `packageManager: pnpm@10.33.0` and a committed `pnpm-lock.yaml`.
  npm's installer hits malformed symlink entries in the
  `pi-coding-agent` tarball and leaves the project unable to resolve
  the type imports; pnpm installs cleanly. Publish target stays npm
  (that's what `pi install` resolves by default).

### Removed

- `PLAN.md`, `PLAN-DELTA.md`, and `TODO.md` from the repo. The reasoning
  they captured is summarized inline in `index.ts` and `AGENTS.md`.
- `AGENTS.md` and `index.ts` references to the deleted plan files.

## [0.1.0] - 2026-06-10

### Added

- Initial release. Registers two providers (`minimax-m3-cache-fixed`,
  `minimax-cn-m3-cache-fixed`) exposing `MiniMax-M3` on the OpenAI-compatible
  endpoint so passive prompt caching works.
- `message_end` hook that strips duplicated thinking output (the
  `<think>…</think>` markers M3 sends alongside the proper thinking block)
  for messages from the two registered providers.
- Reuses the existing `MINIMAX_API_KEY` / `MINIMAX_CN_API_KEY` env vars — no
  new credentials needed.
- Mirrors the upstream fix from
  [`pi-mono@b85b91c9`](https://github.com/badlogic/pi-mono/commit/b85b91c9)
  for pi versions that don't yet include it.

### Known Limitations

- The thinking-strip happens at end-of-stream, so the duplicated thinking is
  briefly visible during streaming before the hook replaces the final
  message. The saved session log is clean.
- Two `MiniMax-M3` entries appear in `/model` (built-in broken + extension
  fixed). Users must pick the one with `(cache-fixed)` in the name.
