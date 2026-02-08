# Quick Terminal Screen Selection on Linux - Implementation Plan

## Overview

Implement the `quick-terminal-screen` config option for Linux/Wayland, allowing users to pin the quick terminal to a specific monitor instead of always following the mouse cursor. This addresses user requests in GitHub discussions #8076 and #7514.

## Current State Analysis

### What Exists Now

1. **Config option defined** (`src/config/Config.zig:2580-2600`):
   ```zig
   @"quick-terminal-screen": QuickTerminalScreen = .main,
   ```

2. **Enum defined** (`src/config/Config.zig:9074-9079`):
   ```zig
   pub const QuickTerminalScreen = enum {
       main,
       mouse,
       @"macos-menu-bar",
   };
   ```

3. **Documentation states**: "Only implemented on macOS"

4. **Linux behavior**: Quick terminal always appears on whichever monitor contains the mouse cursor, with no way to configure this.

### Current Linux Implementation

- Layer-shell initialization at `src/apprt/gtk/winproto/wayland.zig:130-133` calls `layer_shell.initForWindow(window)` without specifying a monitor
- The compositor decides placement (typically mouse cursor location)
- `enteredMonitor()` signal handler at `wayland.zig:481-501` adjusts window size when entering a monitor
- Signal connection for `enter_monitor` at `wayland.zig:288-296`
- `syncQuickTerminal()` at `wayland.zig:405-478` configures layer, keyboard mode, anchoring, margins

### gtk4-layer-shell Bindings

Current bindings in `pkg/gtk4-layer-shell/src/main.zig` (67 lines) expose:
- `isSupported()`, `getProtocolVersion()`, `getLibraryVersion()`
- `initForWindow()`, `setLayer()`, `setAnchor()`, `setMargin()`, `setKeyboardMode()`, `setNamespace()`

Only the `gtk` module is provided to this package (via `SharedDeps.zig:658-661`). The `gdk` module is **not** currently available.

**Missing**: `gtk_layer_set_monitor()` function which exists in the C library with signature:
```c
void gtk_layer_set_monitor(GtkWindow* window, GdkMonitor* monitor);
```

### macOS Reference Implementation

The macOS implementation at `macos/Sources/Features/QuickTerminal/QuickTerminalScreen.swift:24-36` shows how to resolve the enum to an actual screen:
- `.main` → Primary display
- `.mouse` → Display containing mouse cursor
- `.macos-menu-bar` → Display with menu bar (first screen)

## Desired End State

After implementation:

1. Setting `quick-terminal-screen = main` on Linux/Wayland will pin the quick terminal to the primary monitor
2. Setting `quick-terminal-screen = mouse` will use current behavior (monitor containing mouse cursor)
3. Setting `quick-terminal-screen = macos-menu-bar` will map to primary monitor (Linux doesn't have a menu bar equivalent)
4. The quick terminal will appear on the configured monitor regardless of mouse position
5. Window sizing will use the configured monitor's dimensions

### Verification

- Build with `zig build` succeeds
- Run `zig build run` with `quick-terminal-screen = main` and verify quick terminal appears on primary monitor
- Run with `quick-terminal-screen = mouse` and verify current behavior (follows mouse)
- Test with multiple monitors of different resolutions

## What We're NOT Doing

- Adding new Linux-specific screen options (e.g., monitor by name/index)
- Implementing X11 support (X11 backend doesn't support quick terminal)
- Adding per-monitor state caching (macOS has this, but not needed for initial implementation)
- Handling monitor hotplug (if configured monitor disconnects, fall back to compositor default)

## Implementation Approach

The implementation requires three changes:

1. Add `setMonitor()` binding to gtk4-layer-shell
2. Add monitor resolution logic to Wayland handler
3. Call `setMonitor()` during quick terminal initialization

## Phase 1: Add gtk4-layer-shell Binding

### Overview
Add the missing `setMonitor()` function to the Zig bindings for gtk4-layer-shell.

### Changes Required

#### 1. Update gtk4-layer-shell bindings
**File**: `pkg/gtk4-layer-shell/src/main.zig`
**Changes**: Add `setMonitor` function following the existing binding pattern

After line 66, add:
```zig
const gdk = @import("gdk");

pub fn setMonitor(window: *gtk.Window, monitor: ?*gdk.Monitor) void {
    c.gtk_layer_set_monitor(@ptrCast(window), if (monitor) |m| @ptrCast(m) else null);
}
```

Note: The `gdk` module import needs to be added. Check if it's already available or needs to be added to the build configuration.

#### 2. Add `gdk` module to gtk4-layer-shell
**File**: `src/build/SharedDeps.zig`
**Changes**: Add `gdk` module import to the layer_shell_module, similar to how `gtk` is already added at lines 658-661.

After the existing `gtk` import block (line 661), add:
```zig
if (gobject_) |gobject| layer_shell_module.addImport(
    "gdk",
    gobject.module("gdk4"),
);
```

This is required because the new `setMonitor` function needs the `gdk.Monitor` type, and `gdk` is not currently provided to the gtk4-layer-shell package.

### Success Criteria

#### Automated Verification:
- [x] Build succeeds: `zig build`
- [x] No compile errors in `pkg/gtk4-layer-shell/src/main.zig`

#### Manual Verification:
- [x] The new function compiles and links correctly

---

## Phase 2: Add Monitor Resolution Logic

### Overview
Add logic to determine which monitor to use based on the `quick-terminal-screen` config value.

### Changes Required

#### 1. Add monitor resolution function
**File**: `src/apprt/gtk/winproto/wayland.zig`
**Changes**: Add a function to resolve `QuickTerminalScreen` to a `*gdk.Monitor`

Add new function (suggested location: after `initQuickTerminal`, around line 134):
```zig
/// Resolve the quick-terminal-screen config to a specific monitor.
/// Returns null to let the compositor decide (used for .mouse mode).
fn resolveQuickTerminalMonitor(
    apprt_window: *ApprtWindow,
) ?*gdk.Monitor {
    const config = if (apprt_window.getConfig()) |v| v.get() else return null;
    const display = apprt_window.as(gtk.Widget).getDisplay();

    return switch (config.@"quick-terminal-screen") {
        .mouse => null, // Let compositor decide based on mouse position
        .main, .@"macos-menu-bar" => blk: {
            // Get the first monitor as "primary" (GTK4 doesn't have a
            // platform-agnostic primary monitor concept)
            const monitors = display.getMonitors();
            if (monitors.getObject(0)) |item| {
                defer item.unref();
                break :blk gobject.ext.cast(gdk.Monitor, item);
            }
            break :blk null;
        },
    };
}
```

**Notes**:
- GTK4 removed the `gdk_display_get_primary_monitor()` function. The X11-specific `gdk_x11_display_get_primary_monitor()` exists but since quick terminal only works on Wayland, we use the first monitor in the list as the "primary" monitor. This is consistent with common compositor behavior where the first monitor is typically the primary.
- Uses `getObject(0)` + `gobject.ext.cast` pattern, which is the established codebase convention (see `command_palette.zig:334`). The returned object must be unreffed after use.
- The `gio` import is not needed since `getMonitors()` returns its own ListModel type that already has `getObject()` available.

### Success Criteria

#### Automated Verification:
- [x] Build succeeds: `zig build`
- [ ] Unit tests pass: `zig build test`

#### Manual Verification:
- [x] None for this phase

---

## Phase 3: Integrate Monitor Selection

### Overview
Call `layer_shell.setMonitor()` during quick terminal initialization to pin the window to the resolved monitor.

### Changes Required

#### 1. Update initQuickTerminal
**File**: `src/apprt/gtk/winproto/wayland.zig`
**Changes**: Modify `initQuickTerminal` to set the target monitor

Replace the current `initQuickTerminal` function (lines 130-133):

```zig
pub fn initQuickTerminal(_: *App, apprt_window: *ApprtWindow) !void {
    const window = apprt_window.as(gtk.Window);
    layer_shell.initForWindow(window);

    // Set target monitor based on config (null lets compositor decide)
    const monitor = resolveQuickTerminalMonitor(apprt_window);
    layer_shell.setMonitor(window, monitor);
}
```

#### 2. Update enteredMonitor to respect configured monitor
**File**: `src/apprt/gtk/winproto/wayland.zig`
**Changes**: Modify `enteredMonitor` (lines 481-501) to use the configured monitor's geometry for sizing when not in `.mouse` mode

The current implementation uses the monitor from the `enter_monitor` signal. For `.main` mode, we should use the configured monitor's dimensions instead:

```zig
fn enteredMonitor(
    _: *gdk.Surface,
    monitor: *gdk.Monitor,
    apprt_window: *ApprtWindow,
) callconv(.c) void {
    const window = apprt_window.as(gtk.Window);
    const config = if (apprt_window.getConfig()) |v| v.get() else return;

    // Use the configured monitor for sizing if not in mouse mode
    const size_monitor = switch (config.@"quick-terminal-screen") {
        .mouse => monitor, // Use the monitor we entered
        .main, .@"macos-menu-bar" => resolveQuickTerminalMonitor(apprt_window) orelse monitor,
    };

    var monitor_size: gdk.Rectangle = undefined;
    size_monitor.getGeometry(&monitor_size);

    const dims = config.@"quick-terminal-size".calculate(
        config.@"quick-terminal-position",
        .{
            .width = @intCast(monitor_size.f_width),
            .height = @intCast(monitor_size.f_height),
        },
    );

    window.setDefaultSize(@intCast(dims.width), @intCast(dims.height));
}
```

### Success Criteria

#### Automated Verification:
- [x] Build succeeds: `zig build`
- [ ] Unit tests pass: `zig build test`

#### Manual Verification:
- [ ] With `quick-terminal-screen = main`, quick terminal appears on primary monitor regardless of mouse position
- [ ] With `quick-terminal-screen = mouse`, quick terminal appears on monitor containing mouse (current behavior)
- [ ] Quick terminal is correctly sized for the target monitor's resolution
- [ ] Toggle quick terminal multiple times and verify consistent behavior

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation that the manual testing was successful.

---

## Phase 4: Update Documentation

### Overview
Update the config option documentation to reflect Linux support.

### Changes Required

#### 1. Update config documentation
**File**: `src/config/Config.zig`
**Changes**: Update the docstring for `quick-terminal-screen` to indicate Linux support

Replace line 2599 (`/// Only implemented on macOS.`) with:
```zig
/// On macOS, `macos-menu-bar` uses the screen containing the menu bar.
/// On Linux/Wayland, `macos-menu-bar` is treated as equivalent to `main`.
///
/// Note: On Linux, there is no universal concept of a "primary" monitor.
/// The `main` option uses the first monitor reported by the display server,
/// which is typically the primary monitor as configured in your desktop
/// environment settings.
```

### Success Criteria

#### Automated Verification:
- [x] Build succeeds: `zig build`
- [x] Linting passes (if applicable)

#### Manual Verification:
- [ ] Documentation accurately describes the behavior

---

## Testing Strategy

### Unit Tests
No new unit tests required - the functionality integrates with GTK/Wayland APIs that are difficult to mock.

### Integration Tests
None available for this feature.

### Manual Testing Steps

1. **Setup**: Ensure system has multiple monitors connected
2. **Test main mode**:
   - Set `quick-terminal-screen = main` in config
   - Move mouse to secondary monitor
   - Toggle quick terminal
   - Verify it appears on primary monitor
3. **Test mouse mode**:
   - Set `quick-terminal-screen = mouse` in config
   - Move mouse to secondary monitor
   - Toggle quick terminal
   - Verify it appears on secondary monitor
4. **Test sizing**:
   - With monitors of different resolutions
   - Verify quick terminal is correctly sized for its target monitor
5. **Test toggle persistence**:
   - Toggle quick terminal on/off multiple times
   - Verify it consistently appears on the configured monitor

## Performance Considerations

- Monitor resolution happens during `initQuickTerminal` which is called once when the quick terminal is created
- The `getMonitors()` call returns a cached list that doesn't allocate
- No performance impact expected

## Migration Notes

No migration needed. This is a new feature implementation for an existing config option.

## Edge Cases

1. **Single monitor**: Behavior unchanged - quick terminal appears on the only monitor
2. **Monitor disconnected**: If the configured "primary" monitor is disconnected, `getMonitors().getObject(0)` will return a different monitor. This is acceptable fallback behavior.
3. **Hot-plugging monitors**: Monitor list is queried at quick terminal creation time. If monitors change, the next quick terminal creation will pick up the changes.

## References

- Research document: `thoughts/shared/research/2026-01-24-quick-terminal-screen-linux.md`
- Notes: `thoughts/shared/notes/quick-terminal-screen-linux.md`
- GitHub #8076: Quick Terminal should optionally be fixed to opening on a specific display
- GitHub #7514: `non-main` option for `quick-terminal-screen` configuration
- gtk4-layer-shell API: https://wmww.github.io/gtk4-layer-shell/
