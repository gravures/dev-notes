---
project: awrit
---
# Domain Pitfalls

**Domain:** Terminal-based web browser with tmux support (Rust/TypeScript hybrid)
**Researched:** 2026-09-01

## Critical Pitfalls

Mistakes that cause silent rendering failures, data corruption, or architectural rewrites.

### Pitfall 1: DCS Passthrough Nested Escape Sequence Corruption

**What goes wrong:** tmux's DCS passthrough (`\ePtmux;\e...\e\\`) consumes the first `\e\\` it sees as the passthrough terminator, even when that terminator belongs to an inner DCS sequence. This silently truncates nested DCS payloads — images, XTGETTCAP responses, OSC sequences — leaving the outer terminal with incomplete, unparseable data.

**Why it happens:** The passthrough protocol was designed for simple one-way output, not for query/response patterns. tmux's parser is greedy — it matches the first ST terminator regardless of nesting context. This is a fundamental limitation of the passthrough design, not a tmux bug.

**Consequences:** Image transfers silently fail or render partial/corrupted content. Terminal capability queries return garbage. The application receives orphaned payload text as literal keystrokes (e.g., `kitty(0.48.2)` parsed as keypresses `t`, `t`, `:`, etc.).

**Prevention:**
- Never nest DCS sequences inside passthrough. Use raw escape sequences for Kitty graphics when possible.
- For Kitty graphics specifically, use Unicode placeholders (`U+10EEEE`) which are plain text and pass through tmux's grid naturally.
- If you must use DCS passthrough for graphics, batch all data in a single write call — do not interleave with other escape sequences.
- Test with `tmux -vv` logs to verify the passthrough content is complete before writing to pty.

**Detection:** Images render outside tmux but not inside. `strace` on tmux server shows complete escape sequences going out, but the receiving terminal gets truncated data. Check for missing `\e\\` terminators in captured output.

**Phase:** All phases — this is a fundamental architectural constraint that affects every feature touching tmux passthrough.

### Pitfall 2: Missing or Misconfigured allow-passthrough

**What goes wrong:** tmux 3.3+ defaults `allow-passthrough` to `off`. Without it explicitly set to `on`, tmux silently swallows all DCS passthrough sequences. The application sends graphics commands that disappear into a black hole — no error, no feedback, just blank output.

**Why it happens:** Security hardening. Passthrough disables tmux's escape sequence filtering, giving the pane direct terminal access. tmux defaults to the secure posture.

**Consequences:** Complete failure of all tmux-aware rendering. Time wasted debugging escape sequence encoding when the real issue is a tmux config setting.

**Prevention:**
- Document `set -g allow-passthrough on` as a hard requirement in all setup docs.
- On startup, detect `$TMUX` and warn explicitly if passthrough is not configured.
- Use `tmux show -g allow-passthrough` programmatically to check at runtime.
- Consider using Unicode placeholders (`U=1`) which bypass passthrough entirely.

**Detection:** `echo -e '\ePtmux;\e\e]11;?\a\e\\'` produces no response from the terminal. Kitty graphics commands produce no output.

**Phase:** Phase 1 (foundation) — must be validated before any tmux rendering work begins.

### Pitfall 3: Kitty Graphics Ghost Placements (a=T vs a=t)

**What goes wrong:** Using `a=T` (transmit and display) instead of `a=t` (transmit only) + `a=p,U=1` (virtual placement) creates a "real" placement at the cursor position. When the application later re-renders or the pane scrolls, the real placement persists as a ghost — stale image data stuck at absolute coordinates that tmux cannot track per-pane.

**Why it happens:** `a=T` creates both the image data AND a visible placement in one step. The placement is positioned at the cursor, which is an absolute terminal coordinate. tmux doesn't track this placement — it only tracks the Unicode placeholder cells that flow through the grid.

**Consequences:** Ghost images that persist after the content they represent has moved or been replaced. Duplicate images on each re-render. Images leaking to adjacent panes on split.

**Prevention:**
- Always use `a=t` (transmit only) followed by `a=p,U=1` (virtual placement).
- The virtual placement defines the render rectangle but creates no visible output — the `U+10EEEE` placeholder cells drive rendering.
- Never use `a=T` when working through tmux passthrough.

**Detection:** Images appear but don't move when content scrolls. Duplicate images stack on re-render. Split a pane and images from one pane appear in the other.

**Phase:** Phase 2 (Rust tmux passthrough port) — the core graphics pipeline must use correct placement semantics from day one.

### Pitfall 4: Image ID Collision with SGR Extended-Color Introducers

**What goes wrong:** When using the Kitty Unicode placeholder protocol, image IDs are encoded in the foreground color of `U+10EEEE` cells. If the image ID happens to be 38 or 48 (the SGR extended-color introducers), tmux's `setaf` terminfo emits broken escape sequences — `\e[38m` instead of `\e[38;5;38m` — causing the image to silently fail to render.

**Why it happens:** tmux uses `setaf` to emit foreground colors for placeholder cells. For palette indices 38 and 48, `setaf` produces `\e[38m` and `\e[48m` respectively, which tmux's own parser interprets as "start extended foreground color" rather than "set color 38". This happens every ~38th image in a sequential ID scheme.

**Consequences:** Every ~38th image silently fails to render. No error message — the placeholder cells are emitted but the terminal doesn't associate them with any image.

**Prevention:**
- Use `COLOUR_FLAG_256` or equivalent to force tmux to emit `\e[38;5;Nm` (explicit 256-color form) instead of relying on `setaf`.
- Avoid image IDs 38 and 48 in your allocator, or start IDs at a value that doesn't collide.
- The Kitty spec recommends using 24-bit true color (`\e[38;2;R;G;Bm`) for image IDs to avoid all palette collisions.

**Detection:** Images render most of the time but intermittently fail with no pattern. Adding logging shows the placeholder cells are emitted but the terminal doesn't render the image.

**Phase:** Phase 3 (image slot management) — must be addressed in the image ID allocator.

### Pitfall 5: Print-Time vs Render-Time Placeholder Resolution Race

**What goes wrong:** Under tmux passthrough, the Kitty graphics APC data (image transmission + virtual placement creation) and the `U+10EEEE` placeholder text travel on independent paths through tmux. They can arrive at the outer terminal in either order. If placeholder cells arrive before the image data, they fall through to font rendering (showing garbled text or blank) and are never revisited — the image area stays broken until a full repaint.

**Why it happens:** tmux processes passthrough APC sequences synchronously but the placeholder cells are regular text that goes through the normal terminal write path. Under load or with slow image rendering, the ordering guarantee breaks.

**Consequences:** Intermittent rendering failures. Images work sometimes but not others. Hard to reproduce because it depends on timing.

**Prevention:**
- Implement "retro-resolution" — when image data arrives or a virtual placement is created, re-scan the visible screen for existing placeholder cells and retroactively bind them.
- Use synchronized output mode (`?2026h/l`) to batch the APC and placeholder text in a single atomic write.
- Accept that some terminals resolve at print-time (WezTerm) vs render-time (Kitty) and test both.

**Detection:** Images render correctly on first display but break on resize, re-render, or when the terminal is under load. Works outside tmux but intermittently fails inside tmux.

**Phase:** Phase 4 (renderer integration) — the retro-resolution mechanism must be implemented in the rendering pipeline.

## Moderate Pitfalls

Mistakes that cause degraded performance, intermittent bugs, or maintenance burden.

### Pitfall 6: Blocking the Node Event Loop with Synchronous Rust NAPI Calls

**What goes wrong:** Synchronous `#[napi]` functions run on the JavaScript thread. If the Rust code does any meaningful work (string building, ANSI escaping, coordinate math), it freezes the entire Node event loop — no input processing, no rendering, no IPC.

**Why it happens:** The NAPI boundary is convenient to call synchronously, and for small functions the overhead seems negligible. But when you're building escape sequences for every frame, the per-call overhead adds up fast.

**Consequences:** UI freezes. Input lag. Dropped frames. The application feels "laggy" even though the Rust code is fast — the bottleneck is the FFI crossing.

**Prevention:**
- Keep unit of work large: batch operations, cross the boundary once per frame not once per cell.
- Use `#[napi] async fn` for any Rust work that takes >1ms.
- Profile early: measure time in Rust vs time crossing the boundary.
- For escape sequence building, consider building the entire frame buffer in Rust and returning it as a single `Buffer`.

**Detection:** Application feels sluggish. `console.time` shows long gaps between input events. Node profiler shows time spent in NAPI callbacks.

**Phase:** Phase 1 (NAPI bindings) — architectural decisions about sync vs async must be made before building the API surface.

### Pitfall 7: Crossterm Cursor Position Race Condition

**What goes wrong:** Crossterm's `cursor::position()` requires writing an escape sequence and reading the response from stdin. If `EventStream` is simultaneously polling stdin for keyboard/mouse events, the position response gets consumed by the event stream instead of the position query — causing a timeout error.

**Why it happens:** Both operations read from the same file descriptor. Crossterm doesn't provide locking between event polling and cursor queries. This is a fundamental Unix terminal limitation.

**Consequences:** "The cursor position could not be read within a normal duration" panics. Application crashes on resize or when querying pane coordinates.

**Prevention:**
- Never call `cursor::position()` while `EventStream` is active.
- Track cursor position manually — call `position()` once at startup, then maintain it through cursor movement commands.
- Use `poll()` instead of `read()` and drain the event queue before position queries.
- Move UI operations to a single non-async thread.

**Detection:** Random panics with "cursor position could not be read" error. More frequent under load or when terminal is slow to respond.

**Phase:** Phase 2 (Rust tmux passthrough port) — the input handling architecture must account for this constraint.

### Pitfall 8: Locale-Dependent U+10EEEE Placeholder Encoding Failure

**What goes wrong:** When building `U+10EEEE` placeholder cells using `wctomb` or similar locale-dependent encoding functions, non-UTF-8 locales produce empty or invisible cells. The placeholders are emitted but the terminal cannot parse them — images render as blank space.

**Why it happens:** `wctomb` depends on the current locale's character encoding. In non-UTF-8 locales (common on older systems, certain Docker containers, SSH sessions to remote servers), the conversion silently fails or produces incorrect bytes.

**Consequences:** Images fail to render on systems with non-UTF-8 locales. No error message — the cells appear but carry no image information.

**Prevention:**
- Encode `U+10EEEE` and diacritics as literal UTF-8 bytes (`0xF0 0x90 0xAB 0xEE` for U+10EEEE) instead of using locale-dependent conversion.
- Test on at least one non-UTF-8 locale during development.
- Document UTF-8 as a hard requirement.

**Detection:** Works on developer machines (likely UTF-8) but fails on production systems, Docker containers, or SSH sessions to older servers.

**Phase:** Phase 3 (image slot management) — the placeholder encoding must be locale-independent from the start.

### Pitfall 9: Image Leaking to Adjacent Panes on Split

**What goes wrong:** Direct Kitty image placement (without Unicode placeholders) puts pixels at absolute terminal coordinates. When a tmux pane is split, the image from one pane bleeds into the other because tmux cannot track the image's per-pane ownership.

**Why it happens:** Direct placements are positioned by absolute coordinates that tmux doesn't associate with any specific pane. tmux's grid model only tracks text cells — images placed outside the grid model have no pane ownership.

**Consequences:** Visual artifacts on pane split/resize. Images appear in the wrong pane. Stale images persist after the content they represent has been destroyed.

**Prevention:**
- Always use Unicode placeholders (`U=1`) which tie image cells to text characters that flow through tmux's virtual terminal.
- Never use direct placement (`a=T` or `a=p` without `U=1`) when working through tmux.
- Test pane split/resize as a core workflow, not an edge case.

**Detection:** Split a pane and images from one pane appear in the other. Resize a pane and images don't clip correctly.

**Phase:** Phase 2 (Rust tmux passthrough port) — the graphics pipeline must use Unicode placeholders exclusively through tmux.

### Pitfall 10: Extended Keys Corrupting DCS Passthrough Responses

**What goes wrong:** When `extended-keys on` is set in tmux, tmux intercepts certain DCS responses (like XTGETTCAP) and reinterprets them as key events before the passthrough can forward them to the application. The response arrives garbled — missing its DCS introducer and carrying extra bytes from tmux's key event processing.

**Why it happens:** tmux's extended keys mode installs its own response handlers for certain terminal capability queries. These handlers conflict with passthrough — tmux doesn't know whether a DCS response is for its own query or a passthrough from the application.

**Consequences:** Terminal capability queries return garbage. Applications that query the host terminal for features get incorrect results, leading to wrong rendering paths.

**Prevention:**
- Do not use passthrough for query/response patterns — it's not designed for bidirectional communication.
- For terminal capability detection, use tmux's own mechanisms (e.g., `tmux display -p '#{terminal-features}')`.
- Document that passthrough is one-way only.

**Detection:** Queries sent via passthrough return garbled responses. Works without extended-keys but breaks with them.

**Phase:** Phase 4 (renderer integration) — capability detection must be designed around this constraint.

## Minor Pitfalls

Mistakes that cause technical debt, maintenance burden, or edge-case failures.

### Pitfall 11: Generated .d.ts Drifting from Rust Signatures

**What goes wrong:** The `index.d.ts` generated by napi-rs gets out of sync with the actual Rust function signatures. TypeScript callers get type errors at runtime, or worse, silently pass wrong types that crash at the NAPI boundary.

**Why it happens:** Manual edits to generated files are overwritten on next build. Forgetting to regenerate after changing Rust signatures. CI doesn't verify type generation.

**Prevention:**
- Never hand-edit `index.d.ts` — use `ts_args_type`/`ts_return_type` attributes for intentional deviations.
- Add `napi build` to CI and verify `index.d.ts` matches the generated output.
- Use `tsc --noEmit` against generated types in the test suite.

**Detection:** TypeScript compilation errors after changing Rust function signatures. Runtime crashes from type mismatches.

**Phase:** All phases — this is a build/CI concern that affects every NAPI binding change.

### Pitfall 12: tmux Paste-Buffer Escape Sequence Injection

**What goes wrong:** tmux's paste-buffer writes content directly to the pty without filtering escape sequences. If paste content contains `\e[201~` (bracketed paste end marker), it terminates bracketed paste mode early — anything after that marker is interpreted as direct keyboard input.

**Why it happens:** tmux doesn't sanitize paste content. This is a known vulnerability class with CVEs in other terminals.

**Consequences:** Pasted content containing escape sequences can execute commands or inject keystrokes. Low risk for Awrit (paste is limited), but relevant if the browser supports clipboard integration.

**Prevention:**
- Strip or replace `\e[200~` and `\e[201~` sequences from any content before writing to pty.
- Use bracketed paste mode properly — detect and handle the markers.
- For browser clipboard integration, sanitize HTML/text content before pasting.

**Detection:** Pasted text containing escape sequences causes unexpected behavior. Commands execute when pasting multi-line content.

**Phase:** Phase 5 (input handling) — if clipboard/paste integration is added.

### Pitfall 13: Image Cap Off-By-One Error

**What goes wrong:** When implementing an image retention cap (maximum number of images kept in memory), using `==` instead of `>=` for the comparison means only one image is ever freed when the cap is reached, causing unbounded memory growth.

**Why it happens:** Off-by-one in the cap check. The cap should free images when `count >= max`, but `==` only triggers at exactly `max`.

**Consequences:** Memory leak. Each rendered image is heap-allocated but never freed. After enough images, the application runs out of memory.

**Prevention:**
- Use `>=` not `==` for cap checks.
- Test with a loop that renders many images and monitor memory usage.
- Implement proper image lifecycle: allocation → display → replacement → cleanup.

**Detection:** Memory usage grows monotonically during long sessions. `htop` shows increasing RSS without release.

**Phase:** Phase 3 (image slot management) — the image cap must be implemented correctly from the start.

### Pitfall 14: GRID_LINE_WRAPPED Reflow Causing Ghost Images

**What goes wrong:** When image rows are terminated with `screen_write_linefeed` (which marks lines as `GRID_LINE_WRAPPED`), pane reflow joins the following line onto each image line during a split. This drops the placeholder cells and leaves stale images on screen.

**Why it happens:** tmux's `grid_reflow` function joins wrapped lines. Image placeholder cells get absorbed into the wrapped line and disappear from the grid, but the image they reference remains rendered.

**Consequences:** Ghost images persist after the placeholder cells are removed. Visual artifacts on pane split/resize.

**Prevention:**
- Terminate each placeholder row with a real newline, not a linefeed that marks the line as wrapped.
- Test pane split/resize extensively with images displayed.
- Monitor `grid_reflow` behavior in tmux logs during split operations.

**Phase:** Phase 4 (renderer integration) — the rendering pipeline must handle tmux's grid reflow correctly.

### Pitfall 15: NAPI Platform-Specific Binary Distribution

**What goes wrong:** The NAPI `.node` binary is compiled for one platform but fails to load on another. Different Linux distributions (musl vs glibc), different architectures (x64 vs arm64), different Node.js versions can all cause load failures.

**Why it happens:** NAPI binaries are platform-specific. A binary compiled on macOS won't load on Linux. A musl binary won't load on glibc systems.

**Prevention:**
- Use CI matrix builds for all target platforms.
- Use `napi build --platform` for cross-compilation.
- Provide fallback instructions for building from source.
- Document minimum Node.js version (tied to NAPI version).

**Detection:** `require()` fails with platform-specific error messages. "Invalid ELF header" or "wrong ELF class" errors.

**Phase:** Phase 1 (NAPI bindings) — build infrastructure must be set up for multi-platform before any Rust code is written.

## Phase-Specific Warnings

| Phase Topic | Likely Pitfall | Mitigation |
|-------------|---------------|------------|
| NAPI bindings (Phase 1) | Blocking event loop with sync calls (#6) | Design API as async from day one |
| NAPI bindings (Phase 1) | Platform binary distribution (#15) | Set up CI matrix before writing code |
| Rust tmux passthrough (Phase 2) | Ghost placements from a=T (#3) | Enforce a=t + a=p,U=1 in code review |
| Rust tmux passthrough (Phase 2) | Image leaking to adjacent panes (#9) | Unicode placeholders only, never direct placement |
| Image slot management (Phase 3) | SGR color collision (#4) | Force explicit 256-color or truecolor escaping |
| Image slot management (Phase 3) | Locale-dependent encoding (#8) | Literal UTF-8 bytes, not wctomb |
| Image slot management (Phase 3) | Memory leak from cap off-by-one (#13) | Use >= not ==, add memory monitoring tests |
| Renderer integration (Phase 4) | Print-time vs render-time race (#5) | Implement retro-resolution in rendering pipeline |
| Renderer integration (Phase 4) | Extended keys corrupting responses (#10) | One-way passthrough only, no queries |
| Renderer integration (Phase 4) | GRID_LINE_WRAPPED reflow (#14) | Real newlines, not linefeeds, for image rows |
| Input handling (Phase 5) | Crossterm cursor race (#7) | Track position manually, never query during EventStream |
| Input handling (Phase 5) | Paste-buffer injection (#12) | Sanitize paste content, handle bracketed markers |

## Sources

- tmux/tmux#3755 — DCS passthrough nested sequence truncation
- tmux/tmux#3287 — DCS passthrough regression since 3.3a
- tmux/tmux#4386 — Extended keys corrupting passthrough responses
- tmux/tmux#5530 — Local passthrough queries collide with tmux's own probes
- tmux/tmux#4902 — Kitty graphics protocol support in tmux
- tmux/tmux#5274 — Unicode placeholder implementation (grid-resident)
- kovidgoyal/kitty#7655 — Graphics protocol direct mode + tmux stops displaying images
- kovidgoyal/kitty#7070 — Distorted images with unicode placeholders
- kovidgoyal/kitty#6215 — Unicode passthrough over tmux over ssh
- kovidgoyal/kitty/pull/5664 — Unicode placeholder image placement implementation
- earendil-works/pi#2374 — Kitty inline images not rendered inside tmux (comprehensive bug report)
- saka1/mlux#32 — Kitty graphics commands silently dropped inside tmux
- ghostty-org/ghostty#13056 — Unicode placeholder rendering width bug
- napi-rs documentation — Type conversions, error handling, async/concurrency
- crossterm-rs/crossterm#672 — Cursor position race condition with EventStream
- crossterm-rs/crossterm#1039 — EventStream polling lock for viuer
- CVE-2020-27347 — tmux buffer overflow in escape sequence parser
- CVE-2024-38396 — iTerm2 escape sequence injection via tmux integration
