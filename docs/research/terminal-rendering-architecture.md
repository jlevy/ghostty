# Ghostty Terminal Rendering Architecture Research Document

## Executive Summary

This document provides a comprehensive analysis of Ghostty's terminal rendering architecture, examining the complete pipeline from TTY input to visual output. It evaluates the feasibility of implementing advanced terminal features including web iframe overlays, regex-based text hovers, collapsible text blocks, animations, and enhanced font control.

**Key Findings:**
- Ghostty has a well-architected, modular rendering pipeline with clear separation of concerns
- The existing overlay/highlight systems provide extension points for many features
- Some proposed features (iframe overlays, collapsible blocks) would require significant architectural changes
- Font size variations within a terminal session would require fundamental changes to the grid-based rendering model

---

## Part 1: Terminal Architecture Overview

### 1.1 High-Level Architecture Diagram

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
                    │                 │
                    ▼                 │
┌──────────────────────────────────────────────────────────────────────────────┐
│                            Terminal State Layer                               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐ │
│  │   Terminal   │  │    Screen    │  │   PageList   │  │      Page        │ │
│  │(Terminal.zig)│  │ (Screen.zig) │  │(PageList.zig)│  │   (page.zig)     │ │
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────────┘ │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐ │
│  │    Modes     │  │    Cursor    │  │   Selection  │  │     Styles       │ │
│  │ (modes.zig)  │  │              │  │  & Highlight │  │   (style.zig)    │ │
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────────┘ │
└──────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Threading Model

Ghostty uses a multi-threaded architecture:

```
┌────────────────────┐     ┌────────────────────┐     ┌────────────────────┐
│    Main/App        │     │    I/O Thread      │     │   Renderer Thread  │
│    Thread          │     │                    │     │                    │
│                    │     │  - PTY read loop   │     │  - 120 FPS target  │
│  - Event loop      │     │  - Parser/state    │     │  - GPU upload      │
│  - User input      │◀────│    updates         │────▶│  - Draw calls      │
│  - Window mgmt     │     │  - Wakes renderer  │     │  - Cursor blink    │
│                    │     │                    │     │                    │
└────────────────────┘     └────────────────────┘     └────────────────────┘
          │                                                    │
          │              ┌────────────────────┐                │
          └─────────────▶│  Shared State      │◀───────────────┘
                         │  (mutex-protected) │
                         │  - Terminal state  │
                         │  - Render state    │
                         └────────────────────┘
```

**Key Files:**
- `src/termio/Thread.zig` - I/O thread management
- `src/renderer/Thread.zig` - Renderer thread with timer-based updates
- `src/renderer/State.zig` - Thread-safe shared render state

---

## Part 2: TTY-Level Architecture

### 2.1 PTY Communication

**File:** `src/pty.zig`

The PTY (pseudo-terminal) layer handles bidirectional communication with shell processes:

```zig
pub const PosixPty = struct {
    master: std.posix.fd_t,    // Master file descriptor (read/write)
    slave: std.posix.fd_t,     // Slave file descriptor (child process)
    winsize: std.posix.winsize, // Terminal dimensions
};
```

**Data Flow:**
1. Shell process writes to slave fd
2. Ghostty reads from master fd
3. Bytes fed into StreamHandler
4. Parser generates actions
5. Actions update Terminal state

### 2.2 Parser State Machine

**File:** `src/terminal/Parser.zig`

Implements the VT100/VT102 parser from [vt100.net](https://vt100.net/emu/dec_ansi_parser):

```
                    ┌─────────────┐
                    │   Ground    │◀──── Printable chars
                    └─────────────┘
                          │ ESC
                          ▼
                    ┌─────────────┐
              ┌─────│   Escape    │─────┐
              │     └─────────────┘     │
              │ [         │ ]           │ other
              ▼           │             ▼
        ┌───────────┐     │      ┌─────────────┐
        │ CSI Entry │     │      │  ESC Dispatch│
        └───────────┘     │      └─────────────┘
              │           ▼
              ▼     ┌─────────────┐
        ┌───────────┐│ OSC String │
        │ CSI Param │└─────────────┘
        └───────────┘
              │
              ▼
        ┌───────────┐
        │CSI Dispatch│───▶ Execute CSI command
        └───────────┘
```

**Parser Actions:**
```zig
pub const Action = union(enum) {
    print: u21,              // Printable codepoint
    execute: u8,             // C0/C1 control
    csi_dispatch: CSI,       // CSI sequence (cursor, SGR, etc.)
    esc_dispatch: ESC,       // Escape sequence
    osc_dispatch: OSC,       // Operating System Command
    dcs_hook/put/unhook,     // Device Control String
    apc_start/put/end,       // Application Program Command
};
```

### 2.3 Stream Handler Pipeline

**File:** `src/termio/stream_handler.zig`

The StreamHandler coordinates parsing and state updates:

```
PTY bytes → Parser.next() → Action → StreamHandler.vt() → Terminal.*
                                           │
                                           ├── print (98% hot path)
                                           ├── csi_dispatch → CSI handlers
                                           ├── osc_dispatch → OSC handlers
                                           └── esc_dispatch → ESC handlers
```

**Performance Optimization:** Branch hints mark the `print` action as the likely path (98% of bytes are printable characters).

---

## Part 3: Terminal Data Structures

### 3.1 Cell Structure (64-bit packed)

**File:** `src/terminal/page.zig:1962`

```zig
pub const Cell = packed struct(u64) {
    // Content type selector
    content_tag: ContentTag,      // 3 bits: codepoint, grapheme, bg_color, etc.

    // Content payload
    content: packed union {
        codepoint: u21,           // Unicode codepoint
        color_palette: u8,        // Palette index for bg_color
        color_rgb: RGB,           // 24-bit RGB for bg_color
    },

    // Style reference (0 = default style)
    style_id: StyleId,            // Index into page's style_set

    // Width handling
    wide: Wide,                   // narrow, wide, spacer_head, spacer_tail

    // Flags
    protected: bool,              // DECSCA protection
    hyperlink: bool,              // Has OSC8 link
    semantic_content: enum,       // output, input, prompt
    _padding: u16,
};
```

**Design Rationale:**
- Packed into exactly 64 bits for atomic operations
- Style deduplication via reference counting
- Supports grapheme clusters (emoji, combining marks) via separate storage
- Wide character tracking for CJK and emoji

### 3.2 Page Memory Architecture

**File:** `src/terminal/page.zig`

```zig
pub const Page = struct {
    // Single contiguous, page-aligned memory block
    memory: []align(page_size) u8,

    // Row/cell storage
    rows: []*Row,
    cols: size.CellCountInt,

    // Shared data (deduplication)
    style_set: StyleSet,           // Style → StyleId mapping
    grapheme_map: GraphemeMap,     // Extended grapheme storage
    hyperlink_set: HyperlinkSet,   // Hyperlink deduplication
    hyperlink_map: HyperlinkMap,   // Cell → Hyperlink mapping

    // Images (Kitty graphics protocol)
    kitty_images: KittyImageStorage,
};
```

### 3.3 PageList (Scrollback Buffer)

**File:** `src/terminal/PageList.zig`

```
┌─────────────────────────────────────────────────────────────────┐
│                         PageList                                 │
│                                                                  │
│  ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐         │
│  │ Page 1  │◀─▶│ Page 2  │◀─▶│ Page 3  │◀─▶│ Page 4  │         │
│  │(history)│   │(history)│   │(active) │   │(active) │         │
│  └─────────┘   └─────────┘   └─────────┘   └─────────┘         │
│       ▲                           ▲                              │
│       │                           │                              │
│  viewport_pin              active_area                           │
│  (scroll position)         (current screen)                      │
│                                                                  │
│  Memory Pool: Pre-allocated pages for reuse                      │
│  Tracked Pins: References that auto-update on mutations          │
└─────────────────────────────────────────────────────────────────┘
```

### 3.4 Style System

**File:** `src/terminal/style.zig`

```zig
pub const Style = struct {
    fg_color: Color,           // Foreground (none, palette, or RGB)
    bg_color: Color,           // Background
    underline_color: Color,    // Underline color (independent)

    flags: packed struct {
        bold: bool,
        italic: bool,
        faint: bool,
        blink: bool,
        inverse: bool,
        invisible: bool,
        strikethrough: bool,
        overline: bool,
        underline: Underline,  // none, single, double, curly, dotted, dashed
    },
};
```

### 3.5 Screen and Terminal State

**File:** `src/terminal/Screen.zig`, `src/terminal/Terminal.zig`

```zig
// Screen: Active terminal surface
pub const Screen = struct {
    pages: PageList,           // All pages (scrollback + active)
    cursor: Cursor,            // Position, style, hyperlink state
    selection: ?Selection,     // Text selection
    charset: CharsetState,     // G0/G1/GL/GR charset switching
    dirty: Dirty,              // Render dirty flags
};

// Terminal: Complete terminal state
pub const Terminal = struct {
    screens: [2]Screen,        // Primary + Alternate screen
    active_screen: u1,         // Which screen is active
    tabstops: Tabstops,        // Horizontal tab positions
    scrolling_region: Region,  // DECSTBM scroll region
    colors: Colors,            // Dynamic palette
    modes: ModeState,          // DEC/ANSI mode flags
};
```

---

## Part 4: Visual Rendering Pipeline

### 4.1 Renderer Thread Loop

**File:** `src/renderer/Thread.zig`

```
┌─────────────────────────────────────────────────────────────────┐
│                    Renderer Thread Loop                          │
│                                                                  │
│    ┌──────────────┐                                             │
│    │ Render Timer │ (8ms interval = 120 FPS)                    │
│    │              │                                             │
│    └──────┬───────┘                                             │
│           │                                                      │
│           ▼                                                      │
│    ┌──────────────┐     ┌──────────────┐     ┌──────────────┐  │
│    │ updateFrame()│────▶│ rebuildCells │────▶│  drawFrame() │  │
│    │              │     │              │     │              │  │
│    │ Lock mutex   │     │ Convert term │     │ GPU upload   │  │
│    │ Read terminal│     │ state to GPU │     │ Render passes│  │
│    │ Update state │     │ vertex data  │     │ Present      │  │
│    └──────────────┘     └──────────────┘     └──────────────┘  │
│                                                                  │
│    ┌──────────────┐     ┌──────────────┐                        │
│    │ I/O Wakeup   │     │ Cursor Blink │                        │
│    │ (from termio)│     │ Timer (600ms)│                        │
│    └──────────────┘     └──────────────┘                        │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 Cell to GPU Vertex Conversion

**File:** `src/renderer/cell.zig`

```
Terminal Cell                    GPU Vertex Data
┌────────────────────┐          ┌────────────────────────────────┐
│ codepoint: 'A'     │          │ CellText:                      │
│ style_id: 3        │   ───▶   │   glyph_id: 42                 │
│ wide: narrow       │          │   grid_pos: (5, 10)            │
│ hyperlink: true    │          │   bearing: (2, -3)             │
└────────────────────┘          │   size: (8, 16)                │
                                │   color: (255, 255, 255, 255)  │
Style (id=3)                    │   atlas: grayscale             │
┌────────────────────┐          └────────────────────────────────┘
│ fg: RGB(255,0,0)   │
│ bg: palette(16)    │   ───▶   ┌────────────────────────────────┐
│ bold: true         │          │ CellBg:                        │
│ underline: single  │          │   color: (30, 30, 30, 255)     │
└────────────────────┘          │   position: (5, 10)            │
                                └────────────────────────────────┘
```

### 4.3 GPU Render Passes

**File:** `src/renderer/generic.zig`

```
Frame Rendering Pipeline:
─────────────────────────

Pass 0: Background
├── Clear framebuffer
└── Draw background image (if configured)

Pass 1: Cell Backgrounds
└── Draw all CellBg vertices (background colors)

Pass 2: Text & Decorations
├── Draw all CellText vertices (glyphs)
├── Draw underlines
├── Draw strikethroughs
└── Draw overlines

Pass 3: Overlays
├── Draw cursor
├── Draw selection highlights
├── Draw search highlights
└── Draw hyperlink highlights

Pass 4: Images
├── Draw Kitty images (below text)
└── Draw Kitty images (above text)

Pass 5+: Custom Shaders
└── User-provided ShaderToy-style effects
```

### 4.4 Graphics Backend Abstraction

**File:** `src/renderer/backend.zig`

```zig
// Compile-time backend selection
pub const Backend = switch (build_config.renderer) {
    .metal => Metal,    // macOS
    .opengl => OpenGL,  // Linux/Windows
    .webgl => WebGL,    // Browser
};

// Unified interface
pub const GraphicsAPI = struct {
    Target,      // Render target (texture or backbuffer)
    Frame,       // Frame context
    RenderPass,  // Group of draw operations
    Pipeline,    // Shader program
    Buffer,      // Vertex/uniform buffer
    Texture,     // GPU texture
    Sampler,     // Texture sampling config
};
```

### 4.5 Font Atlas System

**File:** `src/font/Atlas.zig`

```
┌───────────────────────────────────────────────────────────────┐
│                      Font Atlas Texture                        │
│                                                                │
│  ┌──────┬──────┬──────┬──────┬──────┬──────┬──────┬─────────┐│
│  │  A   │  B   │  C   │  D   │  E   │  F   │  G   │ (free)  ││
│  ├──────┼──────┼──────┼──────┼──────┼──────┼──────┼─────────┤│
│  │  H   │  I   │  J   │  K   │  L   │  M   │  N   │ (free)  ││
│  ├──────┼──────┼──────┼──────┼──────┼──────┼──────┼─────────┤│
│  │  O   │  P   │  Q   │  R   │  S   │  T   │  U   │ (free)  ││
│  └──────┴──────┴──────┴──────┴──────┴──────┴──────┴─────────┘│
│                                                                │
│  Bin-packing algorithm (Jukka Jylänki)                        │
│  Automatic resize when full                                    │
│  Separate atlases for grayscale vs. color glyphs              │
└───────────────────────────────────────────────────────────────┘
```

---

## Part 5: Existing Extension Points

### 5.1 Highlight System

**File:** `src/terminal/highlight.zig`

The highlight system provides a foundation for marking text regions:

```zig
pub const Highlight = struct {
    start: Pin,    // Start position (tracks through mutations)
    end: Pin,      // End position
    layer: Layer,  // Rendering layer (below/above text)
};

// Already used for:
// - Text selection
// - Search result highlighting
// - Hyperlink hover highlighting
```

### 5.2 Link Detection System

**File:** `src/renderer/link.zig`, `src/input/Link.zig`

```zig
pub const Link = struct {
    regex: Regex,              // Oniguruma regex pattern
    action: Action,            // open, _open_osc8
    highlight: Highlight,      // always, hover, always_mods, hover_mods
};

// Configurable via:
// link = regex:pattern action:open highlight:hover
```

### 5.3 Overlay System (GTK)

**File:** `src/apprt/gtk/class/`

Existing overlay implementations:
- `ResizeOverlay` - Window resize dimensions
- `SearchOverlay` - Search UI with results
- `KeyStateOverlay` - Active key sequence display
- `InspectorWidget` - Dear ImGui debug overlay

### 5.4 Custom Shader System

**File:** `src/renderer/shadertoy.zig`

Supports user-provided GLSL/MSL shaders with uniforms:
```glsl
uniform float iTime;           // Animation time
uniform vec3 iResolution;      // Viewport size
uniform sampler2D iChannel0;   // Terminal content texture
```

### 5.5 Kitty Graphics Protocol

**File:** `src/terminal/kitty/graphics.zig`

Full implementation of inline image display:
- PNG, JPEG, GIF, WebP support
- Three z-layers (below bg, below text, above text)
- Virtual placements
- Unicode placeholders

---

## Part 6: Feature Feasibility Analysis

### 6.1 Web iFrame Overlays

**Complexity: Very High**

**Current Architecture Constraints:**
1. Ghostty uses GPU-accelerated rendering (Metal/OpenGL), not a web view
2. GTK/macOS app layers are separate from the renderer
3. No browser engine is embedded

**Implementation Approaches:**

**Approach A: Native Compositor Overlay**
```
┌─────────────────────────────────────────┐
│            OS Window                     │
│  ┌───────────────────────────────────┐  │
│  │     Ghostty Terminal (OpenGL)     │  │
│  │                                   │  │
│  │   ┌───────────────────────────┐   │  │
│  │   │ WebView (native platform) │   │  │
│  │   │ positioned absolutely     │   │  │
│  │   └───────────────────────────┘   │  │
│  │                                   │  │
│  └───────────────────────────────────┘  │
└─────────────────────────────────────────┘
```

**Required Changes:**
1. Add platform-specific WebView integration:
   - macOS: `WKWebView` in Swift layer
   - GTK: `WebKitWebView` widget
2. Create overlay positioning system that maps terminal coordinates to screen coordinates
3. Handle z-ordering and focus management
4. Implement resize/scroll tracking to reposition overlays

**Estimated Effort:** 3-6 months, requires platform-specific code for each OS

**Approach B: Electron-style Architecture (Major Rewrite)**
- Would require replacing the renderer with a web-based approach
- Not recommended for Ghostty's performance-focused design

**Recommendation:** Approach A is feasible but requires significant platform-specific work. Consider starting with macOS only.

---

### 6.1.1 Detailed macOS WebView Implementation Specification

This section provides a comprehensive technical specification for implementing web overlays on macOS.

#### Current macOS View Hierarchy

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  SwiftUI SurfaceWrapper                                                     │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │  ZStack (Overlay Composition)                                         │  │
│  │                                                                       │  │
│  │  Layer 0:  GeometryReader + SurfaceRepresentable                      │  │
│  │            └── SurfaceScrollView (NSView)                             │  │
│  │                └── NSScrollView                                       │  │
│  │                    └── SurfaceView (NSView with Metal Layer)          │  │
│  │                                                                       │  │
│  │  Layer 1:  SurfaceResizeOverlay (size indicator)                      │  │
│  │  Layer 2:  SurfaceProgressBar (top progress bar)                      │  │
│  │  Layer 3:  ReadonlyBadge (top-right corner)                           │  │
│  │  Layer 4:  KeyStateIndicator (draggable key state pill)               │  │
│  │  Layer 5:  URL Tooltip (bottom-right hover URL)                       │  │
│  │  Layer 6:  SecureInputOverlay (lock indicator)                        │  │
│  │  Layer 7:  SurfaceSearchOverlay (draggable search bar)                │  │
│  │  Layer 8:  BellBorderOverlay (animated border)                        │  │
│  │  Layer 9:  HighlightOverlay (pulsing glow)                            │  │
│  │  Layer 10: Error/Unhealthy state overlay                              │  │
│  │  Layer 11: Unfocused split dimming                                    │  │
│  │  Layer 12: SurfaceGrabHandle (top drag area)                          │  │
│  │                                                                       │  │
│  │  ═══════════════════════════════════════════════════════════════════  │  │
│  │  NEW: Layer 13: WebOverlayContainer (proposed)                        │  │
│  │                                                                       │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Key Files:**
- `macos/Sources/Ghostty/Surface View/SurfaceView.swift` (overlay ZStack, ~1263 lines)
- `macos/Sources/Ghostty/Surface View/SurfaceView_AppKit.swift` (NSView, ~2291 lines)
- `macos/Sources/Ghostty/Surface View/SurfaceScrollView.swift` (scroll management, ~396 lines)

#### Coordinate System Conversion

The coordinate conversion pipeline is critical for positioning overlays correctly:

```
Terminal Coordinates          View Coordinates           Screen Coordinates
(row, col) grid units    →    (x, y) pixels         →   (x, y) screen pixels
Origin: top-left             Origin: bottom-left        Origin: varies
+Y: down                     +Y: up (AppKit)            +Y: up

┌─────────────────┐          ┌─────────────────┐        ┌─────────────────┐
│ (0,0)───────→   │          │        ↑ +Y     │        │ Screen origin   │
│   │             │          │        │        │        │ (may be non-zero)│
│   ↓ +Y          │    ──►   │ (0,0)──┼───→ +X │   ──►  │                 │
│                 │          │        │        │        │                 │
└─────────────────┘          └─────────────────┘        └─────────────────┘
```

**Conversion Code Path (from `SurfaceView_AppKit.swift:1851-1906`):**

```swift
// Step 1: Get pixel coordinates from libghostty (terminal coords → pixels)
var x: Double = 0    // Pixel X from terminal
var y: Double = 0    // Pixel Y (top-left origin, +Y down)
ghostty_surface_ime_point(surface, &x, &y, &width, &height)

// Step 2: Convert to AppKit view coordinates (flip Y axis)
let viewRect = NSMakeRect(
    x,                              // X unchanged
    frame.size.height - y,          // Y flipped: view_y = height - terminal_y
    width,
    max(height, cellSize.height)
)

// Step 3: Convert to window coordinates
let winRect = self.convert(viewRect, to: nil)

// Step 4: Convert to screen coordinates
guard let window = self.window else { return winRect }
return window.convertToScreen(winRect)
```

#### Proposed Implementation: WebOverlayManager

**New Files to Create:**

```
macos/Sources/Ghostty/Overlays/
├── WebOverlay.swift           // WKWebView wrapper
├── WebOverlayManager.swift    // Manages multiple overlays
├── WebOverlayConfig.swift     // Configuration/styling
└── WebOverlayPosition.swift   // Terminal → screen positioning
```

**Core Data Structures:**

```swift
// WebOverlay.swift
import WebKit
import SwiftUI

/// Configuration for a web overlay
struct WebOverlayConfig: Identifiable {
    let id: UUID = UUID()

    /// Anchor point in terminal coordinates
    var anchorRow: Int
    var anchorCol: Int

    /// Size in terminal cells (or fixed pixels)
    enum Size {
        case cells(rows: Int, cols: Int)
        case pixels(width: CGFloat, height: CGFloat)
        case auto  // Size to content
    }
    var size: Size

    /// Position relative to anchor
    enum Anchor {
        case topLeft, topRight, bottomLeft, bottomRight
        case above, below  // Tooltip-style
    }
    var anchor: Anchor = .below

    /// Content source
    enum Content {
        case html(String)
        case url(URL)
        case data(Data, mimeType: String)
    }
    var content: Content

    /// Behavior options
    var dismissOnClickOutside: Bool = true
    var dismissOnEscape: Bool = true
    var dismissOnScroll: Bool = false
    var capturesMouse: Bool = true
    var capturesKeyboard: Bool = false  // Usually keep false for tooltips

    /// Appearance
    var backgroundColor: NSColor = .clear
    var cornerRadius: CGFloat = 8
    var shadow: Bool = true
}

/// The actual WebView overlay
class WebOverlayView: NSView {
    let webView: WKWebView
    let config: WebOverlayConfig
    weak var surfaceView: SurfaceView?

    private var observation: NSKeyValueObservation?

    init(config: WebOverlayConfig, surfaceView: SurfaceView) {
        self.config = config
        self.surfaceView = surfaceView

        // Configure WKWebView
        let webConfig = WKWebViewConfiguration()
        webConfig.preferences.javaScriptEnabled = true
        // Disable scrolling for tooltip-style overlays
        webConfig.preferences.setValue(true, forKey: "allowFileAccessFromFileURLs")

        self.webView = WKWebView(frame: .zero, configuration: webConfig)
        self.webView.isOpaque = false
        self.webView.setValue(false, forKey: "drawsBackground")

        super.init(frame: .zero)

        setupView()
        loadContent()
        positionOverlay()
    }

    private func setupView() {
        wantsLayer = true
        layer?.cornerRadius = config.cornerRadius
        layer?.masksToBounds = true

        if config.shadow {
            layer?.shadowColor = NSColor.black.cgColor
            layer?.shadowOpacity = 0.3
            layer?.shadowOffset = CGSize(width: 0, height: -2)
            layer?.shadowRadius = 8
        }

        addSubview(webView)
        webView.translatesAutoresizingMaskIntoConstraints = false
        NSLayoutConstraint.activate([
            webView.topAnchor.constraint(equalTo: topAnchor),
            webView.bottomAnchor.constraint(equalTo: bottomAnchor),
            webView.leadingAnchor.constraint(equalTo: leadingAnchor),
            webView.trailingAnchor.constraint(equalTo: trailingAnchor),
        ])
    }

    private func loadContent() {
        switch config.content {
        case .html(let html):
            webView.loadHTMLString(html, baseURL: nil)
        case .url(let url):
            webView.load(URLRequest(url: url))
        case .data(let data, let mimeType):
            webView.load(data, mimeType: mimeType,
                        characterEncodingName: "utf-8", baseURL: nil)
        }
    }

    /// Position the overlay relative to terminal coordinates
    func positionOverlay() {
        guard let surfaceView = surfaceView else { return }

        // Calculate pixel position from terminal coordinates
        let cellWidth = surfaceView.cellSize.width
        let cellHeight = surfaceView.cellSize.height

        // Terminal pixel position (top-left origin)
        let terminalX = CGFloat(config.anchorCol) * cellWidth
        let terminalY = CGFloat(config.anchorRow) * cellHeight

        // Convert to view coordinates (flip Y)
        let viewX = terminalX
        let viewY = surfaceView.frame.height - terminalY

        // Calculate overlay size
        let overlaySize: CGSize
        switch config.size {
        case .cells(let rows, let cols):
            overlaySize = CGSize(
                width: CGFloat(cols) * cellWidth,
                height: CGFloat(rows) * cellHeight
            )
        case .pixels(let width, let height):
            overlaySize = CGSize(width: width, height: height)
        case .auto:
            overlaySize = CGSize(width: 300, height: 200) // Default
        }

        // Apply anchor offset
        var origin = CGPoint(x: viewX, y: viewY)
        switch config.anchor {
        case .topLeft:
            break // No adjustment
        case .topRight:
            origin.x -= overlaySize.width
        case .bottomLeft:
            origin.y -= overlaySize.height
        case .bottomRight:
            origin.x -= overlaySize.width
            origin.y -= overlaySize.height
        case .above:
            origin.y += cellHeight  // Move up (in flipped coords)
        case .below:
            origin.y -= overlaySize.height + cellHeight
        }

        // Set frame
        self.frame = NSRect(origin: origin, size: overlaySize)
    }

    required init?(coder: NSCoder) {
        fatalError("init(coder:) not implemented")
    }
}
```

**Manager Class:**

```swift
// WebOverlayManager.swift

/// Manages all web overlays for a terminal surface
class WebOverlayManager: ObservableObject {
    @Published private(set) var overlays: [UUID: WebOverlayView] = [:]

    weak var surfaceView: SurfaceView?

    init(surfaceView: SurfaceView) {
        self.surfaceView = surfaceView

        // Observe scroll position changes to reposition overlays
        // (or dismiss if configured)
        setupScrollObservation()
    }

    /// Show a new overlay
    func show(_ config: WebOverlayConfig) -> UUID {
        guard let surfaceView = surfaceView else { return config.id }

        let overlay = WebOverlayView(config: config, surfaceView: surfaceView)
        overlays[config.id] = overlay

        // Add to surface view's superview (so it floats above terminal)
        surfaceView.superview?.addSubview(overlay)

        // Animate in
        overlay.alphaValue = 0
        NSAnimationContext.runAnimationGroup { context in
            context.duration = 0.15
            overlay.animator().alphaValue = 1
        }

        return config.id
    }

    /// Dismiss an overlay
    func dismiss(_ id: UUID, animated: Bool = true) {
        guard let overlay = overlays[id] else { return }

        if animated {
            NSAnimationContext.runAnimationGroup { context in
                context.duration = 0.15
                overlay.animator().alphaValue = 0
            } completionHandler: {
                overlay.removeFromSuperview()
                self.overlays.removeValue(forKey: id)
            }
        } else {
            overlay.removeFromSuperview()
            overlays.removeValue(forKey: id)
        }
    }

    /// Dismiss all overlays
    func dismissAll() {
        for id in overlays.keys {
            dismiss(id)
        }
    }

    /// Update overlay positions (call on terminal resize/scroll)
    func repositionAll() {
        for overlay in overlays.values {
            overlay.positionOverlay()
        }
    }

    private func setupScrollObservation() {
        // Would observe scrollbar changes from surfaceView
        // and call repositionAll() or dismissAll() based on config
    }
}
```

**SwiftUI Integration:**

```swift
// In SurfaceView.swift, add to the ZStack after existing overlays:

struct SurfaceWrapper: View {
    @ObservedObject var surfaceView: Ghostty.SurfaceView
    @StateObject private var webOverlayManager: WebOverlayManager

    init(surfaceView: Ghostty.SurfaceView) {
        self.surfaceView = surfaceView
        _webOverlayManager = StateObject(
            wrappedValue: WebOverlayManager(surfaceView: surfaceView)
        )
    }

    var body: some View {
        ZStack {
            // ... existing layers 0-12 ...

            // Layer 13: Web Overlays (rendered via NSViewRepresentable)
            WebOverlayContainerView(manager: webOverlayManager)
                .allowsHitTesting(true)  // Overlays capture input
        }
        .environmentObject(webOverlayManager)
    }
}
```

#### API for Triggering Overlays

**Option A: Escape Sequence Protocol (Shell Integration)**

Define a new OSC sequence for web overlays:

```
OSC 1337 ; WebOverlay ; action=show ; row=R ; col=C ; html=<base64> ST
OSC 1337 ; WebOverlay ; action=show ; row=R ; col=C ; url=<url> ST
OSC 1337 ; WebOverlay ; action=dismiss ; id=<uuid> ST
OSC 1337 ; WebOverlay ; action=dismissAll ST
```

**Changes Required:**
1. Add parser handler in `src/terminal/Parser.zig`
2. Add OSC handler in `src/termio/stream_handler.zig`
3. Forward to apprt via message system
4. Handle in Swift layer to create overlay

**Option B: libghostty C API Extension**

```c
// In include/ghostty.h

typedef struct {
    int32_t row;
    int32_t col;
    const char* html;       // NULL if using url
    const char* url;        // NULL if using html
    int32_t width_cells;    // 0 for auto
    int32_t height_cells;   // 0 for auto
    bool dismiss_on_outside_click;
    bool dismiss_on_scroll;
} ghostty_web_overlay_config_s;

typedef void* ghostty_web_overlay_t;

ghostty_web_overlay_t ghostty_surface_show_web_overlay(
    ghostty_surface_t surface,
    const ghostty_web_overlay_config_s* config
);

void ghostty_surface_dismiss_web_overlay(
    ghostty_surface_t surface,
    ghostty_web_overlay_t overlay
);

void ghostty_surface_dismiss_all_web_overlays(ghostty_surface_t surface);
```

**Option C: Configuration-Driven (Regex Triggers)**

Extend the existing link system:

```
# In ghostty config
link = regex:https?://github\.com/[^/]+/[^/]+/pull/\d+ \
       action:none \
       hover:web-preview \
       hover-url:https://ghpreview.example.com/?url=$0
```

This would automatically show a web preview when hovering over GitHub PR URLs.

#### Focus and Event Handling

**Challenge:** When a WebView overlay is visible, keyboard/mouse events need careful routing.

```
Event Flow with Overlays:

Mouse Click
    │
    ▼
┌─────────────────┐
│ Hit Test:       │
│ Is click in     │──── Yes ───▶ WebView handles event
│ WebOverlay?     │             (links, scroll, etc.)
└────────┬────────┘
         │ No
         ▼
┌─────────────────┐
│ Click outside   │──── dismissOnClickOutside? ───▶ Dismiss overlay
│ overlay         │
└────────┬────────┘
         │
         ▼
   Terminal handles event
   (selection, links, etc.)


Keyboard Event
    │
    ▼
┌─────────────────┐
│ ESC key?        │──── dismissOnEscape? ───▶ Dismiss overlay
└────────┬────────┘
         │ No
         ▼
┌─────────────────┐
│ capturesKeyboard│──── Yes ───▶ WebView handles event
│ = true?         │
└────────┬────────┘
         │ No
         ▼
   Terminal handles event
```

**Implementation:**

```swift
// In WebOverlayView
override func hitTest(_ point: NSPoint) -> NSView? {
    // Only capture clicks if configured to do so
    if config.capturesMouse {
        let localPoint = convert(point, from: superview)
        if bounds.contains(localPoint) {
            return super.hitTest(point)
        }
    }
    return nil  // Pass through to terminal
}

// Handle escape key to dismiss
override func keyDown(with event: NSEvent) {
    if event.keyCode == 53 && config.dismissOnEscape {  // ESC key
        NotificationCenter.default.post(
            name: .dismissWebOverlay,
            object: config.id
        )
        return
    }

    if config.capturesKeyboard {
        super.keyDown(with: event)
    } else {
        nextResponder?.keyDown(with: event)
    }
}
```

#### Scroll Synchronization

When the terminal scrolls, overlays need to either:
1. Move with the content (if attached to a specific row)
2. Stay fixed (if attached to viewport)
3. Dismiss (for tooltip-style overlays)

```swift
// In WebOverlayManager
func handleScroll(newOffset: Int, oldOffset: Int) {
    for (id, overlay) in overlays {
        if overlay.config.dismissOnScroll {
            dismiss(id)
        } else {
            // Reposition based on new scroll offset
            overlay.positionOverlay()
        }
    }
}
```

#### Complete Implementation Checklist

| Component | Files | Effort |
|-----------|-------|--------|
| **WebOverlayView** | New Swift file | 2-3 days |
| **WebOverlayManager** | New Swift file | 2-3 days |
| **Position calculation** | Extend SurfaceView | 1-2 days |
| **SwiftUI integration** | Modify SurfaceView.swift | 1 day |
| **Event handling** | WebOverlayView | 2-3 days |
| **Scroll sync** | SurfaceScrollView + Manager | 2-3 days |
| **OSC protocol** | Parser + stream_handler | 3-5 days |
| **C API bridge** | embedded.zig + ghostty.h | 2-3 days |
| **Testing/polish** | Various | 3-5 days |
| **Documentation** | Config docs, man pages | 1-2 days |

**Total Estimated Effort:** 3-5 weeks for macOS implementation

#### Security Considerations

1. **Content Security Policy:** WebViews loading arbitrary HTML could execute malicious scripts
   - Solution: Sandboxed WKWebView configuration
   - Disable JavaScript for untrusted content
   - Content-Security-Policy headers for loaded URLs

2. **URL Validation:** Only allow specific URL patterns
   - Whitelist trusted domains
   - Block file:// URLs unless explicitly allowed

3. **Resource Limits:** Prevent overlays from consuming excessive memory
   - Limit number of concurrent overlays
   - Auto-dismiss after timeout
   - Monitor WebView memory usage

---

### 6.2 Text Hovers/Highlights on Regex Patterns

**Complexity: Low-Medium**

**Current Support:** Already implemented!

**File:** `src/input/Link.zig`, `src/renderer/link.zig`

**Existing Capabilities:**
```zig
// Configuration option
link = regex:pattern action:open highlight:hover
```

**What Works Today:**
- Regex-based pattern matching
- Hover-triggered highlighting
- Modifier-based highlighting (e.g., Cmd+hover)
- Customizable actions on click

**Enhancement Opportunities:**

1. **Custom Highlight Styles** (Medium effort):
   - Currently highlights use a fixed style
   - Could add per-link highlight colors

   ```zig
   // Proposed config extension
   link = regex:TODO action:none highlight:always style:bg=#ffff00
   ```

   **Changes Required:**
   - Extend `Link` struct with optional style
   - Modify `link.zig` to apply custom styles to highlighted cells
   - Add config parsing for style parameter

2. **Tooltip Content** (Medium effort):
   - Show custom tooltip text on hover
   - Already have `mouse_hover_url` infrastructure

   **Changes Required:**
   - Extend Link struct with `tooltip: ?[]const u8`
   - Pass tooltip through apprt message system
   - Render tooltip in platform UI layer

3. **Multiple Highlight Layers** (Low effort):
   - Already supported via highlight system
   - Just needs config exposure

**Estimated Effort:** 1-4 weeks depending on feature scope

---

### 6.3 Collapsible/Toggleable Text Blocks

**Complexity: Very High**

**Fundamental Challenge:** Terminal emulation is based on a fixed character grid. Collapsing text would require:

1. **Variable Row Heights** - Currently impossible:
   ```
   Current Model:          Proposed Model:
   ┌─────────────────┐    ┌─────────────────┐
   │ Row 1 (16px)    │    │ Row 1 (16px)    │
   │ Row 2 (16px)    │    │ ▼ Collapsed (8px)│
   │ Row 3 (16px)    │    │ Row 3 (16px)    │
   │ Row 4 (16px)    │    │ Row 4 (16px)    │
   └─────────────────┘    └─────────────────┘
   ```

2. **Content Reflow** - What happens to hidden content?
3. **PTY Synchronization** - Shell doesn't know about collapse state

**Possible Implementations:**

**Approach A: Display-Only Collapsing (Viewport Filter)**
```
Terminal State (unchanged):     Viewport (filtered):
┌─────────────────────────┐    ┌─────────────────────────┐
│ Line 1                  │    │ Line 1                  │
│ ── BEGIN FOLD ──        │    │ ▶ [3 lines hidden]      │
│ Line 2                  │    │ Line 5                  │
│ Line 3                  │    │ Line 6                  │
│ Line 4                  │    └─────────────────────────┘
│ ── END FOLD ──          │
│ Line 5                  │
│ Line 6                  │
└─────────────────────────┘
```

**Required Changes:**
1. Add fold marker detection (via escape sequence or config pattern)
2. Create fold state tracking in Screen
3. Modify `RenderState` to filter/collapse rows
4. Handle selection across fold boundaries
5. Implement fold toggle UI (click handler)

**Approach B: Shell Integration Protocol**
- Define new OSC sequences for fold regions
- Shell/application marks foldable regions explicitly
- Similar to semantic prompts (OSC 133)

```
OSC 1337 ; FoldStart ; id=1 ST
... foldable content ...
OSC 1337 ; FoldEnd ; id=1 ST
```

**Challenges:**
- Selection behavior across folds
- Search within folded content
- Scrollback handling (fold state persistence)
- Copy/paste of folded regions

**Estimated Effort:** 2-4 months for basic implementation

---

### 6.4 Animation Support

**Complexity: Medium**

**Current Animation Capabilities:**

1. **Cursor Blink** - Already implemented (600ms timer)
2. **Custom Shaders** - Support time-based animations via `iTime` uniform
3. **Render Timer** - 120 FPS capable

**What's Missing for Smooth Animations:**

1. **Scroll Animation**
   - Currently: Instant viewport jump
   - Desired: Smooth scroll interpolation

   **Implementation:**
   ```zig
   // In renderer state
   scroll_animation: struct {
       start_offset: f32,
       target_offset: f32,
       start_time: u64,
       duration_ms: u32,
   },
   ```

   **Changes Required:**
   - Add animation state to renderer
   - Interpolate viewport position in `updateFrame()`
   - Trigger animation on scroll events
   - Use easing functions (ease-out, etc.)

2. **Fold/Expand Animation**
   - Requires variable row heights (see 6.3)
   - Could animate row height from 0 to full

3. **Highlight Fade Animation**
   - Animate alpha of highlight overlays
   - Already have per-frame updates

**Estimated Effort:**
- Scroll animation: 1-2 weeks
- Other animations: Depends on underlying feature

---

### 6.5 Font Size and Style Variations

**Complexity: Very High (for mixed sizes)**

**Current Font System:**

```zig
// Font metrics (src/font/Metrics.zig)
pub const Metrics = struct {
    cell_width: u32,      // All cells same width
    cell_height: u32,     // All cells same height
    // ...
};
```

**Constraint:** The entire terminal uses a single cell size. This is fundamental to:
- Cursor positioning
- Selection rectangles
- Text alignment
- GPU vertex generation

**What's Currently Supported:**

1. **Bold/Italic** - Different font faces, same metrics
2. **Font Features** - OpenType features (ligatures, etc.)
3. **Global Font Size** - Configurable, uniform across terminal

**Double-Width/Double-Height (DECDWL/DECDHL):**

VT terminals supported these via escape sequences:
```
ESC # 6  - Double-width single-height (DECDWL)
ESC # 3  - Double-height top half (DECDHL)
ESC # 4  - Double-height bottom half (DECDHL)
```

**Ghostty Status:** NOT IMPLEMENTED

**Implementation Approach:**

```
Normal Line:        DECDWL Line:        DECDHL Lines:
┌───┬───┬───┬───┐  ┌───────┬───────┐   ┌───────┬───────┐
│ A │ B │ C │ D │  │   A   │   B   │   │   A   │   B   │ ← Top half
└───┴───┴───┴───┘  └───────┴───────┘   ├───────┼───────┤
                                        │   A   │   B   │ ← Bottom half
                                        └───────┴───────┘
```

**Required Changes:**

1. **Row Metadata Extension:**
   ```zig
   pub const Row = struct {
       cells: []Cell,
       line_attribute: enum {
           normal,
           double_width,
           double_height_top,
           double_height_bottom,
       },
   };
   ```

2. **Parser Support:**
   - Add ESC # sequence handling
   - Track line attributes per row

3. **Renderer Changes:**
   - Scale glyphs 2x when rendering DECDWL/DECDHL rows
   - Adjust cell width calculations for cursor
   - Handle selection across mixed-width lines

4. **Font Atlas:**
   - Generate 2x scaled glyphs (or scale at render time)

**Mixed Font Sizes (e.g., per-cell size):**

This would require fundamental changes:
- Variable-width cell grid
- Complex text layout engine
- Reflow on resize
- Essentially building a rich text editor

**Not recommended** for terminal emulation.

**Estimated Effort:**
- DECDWL/DECDHL: 2-4 weeks
- Per-cell font sizes: Not feasible within terminal model

---

## Part 7: Architecture Recommendations

### 7.1 Extension Architecture

For adding rich features without breaking terminal semantics:

```
┌─────────────────────────────────────────────────────────────────┐
│                    Proposed Extension Layer                      │
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │   Overlay    │  │   Tooltip    │  │    Fold/Collapse     │  │
│  │   Manager    │  │   System     │  │      Controller      │  │
│  └──────────────┘  └──────────────┘  └──────────────────────┘  │
│         │                 │                    │                │
│         └─────────────────┼────────────────────┘                │
│                           │                                      │
│                           ▼                                      │
│                  ┌──────────────────┐                           │
│                  │  Region Tracker  │                           │
│                  │  (Pin-based)     │                           │
│                  └──────────────────┘                           │
│                           │                                      │
└───────────────────────────┼─────────────────────────────────────┘
                            │
                            ▼
              ┌─────────────────────────┐
              │    Terminal Core        │
              │    (minimal changes)    │
              └─────────────────────────┘
```

### 7.2 Recommended Implementation Order

1. **Phase 1: Enhanced Regex Highlights** (Easiest)
   - Custom highlight styles
   - Tooltip support
   - Already has foundation

2. **Phase 2: DECDWL/DECDHL Support** (Medium)
   - Well-defined VT standard
   - Localized changes to row handling
   - Good test case for variable rendering

3. **Phase 3: Scroll Animation** (Medium)
   - Pure renderer change
   - No terminal state impact

4. **Phase 4: Fold/Collapse** (Hard)
   - Requires careful design
   - Consider shell integration protocol
   - May need viewport filtering layer

5. **Phase 5: Web Overlays** (Hardest)
   - Platform-specific implementations
   - Consider macOS-first approach
   - May want as separate project/plugin

### 7.3 Key Files for Each Feature

| Feature | Primary Files |
|---------|---------------|
| Regex highlights | `src/input/Link.zig`, `src/renderer/link.zig` |
| Tooltips | `src/apprt/gtk/class/surface.zig`, macOS Swift files |
| DECDWL/DECDHL | `src/terminal/page.zig`, `src/renderer/cell.zig` |
| Scroll animation | `src/renderer/Thread.zig`, `src/renderer/generic.zig` |
| Fold/collapse | `src/terminal/Screen.zig`, `src/terminal/render.zig` |
| Web overlays | `src/apprt/gtk/`, `macos/Sources/Ghostty/` |

---

## Part 8: Detailed Code References

### 8.1 Key Entry Points

| Component | File | Line | Description |
|-----------|------|------|-------------|
| PTY read | `src/termio/Termio.zig` | 660 | `processOutput()` entry |
| Parser | `src/terminal/Parser.zig` | 1 | VT state machine |
| Cell definition | `src/terminal/page.zig` | 1962 | `Cell` packed struct |
| Style system | `src/terminal/style.zig` | 20 | `Style` struct |
| Render loop | `src/renderer/Thread.zig` | 198 | `threadMain()` |
| Frame update | `src/renderer/generic.zig` | 1110 | `updateFrame()` |
| Frame draw | `src/renderer/generic.zig` | 1393 | `drawFrame()` |
| Cell building | `src/renderer/cell.zig` | 41 | `Contents` struct |
| Link detection | `src/renderer/link.zig` | 1 | Regex link system |
| Font atlas | `src/font/Atlas.zig` | 1 | Glyph texture packing |
| Mouse handling | `src/Surface.zig` | 4343 | `linkAtPos()` |

### 8.2 Configuration Options

Relevant existing options (`src/config/Config.zig`):

```zig
// Link configuration
@"link-url": bool = true,                    // Enable URL detection
@"link-previews": LinkPreviews = .true,      // Show URL preview on hover
links: std.ArrayListUnmanaged(Link) = .{},   // Custom link patterns

// Font configuration
@"font-family": ?[:0]const u8 = null,
@"font-size": f32 = 13,
@"font-variation": std.ArrayListUnmanaged(Variation) = .{},

// Visual configuration
@"background-opacity": f32 = 1.0,
@"custom-shader": ?[:0]const u8 = null,
@"custom-shader-animation": Animation = .auto,
```

---

## Conclusion

Ghostty's architecture is well-designed for terminal emulation with excellent performance characteristics. The separation between terminal state, rendering, and platform integration provides clear extension points.

**Most Feasible Enhancements:**
1. Enhanced regex highlighting with custom styles ✓
2. Tooltip system for hovers ✓
3. Scroll animations ✓
4. DECDWL/DECDHL character doubling ✓

**Significant Effort Required:**
5. Collapsible text blocks (display-only approach)
6. Web iframe overlays (platform-specific)

**Not Recommended:**
7. Per-cell font size variations (breaks grid model)

The existing highlight, link, and overlay systems provide a solid foundation for many rich terminal features without requiring fundamental architectural changes.
