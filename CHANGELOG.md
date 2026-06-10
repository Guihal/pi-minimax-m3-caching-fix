# Changelog

All notable changes to `pi-minimax-m3` are documented here.
Format follows [Keep a Changelog](https://keepachangelog.com/).

## [Unreleased]

## [0.1.0] - 2026-06-10

### Added

- Initial release. Registers two providers (`minimax-m3`, `minimax-cn-m3`)
  exposing `MiniMax-M3` on the OpenAI-compatible endpoint so passive prompt
  caching works.
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
  fixed). Users must pick the one with `(passive cache)` in the name.
