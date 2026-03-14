# VSNC (VS Not Code)

## Master Architecture & Engineering Specification

**Version:** 1.0 (Living Document)  
**Core Philosophy:** A completely native, GPU-accelerated, pure-Rust code editor. VSNC delivers the core workflow tools of VS Code (LSP, Git, Terminals, File Management, Fuzzy Finding) operating at single-digit millisecond latencies. It entirely discards the DOM, Node.js, and Electron in favor of a `wgpu`-driven rendering pipeline and multi-threaded Rust architecture.

---

### 1. The Application Heartbeat (The Game Loop)

VSNC operates on a retained-mode data structure but renders using a game-loop paradigm to ensure absolute control over the GPU and zero UI thread blocking.

- **The Main Thread (UI & Render):** Solely responsible for polling OS events (`winit`), walking the UI layout tree, and submitting rendering commands to the GPU (`wgpu`). *Nothing on this thread is allowed to block or perform heavy I/O.*
- **The Async Message Bus:** The nervous system of the editor. Powered by an async runtime (`tokio`) and MPSC channels (`crossbeam-channel`). All heavy computing (I/O, LSP parsing, Git diffs, Fuzzy matching) happens on background thread pools.
- **The Frame Lifecycle:**
  1. **Event Ingestion:** Consume `winit` inputs (mouse/keyboard/resize) and async inbox messages (e.g., `LspResponse`, `FileChanged`, `PtyBytes`).
  2. **State Mutation:** Update the internal Data Models (Ropes, Trees, Terminal Grids).
  3. **Layout Pass:** Push the updated UI tree to the layout engine to calculate exact X/Y/W/H bounds.
  4. **Render Pass:** Generate quads (vertices/indices) for shapes and glyphs.
  5. **Submission:** Flush the command buffer to `wgpu` and present to the screen.

---

### 2. The Graphics & UI Engine

Since VSNC lacks a DOM, it implements its own highly optimized, text-editor-specific UI framework.

- **Windowing & Context:** `winit` manages cross-platform window creation, raw input events, and DPI scaling factors.
- **Graphics API:** `wgpu` interfaces with Vulkan, Metal, or DX12. The entire editor is drawn using:
  - Colored Quads (Backgrounds, borders, selections, cursors).
  - Textured Quads (Icons, images, terminal grid backgrounds).
  - Glyph Quads (Text).
- **Layout Mathematics:** `taffy` calculates the flexbox/CSS-grid-style bounding boxes of every UI element. Elements are defined in a scene graph consisting of Nodes (SplitPanes, Tabs, Buttons, ScrollViews).
- **Text Rasterization & Shaping:** Text is the most expensive thing to draw. VSNC uses `cosmic-text` for text shaping (ligatures, bidirectional text, font fallback) and `glyphon` to rasterize glyphs into a dynamic GPU Texture Atlas. This allows instantaneous rendering of thousands of lines of code.
- **Paneling Logic & Window Management:** The main workspace is a binary tree of Split Views. Each leaf node is a `TabGroup` containing `View` trait objects (which could be an Editor, a Terminal, or a Settings screen). Drag-and-drop modifies the tree structure dynamically, triggering a `taffy` recalculation.

---

### 3. The Data Model: Buffer & Project Management

VSNC fiercely decouples *View* (what is on screen) from *Model* (what is in memory).

- **The Buffer Manager:** A centralized registry mapping `BufferId` to file contents. If `main.rs` is open in three different split panes, there is only *one* Buffer in memory.
- **The Text Structure (Rope):** File contents are stored using the `ropey` (or `crop`) crate. A Rope data structure guarantees sub-millisecond insertions/deletions regardless of file size (up to multi-gigabyte files) and allows instant querying of `(line, column)` to byte offsets.
- **Multiple Cursors & Selections:** Each `EditorView` maintains a `Vec<Selection>`, where a Selection contains an `anchor` and `head` position. All typing and mutating commands iterate over these cursors simultaneously.
- **Undo/Redo History:** Implemented as a persistent tree or stack of diffs applied to the Rope. This allows for branching undo histories.

---

### 4. Editor Intelligence & Parsing

Syntax highlighting and language intelligence are handled natively, providing context-aware tools without the slow regex parsers of older editors.

- **Syntax Highlighting (`tree-sitter`):** Every buffer is paired with a background `tree-sitter` parser. As the user types, the concrete syntax tree is incrementally updated. The tree nodes (e.g., `function_item`, `string_literal`) are mapped to theme colors.
- **Fallback Highlighting (`syntect`):** For obscure languages lacking a tree-sitter grammar, VSNC parses standard `.tmLanguage` files (TextMate grammars) using `syntect`.
- **Language Server Protocol (LSP):** The editor acts as an LSP client. When a Rust or Python file is opened, VSNC spawns `rust-analyzer` or `pyright` in the background. It communicates via JSON-RPC over `stdin/stdout` to provide:
  - Autocomplete popups.
  - Go to Definition / Find References.
  - Hover documentation.
  - Inline diagnostics (red/yellow squiggly lines rendered via `wgpu` under text).
- **Debug Adapter Protocol (DAP):** Implements standard communication with debuggers (`lldb`, `gdb`), allowing visual breakpoints in the gutter, step-over execution, and a variables panel.

---

### 5. Core Tooling Modules

These are the non-editor panels that make VSNC a complete IDE.

#### A. File Explorer & Workspace

- **Filesystem Watching (`notify`):** Monitors the project directory for external changes (git branch swaps, terminal creations) and updates the in-memory tree.
- **Directory Reading (`ignore`):** Reads the workspace tree rapidly by immediately disregarding anything in `.gitignore` (like `target/` or `node_modules/`), preventing I/O lockups.
- **Tree UI:** An interactive, collapsible list view supporting drag-and-drop, rename, delete, and create operations.

#### B. Integrated Terminal Engine

- **Pseudo-Terminal Spawning (`portable-pty`):** Forks a system shell (bash, zsh, powershell) and attaches it to a hidden PTY.
- **Escape Code Parsing (`alacritty_terminal` core):** Reads the raw byte stream from the PTY, parses ANSI escape codes (colors, cursor movements), and maintains a virtual grid of characters.
- **Rendering:** The virtual grid is handed to `glyphon` and `wgpu` to render as a fast, hardware-accelerated terminal panel directly within the VSNC split-tree.

#### C. Git Integration (`gix` / Gitoxide)

- **The Engine:** Powered entirely by `gix`, a pure-Rust implementation of Git. Zero shelling out to the `git` CLI executable.
- **Gutter Markers:** On every buffer mutation, a fast diff is run against the git `HEAD`. Blue/Green/Red thin quads are drawn in the editor's left margin to indicate modified, added, or deleted lines.
- **Source Control Panel:** A dedicated panel listing staged and unstaged files, allowing users to commit, push, and pull directly from the UI.

#### D. Global Search & Navigation

- **Fuzzy Finding (`nucleo`):** Powers the `Ctrl+P` (Find File) and `Ctrl+Shift+P` (Command Palette) features. `nucleo` can filter and rank millions of strings in microseconds, providing instant keystroke feedback.
- **Project Search (`ripgrep` core crates):** `grep-regex` and `grep-searcher` run heavily optimized, multi-threaded text searches across the entire workspace, feeding results back to a Search Panel view.

---

### 6. Input Routing & Keymaps

Input handling handles the complex context requirements of an IDE.

- **Action System:** Keystrokes are not mapped to hardcoded functions. They map to an `Action` enum (e.g., `Action::SplitRight`, `Action::MoveLineDown`).
- **Contextual Routing:** When a key is pressed, the Input Router checks the currently focused UI node. It cascades the keypress upwards. If the Terminal is focused, `Ctrl+C` sends a `SIGINT`. If the Editor is focused, `Ctrl+C` triggers `Action::Copy`.
- **Chord Support:** The router holds state for multi-keypress sequences (e.g., `Ctrl+K` followed by `Ctrl+S`).
- **Vim Mode (Optional Sub-system):** Because actions are routed through an engine, a state-machine can be toggled to interpret inputs as Vim commands (Normal, Insert, Visual modes) before dispatching the final `Action`.

---

### 7. Configuration & Theming Engine

- **Hot-Reloadable Config:** Settings are defined in a `.toml` or `.json` file. The `notify` crate watches this file. When saved, `serde` deserializes the new settings, and the global `Config` struct is atomically updated via a read-write lock (`RwLock`).
- **Dynamic Layout Trigger:** Changes to `font_size`, `line_height`, or `tab_width` immediately trigger a `taffy` layout invalidation and a `cosmic-text` re-shape, redrawing the UI on the next frame.
- **Theming:** A centralized `Theme` struct dictates all UI colors. Standard VS Code `.tmTheme` files can be parsed and converted into VSNC themes.

---

### 8. Extensibility: The WASM Plugin Architecture

To maintain absolute stability and performance while allowing community extensions, VSNC utilizes WebAssembly instead of Node.js.

- **The WASM Runtime (`wasmtime`):** Plugins are compiled to `.wasm` modules.
- **The Sandbox:** Plugins execute in a strictly sandboxed environment. They cannot access the filesystem, network, or OS directly.
- **Host Functions:** VSNC exposes specific Rust host functions to the WASM module (e.g., `vsnc_get_selection`, `vsnc_insert_text`, `vsnc_spawn_notification`).
- **Language Agnostic:** Because it uses WASM, users can write VSNC plugins in Rust, Go, C, or Zig.

---

## Summary of the Data Flow (End-to-End Example)

*User types `a` in the editor:*

1. `winit` detects the `A` key press.
2. Input Router checks the keymap, sees `A` without modifiers, maps it to `Action::InsertCharacter('a')`.
3. The Action is sent to the currently focused `EditorView`.
4. The `EditorView` tells the `BufferManager` to insert `'a'` at the current cursors.
5. `ropey` performs the insertion in memory.
6. The `BufferManager` fires an async event to the background `tree-sitter` worker to parse the new syntax tree.
7. The `BufferManager` fires an async event to the `gix` worker to diff the line.
8. The `BufferManager` fires an event to the background LSP worker to request autocomplete suggestions for the new context.
9. The UI Main Thread continues; `EditorView` invalidates its layout.
10. `taffy` re-measures the text line.
11. `glyphon` caches the glyph for `'a'` if not already on the GPU.
12. `wgpu` draws the new quad.

*(All of this happens in less than 4 milliseconds).*
