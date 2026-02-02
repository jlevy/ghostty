# Research: Ghostty Terminal Rendering Architecture & Rich Terminal Features

**Date:** 2026-02-02 (last updated 2026-02-02)

**Author:** Claude (AI Research Assistant)

**Status:** Complete

## Overview

This research document provides a comprehensive analysis of Ghostty's terminal rendering architecture to assess the feasibility of implementing advanced terminal features. The goal is to understand the complete rendering pipeline from TTY input to visual output and evaluate what architectural changes would be required for rich terminal enhancements.

## Questions to Answer

1. How does Ghostty's rendering pipeline work from TTY to screen?
2. What terminal data structures exist and how flexible are they?
3. What extension points exist for adding visual overlays and interactions?
4. Is it feasible to embed web iframe overlays on top of terminal content?
5. Can we implement text hovers/highlights on regex patterns?
6. Can we implement collapsible/toggleable text blocks?
7. What animation support exists (smooth scroll, expand/collapse)?
8. What font size/style control is possible (e.g., DECDWL/DECDHL double-width/height characters)?

## Scope

**Included:**
- Complete analysis of Ghostty's rendering architecture
- TTY-level data flow (PTY, parser, stream handler)
- Visual rendering pipeline (Metal/OpenGL, shaders, font system)
- Terminal data structures (Cell, Page, PageList, Screen)
- Existing overlay and extension systems
- Detailed feasibility analysis for each proposed feature
- Specific implementation specifications for macOS WebView overlays

**Excluded:**
- Actual implementation of proposed features
- Performance benchmarking
- Third-party terminal comparisons

---

## Findings

### 1. High-Level Architecture

Ghostty has a well-architected, modular rendering pipeline with clear separation of concerns:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              Application Layer                               │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │  macOS (Swift/AppKit)  │  GTK (Zig/GObject)  │  libghostty (Library)   ││
│  └─────────────────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                               Core Surface Layer                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐│
│  │   Surface    │  │    Input     │  │   Config     │  │     Actions      ││
│  │  (Surface.zig)│  │  Handling   │  │   System     │  │   & Messages     ││
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────────┘│
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                    ┌─────────────────┼─────────────────┐
                    ▼                 ▼                 ▼
┌──────────────────────┐  ┌──────────────────┐  ┌──────────────────────────┐
│     Terminal I/O     │  │    Renderer      │  │      Font System         │
│  ┌────────────────┐  │  │  ┌────────────┐  │  │  ┌──────────────────┐   │
│  │  PTY Master    │  │  │  │  Generic   │  │  │  │  Discovery       │   │
│  │  (pty.zig)     │  │  │  │  Renderer  │  │  │  │  (fontconfig/    │   │
│  └────────────────┘  │  │  │ (generic.zig)│ │  │  │   coretext)      │   │
│  ┌────────────────┐  │  │  └────────────┘  │  │  └──────────────────┘   │
│  │  StreamHandler │  │  │  ┌────────────┐  │  │  ┌──────────────────┐   │
│  │ (stream_handler│  │  │  │  Metal/    │  │  │  │  Shaper          │   │
│  │      .zig)     │  │  │  │  OpenGL/   │  │  │  │  (HarfBuzz/      │   │
│  └────────────────┘  │  │  │  WebGL     │  │  │  │   CoreText)      │   │
│  ┌────────────────┐  │  │  └────────────┘  │  │  └──────────────────┘   │
│  │    Parser      │  │  │  ┌────────────┐  │  │  ┌──────────────────┐   │
│  │  (Parser.zig)  │  │  │  │   Shaders  │  │  │  │  Atlas           │   │
│  └────────────────┘  │  │  │ (GLSL/MSL) │  │  │  │  (texture pack)  │   │
│                      │  │  └────────────┘  │  │  └──────────────────┘   │
└──────────────────────┘  └──────────────────┘  └──────────────────────────┘
```

### 2. Threading Model

Ghostty uses a multi-threaded architecture:

| Thread | Responsibility | Key Files |
|--------|---------------|-----------|
| **Main/App** | Event loop, user input, window management | Platform-specific |
| **I/O Thread** | PTY read loop, parser/state updates, wakes renderer | `src/termio/Thread.zig` |
| **Renderer Thread** | 120 FPS target, GPU upload, draw calls, cursor blink | `src/renderer/Thread.zig` |

### 3. TTY-Level Architecture

#### PTY Communication
**File:** `src/pty.zig`

The PTY layer handles bidirectional communication with shell processes:
- Master fd for reading/writing to the terminal
- Slave fd for the child shell process
- Window size tracking for resize handling

#### Parser State Machine
**File:** `src/terminal/Parser.zig`

Implements the VT100/VT102 parser from [vt100.net](https://vt100.net/emu/dec_ansi_parser):

```
Ground → Escape → CSI Entry → CSI Param → CSI Dispatch
                    ↓
             OSC String (Operating System Commands)
```

**Parser Actions:**
- `print` (98% hot path - printable characters)
- `csi_dispatch` (cursor movement, SGR colors, etc.)
- `osc_dispatch` (OSC 8 hyperlinks, title, etc.)
- `esc_dispatch` (DEC private modes)

### 4. Terminal Data Structures

#### Cell Structure (64-bit packed)
**File:** `src/terminal/page.zig:1962`

```zig
pub const Cell = packed struct(u64) {
    content_tag: ContentTag,      // 3 bits: codepoint, grapheme, bg_color
    content: packed union {
        codepoint: u21,           // Unicode codepoint
        color_palette: u8,        // Palette index
        color_rgb: RGB,           // 24-bit RGB
    },
    style_id: StyleId,            // Index into page's style_set
    wide: Wide,                   // narrow, wide, spacer_head, spacer_tail
    protected: bool,              // DECSCA protection
    hyperlink: bool,              // Has OSC8 link
    semantic_content: enum,       // output, input, prompt
};
```

**Design Rationale:**
- Packed into exactly 64 bits for atomic operations
- Style deduplication via reference counting (styles are shared)
- Supports grapheme clusters via separate storage
- Wide character tracking for CJK and emoji

#### Page Memory Architecture
**File:** `src/terminal/page.zig`

Pages use single contiguous, page-aligned memory blocks containing:
- Row/cell storage
- StyleSet (style → StyleId mapping, deduplicated)
- GraphemeMap (extended grapheme storage for emoji/combining marks)
- HyperlinkSet/Map (hyperlink deduplication)
- KittyImageStorage (inline images)

#### PageList (Scrollback Buffer)
**File:** `src/terminal/PageList.zig`

Doubly-linked list of Pages with:
- Tracked pins that auto-update on mutations
- Memory pool for page reuse
- Viewport pin for scroll position tracking

### 5. Visual Rendering Pipeline

#### Renderer Thread Loop
**File:** `src/renderer/Thread.zig`

```
┌─────────────────────────────────────────────────────────────────┐
│                    Renderer Thread Loop                          │
│                                                                  │
│    Render Timer (8ms = 120 FPS)                                 │
│           │                                                      │
│           ▼                                                      │
│    updateFrame() → rebuildCells() → drawFrame()                 │
│    (lock mutex,   (convert term    (GPU upload,                 │
│     read term     state to GPU     render passes,               │
│     state)        vertex data)     present)                     │
└─────────────────────────────────────────────────────────────────┘
```

#### GPU Render Passes
**File:** `src/renderer/generic.zig`

| Pass | Content |
|------|---------|
| 0 | Background (clear + background image) |
| 1 | Cell backgrounds (CellBg vertices) |
| 2 | Text & decorations (glyphs, underlines, strikethroughs) |
| 3 | Overlays (cursor, selection, search, hyperlink highlights) |
| 4 | Images (Kitty graphics below/above text) |
| 5+ | Custom shaders (ShaderToy-style effects) |

#### Graphics Backend Abstraction
**File:** `src/renderer/backend.zig`

| Platform | Backend | Notes |
|----------|---------|-------|
| macOS | Metal | Triple buffering, MSL shaders |
| Linux/Windows | OpenGL 4.3+ | Single buffering, GLSL shaders |
| Browser | WebGL | Minimal implementation |

### 6. Existing Extension Points

#### Highlight System
**File:** `src/terminal/highlight.zig`

Pin-based region tracking for:
- Text selection
- Search result highlighting
- Hyperlink hover highlighting

#### Link Detection System
**Files:** `src/input/Link.zig`, `src/renderer/link.zig`

Regex-based link detection with configurable highlighting:
```
link = regex:pattern action:open highlight:hover
```

#### Overlay System (GTK)
**File:** `src/apprt/gtk/class/`

Existing overlays: ResizeOverlay, SearchOverlay, KeyStateOverlay, InspectorWidget

#### Custom Shader System
**File:** `src/renderer/shadertoy.zig`

Supports user-provided GLSL/MSL shaders with `iTime` uniform for animations.

#### Kitty Graphics Protocol
**File:** `src/terminal/kitty/graphics.zig`

Full inline image support: PNG, JPEG, GIF, WebP with three z-layers.

### 7. macOS View Hierarchy (Detailed)

**Files:**
- `macos/Sources/Ghostty/Surface View/SurfaceView.swift`
- `macos/Sources/Ghostty/Surface View/SurfaceView_AppKit.swift`
- `macos/Sources/Ghostty/Surface View/SurfaceScrollView.swift`

```
SwiftUI SurfaceWrapper
└── ZStack (Overlay Composition)
    ├── Layer 0:  GeometryReader + SurfaceRepresentable
    │             └── SurfaceScrollView (NSView)
    │                 └── NSScrollView
    │                     └── SurfaceView (NSView with Metal Layer)
    ├── Layer 1:  SurfaceResizeOverlay (size indicator)
    ├── Layer 2:  SurfaceProgressBar (top progress bar)
    ├── Layer 3:  ReadonlyBadge (top-right corner)
    ├── Layer 4:  KeyStateIndicator (draggable key state pill)
    ├── Layer 5:  URL Tooltip (bottom-right hover URL)
    ├── Layer 6:  SecureInputOverlay (lock indicator)
    ├── Layer 7:  SurfaceSearchOverlay (draggable search bar)
    ├── Layer 8:  BellBorderOverlay (animated border)
    ├── Layer 9:  HighlightOverlay (pulsing glow)
    ├── Layer 10: Error/Unhealthy state overlay
    ├── Layer 11: Unfocused split dimming
    └── Layer 12: SurfaceGrabHandle (top drag area)
```

### 8. Coordinate System Conversion (macOS)

Critical for positioning overlays:

```
Terminal Coordinates          View Coordinates           Screen Coordinates
(row, col) grid units    →    (x, y) pixels         →   (x, y) screen pixels
Origin: top-left             Origin: bottom-left        Origin: varies
+Y: down                     +Y: up (AppKit)            +Y: up
```

**Conversion Code Path (`SurfaceView_AppKit.swift:1851-1906`):**
1. Get pixel coordinates from libghostty (`ghostty_surface_ime_point`)
2. Flip Y axis: `view_y = frame.height - terminal_y`
3. Convert to window coordinates: `convert(viewRect, to: nil)`
4. Convert to screen coordinates: `window.convertToScreen()`

---

## Options Considered

### Option A: Web iFrame Overlays via Native Compositor

**Description:** Layer native WebView (WKWebView on macOS, WebKitWebView on GTK) on top of the Metal/OpenGL terminal renderer using the existing SwiftUI ZStack overlay pattern.

**Pros:**
- Uses existing overlay architecture pattern
- Full web rendering capabilities (HTML, CSS, JavaScript)
- Can display rich content (tooltips, documentation, previews)
- Platform-native WebView is well-optimized

**Cons:**
- Requires platform-specific implementation (macOS, GTK, Windows separately)
- Complex coordinate system conversion
- Focus/event routing complexity
- Security considerations (CSP, sandboxing)
- Memory overhead of WebView instances

**Estimated Effort:** 3-5 weeks for macOS, additional 3-5 weeks per platform

### Option B: Regex Text Hovers/Highlights (Enhancement)

**Description:** Extend existing link system with custom highlight styles and tooltip content.

**Pros:**
- Foundation already exists and works
- Minimal architectural changes
- Low risk, incremental improvement

**Cons:**
- Limited to what the existing system can express
- No rich content (just styled text)

**Estimated Effort:** 1-4 weeks

### Option C: Collapsible Text Blocks (Display-Only)

**Description:** Implement viewport filtering to hide/show content based on fold markers, without changing terminal state.

**Pros:**
- Doesn't break terminal semantics
- Shell doesn't need to know about folds
- Could use OSC sequences for fold markers

**Cons:**
- Very complex to implement correctly
- Selection behavior across folds is tricky
- Search within folded content is problematic
- Scrollback handling adds complexity

**Estimated Effort:** 2-4 months

### Option D: DECDWL/DECDHL Double-Width/Height Characters

**Description:** Implement VT standard escape sequences for double-width and double-height characters.

**Pros:**
- Well-defined VT standard
- Localized changes to row handling
- Good test case for variable rendering
- Some programs actually use this (BBS art, etc.)

**Cons:**
- Requires row metadata extension
- Parser and renderer changes
- Selection behavior changes needed

**Estimated Effort:** 2-4 weeks

### Option E: Scroll Animation

**Description:** Interpolate viewport position over time for smooth scrolling.

**Pros:**
- Pure renderer change
- No terminal state impact
- Modern UI expectation

**Cons:**
- May feel sluggish if overdone
- Need to handle interruption cleanly

**Estimated Effort:** 1-2 weeks

---

## Recommendations

Based on the findings, we recommend the following prioritized implementation order:

### Phase 1: Low-Hanging Fruit (1-2 months)
1. **Enhanced regex highlights with custom styles** - Extend existing Link struct
2. **Scroll animations** - Pure renderer change, no terminal state impact
3. **DECDWL/DECDHL support** - Well-defined VT standard

### Phase 2: Medium Complexity (2-3 months)
4. **Tooltip system for hovers** - Extend hoverUrl infrastructure
5. **macOS WebView overlays** - Platform-specific but high value

### Phase 3: High Complexity (3-6 months)
6. **GTK WebView overlays** - Port macOS implementation
7. **Collapsible text blocks** - Requires careful design, consider shell integration protocol

### Not Recommended
- **Per-cell font sizes** - Fundamentally breaks grid model, would require rich text editor architecture

---

## Detailed Implementation Specification: macOS WebView Overlays

### Proposed Files

```
macos/Sources/Ghostty/Overlays/
├── WebOverlay.swift           // WKWebView wrapper
├── WebOverlayManager.swift    // Manages multiple overlays
├── WebOverlayConfig.swift     // Configuration/styling
└── WebOverlayPosition.swift   // Terminal → screen positioning
```

### Core Data Structures

```swift
struct WebOverlayConfig: Identifiable {
    let id: UUID = UUID()

    // Anchor point in terminal coordinates
    var anchorRow: Int
    var anchorCol: Int

    // Size
    enum Size {
        case cells(rows: Int, cols: Int)
        case pixels(width: CGFloat, height: CGFloat)
        case auto
    }
    var size: Size

    // Position relative to anchor
    enum Anchor {
        case topLeft, topRight, bottomLeft, bottomRight
        case above, below
    }
    var anchor: Anchor = .below

    // Content
    enum Content {
        case html(String)
        case url(URL)
        case data(Data, mimeType: String)
    }
    var content: Content

    // Behavior
    var dismissOnClickOutside: Bool = true
    var dismissOnEscape: Bool = true
    var dismissOnScroll: Bool = false
    var capturesMouse: Bool = true
    var capturesKeyboard: Bool = false

    // Appearance
    var backgroundColor: NSColor = .clear
    var cornerRadius: CGFloat = 8
    var shadow: Bool = true
}
```

### API Options

| Option | Description | Example |
|--------|-------------|---------|
| **OSC Protocol** | Shell can trigger overlays | `OSC 1337 ; WebOverlay ; row=5 ; col=10 ; html=<base64> ST` |
| **C API** | libghostty function | `ghostty_surface_show_web_overlay(surface, &config)` |
| **Config-Driven** | Regex hover triggers | `link = regex:pattern hover:web-preview hover-url:...` |

### Event Handling

```
Mouse Click → Hit Test → In Overlay?
    → Yes: WebView handles (links, scroll)
    → No: dismissOnClickOutside? → Dismiss
         → Terminal handles event

Keyboard → ESC? → dismissOnEscape? → Dismiss
        → capturesKeyboard? → WebView handles
        → Terminal handles
```

### Implementation Checklist

| Component | Files | Effort |
|-----------|-------|--------|
| WebOverlayView | New Swift file | 2-3 days |
| WebOverlayManager | New Swift file | 2-3 days |
| Position calculation | Extend SurfaceView | 1-2 days |
| SwiftUI integration | Modify SurfaceView.swift | 1 day |
| Event handling | WebOverlayView | 2-3 days |
| Scroll sync | SurfaceScrollView + Manager | 2-3 days |
| OSC protocol | Parser + stream_handler | 3-5 days |
| C API bridge | embedded.zig + ghostty.h | 2-3 days |
| Testing/polish | Various | 3-5 days |
| Documentation | Config docs, man pages | 1-2 days |

**Total: 3-5 weeks**

### Security Considerations

1. **Content Security Policy** - Sandboxed WKWebView, disable JS for untrusted content
2. **URL Validation** - Whitelist trusted domains, block file:// URLs
3. **Resource Limits** - Max concurrent overlays, auto-dismiss timeout, memory monitoring

---

## Next Steps

- [ ] Discuss priorities with project maintainer
- [ ] Create beads for approved implementation phases
- [ ] Prototype enhanced regex highlights (Phase 1)
- [ ] Prototype scroll animations (Phase 1)
- [ ] Design OSC protocol for web overlays

## References

### Key Source Files

| Component | File | Purpose |
|-----------|------|---------|
| PTY | `src/pty.zig` | PTY communication |
| Parser | `src/terminal/Parser.zig` | VT state machine |
| Cell | `src/terminal/page.zig:1962` | Cell data structure |
| Style | `src/terminal/style.zig` | Style system |
| Render loop | `src/renderer/Thread.zig:198` | Renderer thread |
| Frame update | `src/renderer/generic.zig:1110` | Frame building |
| Frame draw | `src/renderer/generic.zig:1393` | GPU rendering |
| Cell building | `src/renderer/cell.zig` | Cell → GPU conversion |
| Link detection | `src/renderer/link.zig` | Regex link system |
| Font atlas | `src/font/Atlas.zig` | Glyph texture packing |
| macOS Surface | `macos/Sources/Ghostty/Surface View/` | Swift UI layer |
| GTK Surface | `src/apprt/gtk/class/surface.zig` | GTK widget |

### External Resources

- [VT100 Parser State Machine](https://vt100.net/emu/dec_ansi_parser)
- [Kitty Graphics Protocol](https://sw.kovidgoyal.net/kitty/graphics-protocol/)
- [OSC 8 Hyperlinks](https://gist.github.com/egmontkob/eb114294efbcd5adb1944c9f3cb5feda)
