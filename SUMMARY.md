---
project: awrit
---
# Project Research Summary

**Project:** Awrit Tmux Hybrid — terminal web browser with tmux support
**Domain:** Terminal-based web browser hybridizing Rust/TypeScript for tmux passthrough
**Researched:** 2026-09-01
**Confidence:** HIGH

## Executive Summary

This is NOT a greenfield project. Awrit already has a working stack (Electron 37, TypeScript with Bun, Rust via NAPI-RS 3.x, crossterm for terminal I/O). The project's goal is to hybridize two independent forks: the `tmux` branch (7 commits, fast Rust ANSI escaping) and the `tmux-pane` branch (13 commits, comprehensive renderer lifecycle, visibility tracking, and tests). The core work is cherry-picking Rust tmux passthrough code from the `tmux` branch into the existing `tmux-pane` branch — not choosing new technologies. The current stack is already well-chosen and recent; versions are verified and compatible.

The recommended approach is layered responsibility: Rust handles low-level terminal I/O and tmux passthrough via NAPI bindings, TypeScript manages renderer lifecycle, protocol encoding, and pane state queries. The key architectural insight is that tmux doesn't natively support Kitty graphics passthrough as of stable releases — the project must use Unicode placeholders (`U+10EEEE`) tied to virtual placements (`a=t` + `a=p,U=1`) to make images grid-resident and tmux-aware. This is the Kitty spec-recommended approach for multiplexers. The bash wrapper handles pre-launch tmux setup (enabling `allow-passthrough`) and restoration on exit.

The critical risks are all well-documented tmux protocol limitations: DCS passthrough silently truncates nested escape sequences (architectural constraint, not a bug), `allow-passthrough` defaults to `off` in tmux 3.3+, and Kitty graphics placement semantics (`a=T` vs `a=t`) cause ghost images if done incorrectly. These are avoidable with the right approach — Unicode placeholders bypass passthrough entirely for the grid-resident cells, and virtual placements prevent stale frames. The project has clear anti-features (no Sixel, no video, no extensions beyond unpacked) and a focused scope (hybridization only, no new features).

## Key Findings

### Recommended Stack

The existing stack is current and requires no changes. All dependencies are verified compatible as of September 2026.

**Core technologies:**
- **Electron ^37.3.1** — App shell, web content rendering; already in use, recent stable
- **Node.js >=22.11.0** — Runtime for TypeScript; required by Electron 37 (bundles Node 22.16.0)
- **Bun** — TypeScript execution and testing; already in use, fast runtime
- **TypeScript ^5.9.2** — Type safety, renderer logic; already in use
- **Rust Edition 2021** — Performance-critical terminal I/O; already in use
- **NAPI-RS 3.2.4** — Rust↔TypeScript FFI; already pinned, proven stable
- **Biome 2.2.2** — Formatting and linting; already in use

**Terminal protocol:**
- **crossterm (forked subtree)** — Terminal manipulation, raw mode, keyboard/mouse; forked for customization, tracks upstream 0.29.0
- **Kitty Graphics Protocol (APC sequences)** — Image rendering to terminal; already implemented in tmuxProtocol.ts
- **tmux DCS Passthrough** — Escaping ANSI through tmux; required, tmux 3.3+ mandatory
- **Unicode placeholders (U+10EEEE)** — Grid-resident image cells for tmux pane switching, scrolling, clipping

### Expected Features

**Must have (table stakes):**
- Kitty graphics rendering — core value proposition, already exists in both branches
- DCS passthrough wrapping — tmux swallows raw Kitty APC sequences; must wrap in `\x1bPtmux;...\x1b\\`
- Unicode placeholder support (U+10EEEE) — prevents ghost images on pane switch; overlay approach ghosts on redraw
- Virtual placements (a=p,U=1) — prevents stale frames; real placements (a=T) leave ghost images in tmux
- Mouse click forwarding — users click links, buttons, form elements; tmux-pane branch has coordinate normalization
- Scroll support, keyboard navigation, URL bar, back/forward history, tab management, page reload, form handling

**Should have (differentiators):**
- Full Chromium rendering via headless CDP — real HTML5/CSS3/JS, not text approximation like Browsh
- Pane visibility tracking — only render when pane is visible, saves CPU/GPU
- Image slot management — reuse image IDs to prevent memory leaks
- Mouse coordinate normalization — convert tmux cell coordinates to pixel coordinates for Chromium
- Vimium-style link hints — keyboard-driven link following
- Custom keybindings, persistent bookmarks, page search (ctrl+f)

**Defer to v2+:**
- Extension support — not needed for hybridization focus
- Video playback — technical limitations make this impractical over tmux passthrough
- DevTools integration — nice-to-have, not essential for MVP
- WebRTC/audio — terminal has no audio output mechanism

### Architecture Approach

The system uses a **layered responsibility** pattern: Electron BrowserWindow handles web content rendering, paint.ts routes paint events to the appropriate renderer (TmuxRenderer or KittyGraphics), the TTY Layer manages protocol encoding and tmux queries, awrit-native-rs provides Rust NAPI bindings for terminal I/O and tmux passthrough, crossterm (forked) handles low-level terminal manipulation, and the bash wrapper handles pre-launch tmux setup. The key pattern is dual code paths — branch on `isTmuxSession()` to use appropriate rendering path.

**Major components:**
1. **TmuxRenderer** — Slot-based image rendering with visibility gating; manages pending/displayed image slots per pane
2. **tmuxProtocol.ts** — Kitty protocol encoding for tmux passthrough; builds wrapped upload/delete commands with chunking
3. **awrit-native-rs (Rust)** — Terminal I/O, shared memory, tmux passthrough; NAPI exports to TypeScript
4. **crossterm tmux.rs** — ANSI escaping, passthrough wrapping, extended keys; Rust NAPI for performance-critical I/O
5. **Bash wrapper (awrit)** — Pre-launch tmux setup, passthrough enable/restore, TMUX_PANE validation

**Key patterns to follow:**
- Slot-based image management (pending/latest/displayed per slot)
- Visibility-gated rendering (skip flush if invisible, retry on visibility change)
- Rust NAPI for performance-critical I/O (batch operations, cross boundary once per frame)
- Bash wrapper for tmux setup (save/restore passthrough state with EXIT trap)

### Critical Pitfalls

1. **DCS Passthrough Nested Escape Sequence Corruption** — tmux's passthrough consumes the first `\e\\` terminator regardless of nesting; never nest DCS sequences inside passthrough; use Unicode placeholders which bypass passthrough entirely; test with `tmux -vv` logs
2. **Missing or Misconfigured allow-passthrough** — defaults to `off` in tmux 3.3+; detect `$TMUX` at startup and warn if not configured; programmatically check with `tmux show -g allow-passthrough`
3. **Kitty Graphics Ghost Placements (a=T vs a=t)** — `a=T` creates real placements at absolute coordinates tmux can't track; always use `a=t` (transmit only) + `a=p,U=1` (virtual placement); never use `a=T` through tmux passthrough
4. **Image ID Collision with SGR Extended-Color Introducers** — image IDs 38 and 48 cause tmux `setaf` to emit broken escape sequences; use 24-bit truecolor (`\e[38;2;R;G;Bm`) for image IDs to avoid all palette collisions
5. **Print-Time vs Render-Time Placeholder Resolution Race** — APC data and placeholder text travel on independent paths through tmux; use synchronized output mode (`?2026h/l`) to batch in a single atomic write

## Implications for Roadmap

Based on combined research (ARCHITECTURE.md build order + FEATURES.md dependencies + PITFALLS.md phase warnings), the suggested phase structure follows the dependency chain: Rust tmux module → NAPI bindings → TypeScript integration → Renderer integration → Testing.

### Phase 1: Rust tmux Module Port
**Rationale:** Foundation for all tmux work; must be first because NAPI bindings depend on the Rust module. This is the cherry-pick from the tmux branch.
**Delivers:** `tmux.rs` with `is_tmux()`, `tmux_escape()`, `tmux_passthrough()`, `MaybeTmux`, `TmuxBeginPassthrough`, `TmuxEndPassthrough`, `TmuxSetExtendedKeysMode`
**Addresses:** DCS passthrough wrapping (FEATURES.md table stakes), Kitty graphics rendering prerequisite
**Avoids:** Pitfall #2 (allow-passthrough configuration — validated in this phase), Pitfall #6 (blocking event loop — design API as async from day one), Pitfall #15 (platform binary distribution — set up CI matrix before writing code)

### Phase 2: NAPI Bindings Extension
**Rationale:** Depends on Phase 1 Rust module; provides the TypeScript→Rust bridge needed by all higher-level code.
**Delivers:** `isTmux()`, `passthroughTmux()`, `writeMaybeTmux()`, tmux-aware `termEnableFeatures()`/`termDisableFeatures()`, TypeScript type definitions
**Addresses:** DCS passthrough wrapping prerequisite, extended-keys NAPI call
**Avoids:** Pitfall #11 (generated .d.ts drift — add `napi build` to CI verification)

### Phase 3: TypeScript Integration
**Rationale:** Depends on Phase 2 NAPI bindings; replaces TypeScript tmuxWrap() with Rust passthrough calls and shell exec() with NAPI calls.
**Delivers:** Replace `tmuxWrap()` with `passthroughTmux()`, replace shell exec for extended-keys with Rust NAPI call, preserve both tmux and non-tmux code paths
**Addresses:** Mouse click forwarding (coordinate normalization already exists), scroll support, keyboard navigation
**Avoids:** Pitfall #3 (ghost placements — enforce `a=t` + `a=p,U=1` in code review), Pitfall #9 (image leaking to adjacent panes — Unicode placeholders only)

### Phase 4: Image Slot Management & Renderer Integration
**Rationale:** Depends on Phase 3 TypeScript integration; the rendering pipeline must be correct from day one with proper placement semantics.
**Delivers:** Retro-resolution mechanism (Pitfall #5), image ID allocator avoiding SGR collisions (Pitfall #4), locale-independent placeholder encoding (Pitfall #8), correct image cap (`>=` not `==`, Pitfall #13), GRID_LINE_WRAPPED handling (Pitfall #14)
**Addresses:** Pane visibility tracking, image slot management, renderer lifecycle management, virtual placements
**Avoids:** Pitfall #4 (SGR color collision — use truecolor encoding), Pitfall #5 (print-time vs render-time race — implement retro-resolution), Pitfall #8 (locale-dependent encoding — literal UTF-8 bytes), Pitfall #10 (extended keys corrupting responses — one-way passthrough only), Pitfall #14 (GRID_LINE_WRAPPED reflow — real newlines for image rows)

### Phase 5: Input Handling & Bug Fixes
**Rationale:** Depends on all previous phases; input forwarding must work correctly with the new Rust passthrough.
**Delivers:** Crossterm cursor position race prevention (Pitfall #7), paste-buffer sanitization (Pitfall #12), comprehensive manual testing, automated test preservation, bug fixes
**Addresses:** Form handling (requires JS execution + input forwarding), all edge cases
**Avoids:** Pitfall #7 (cursor race — track position manually, never query during EventStream), Pitfall #12 (paste-buffer injection — sanitize paste content)

### Phase Ordering Rationale

- **Dependency chain enforced:** Rust → NAPI → TypeScript → Renderer → Input (each phase builds on previous)
- **Architecture patterns applied:** Dual code paths (tmux vs native) designed in Phase 1, Rust NAPI for hot path in Phase 1-2, visibility-gated rendering in Phase 4
- **Pitfall avoidance built-in:** allow-passthrough validated in Phase 1, ghost placements prevented in Phase 3, SGR collisions addressed in Phase 4, cursor race handled in Phase 5
- **Scoped to hybridization:** No new features beyond tmux support, no unnecessary refactoring, manual testing approach

### Research Flags

**Needs research during planning:**
- **Phase 1:** Rust tmux module port — needs `gsd-plan-phase --research-phase 1` to verify exact file contents from tmux branch and NAPI-RS binding patterns
- **Phase 4:** Renderer integration — needs `gsd-plan-phase --research-phase 4` to verify retro-resolution mechanism and GRID_LINE_WRAPPED behavior across tmux versions

**Standard patterns (skip research-phase):**
- **Phase 2:** NAPI bindings — well-documented NAPI-RS patterns, straightforward extension
- **Phase 3:** TypeScript integration — replacement pattern is clear (import NAPI functions, replace existing calls)
- **Phase 5:** Testing — manual testing approach defined in PROJECT.md constraints

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | HIGH | All versions verified, already in use, Electron 37 + NAPI-RS 3.2.4 + crossterm fork confirmed compatible |
| Features | HIGH | Clear table stakes from PROJECT.md requirements, well-scoped to hybridization only |
| Architecture | HIGH | Layered responsibility pattern is established, dual code paths designed, build order dependency chain verified |
| Pitfalls | HIGH | 15 pitfalls documented with specific prevention strategies, phase-specific warnings mapped, sourced from tmux/kitty/napi-rs issue trackers |

**Overall confidence:** HIGH

### Gaps to Address

- **tmux version compatibility testing:** The project requires tmux 3.3+ for DCS passthrough, but testing across tmux versions (3.3, 3.3a, 3.4, 3.5) is not yet planned. During Phase 1, add a tmux version check in the bash wrapper and warn if < 3.4.
- **Outer terminal compatibility matrix:** DCS passthrough support varies by terminal (Kitty, iTerm2, WezTerm, Ghostty support it; Alacritty, foot, xterm do not). Document supported outer terminals in README, test with at least Kitty and WezTerm.
- **Unicode placeholder behavior across tmux versions:** The U+10EEEE placeholder approach is from the tmux Kitty PR (#5274) which is not yet in stable releases. Verify which tmux version first supports Unicode placeholders; may need to fall back to overlay approach for tmux < 3.4.
- **Automated test coverage:** PROJECT.md specifies manual testing (dogfood-ready quality bar), but automated tests exist in tmux-pane branch. Preserve existing tests and add integration tests for the new Rust passthrough code during Phase 5.
- **NAPI-RS async vs sync decision:** Pitfall #6 warns about blocking the event loop with sync Rust calls. The research recommends `#[napi] async fn` for any Rust work >1ms, but the exact API surface design needs validation during Phase 1 planning.

## Sources

### Primary (HIGH confidence)
- Kitty graphics protocol spec: https://sw.kovidgoyal.net/kitty/graphics-protocol/
- tmux DCS passthrough spec: https://vtdn.dev/docs/dcs/tmux-passthrough
- NAPI-RS releases and docs: https://github.com/napi-rs/napi-rs/releases
- crossterm releases: https://github.com/crossterm-rs/crossterm/releases
- Electron schedule: https://releases.electronjs.org/schedule
- tmux Kitty support PR: https://github.com/tmux/tmux/pull/5274

### Secondary (MEDIUM confidence)
- tmux issue trackers for DCS passthrough limitations (tmux/tmux#3755, #3287, #4386, #5530)
- Kitty issue trackers for tmux compatibility (kovidgoyal/kitty#7655, #7070, #6215)
- casty browser: https://github.com/cashmeredev/casty
- awrit original: https://github.com/chase/awrit
- Chawan TUI browser: https://chawan.net/
- Browsh text browser: https://www.brow.sh/
- kitty-graphics.el tmux support: https://cashmere.rs/blog/kitty-graphicsel-v060-video-browser-and-improved-tmux/

### Tertiary (LOW confidence)
- tmux Unicode placeholder implementation status (PR #5274 — not yet merged as of research date)
- Terminal-specific passthrough support matrices (may change with new tmux releases)

---
*Research completed: 2026-09-01*
*Ready for roadmap: yes*
