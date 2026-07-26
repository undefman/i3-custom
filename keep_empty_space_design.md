# Keep Empty Space Feature in i3

This document summarizes the design and implementation of the `keep_empty_space` feature added to i3.

## Overview
Normally, when a window is closed in i3, the tiling layout collapses and the remaining windows expand to fill the empty space. The `keep_empty_space` feature prevents this collapse by turning closed tiling windows into invisible placeholders, preserving the layout structure. When a new window is opened, it automatically fills the available placeholder space.

### Core Requirements implemented:
1. **Config file directive**: `keep_empty_space on|off` to enable or disable the mode. Enabled (`on`) by default on startup.
2. **Runtime toggle/command**: `keep_empty_space on|off|toggle|enable|disable` (via `i3-msg`) to modify the mode dynamically.
3. **Keep space on close**: Closed tiling windows turn into empty placeholders, taking up the exact same width, height, and position. They are rendered as transparent empty space (showing the desktop wallpaper underneath).
   - **Adjacent Placeholder Merging**: If multiple adjacent tiling windows are closed, their placeholders are automatically merged into a single larger placeholder taking up the combined screen space (percentage). Non-adjacent placeholders remain separate.
4. **Fills empty slots**: When a new tiling window is opened on a workspace with placeholders:
   - It will automatically reuse the focused placeholder container.
   - If no placeholder is focused, it will reuse the first available placeholder on the target workspace.
5. **Deleting slots manually**: Focusing a placeholder and running the `kill` command (e.g., `$mod+Shift+q`) deletes the empty slot normally, allowing surrounding windows to expand and reclaim the space.

---

## File Changes & Code Structure

The implementation spans the following files:

### 1. Structure Definitions
* **File**: `include/configuration.h`
  - Added `bool keep_empty_space;` to the global `struct Config` to store the active state.
* **File**: `include/data.h`
  - Added `bool is_placeholder;` to `struct Con` to track whether a container represents an empty slot.
* **File**: `src/config.c`
  - Initialized `config.keep_empty_space = false;` on startup.

### 2. Configuration & Command Parsing
* **File**: `parser-specs/config.spec`
  - Registered the `keep_empty_space` config directive and mapped it to parse boolean values (`on`/`off`/`yes`/`no`).
* **File**: `include/config_directives.h` & `src/config_directives.c`
  - Declared and implemented `CFGFUN(keep_empty_space, const char *value)` to parse the config option.
* **File**: `parser-specs/commands.spec`
  - Registered the `keep_empty_space` runtime command and mapped it to state `KEEP_EMPTY_SPACE` with values `on`, `off`, `toggle`, `enable`, `disable`.
* **File**: `include/commands.h` & `src/commands.c`
  - Declared and implemented `cmd_keep_empty_space(I3_CMD, const char *keep_empty_space_mode)` to update the config dynamically at runtime.

### 3. Window Closing Hook (Placeholder Creation)
* **File**: `src/tree.c`
  - Modified `tree_close_internal` so that when a tiling window is closed while `keep_empty_space` is active:
    - The X11 client window is unmapped and reparented to the root window (removed from screen).
    - The window struct is freed and set to `NULL`.
    - The container `con` itself is NOT detached or freed; it is marked with `con->is_placeholder = true`, renamed to `"[Empty]"`, and set to `con->mapped = false` (unmapping its WM frame so it shows up as empty background space).
    - **Adjacent Placeholder Merging & Memory Safety**:
      - It checks if the adjacent containers (prev/next sibling in the parent) are already placeholders. If so, they are merged by adding the closed container's space percentage to the sibling, then detaching and freeing the redundant container.
      - **Focus Safety**: If a container being freed is currently focused, it shifts focus to the kept placeholder (`con_activate`) first, avoiding dangling pointers in the global `focused` variable that would cause segfaults.
      - **X11 Cleanup**: Explicitly calls `x_con_kill` before freeing to cleanly unregister and destroy the X11 window frame of the freed placeholder.
    - **Excluding Dock Clients**: Dock client containers (whose parent is of type `CT_DOCKAREA`) are excluded from being converted to placeholders so they close normally without leaving empty spaces.
    - If the user runs the `kill` command on an empty slot, `con->window` is already `NULL`, which skips the intercept logic and cleanly destroys the placeholder container.

### 4. Window Mapping Hook (Placeholder Reuse)
* **File**: `src/manage.c`
  - Implemented `find_placeholder_recursive(Con *con)` helper to locate placeholders.
  - Modified `manage_window` to check if a workspace has placeholders before creating a new container:
    - If `keep_empty_space` is active and there are placeholder containers:
      - It checks if the currently focused container is a placeholder on the target workspace.
      - Otherwise, it searches the target workspace recursively for the first available placeholder.
      - If a placeholder is found, it reuses that container (`nc`), resets `is_placeholder = false`, maps it (`mapped = true`), and places the new client window inside it.

### 5. Layout Serialization & Deserialization (In-Place Restart Preservation)
* **Files**: `src/ipc.c` and `src/load_layout.c`
  - Added `is_placeholder` field serialization in `dump_node` inside `src/ipc.c`.
  - Added `is_placeholder` parsing logic in `json_bool` inside `src/load_layout.c` to properly restore placeholder state when i3 restarts in-place.

### 6. Dockarea Sorting Robustness
* **File**: `src/con.c`
  - Added a defensive null check in `_con_attach` to skip containers with `window == NULL` (such as placeholders) when sorting dock clients, preventing null pointer dereferences.

---

## How to Use

### 1. In `~/.config/i3/config`
To enable this feature by default on startup, add:
```
keep_empty_space on
```

### 2. At Runtime via `i3-msg`
You can run commands to toggle or update the mode dynamically:
* **Toggle**: `i3-msg keep_empty_space toggle`
* **Enable**: `i3-msg keep_empty_space on` (or `i3-msg keep_empty_space enable`)
* **Disable**: `i3-msg keep_empty_space off` (or `i3-msg keep_empty_space disable`)

### 3. Bind to a hotkey
You can add a keybinding to toggle the mode in your config file:
```
bindsym $mod+Shift+s keep_empty_space toggle
```
