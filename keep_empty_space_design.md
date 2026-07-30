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
6. **Empty Workspace Cleanup**: Workspaces that contain only placeholder containers (i.e. all windows have been closed) are automatically cleaned up and closed when switching away from them (just like normal empty workspaces).
7. **Auto-Cleanup on Squeeze**: If a placeholder is resized (via keyboard or mouse) below a width or height threshold of 40 pixels, it is automatically destroyed, allowing surrounding windows to expand and reclaim the space.
8. **Failed Window Mapping Protection**: If a window mapping or reparenting fails (e.g., temporary helper/transient windows opened and quickly closed by applications during splits/restores), the container is deleted immediately without leaving unwanted empty placeholders.
9. **Client Reparenting Detection (UnmapNotify Bypass)**: If a client window is unmapped because it was reparented internally by the client itself (e.g., splitting terminal pane in Konsole) rather than being closed/destroyed, i3 queries the window's parent. If it is no longer under our frame window, it bypasses the placeholder conversion, preventing unwanted empty spaces.
10. **Short-Lived Window Filtering**: To prevent temporary dummy/helper windows (e.g., created by Konsole when splitting views or DPI testing) from leaving placeholders if they are destroyed immediately after being mapped, i3 checks if the window's lifetime is 200 milliseconds or less. Such short-lived windows do not trigger placeholder creation. (A 200ms threshold is selected because it is long enough for programmatic windows, yet short enough that no human-initiated window close could occur within it, preventing any user-facing inconsistencies).
11. **Mouse Over-Resize to Fill**: During interactive graphical (mouse) resizing, if the user drags the border towards a placeholder and past its boundary (making its size less than 40px, or trying to drag past it entirely), the placeholder is automatically destroyed, and the resized window expands to reclaim the space, seamlessly filling it.
12. **Click-to-Focus Placeholders**: Users can click directly on the empty space `[Empty]` using the mouse to focus it. i3 intercepts clicks on the placeholder's frame window and activates focus on the container.

---

## File Changes & Code Structure

The implementation spans the following files:

### 1. Structure Definitions
* **File**: `include/configuration.h`
  - Added `bool keep_empty_space;` to the global `struct Config` to store the active state.
* **File**: `include/data.h`
  - Added `bool is_placeholder;` to `struct Con` to track whether a container represents an empty slot.
  - Added `struct timeval managed_since_tv;` to `struct Window` to record high-resolution timestamps.
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
    - Records the mapping timestamp using `gettimeofday(&cwindow->managed_since_tv, NULL)`.

### 5. Layout Serialization & Deserialization (In-Place Restart Preservation)
* **Files**: `src/ipc.c` and `src/load_layout.c`
  - Added `is_placeholder` field serialization in `dump_node` inside `src/ipc.c`.
  - Added `is_placeholder` parsing logic in `json_bool` inside `src/load_layout.c` to properly restore placeholder state when i3 restarts in-place.

### 6. Dockarea Sorting Robustness
* **File**: `src/con.c`
  - Added a defensive null check in `_con_attach` to skip containers with `window == NULL` (such as placeholders) when sorting dock clients, preventing null pointer dereferences.

### 7. Empty Workspace Cleanup
* **File**: `src/workspace.c`
  - Implemented `workspace_is_empty_or_only_placeholders` helper to recursively scan the workspace's tiling tree. If all leaf containers are placeholders and there are no active windows or floating windows, the workspace is treated as empty.
  - Updated the workspace switching hook in `workspace_show` to close the switched-away workspace if it only contains placeholders.

### 8. Auto-Cleanup on Squeeze
* **File**: `src/tree.c`
  - Modified `tree_render` to scan the global container list `all_cons` for any active placeholder containers whose parent is a split container (`CT_CON`) or a workspace (`CT_WORKSPACE`).
  - If a placeholder's width (for horizontal split) or height (for vertical split) is reduced below 40 pixels, it calls `tree_close_internal` to destroy it, letting adjacent tiling windows expand to fill its space. Used a recursion guard to prevent nested rendering loops.

### 9. Failed Window Mapping Protection
* **File**: `src/manage.c`
  - Updated the window mapping function `manage_window` to clean up its newly created container (`nc`) and free the window metadata (`cwindow`) immediately if the X11 reparenting request fails.
  - Setting `nc->window = NULL` before calling `tree_close_internal` bypasses the `keep_empty_space` intercept logic, ensuring the failed container is completely deleted instead of turning into an unwanted placeholder.

### 10. Client Reparenting Detection
* **File**: `src/handlers.c`
  - Modified `handle_unmap_notify_event` to query the window parent via `xcb_query_tree` upon receiving an `UnmapNotify` event.
  - If the window's parent is not our container frame (`con->frame.id`), it indicates the window was reparented internally by the client (such as a split pane in Konsole). i3 temporarily disables `config.keep_empty_space` before calling `tree_close_internal`, ensuring the container is deleted normally without creating a placeholder.

### 11. Short-Lived Window Filtering
* **File**: `src/tree.c`
  - Modified `tree_close_internal` to check the age of the window being closed using `gettimeofday` compared to `con->window->managed_since_tv`.
  - Calculates the elapsed time in milliseconds. If the window was managed for 200 milliseconds or less, it is treated as a programmatic temporary window (such as those created and immediately destroyed by Konsole during multiple split pane actions) rather than a user-initiated window close. This prevents it from being converted into a placeholder, while ensuring manual quick closes (which always take >500ms) consistently leave placeholders.

### 12. Mouse Over-Resize to Fill
* **File**: `src/resize.c`
  - Modified the graphical mouse resize handler `resize_graphical_handler` to monitor if one of the resized containers is a placeholder.
  - If a placeholder is being shrunk, and the dragging movement reduces the placeholder's size below 40 pixels (or pushes past it entirely), it automatically destroys the placeholder using `tree_close_internal` and triggers `tree_render()`. This allows the expanding window to immediately reclaim and fill 100% of the space.

### 13. Click-to-Focus Placeholders
* **Files**: `src/bindings.c`, `src/tree.c`, `src/manage.c`, `src/x.c`, `src/handlers.c`, `libi3/draw_util.c`, `include/resize.h`, `src/resize.c`, `src/click.c`
  - Modified `regrab_all_buttons` in `src/bindings.c` to grab mouse buttons on the frame window `con->frame.id` if the container is a placeholder (`con->is_placeholder == true`). If a container is not a placeholder, it explicitly ungrabs buttons on the frame to prevent dangling grabs.
  - Added calls to `regrab_all_buttons` when a container is converted to a placeholder in `src/tree.c` or when a placeholder container is reused for a new client window in `src/manage.c`.
  - Reset the container's border style and border width to `config.default_border` and `config.default_border_width` in `src/manage.c` when reusing a placeholder, restoring normal window decorations and titlebars.
  - Modified the placeholder search in `src/manage.c` to pre-calculate if the incoming window wants to be floating (based on `_NET_WM_WINDOW_TYPE` dialogs, transients, etc.) and bypass placeholder reuse for floating windows so they do not close/disrupt tiling placeholders.
  - Modified `x_push_node` in `src/x.c` to not force `con->mapped = false` for containers without windows if the container is a placeholder, ensuring its frame window remains mapped to the screen.
  - Set the placeholder container's border style to `BS_NONE` and `current_border_width = 0` in `src/tree.c` when creating it to hide all borders/titlebars.
  - Freed the container's decoration frame buffer `frame_buffer` using `draw_util_surface_free` and explicitly set `con->frame_buffer.id = XCB_NONE` when converting it to a placeholder in `src/tree.c`, indicating to the system that the pixmap must be recreated when the container is reused.
  - Configured the placeholder frame window's X11 background pixmap to `XCB_BACK_PIXMAP_PARENT_RELATIVE` in both `src/tree.c` and `src/x.c` (`x_push_node`) to make it completely transparent (revealing the desktop).
  - Added an `xcb_clear_area` call inside `x_push_node` in `src/x.c` for placeholders to force the X server to repaint/clear the window frame area, copying the parent (desktop) background immediately when mapped.
  - Bypass surface copying in `x_draw_decoration` and `x_push_node` (`src/x.c`) if the container is a placeholder, preventing copying from freed frame buffers.
  - Updated `handle_expose_event` in `src/handlers.c` to check if the parent container is a placeholder; if so, it clears the area using `xcb_clear_area` and returns immediately rather than attempting to copy from the freed `frame_buffer` surface.
  - Modified `surface_initialized` in `libi3/draw_util.c` to return `false` if `surface->surface` is `NULL`. This makes drawing operations safe against freed buffers (such as when `con->frame_buffer` is freed) and prevents Cairo segmentation fault crashes, while keeping `surface->id` intact for container state lookups.
  - Modified titlebar generation in `src/x.c` to use `con->name` (which is `"[Empty]"`) instead of falling back to `"i3: nowin"` for placeholders.
  - When a user clicks on the empty space, X11 sends a `ButtonPress` event on the frame window, which is mapped back to the placeholder container and handled by `route_click` in `src/click.c` to focus it via `con_activate`.
  - Implemented 2D tiling resize using the mouse in `src/click.c` and `src/resize.c`. When the user holds the floating modifier (Mod key) and right-clicks drag close to a corner of a tiling window, i3 detects the corner click and initiates a 2D resize drag, showing crossed vertical and horizontal indicator lines and resizing both the width and height of neighboring containers simultaneously.

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
