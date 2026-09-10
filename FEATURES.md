# Feature Landscape

**Domain:** Terminal web browser with Kitty graphics protocol and tmux support
**Researched:** 2026-09-01
**Context:** Hybridizing two Awrit forks (Rust tmux branch + TypeScript tmux-pane branch)

## Table Stakes

Features users expect from a Kitty-based terminal browser with tmux support. Missing = product feels broken.

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Kitty graphics rendering | Core value proposition — renders Chromium output as pixel graphics | High | Existing in both branches |
| DCS passthrough wrapping | tmux swallows raw Kitty APC sequences; must wrap in `\x1bPtmux;...\x1b\\` | Medium | Port from Rust tmux branch |
| Unicode placeholder support (U+10EEEE) | Images must be grid-resident for tmux pane switching, scrolling, clipping | High | Critical for tmux — overlay approach ghosts on redraw |
| Virtual placements (a=p,U=1) | Prevents ghost images; real placements (a=p,C=1) leave stale frames on tmux | Medium | Use a=t + a=p,U=1, never a=T with cursor placement |
| Mouse click forwarding | Users click links, buttons, form elements | Medium | tmux-pane branch has coordinate normalization |
| Scroll support | Fundamental navigation | Low | Both branches have basic support |
| Keyboard navigation | Vim-like keys, arrow keys, tab cycling | Low | tmux-pane branch has input handling |
| URL bar | Enter URLs, search queries | Low | casty has urlbar.js reference |
| Back/forward history | Browser navigation basics | Low | Standard feature |
| Tab management | Open/close/switch tabs | Medium | tmux-pane branch tracks pane state |
| Page reload | Refresh stale content | Low | Standard feature |
| Form handling | Fill fields, submit forms | High | Requires JS execution + input forwarding |

## Differentiators

Features that set Awrit apart from text-only browsers (lynx, w3m, links) and other Kitty-based tools.

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Full Chromium rendering | Real HTML5/CSS3/JS support — not text approximation like Browsh | Very High | Core differentiator — headless Chromium + CDP |
| Pane visibility tracking | Only render when pane is visible — saves CPU/GPU | Medium | tmux-pane branch has this |
| Image slot management | Reuse image IDs to prevent memory leaks | Medium | tmux-pane branch has imageIds.ts |
| Renderer lifecycle management | Proper setup/teardown on pane focus/blur | Medium | tmux-pane branch has tmuxRenderer.ts |
| Mouse coordinate normalization | Convert tmux cell coordinates to pixel coordinates for Chromium | High | tmux-pane branch has mouseCoordinates.ts |
| Vimium-style link hints | Keyboard-driven link following (f + hint keys) | Medium | casty has hints.js reference |
| Custom keybindings | Power users want to remap keys | Low | awrit has config.js |
| Persistent bookmarks | Save frequently visited URLs | Low | casty has bookmarks.js |
| Page search (ctrl+f) | Find text on page | Medium | Standard browser feature |
| DevTools integration | Debug web pages from terminal | High | casty uses CDP WebSocket |
| Extension support | Run Chrome extensions for ad blocking, etc. | Very High | glimpse-tty has userExtensions config |

## Anti-Features

Features to explicitly NOT build — either out of scope, wrong architecture, or technical impossibility.

| Anti-Feature | Why Avoid | What to Do Instead |
|--------------|-----------|-------------------|
| Sixel support | Awrit is Kitty-native; Sixel requires different encoding pipeline | Use Kitty protocol exclusively |
| Video playback in terminal | Frame rate too low for smooth video over tmux passthrough | Document limitation; offer download links |
| WebRTC/audio | Terminal has no audio output mechanism | Pass through to system browser for media |
| Browser extension ecosystem | Chrome extensions need full browser UI, not headless | Support unpacked extensions only (like glimpse-tty) |
| Print functionality | Terminal output is not printable in traditional sense | Offer "save as PDF" or HTML source export |
| Download management UI | Complex file handling; terminal not suited | Shell out to `wget`/`curl` with progress |
| Multi-window support | tmux already provides windowing; don't reinvent | Leverage tmux windows/panes |
| Session restore across reconnect | tmux sessions persist, but Kitty graphics don't survive reattach | Document limitation; images need retransmission |
| CSS grid/flexbox layout | Chromium handles layout; terminal doesn't need to | Let Chromium render, capture as pixels |
| Accessibility tree | Headless Chromium has no screen reader integration | Not applicable in terminal context |

## Feature Dependencies

```
Kitty graphics rendering → DCS passthrough wrapping (tmux)
Kitty graphics rendering → Unicode placeholder support (tmux)
Kitty graphics rendering → Virtual placements (tmux)
Kitty graphics rendering → Image slot management
Mouse click forwarding → Mouse coordinate normalization
Tab management → Pane visibility tracking
Tab management → Renderer lifecycle management
Form handling → Keyboard navigation (input forwarding)
```

## tmux-Specific Feature Matrix

| Feature | tmux behavior | Implementation required |
|---------|---------------|------------------------|
| Image rendering | APC sequences must be wrapped in DCS passthrough | `passthroughTmux()` Rust NAPI call |
| Image placement | Use virtual placements (U=1) with Unicode placeholders | `a=t` + `a=p,U=1`, write U+10EEEE cells |
| Image cleanup | Delete images when pane loses focus or scrolls | `d=i` or `d=I` delete commands |
| Pane switching | Images may ghost if not using placeholders | Unicode placeholder approach prevents this |
| Scrolling | tmux is not pixel-aware; images can stick until cells overwritten | Document as upstream limitation |
| Resize events | Terminal size changes need re-render | Listen to SIGWINCH, recalculate placement |
| Detach/reattach | Images lost on reattach — need retransmission | Future: per-client transmit tracking |
| Nested panes | Recursive DCS passthrough required | Test with tmux inside tmux |

## MVP Recommendation

Prioritize:
1. **DCS passthrough wrapping** — without this, nothing works in tmux
2. **Unicode placeholder support** — prevents ghost images on pane switch
3. **Virtual placements** — proper image lifecycle management
4. **Pane visibility tracking** — only render when visible

Defer:
- **Extension support**: Not needed for hybridization focus
- **Video playback**: Technical limitations make this impractical
- **DevTools integration**: Nice-to-have, not essential for MVP

## Sources

- Kitty graphics protocol: https://sw.kovidgoyal.net/kitty/graphics-protocol/
- tmux Kitty support PR: https://github.com/tmux/tmux/pull/5274
- tmux DCS passthrough: https://github.com/tmux/tmux/issues/4902
- casty browser: https://github.com/cashmeredev/casty
- awrit original: https://github.com/chase/awrit
- Chawan TUI browser: https://chawan.net/
- Browsh text browser: https://www.brow.sh/
- kitty-graphics.el tmux support: https://cashmere.rs/blog/kitty-graphicsel-v060-video-browser-and-improved-tmux/
