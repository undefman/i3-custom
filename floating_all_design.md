# Floating All Windows Feature in i3

This document summarizes the design and implementation of the `floating_all` feature added to i3.

## Overview
The `floating_all` feature allows configuring i3 to make all new windows float by default (similar to traditional desktop environments like GNOME).

### Core Requirements implemented:
1. **Config file directive**: `floating_all on|off` to enable or disable the mode by default on startup.
2. **Runtime toggle/command**: `floating_all on|off|toggle|enable|disable` (via `i3-msg`) to modify the mode dynamically.
3. **Preserve size on enabling**: If `floating_all` is turned on at runtime, currently open tiling windows are floated but retain their exact screen size and position (so they do not jump around or resize).
4. **Auto-centering for new windows**: New windows opened while `floating_all` is active are automatically centered on the current workspace using their normal/requested size.
5. **No reverting on disabling**: Turning `floating_all` off at runtime leaves currently open floating windows as floating, only applying the tiling layout to windows opened afterward.

---

## File Changes & Code Structure

The implementation spans the following files:

### 1. Configuration Declaration
* **File**: `include/configuration.h`
* Declared `bool floating_all` inside the global `struct Config` to store the active state.
* **File**: `src/config.c`
* Initialized `config.floating_all = false` during configuration resets.

### 2. Configuration Parsing
* **File**: `parser-specs/config.spec`
* Registered the keyword `floating_all` under the `INITIAL` parsing state and mapped it to the `FLOATING_ALL` state to parse booleans like `on`/`off`/`yes`/`no`.
* **File**: `include/config_directives.h`
* Declared the configuration function prototype: `CFGFUN(floating_all, const char *value);`.
* **File**: `src/config_directives.c`
* Implemented `CFGFUN(floating_all, const char *value)` to parse and set `config.floating_all`.

### 3. Runtime Command Parsing
* **File**: `parser-specs/commands.spec`
* Swapped order of `floating` and `floating_all` keyword registration in `INITIAL` state to avoid prefix matching collision.
* Registered the runtime command `floating_all` and mapped it to transition to state `FLOATING_ALL` with values `on`, `off`, `toggle`, `enable`, `disable`.
* **File**: `include/commands.h`
* Declared the command prototype: `void cmd_floating_all(I3_CMD, const char *floating_all_mode);`.
* **File**: `src/commands.c`
* Implemented `cmd_floating_all` which updates `config.floating_all` and, if turned on, collects all tiling windows on non-internal workspaces and floats them while preserving their current layout rect/size.

### 4. Floating & Sizing Logic
* **File**: `include/floating.h` & `src/floating.c`
* Updated the `floating_enable` signature to accept a third argument: `bool use_current_size`.
* Inside `floating_enable`:
  * If `use_current_size` is `true`, it sets the new floating container size `nc->rect` to `con->rect` directly (preserving screen layout and position) and skips decoration/size limit adjustments.
  * If `use_current_size` is `false`, it uses the application geometry and centers the window on the active workspace (via `floating_center`) if `config.floating_all` is enabled or if the requested coordinates are `(0, 0)`.
* **File**: `src/manage.c`
* Defaulted `want_floating` to `config.floating_all` for new managed windows.
* Passes `false` as the `use_current_size` argument to `floating_enable` for new windows.
* **Other updated files** (passing default `false` to `use_current_size` parameter):
  * `src/scratchpad.c`
  * `src/load_layout.c`
  * `src/handlers.c`

---

## How to Use

### 1. In `~/.config/i3/config`
To enable this feature by default on startup, add:
```
floating_all on
```
To keep standard tiling behavior (default):
```
floating_all off
```

### 2. At Runtime via `i3-msg`
You can run commands to toggle or update the mode dynamically:
* **Toggle**: `i3-msg floating_all toggle`
* **Enable**: `i3-msg floating_all on` (or `i3-msg floating_all enable`)
* **Disable**: `i3-msg floating_all off` (or `i3-msg floating_all disable`)

### 3. Bind to a hotkey
You can add a keybinding to toggle the mode in your config file:
```
bindsym $mod+Shift+f floating_all toggle
```
