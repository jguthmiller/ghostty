# KDE Output Order Protocol for Primary Monitor Detection

## Overview

Add `kde_output_order_v1` Wayland protocol support so `quick-terminal-screen = main` correctly identifies the primary monitor on Linux. GTK4 has no primary monitor API; this protocol provides the compositor's monitor priority ordering and is supported by KDE Plasma, GNOME/Mutter, Hyprland, Sway, and others.

## Current State Analysis

- `resolveQuickTerminalMonitor()` (`wayland.zig:139-170`) uses a coordinate heuristic (monitor at 0,0) that fails when the primary monitor isn't at the display origin.
- Three KDE protocols (blur, server-decoration, slide) are already integrated following a consistent pattern: XML registration, binding generation, Context field, auto-bind via `registryListener()`, listener setup in `App.init()`.
- `kde-output-order-v1.xml` exists in the `plasma_wayland_protocols` dependency at `src/protocols/kde-output-order-v1.xml`.
- The protocol sends `output(name)` events in priority order (first = primary), then `done`. It resends on changes.
- `gdk_monitor_get_connector()` returns connector names (e.g., `"DP-1"`) matching the protocol's output names.

## Desired End State

`quick-terminal-screen = main` uses the compositor-reported primary monitor. Falls back to first GDK monitor if the protocol is unavailable.

## What We're NOT Doing

- Per-name monitor selection (e.g., `quick-terminal-screen = DP-1`)
- X11 support (quick terminal is Wayland-only)

## Phase 1: Add Protocol Binding

**File**: `src/build/SharedDeps.zig`

After line 635 (the `slide.xml` block), add:
```zig
scanner.addCustomProtocol(
    plasma_wayland_protocols_dep.path("src/protocols/kde-output-order-v1.xml"),
);
```

After line 642 (the `org_kde_kwin_slide_manager` generate), add:
```zig
scanner.generate("kde_output_order_v1", 1);
```

### Success Criteria

#### Automated Verification:
- [x] Build succeeds: `zig build`

---

## Phase 2: Bind Protocol and Use for Monitor Resolution

### 2a. Add Context fields

**File**: `src/apprt/gtk/winproto/wayland.zig`

Add to the `Context` struct after `kde_slide_manager` (line 34):
```zig
kde_output_order: ?*org.KdeOutputOrderV1 = null,

/// Connector name of the primary output (e.g., "DP-1") as reported
/// by kde_output_order_v1. The first output in each priority list
/// is the primary.
primary_output_name: ?[63:0]u8 = null,

/// Tracks the output order event cycle. Set to true after a `done`
/// event so the next `output` event is captured as the new primary.
/// Initialized to true so the first event after binding is captured.
output_order_done: bool = true,
```

The `kde_output_order` field is an optional pointer to a Wayland interface type, so `registryListener()` auto-binds it via `getInterfaceType()`. No registry changes needed.

### 2b. Add event listener

**File**: `src/apprt/gtk/winproto/wayland.zig`

Add after `decoManagerListener` (after line 223):
```zig
fn outputOrderListener(
    _: *org.KdeOutputOrderV1,
    event: org.KdeOutputOrderV1.Event,
    context: *Context,
) void {
    switch (event) {
        .output => |v| {
            if (context.output_order_done) {
                context.output_order_done = false;
                const name = std.mem.sliceTo(v.output_name, 0);
                if (name.len <= 63) {
                    var buf: [63:0]u8 = @splat(0);
                    @memcpy(buf[0..name.len], name);
                    context.primary_output_name = buf;
                    log.debug("primary output: {s}", .{name});
                }
            }
        },
        .done => {
            context.output_order_done = true;
        },
    }
}
```

### 2c. Set listener in App.init()

**File**: `src/apprt/gtk/winproto/wayland.zig`

Add after the decoration manager listener block (after line 90):
```zig
if (context.kde_output_order) |output_order| {
    output_order.setListener(*Context, outputOrderListener, context);
    if (display.roundtrip() != .SUCCESS) return error.RoundtripFailed;
}
```

### 2d. Update resolveQuickTerminalMonitor

**File**: `src/apprt/gtk/winproto/wayland.zig`

Change `initQuickTerminal` signature from `_: *App` to `self: *App`, and pass `self.context` to `resolveQuickTerminalMonitor`.

Change `resolveQuickTerminalMonitor` to take `*Context` and match by connector name:

```zig
fn resolveQuickTerminalMonitor(
    context: *Context,
    apprt_window: *ApprtWindow,
) ?*gdk.Monitor {
    const config = if (apprt_window.getConfig()) |v| v.get() else return null;
    const display = apprt_window.as(gtk.Widget).getDisplay();

    return switch (config.@"quick-terminal-screen") {
        .mouse => null,
        .main, .@"macos-menu-bar" => blk: {
            const monitors = display.getMonitors();
            const primary_name: ?[]const u8 = if (context.primary_output_name) |*buf|
                std.mem.sliceTo(buf, 0)
            else
                null;

            var fallback: ?*gdk.Monitor = null;
            var i: u32 = 0;
            while (monitors.getObject(i)) |item| : (i += 1) {
                defer item.unref();
                const monitor = gobject.ext.cast(gdk.Monitor, item) orelse continue;
                if (fallback == null) fallback = monitor;

                if (primary_name) |name| {
                    const connector = std.mem.sliceTo(
                        monitor.getConnector() orelse continue,
                        0,
                    );
                    if (std.mem.eql(u8, connector, name)) {
                        break :blk monitor;
                    }
                }
            }
            break :blk fallback;
        },
    };
}
```

### 2e. Update call sites

**`initQuickTerminal`** (line 130): Change `_: *App` to `self: *App`, call `resolveQuickTerminalMonitor(self.context, apprt_window)`.

**`enteredMonitor`** (line 529): Access context via `apprt_window.winproto()`:
```zig
const context = switch (apprt_window.winproto().*) {
    .wayland => |*wl| wl.app_context,
    else => null,
};
const size_monitor = switch (config.@"quick-terminal-screen") {
    .mouse => monitor,
    .main, .@"macos-menu-bar" => if (context) |ctx|
        resolveQuickTerminalMonitor(ctx, apprt_window) orelse monitor
    else
        monitor,
};
```

### Success Criteria

#### Automated Verification:
- [x] Build succeeds: `zig build`

#### Manual Verification:
- [x] Debug log shows `primary output: DP-1` (or equivalent) on startup
- [x] With `quick-terminal-screen = main`, quick terminal appears on KDE-configured primary monitor regardless of mouse position
- [x] With `quick-terminal-screen = mouse`, quick terminal follows mouse (unchanged)
- [x] Quick terminal is correctly sized for the target monitor
- [x] Toggle multiple times for consistent behavior

---

## References

- Parent plan: `thoughts/shared/plans/2026-01-25-quick-terminal-screen-linux.md`
- Protocol spec: `plasma_wayland_protocols/src/protocols/kde-output-order-v1.xml`
- Protocol docs: https://wayland.app/protocols/kde-output-order-v1
