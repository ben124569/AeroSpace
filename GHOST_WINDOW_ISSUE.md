# Ghost Window Issue - Investigation and Attempted Fixes

## Problem Description
When exiting fullscreen mode with any tiled app in AeroSpace, a "ghost window" appears - a frozen/stuck image of the window that remains visible on screen. This affects all tiled applications, not just specific ones.

### Key Observations
1. The ghost window appears for ANY tiled app when exiting fullscreen
2. User reported seeing a flash of the ghost window BEFORE entering fullscreen, not just when exiting
3. This suggests the issue might be related to how windows are captured/positioned when transitioning states

## Investigation Findings

### Code Areas Examined
1. **MacosNativeFullscreenCommand.swift** - Handles native macOS fullscreen transitions
2. **FullscreenCommand.swift** - AeroSpace's own fullscreen implementation  
3. **normalizeLayoutReason.swift** - Contains `exitMacOsNativeUnconventionalState` function
4. **layoutRecursive.swift** - Handles window layout and positioning
5. **refresh.swift** - Contains workspace refresh and window hide/unhide logic

### Key Functions Involved
- `setNativeFullscreen()` - Sets the macOS fullscreen state
- `exitMacOsNativeUnconventionalState()` - Handles transition out of fullscreen
- `relayoutWindow()` - Repositions windows after state changes
- `layoutWorkspace()` - Layouts all windows in a workspace

## Attempted Fixes

### 1. Exit Delays (FAILED)
**Theory**: macOS needs time to complete fullscreen exit animation before repositioning windows
- Added 300ms delay after fullscreen exit - No effect
- Increased to 500ms - No effect  
- Increased to 800ms - No effect
- Added additional 300ms delay in `exitMacOsNativeUnconventionalState` - No effect

**Result**: Delays did not fix the ghost window issue

### 2. Force Workspace Layout (FAILED)
**Theory**: Windows need explicit relayout after fullscreen exit
- Added `workspace.layoutWorkspace()` call after exiting fullscreen
- This forces all windows to recalculate positions

**Result**: Did not fix the ghost window issue

### 3. Entry Delay (PARTIAL/UNCLEAR)
**Theory**: Based on user observation of flash BEFORE entering fullscreen
- Added 100ms delay before entering fullscreen to let window position settle
- Aimed to prevent window from being captured in intermediate state

**Result**: User reports issue still persists, though behavior may have changed slightly

## Current State
The ghost window issue remains unresolved. The problem appears to be more complex than simple timing or layout issues. It may be related to:
- How macOS captures window state during transitions
- AeroSpace's window hiding/unhiding mechanism
- The interaction between AeroSpace's tiling and macOS native fullscreen

## Next Steps to Try
1. Investigate window hide/unhide mechanism more deeply
2. Check if windows are being properly unhidden from corners before fullscreen
3. Look into forcing a complete refresh session after fullscreen transitions
4. Consider if the issue is related to how AeroSpace tracks window state
5. Test with different window management approaches (floating vs tiled)

## Files Modified in This Branch
- `/Sources/AppBundle/command/impl/MacosNativeFullscreenCommand.swift`
- `/Sources/AppBundle/normalizeLayoutReason.swift` (reverted changes)

## Related Issues
- Problem ID-B6E178F2: macOS native fullscreen is not a first-class citizen in AeroSpace model
- The command interacts with macOS API directly which can cause delays and flicker