---
project: awrit
---
<!-- refreshed: 2026-09-01 -->
# Codebase Concerns

**Analysis Date:** 2026-09-01

## Tech Debt

**Global State & Module Singletons:**
- Issue: Heavy reliance on module-level singletons for critical state management across the application
- Files: `src/features.ts` (global features), `src/windows.ts` (focusedView, windowViews), `src/keybindings.ts` (bindings, currentSequence, timeoutId), `src/paint.ts` (weakPaintedContents_, tmuxRenderer), `src/tty/kittyGraphics.ts` (imageId_)
- Impact: Makes testing difficult, creates hidden dependencies, prevents multi-window support
- Fix approach: Refactor to dependency injection or context-based state management

**Config File Requires CommonJS Pattern:**
- Issue: `config.js` uses `module.exports` with CommonJS while the rest of the codebase uses ES modules, loaded via `require()` in `src/index.ts`
- Files: `config.js`, `src/index.ts:41`
- Impact: Creates module system inconsistency, cache invalidation hacks (line 46), and potential ESM compatibility issues
- Fix approach: Convert config to ES module or use dynamic import()

**TypeScript Suppression Comments:**
- Issue: 19 occurrences of `@ts-expect-error`, `@ts-ignore`, and `as any` across the codebase
- Files: `src/windows.ts:181,184,195,209,230`, `src/args.ts:52,54`, `src/abort.ts:1`, `src/runner/index.ts:105`
- Impact: Reduces type safety, hides potential runtime errors
- Fix approach: Create proper type definitions or interfaces for Electron APIs

**Zoom State Persistence Duplication:**
- Issue: Zoom state reading/writing logic duplicated between `config.js` (lines 302-387) and `src/zoom-state.ts`
- Files: `config.js:302-387`, `src/zoom-state.ts`
- Impact: Inconsistent behavior, maintenance burden, potential data corruption
- Fix approach: Extract shared zoom state logic into `src/zoom-state.ts` and use from both locations

**Console Logging Disabled Globally:**
- Issue: All console logging disabled in `src/console.ts:3`, making debugging difficult
- Files: `src/console.ts`
- Impact: No runtime logging available, errors only captured to `awrit_error.txt` via shell redirect
- Fix approach: Implement proper logging framework with configurable levels

## Known Bugs

**Terminal Color Query Reliability:**
- Symptoms: Some users cannot get terminal colors, resulting in empty `kitty.css` placeholder
- Files: `src/runner/index.ts:96`, `src/runner/kittyColors.ts:55-60`
- Trigger: Running awrit in terminals that don't respond to OSC 21 color queries within 100ms timeout
- Workaround: Empty placeholder CSS is written, toolbar uses default colors

**Content-Ready Event Timing:**
- Symptoms: Race condition where content might not be revealed if content-ready event fires after suppression timeout
- Files: `src/windows.ts:230-234`
- Trigger: Slow page loads or network latency
- Workaround: 200ms safety timeout forces reveal even if content-ready not received

**Zoom Factor Reset on Navigation:**
- Symptoms: Zoom level resets to 1.0 on frame navigation before restoring from saved state
- Files: `src/windows.ts:84-88, 246-253`
- Trigger: Navigating to a new page
- Workaround: Zoom is reapplied after `did-navigate` event

## Security Considerations

**Config File Code Execution:**
- Risk: `config.js` is loaded via `require()` and executed as code, potential for injection if file is writable by others
- Files: `config.js:1-411`, `src/index.ts:41-60`
- Current mitigation: File is loaded from application directory, assumes user owns the file
- Recommendations: Validate config structure, add integrity checks, consider JSON config format

**Shell Command Execution in Config:**
- Risk: `config.js` uses `child_process.exec()` for clipboard operations
- Files: `config.js:107, 128-142`
- Current mitigation: Timeout protection (500ms), command whitelisting (pbcopy, wl-copy, xclip)
- Recommendations: Audit clipboard command sources, consider using Electron clipboard API exclusively

**TMUX_PANE Injection Protection:**
- Risk: TMUX_PANE environment variable could be maliciously set
- Files: `src/tty/tmux.ts:27-36`, `awrit:39-42`
- Current mitigation: Regex validation `/^%\d+$/` in both TypeScript and bash
- Recommendations: Validation is sufficient, but consider sanitizing all tmux command outputs

**Shared Memory Naming:**
- Risk: Shared memory segments (`/awrit_*`) created with predictable timestamp-based names
- Files: `awrit-native-rs/src/lib.rs:46-61`
- Current mitigation: Nanosecond timestamps provide reasonable uniqueness
- Recommendations: Add random component to names for enhanced uniqueness

## Performance Bottlenecks

**TMUX Pane State Queries:**
- Problem: `execFileSync('tmux', ...)` called frequently for pane size/status checks
- Files: `src/tty/tmux.ts:19-25, 80-98, 109-130`
- Cause: Synchronous file execution blocks event loop, up to 250ms cache TTL
- Improvement path: Increase cache TTL, batch queries, or use async execution

**Image Buffer Reallocation:**
- Problem: ShmGraphicBuffer reallocated when image size increases, but old buffers not immediately freed
- Files: `src/paint.ts:100-106, 167-171, 225-234`
- Cause: Conservative allocation strategy to avoid frequent reallocations
- Improvement path: Implement buffer pooling or lazy cleanup

**Layout Calculation Recalculation:**
- Problem: Full layout tree recalculated on every SIGWINCH signal
- Files: `src/windows.ts:387-399`, `src/layout.ts:491-518`
- Cause: BFS traversal of entire layout tree, though debounced at 100ms
- Improvement path: Implement incremental layout updates for simple size changes

**Config File Polling:**
- Problem: `fs.watchFile()` polls config file every 200ms
- Files: `src/index.ts:43`
- Cause: `fs.watch()` unreliable across platforms, polling ensures consistency
- Improvement path: Consider using `fs.watch()` with fallback, or increase poll interval

## Fragile Areas

**Electron API Dependencies:**
- Files: `src/windows.ts`, `src/index.ts`, `src/extensions.ts`
- Why fragile: Relies on undocumented Electron APIs (`content.isSuppressingPaint`, `paintCount`, `content-ready` event)
- Safe modification: Add type declarations for undocumented APIs, test across Electron versions
- Test coverage: No integration tests for Electron API interactions

**Kitty Graphics Protocol:**
- Files: `src/tty/kittyGraphics.ts`, `src/tty/tmuxProtocol.ts`
- Why fragile: Terminal-specific protocol implementation, varies between terminal emulators
- Safe modification: Test against multiple terminal implementations, add feature detection
- Test coverage: Unit tests exist for protocol building, no end-to-end tests

**TMUX Passthrough Mode:**
- Files: `src/tty/tmux.ts:136-142`, `src/tty/tmuxRenderer.ts`
- Why fragile: Complex escape sequence wrapping, version-dependent behavior
- Safe modification: Test with tmux 3.4+, validate passthrough mode before use
- Test coverage: Tests exist for basic wrapping, not for real tmux integration

**Native Module Integration:**
- Files: `awrit-native-rs/src/lib.rs`, `awrit-native-rs/src/input.rs`, `awrit-native-rs/src/term.rs`
- Why fragile: Rust/Node.js FFI boundary, cross-platform compilation
- Safe modification: Use NAPI-RS type safety, test on all target platforms
- Test coverage: No Rust unit tests visible, relies on TypeScript integration tests

## Scaling Limits

**Image ID Space:**
- Current capacity: 32-bit image IDs (4 billion unique IDs)
- Limit: Image ID wraps around at `MAX_IMAGE_ID` (0xffffffff)
- Scaling path: IDs are recycled after deletion, wrap-around is handled

**TMUX Placeholder Grid:**
- Current capacity: Limited by `ROW_COLUMN_DIACRITICS` array (299 entries)
- Limit: Visible image grid cannot exceed 299x299 cells
- Scaling path: Add more diacritics to array or implement alternative encoding

**Shared Memory Segments:**
- Current capacity: System default limit (typically 64KB per segment, 4MB total)
- Limit: Very large images may exceed shared memory limits
- Scaling path: Implement chunked transfers for large images

## Dependencies at Risk

**Electron:**
- Risk: Major version updates may break undocumented APIs used in codebase
- Impact: Window management, offscreen rendering, extension support
- Migration plan: Pin to minor versions, test upgrades against documented APIs

**electron-chrome-extensions:**
- Risk: Third-party package for Chrome extension compatibility, may lag behind Electron updates
- Impact: Extension support (currently uBlock Origin Lite only)
- Migration plan: Monitor updates, consider direct Chrome DevTools Protocol integration

**awrit-native-rs:**
- Risk: Custom native module with cross-platform compilation requirements
- Impact: Terminal input handling, shared memory graphics, terminal feature detection
- Migration plan: Maintain Rust expertise, ensure CI builds for all targets

## Missing Critical Features

**Multi-Window Support:**
- Problem: `managedViews` array and `focusedView` singleton prevent multiple browser windows
- Blocks: Users cannot open multiple awrit instances with proper coordination

**Session Persistence:**
- Problem: No browser session save/restore functionality
- Blocks: Users lose open tabs and navigation history on restart

**Error Recovery:**
- Problem: Limited graceful degradation when terminal features unavailable
- Blocks: Users cannot use awrit in terminals without full Kitty graphics support

## Test Coverage Gaps

**Electron Integration:**
- What's not tested: BrowserWindow creation, IPC communication, extension loading
- Files: `src/windows.ts`, `src/extensions.ts`, `src/preload.js`
- Risk: Electron API changes could break core functionality undetected
- Priority: High

**Terminal Protocol:**
- What's not tested: Real terminal I/O, escape sequence parsing, Kitty graphics rendering
- Files: `src/tty/output.ts`, `src/tty/kittyGraphics.ts`
- Risk: Terminal compatibility issues not caught until user testing
- Priority: High

**Configuration Loading:**
- What's not tested: Config file parsing, hot-reload, error recovery
- Files: `config.js`, `src/index.ts:25-60`
- Risk: Malformed config could crash application or cause undefined behavior
- Priority: Medium

**Native Module FFI:**
- What's not tested: Rust/JavaScript boundary, shared memory operations, input handling
- Files: `awrit-native-rs/src/lib.rs`, `awrit-native-rs/src/input.rs`
- Risk: Memory safety issues, platform-specific bugs
- Priority: Medium

---

*Concerns audit: 2026-09-01*
