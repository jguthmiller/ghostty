---
date: 2026-01-24T23:59:46-06:00
researcher: Claude
git_commit: f479210daf070e487df640d4236522532b13d417
branch: main
repository: ghostty
topic: "Quick Terminal Screen Selection on Linux - Implementation Analysis"
tags: [research, codebase, quick-terminal, wayland, layer-shell, gtk, multi-monitor]
status: complete
last_updated: 2026-01-24
last_updated_by: Claude
---

# Research: Quick Terminal Screen Selection on Linux

**Date**: 2026-01-24T23:59:46-06:00
**Researcher**: Claude
**Git Commit**: f479210daf070e487df640d4236522532b13d417
**Branch**: main
**Repository**: ghostty

## Research Question

Navigate, research, and document the context related to the `quick-terminal-screen` config option on Linux, including why it's not implemented, how the current implementation works, and what would be needed to add support.

## Summary

The `quick-terminal-screen` config option exists in Ghostty's configuration but is **only implemented on macOS**. On Linux/Wayland, the quick terminal always appears on whichever monitor the mouse cursor is located, with no way to pin it to a specific display. This is because:

1. The Linux quick terminal uses the **wlr-layer-shell** Wayland protocol via gtk4-layer-shell
2. The current Zig bindings for gtk4-layer-shell do not expose a monitor/output selection function
3. Layer-shell surfaces are overlay surfaces that don't respond to window manager rules (like KWin rules)

Implementing this feature would require adding the `gtk_layer_set_monitor()` function to the Zig bindings and integrating it with the existing config option.

## Detailed Findings

### Configuration Definition

The `quick-terminal-screen` option is defined in the core configuration:

**Location**: `src/config/Config.zig:2580-2600`

```zig
/// The screen where the quick terminal should show up.
/// ...
/// Only implemented on macOS.
@"quick-terminal-screen": QuickTerminalScreen = .main,
```

**Enum Definition** (`src/config/Config.zig:9075-9079`):
```zig
pub const QuickTerminalScreen = enum {
    main,
    mouse,
    @"macos-menu-bar",
};
```

The option can be set on any platform but only has effect on macOS.

### Linux/Wayland Implementation

#### Layer-Shell Surface Creation

The quick terminal on Linux uses wlr-layer-shell protocol. Initialization happens at:

**Location**: `src/apprt/gtk/winproto/wayland.zig:130-133`

```zig
pub fn initQuickTerminal(_: *App, apprt_window: *ApprtWindow) !void {
    const window = apprt_window.as(gtk.Window);
    layer_shell.initForWindow(window);
}
```

This is called from `src/apprt/gtk/class/window.zig:1106-1112` when the `quick_terminal` property is set.

#### Current Monitor Behavior

The quick terminal has **no explicit monitor selection**. The compositor decides placement (typically where the mouse cursor is). When the surface enters a monitor, Ghostty adapts to that monitor's size:

**Location**: `src/apprt/gtk/winproto/wayland.zig:480-501`

```zig
fn enteredMonitor(
    _: *gdk.Surface,
    monitor: *gdk.Monitor,
    apprt_window: *ApprtWindow,
) callconv(.c) void {
    // ... resize window based on monitor dimensions
}
```

#### Layer-Shell Configuration

The `syncQuickTerminal()` method at `wayland.zig:405-478` configures:
- Layer (overlay/top/bottom/background)
- Namespace identifier
- Keyboard interactivity mode
- Edge anchoring and margins
- KDE slide animations (if available)

**No monitor/output is specified.**

### gtk4-layer-shell Bindings

**Location**: `pkg/gtk4-layer-shell/src/main.zig`

The bindings expose 6 configuration functions:
1. `initForWindow()` - Initialize layer-shell for a window
2. `setLayer()` - Set which layer (overlay, top, bottom, background)
3. `setAnchor()` - Anchor to screen edges
4. `setMargin()` - Set margins from edges
5. `setKeyboardMode()` - Set keyboard interactivity
6. `setNamespace()` - Set layer-shell namespace

**Missing**: There is no `setMonitor()` or equivalent function exposed in these bindings.

### macOS Implementation (Reference)

The macOS implementation shows how screen selection should work:

**Screen Enum** (`macos/Sources/Features/QuickTerminal/QuickTerminalScreen.swift:3-37`):

```swift
enum QuickTerminalScreen {
    case main
    case mouse
    case menuBar

    var screen: NSScreen? {
        switch (self) {
        case .main:
            return NSScreen.main
        case .mouse:
            let mouseLoc = NSEvent.mouseLocation
            return NSScreen.screens.first(where: { $0.frame.contains(mouseLoc) })
        case .menuBar:
            return NSScreen.screens.first
        }
    }
}
```

**Usage** (`macos/Sources/Features/QuickTerminal/QuickTerminalController.swift:419`):

```swift
guard let screen = derivedConfig.quickTerminalScreen.screen else { return }
```

The macOS implementation:
1. Resolves the enum to an actual `NSScreen` at animation time
2. Positions the window on that specific screen
3. Maintains per-screen state cache via `QuickTerminalScreenStateCache`

### Why KWin Rules Don't Work

As documented in `thoughts/shared/notes/quick-terminal-screen-linux.md:26-30`:

> The quick terminal on Linux uses the **wlr-layer-shell** Wayland protocol, which creates overlay surfaces (like panels or docks) rather than regular windows. KWin window rules do not apply to layer-shell surfaces.

Attempted workarounds that failed:
- Setting `screen=0` / `screenrule=2` in `~/.config/kwinrulesrc`
- Position rules to force x=0
- Various wmclass matching patterns

## Code References

### Core Implementation Files

| File | Description |
|------|-------------|
| `src/config/Config.zig:2600` | Config option definition |
| `src/config/Config.zig:9075-9079` | QuickTerminalScreen enum |
| `src/apprt/gtk/winproto/wayland.zig` | Wayland/layer-shell implementation |
| `src/apprt/gtk/winproto/wayland.zig:130-133` | Layer-shell initialization |
| `src/apprt/gtk/winproto/wayland.zig:405-478` | Layer-shell configuration |
| `src/apprt/gtk/winproto/wayland.zig:480-501` | Monitor enter handling |
| `pkg/gtk4-layer-shell/src/main.zig` | gtk4-layer-shell Zig bindings |
| `src/apprt/gtk/class/window.zig:1106-1112` | Quick terminal property handler |

### macOS Reference Implementation

| File | Description |
|------|-------------|
| `macos/Sources/Features/QuickTerminal/QuickTerminalScreen.swift` | Screen selection enum |
| `macos/Sources/Features/QuickTerminal/QuickTerminalController.swift` | Controller using screen config |
| `macos/Sources/Features/QuickTerminal/QuickTerminalScreenStateCache.swift` | Per-screen state caching |
| `macos/Sources/Ghostty/Ghostty.Config.swift:487-495` | Config bridge for Swift |

## Architecture Insights

### Layer-Shell Protocol

The wlr-layer-shell protocol creates surfaces that:
- Exist outside normal window management
- Can be anchored to screen edges
- Support different layers (background to overlay)
- Are typically used for panels, docks, notifications

The protocol **does** support specifying an output (monitor), but this functionality is not exposed in Ghostty's current bindings.

### Cross-Platform Config Pattern

Ghostty uses a pattern where:
1. Config options are defined in Zig (`src/config/Config.zig`)
2. Values are exposed via C API (`ghostty_config_get`)
3. Platform implementations read values through this API
4. Each platform implements the option as appropriate

For `quick-terminal-screen`:
- Zig defines the enum and stores the value
- macOS Swift code reads and implements it
- Linux/GTK code **does not read or use it**

## Historical Context (from thoughts/)

### Related Notes

- `thoughts/shared/notes/quick-terminal-screen-linux.md` - Primary documentation of this limitation

### Related GitHub Discussions

- **#8076**: Quick Terminal should optionally be fixed to opening on a specific display
- **#7514**: `non-main` option for `quick-terminal-screen` configuration
- **#4624**: Quick terminal on Linux (closed - implemented in 1.2.0 without screen selection)

## Implementation Path

To implement `quick-terminal-screen` on Linux:

### 1. Update gtk4-layer-shell Bindings

Add to `pkg/gtk4-layer-shell/src/main.zig`:

```zig
pub fn setMonitor(window: *gtk.Window, monitor: ?*gdk.Monitor) void {
    c.gtk_layer_set_monitor(@ptrCast(window), if (monitor) |m| @ptrCast(m) else null);
}
```

### 2. Add Monitor Resolution Logic

Add to `src/apprt/gtk/winproto/wayland.zig`:

```zig
fn resolveQuickTerminalMonitor(config: Config, display: *gdk.Display) ?*gdk.Monitor {
    return switch (config.@"quick-terminal-screen") {
        .main => // Get primary/default monitor
        .mouse => // Get monitor at mouse position
        .@"macos-menu-bar" => // Map to primary monitor on Linux
    };
}
```

### 3. Call setMonitor During Initialization

In `syncQuickTerminal()` or `initQuickTerminal()`, call the new function to set the target monitor before the window is shown.

### 4. Handle Monitor Changes

Consider whether the quick terminal should:
- Stay on its configured monitor (ignore `enter_monitor` for resizing)
- Move to a different monitor if the configured one disconnects

## Open Questions

1. **Does gtk4-layer-shell expose `gtk_layer_set_monitor()`?** - Need to verify the C library supports this function
2. **What should happen when the configured monitor is disconnected?** - Fall back to mouse position? Primary?
3. **Should `macos-menu-bar` map to primary monitor on Linux?** - Or should a Linux-specific option be added?
4. **X11 support?** - The X11 backend (`src/apprt/gtk/winproto/x11.zig`) would need separate implementation

## Related Research

- None found in `thoughts/shared/research/`
