# Spacebar Propagation Fix for Hotkeys

## Problem

When using space-based hotkeys like `Option+Space` (macOS) or `Ctrl+Space` (Windows/Linux) to trigger the Handy app's listening mode, the spacebar key event was propagating to other applications or text fields before the hotkey blocking could take effect. This resulted in unwanted space characters being typed in the active text field.

## Root Cause

The `handy-keys` library provides hotkey blocking functionality that prevents registered hotkeys from reaching other applications. However, there's a timing issue where the space key event may reach other applications before the blocking mechanism intercepts it. This is especially noticeable on systems with:
- Slower event processing
- High system load
- Certain keyboard configurations

## Solution

This fix implements a **cleanup mechanism** that sends a backspace keystroke immediately after the hotkey is detected to remove any accidentally typed space character.

### Implementation Details

1. **New function in `input.rs`**: `cleanup_space_from_hotkey()`
   - Checks if the triggered hotkey contains "space"
   - Sends a backspace keystroke to delete the unwanted space
   - Uses a small delay (10ms) to ensure the space has been typed before deletion

2. **Modified `shortcut/handler.rs`**: 
   - Calls the cleanup function when a hotkey press event is detected
   - Only runs when `suppress_space_on_hotkey` setting is enabled

3. **New setting**: `suppress_space_on_hotkey`
   - Default: `true` (enabled)
   - Can be disabled if users experience issues or prefer the original behavior
   - Accessible via `change_suppress_space_on_hotkey_setting` command

### Files Modified

- `src-tauri/src/input.rs` - Added `send_backspace()` and `cleanup_space_from_hotkey()` functions
- `src-tauri/src/shortcut/handler.rs` - Integrated space cleanup into hotkey event handling
- `src-tauri/src/settings.rs` - Added `suppress_space_on_hotkey` setting
- `src-tauri/src/shortcut/mod.rs` - Added command to change the setting
- `src-tauri/src/lib.rs` - Registered the new command

## Build and Test Instructions

### Prerequisites

Follow the instructions in [BUILD.md](BUILD.md) to set up your development environment.

### Building

```bash
# Install dependencies
bun install

# Development build
bun tauri dev

# Production build
bun tauri build
```

### Testing the Fix

1. Start the application
2. Open any text editor or text field (e.g., Notes app, browser search bar)
3. Press the hotkey (default: `Option+Space` on macOS, `Ctrl+Space` on Windows/Linux)
4. Verify that:
   - The app starts listening (overlay appears)
   - No space character is typed in the text field
5. Release the hotkey to stop recording
6. Repeat the test several times to ensure consistency

### Disabling the Fix

If you experience issues with the fix (e.g., unwanted backspaces in certain applications), you can disable it:

1. Through the app's settings (if UI is available)
2. Or by modifying the settings file directly:
   - macOS: `~/Library/Application Support/com.handy.app/settings.json`
   - Windows: `%APPDATA%\com.handy.app\settings.json`
   - Linux: `~/.config/com.handy.app/settings.json`

Set `"suppress_space_on_hotkey": false` in the settings file.

## Limitations

- The fix works by sending a backspace after the space is typed, which may cause a brief visual flicker in some applications
- In rare cases, if the text cursor is at the beginning of a text field with no preceding characters, the backspace will have no effect (which is harmless)
- The fix only applies to hotkeys containing "space" in their definition

## Alternative Solutions Considered

1. **Using a different hotkey**: Users can change the hotkey to one that doesn't use the space key (e.g., `Option+Shift+H`)
2. **Waiting for upstream fix**: The `handy-keys` library may improve its blocking timing in future versions
3. **Platform-specific event suppression**: Would require significant platform-specific code and may not be reliable across all scenarios

The implemented cleanup approach was chosen as it provides a reliable cross-platform solution without requiring changes to the underlying hotkey library.
