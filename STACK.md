# Technology Stack

**Project:** Awrit Tmux Hybrid — terminal web browser with tmux support
**Researched:** 2026-09-01
**Context:** Existing Electron-based browser hybridizing Rust + TypeScript branches

## Executive Summary

This is NOT a greenfield project. Awrit already has a working stack: Electron 37, TypeScript with Bun, Rust via NAPI-RS 3.x, and crossterm for terminal I/O. The research question is about verifying that the current dependencies are current and understanding the tmux passthrough protocol requirements.

**Key finding:** The current stack is already well-chosen and recent. The main work is cherry-picking Rust tmux code from the `tmux` branch into the existing `tmux-pane` branch, not choosing new technologies.

## Recommended Stack (Current State + Required Versions)

### Core Runtime
| Technology | Version | Purpose | Confidence | Why |
|------------|---------|---------|------------|-----|
| Electron | ^37.3.1 | App shell, web content rendering | HIGH | Already in use, recent stable |
| Node.js | >=22.11.0 | Runtime for TypeScript | HIGH | Required by Electron 37 (bundles Node 22.16.0) |
| Bun | Latest | TypeScript execution, testing | HIGH | Already in use, fast runtime |

### Language & Build
| Technology | Version | Purpose | Confidence | Why |
|------------|---------|---------|------------|-----|
| TypeScript | ^5.9.2 | Type safety, renderer logic | HIGH | Already in use |
| Rust | Edition 2021 | Performance-critical terminal I/O | HIGH | Already in use |
| NAPI-RS | =3.2.4 (Rust), ^3.2.4 (CLI) | Rust↔TypeScript FFI | HIGH | Already pinned, proven stable |
| Biome | 2.2.2 | Formatting, linting | HIGH | Already in use |

### Terminal Protocol
| Technology | Version | Purpose | Confidence | Why |
|------------|---------|---------|------------|-----|
| crossterm | Forked (subtree) | Terminal manipulation, raw mode, keyboard/mouse | HIGH | Forked for customization, tracks upstream |
| Kitty Graphics Protocol | APC sequences | Image rendering to terminal | HIGH | Already implemented in tmuxProtocol.ts |
| tmux DCS Passthrough | `allow-passthrough on` | Escaping ANSI through tmux | HIGH | Required, tmux 3.3+ |

### Native Modules
| Technology | Version | Purpose | Confidence | Why |
|------------|---------|---------|------------|-----|
| awrit-native-rs | 2.0.3 | Rust NAPI bindings | HIGH | Already in use |
| nix | 0.29.0 | Unix system calls (shared memory) | HIGH | Already in use |
| base64 | 0.22.1 | Encoding for graphics protocol | HIGH | Already in use |
| bgra-to-rgba | local crate | Image format conversion | HIGH | Already in use |

## tmux Protocol Requirements

### DCS Passthrough (tmux 3.3+)
The tmux DCS passthrough wraps escape sequences in `\x1bPtmux;...\x1b\\` with ESC bytes doubled.

**Critical details:**
- Disabled by default (security). Must be enabled: `set -g allow-passthrough on` or `set -p allow-passthrough on`
- tmux 3.3a+ supports `allow-passthrough all` for all panes
- Every ESC byte (0x1B) in the inner sequence must be doubled to ESC ESC
- tmux reduces ESC ESC back to single ESC when forwarding
- **Not all terminals support passthrough** — Kitty, iTerm2, WezTerm, Ghostty do; Alacritty, foot, xterm do not

### Kitty Graphics in tmux
**Key insight from research:** tmux does NOT natively support Kitty graphics passthrough as of stable releases. The `tmux` branch on GitHub has experimental Kitty image support using Unicode placeholders (`U+10EEEE`), but this is not yet released.

**Current approach (tmux-pane branch):**
1. Transmit image data via DCS passthrough to outer terminal
2. Write Unicode placeholder cells (`U+10EEEE`) into tmux's grid
3. Image ID encoded in foreground color, row/column in diacritics
4. tmux scrolls/clips placeholder cells as text, outer terminal composites image

**This is the correct approach** — it's what the Kitty spec recommends for multiplexers.

## What NOT to Use

| Anti-Pattern | Why |
|--------------|-----|
| OSC 133 for image data | Not supported by tmux passthrough |
| Direct tmux control mode | Overkill, breaks terminal abstraction |
| Sixel protocol | tmux has partial support but Kitty is primary target |
| Pre-built tmux bindings | Use `execFileSync('tmux', ...)` for simplicity |
| Upgrade NAPI-RS to 3.12.x | Current 3.2.4 is pinned for stability; upgrade only if needed |

## Porting Strategy (from tmux branch)

### Files to Port
```bash
# Rust tmux utilities (from tmux branch)
awrit-native-rs/crates/crossterm/src/tmux.rs  # is_tmux(), tmux_passthrough(), MaybeTmux

# NAPI bindings (from tmux branch)  
awrit-native-rs/src/term.rs  # Add isTmux(), passthroughTmux(), writeMaybeTmux()

# TypeScript type definitions
awrit-native-rs/index.d.ts  # Add exported types
```

### NAPI-RS Binding Pattern
```rust
#[napi]
pub fn is_tmux() -> bool {
    crossterm::tmux::is_tmux()
}

#[napi]
pub fn passthrough_tmux(sequence: String) -> napi::Result<String> {
    crossterm::tmux::tmux_passthrough(sequence.as_bytes())
        .map(|bytes| String::from_utf8(bytes).unwrap_or_default())
        .map_err(|e| napi::Error::from_reason(e.to_string()))
}
```

### TypeScript Replacement
```typescript
// Replace tmuxWrap() calls with:
import { isTmux, passthroughTmux } from 'awrit-native-rs';

function tmuxWrap(sequence: string): string {
  return isTmux() ? passthroughTmux(sequence) : sequence;
}
```

## Confidence Assessment

| Area | Level | Reason |
|------|-------|--------|
| Core Stack | HIGH | Already in use, versions verified |
| NAPI-RS | HIGH | v3.2.4 is stable, v3.12.2 available but not needed |
| crossterm | HIGH | Forked subtree, tracks upstream 0.29.0 |
| tmux DCS | HIGH | Protocol spec well-documented, tmux 3.3+ required |
| Kitty Graphics | HIGH | Unicode placeholder approach is spec-recommended |
| Electron Compatibility | HIGH | Electron 37 bundles Node 22, NAPI-RS v3 supports it |

## Sources

- tmux DCS passthrough spec: https://vtdn.dev/docs/dcs/tmux-passthrough
- Kitty graphics protocol: https://sw.kovidgoyal.net/kitty/graphics-protocol/
- NAPI-RS releases: https://github.com/napi-rs/napi-rs/releases
- crossterm releases: https://github.com/crossterm-rs/crossterm/releases
- Electron schedule: https://releases.electronjs.org/schedule
- tmux Kitty support PR: https://github.com/tmux/tmux/pull/5274
