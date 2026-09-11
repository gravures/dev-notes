---
project: awrit
---
# Session Summary: Awrit Tmux Hybrid Analysis

## Context

**Awrit** is a terminal-based web browser built on Electron that renders HTML to the terminal using the Kitty graphics protocol. It works natively in terminals like Kitty but does **not** work inside **tmux** (a terminal multiplexer) because tmux intercepts the escape sequences needed for graphics and input.

Two independent forks attempted to add tmux support:
- **`tmux` branch** — Rust implementation (7 commits)
- **`tmux-pane` branch** — TypeScript implementation (13 commits)

The goal of this session was to determine whether these two implementations could be hybridized, or whether one should be ported to use the other's approach.

## What Was Done

1. **Explored both branches** — examined every modified file in both `tmux` and `tmux-pane` branches relative to the upstream `electron` branch
2. **Compared architectures** — mapped out exactly what each branch implements and what each is missing
3. **Produced a recommendation** — start from `tmux-pane` (the more complete implementation) and selectively bring in Rust-level tmux passthrough from the `tmux` branch

## Recommendation

**Do not merge the branches directly.** Instead, port the `tmux` branch's Rust tmux passthrough code into `tmux-pane`'s codebase, keeping `tmux-pane`'s TypeScript layer for higher-level concerns (renderer lifecycle, pane queries, mouse normalization, tests).

Rationale:
- `tmux-pane` has 8 iterative fix commits addressing rendering lifecycle, visibility, and cursor flickering — battle-tested edge cases
- `tmux-pane` has comprehensive test coverage; `tmux` has none
- The Rust code's main advantage (fast ANSI escaping for tmux passthrough) can be cherry-picked without adopting the entire branch
- The two branches diverge from different upstream commits, making a direct merge error-prone

## State of the Program

### What we know about `tmux` branch (Rust)

| File | What it does |
|------|-------------|
| `awrit-native-rs/crates/crossterm/src/tmux.rs` | Core tmux utilities in Rust: `is_tmux()`, `tmux_escape()`, `tmux_passthrough()`, `MaybeTmux` struct, `TmuxBeginPassthrough`/`TmuxEndPassthrough` commands, `TmuxSetExtendedKeysMode` |
| `awrit-native-rs/src/term.rs` | NAPI bindings exposing Rust tmux functions to JavaScript: `isTmux()`, `passthroughTmux()`, `writeMaybeTmux()`, plus tmux-aware `termEnableFeatures()`/`termDisableFeatures()` |
| `awrit-native-rs/index.d.ts` | TypeScript type definitions for the NAPI bindings |
| `src/runner/index.ts` | Sets tmux `extended-keys-format` via shell `exec()` and restores it on exit |
| `src/registerPaintedContent.ts` | Minimal changes to paint handler — no tmux-specific rendering logic |
| `src/inputHandler.ts` | No tmux-specific changes (mouse coordinates not normalized) |
| `src/keybindings.ts` | No tmux-specific changes |

**What `tmux` branch is missing:**
- No pane visibility tracking (cannot detect when pane is hidden/zoomed)
- No mouse coordinate normalization (cell vs pixel mode detection)
- No image slot management or ID allocation
- No `TmuxRenderer` (batching, flush, retry, visibility-gated rendering)
- No tmux protocol encoding for image upload
- No placeholder rendering for image display
- No zoom state persistence
- No wrapper script handling tmux passthrough enable/restore
- No test coverage

### What we know about `tmux-pane` branch (TypeScript)

| File | What it does |
|------|-------------|
| `src/tty/tmux.ts` | Core tmux utilities via CLI calls: `isTmuxSession()`, `getPaneSize()` (with cache), `getPaneState()` (active/inactive/invisible), `getPaneStatus()`, `tmuxWrap()` for ANSI escaping |
| `src/tty/tmuxRenderer.ts` | Full renderer class: slot-based image lifecycle, batched flushing with sync marks (`\x1b[?2026h/l`), visibility-gated retries, pending/latest separation, periodic visibility polling |
| `src/tty/tmuxProtocol.ts` | Kitty protocol encoding: `buildTmuxUploadCommands()` (chunked base64), `buildTmuxDeleteImageCommand()`, `buildTmuxPlaceholderLines()` (Unicode placeholder characters) |
| `src/tty/mouseCoordinates.ts` | Auto-detects cell vs pixel coordinate mode, normalizes mouse events via `normalizeTmuxMouseCoordinates()` |
| `src/tty/imageIds.ts` | Sequential image ID allocator with wraparound |
| `src/paint.ts` | Adds `registerPaintedContentTmux()` — converts paint events to PNG, calculates cell positions, passes to `TmuxRenderer` |
| `src/inputHandler.ts` | Tmux-aware mouse normalization via `maybeNormalizeTmuxMouseCoordinates()` |
| `src/windows.ts` | Chooses `registerPaintedContentTmux` when in tmux, handles `SIGWINCH` re-registration |
| `src/index.ts` | Main entry point with tmux capability checks (passthrough, mouse, version) |
| `src/args.ts` | CLI args including `--tmux-dump` for debugging |
| `src/zoom-state.ts` | Persists per-origin zoom factors to disk |
| `src/paths.ts` | Platform-specific app data paths |
| `config.js` | Keybindings configuration with platform variants |
| `awrit` | Bash wrapper script that enables `allow-passthrough all` and restores on exit |
| `src/tty/tmux.test.ts` | Tests for tmux helpers (passthrough, pane size cache, pane state) |
| `src/tty/tmuxRenderer.test.ts` | Tests for renderer lifecycle (slot release, close, visibility flush) |
| `src/tty/tmuxProtocol.test.ts` | Tests for protocol encoding |

**What `tmux-pane` branch is missing:**
- No Rust-level tmux passthrough (all ANSI escaping done in TypeScript `tmuxWrap()`)
- No Rust-level `extended-keys-format` management (done via shell `exec()`)

### What we know about the upstream `electron` branch

The upstream code (last snapshot before archive) has no tmux support at all. Key baseline files:
- `src/paint.ts` — Only `registerPaintedContent()` (Kitty protocol) and `registerPaintedContentFallback()` (no animation)
- `src/inputHandler.ts` — No mouse coordinate normalization
- `src/windows.ts` — No tmux renderer selection
- `src/index.ts` — Requires keyboard enhancement and Kitty graphics (exits if unavailable)
- `src/runner/index.ts` — No tmux extended-keys handling

## Plan (Not Yet Executed)

### Phase 1: Bring Rust tmux passthrough into tmux-pane's awrit-native-rs

1. Cherry-pick the tmux.rs additions from `awrit-native-rs/crates/crossterm/src/tmux.rs` into tmux-pane's copy of awrit-native-rs
2. Cherry-pick the NAPI bindings from `awrit-native-rs/src/term.rs` (the `isTmux()`, `passthroughTmux()`, `writeMaybeTmux()` functions, and tmux-aware `termEnableFeatures`/`termDisableFeatures`)
3. Update `src/tty/tmux.ts` — replace `tmuxWrap()` with calls to Rust `passthroughTmux()`
4. Update `src/runner/index.ts` — replace shell `exec()` for extended-keys with Rust NAPI call
5. Verify tests still pass

### Phase 2: Keep tmux-pane's TypeScript layer unchanged

Keep these files as-is (they handle higher-level concerns not worth moving to Rust):
- `src/tty/tmuxRenderer.ts` — lifecycle management, scheduling
- `src/tty/tmuxProtocol.ts` — protocol encoding (one-time, not hot path)
- `src/tty/tmux.ts` — pane queries via CLI (infrequent)
- `src/tty/mouseCoordinates.ts` — arithmetic (trivial)
- `src/paint.ts` — paint event handling
- `src/inputHandler.ts` — input dispatch
- `awrit` — wrapper script

### Phase 3: Optional optimization

If profiling shows mouse coordinate normalization is a bottleneck (unlikely), move it to Rust. Otherwise keep in TypeScript.

## References

### Project files (tmux-pane branch, current working tree)
- `src/tty/tmux.ts` — tmux session/pane utilities
- `src/tty/tmuxRenderer.ts` — tmux image renderer
- `src/tty/tmuxProtocol.ts` — kitty protocol encoding for tmux
- `src/tty/mouseCoordinates.ts` — mouse coordinate normalization
- `src/tty/imageIds.ts` — image ID allocation
- `src/tty/kittyGraphics.ts` — kitty graphics protocol (shared by both paths)
- `src/paint.ts` — paint event handlers
- `src/inputHandler.ts` — input event dispatch
- `src/keybindings.ts` — keybinding parser and handler
- `src/windows.ts` — window creation and layout
- `src/index.ts` — application entry point
- `src/args.ts` — CLI argument parsing
- `src/paths.ts` — platform paths
- `src/zoom-state.ts` — zoom persistence
- `src/runner/index.ts` — runner/build script
- `config.js` — user configuration
- `awrit` — bash launcher wrapper
- `package.json` — project manifest

### Project files (tmux branch, accessed via git)
- `awrit-native-rs/crates/crossterm/src/tmux.rs` — Rust tmux utilities
- `awrit-native-rs/src/term.rs` — Rust NAPI tmux bindings
- `awrit-native-rs/index.d.ts` — NAPI TypeScript definitions

### Test files
- `src/tty/tmux.test.ts` — tmux helper tests
- `src/tty/tmuxRenderer.test.ts` — renderer lifecycle tests
- `src/tty/tmuxProtocol.test.ts` — protocol encoding tests
- `src/inputHandler.test.ts` — input handler tests
- `src/tty/imageIds.test.ts` — image ID allocator tests

### External references
- `/home/gilles/.agents/skills/understand/SKILL.md` — understand-anything skill definition
- `/home/gilles/.ua/knowledge-graph.json` — knowledge graph for tmux-pane branch
