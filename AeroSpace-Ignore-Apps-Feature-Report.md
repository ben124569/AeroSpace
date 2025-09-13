# AeroSpace "Ignore Application" Feature Implementation Report

## Executive Summary

After analyzing the AeroSpace source code, implementing an "ignore application" feature is **feasible and relatively straightforward**. The architecture already supports similar concepts through floating windows and the `check-further-callbacks` mechanism. The implementation would allow certain applications (like DaVinci Resolve) to be completely excluded from AeroSpace's window management, letting them manage their own windows without interference.

## Current Architecture Analysis

### Window Detection Pipeline

1. **Application Registration** (`MacApp.swift:40-79`)
   - Apps are registered when detected by macOS
   - Each app gets an AX (Accessibility) observer thread
   - Subscribes to window creation/focus change notifications

2. **Window Registration** (`MacWindow.swift:18-41`)
   - Windows are registered via `MacWindow.getOrRegister()`
   - Initial binding determined by `unbindAndGetBindingDataForNewWindow()`
   - After registration, `tryOnWindowDetected()` processes window rules

3. **Window Detection Callbacks** (`parseOnWindowDetected.swift`)
   - Config system already supports `on-window-detected` rules
   - Currently limited to `layout floating`, `layout tiling`, and `move-node-to-workspace`
   - Has `check-further-callbacks` to stop processing additional rules

### Key Insight: Early Exit Points

The architecture has multiple points where we can "ignore" windows:

1. **App Level**: Skip registration entirely (like lock screen apps)
2. **Window Level**: Register but don't manage (similar to popups)
3. **Callback Level**: Use existing callback system with new command

## Proposed Implementation

### Option 1: App-Level Ignore (Recommended)

**Pros:**
- Most efficient - no resources wasted on ignored apps
- Cleanest implementation
- Completely invisible to AeroSpace

**Implementation:**

1. Add to config parser (`parseConfig.swift`):
```swift
// Add new config field
ignoredApps: [String] = [] // Bundle IDs to ignore
```

2. Modify `MacApp.getOrRegister()` (`MacApp.swift:40`):
```swift
static func getOrRegister(_ nsApp: NSRunningApplication) async throws -> MacApp? {
    // Existing lock screen check
    if nsApp.bundleIdentifier == lockScreenAppBundleId { return nil }
    
    // NEW: Check ignored apps list
    if let bundleId = nsApp.bundleIdentifier, 
       config.ignoredApps.contains(bundleId) { 
        return nil 
    }
    
    // Rest of existing code...
}
```

3. Config usage:
```toml
# aerospace.toml
ignored-apps = [
    'com.blackmagic-design.DaVinciResolve',
    'com.apple.dt.Xcode'  # Example: Also ignore Xcode
]
```

### Option 2: Window-Level Ignore Command

**Pros:**
- More flexible - can use pattern matching
- Reuses existing callback infrastructure
- Can be conditional based on window properties

**Implementation:**

1. Create new command (`IgnoreWindowCommand.swift`):
```swift
struct IgnoreWindowCommand: Command {
    func run(_ env: CmdEnv, _ stdin: CmdStdin) async throws -> CmdResult {
        guard let window = MacWindow.get(byId: env.windowId) else {
            return .failure("Window not found")
        }
        
        // Remove from management
        window.garbageCollect(skipClosedWindowsCache: true)
        
        // Add to ignored windows set
        ignoredWindowIds.insert(window.windowId)
        
        return .success
    }
}
```

2. Modify window detection to skip ignored windows:
```swift
// In MacWindow.getOrRegister()
if ignoredWindowIds.contains(windowId) { 
    return nil 
}
```

3. Config usage:
```toml
[[on-window-detected]]
if.app-id = 'com.blackmagic-design.DaVinciResolve'
check-further-callbacks = false
run = 'ignore-window'  # New command
```

### Option 3: Hybrid Approach

Combine both options:
- Quick ignore list for known apps (Option 1)
- Dynamic ignore command for complex rules (Option 2)

## Implementation Complexity

### Estimated Effort

| Component | Lines of Code | Complexity |
|-----------|--------------|------------|
| Config Parser | ~20 lines | Low |
| App-Level Ignore | ~10 lines | Low |
| Window Command | ~50 lines | Medium |
| Tests | ~100 lines | Medium |
| **Total** | **~180 lines** | **Low-Medium** |

### Files to Modify

1. **Core Changes:**
   - `Sources/AppBundle/config/Config.swift` - Add ignored-apps field
   - `Sources/AppBundle/config/parseConfig.swift` - Parse ignored-apps
   - `Sources/AppBundle/tree/MacApp.swift` - Check ignore list

2. **Optional (for window-level):**
   - `Sources/Common/cmdArgs/impl/IgnoreWindowCmdArgs.swift` - New command args
   - `Sources/AppBundle/command/impl/IgnoreWindowCommand.swift` - New command
   - `Sources/AppBundle/config/parseOnWindowDetected.swift` - Allow new command

## Testing Strategy

1. **Unit Tests:**
   - Config parsing with ignored-apps
   - Window detection skipping
   - Command execution

2. **Integration Tests:**
   - Launch ignored app, verify no windows registered
   - Switch between ignored and managed apps
   - Fullscreen behavior of ignored apps

3. **Manual Testing:**
   - DaVinci Resolve fullscreen workflow
   - Multiple monitor scenarios
   - App restart scenarios

## Potential Challenges

1. **Window Focus Events**
   - Ignored app windows might still trigger focus events
   - Solution: Filter events at the observer level

2. **Workspace Switching**
   - Need to handle workspace changes with ignored apps
   - Solution: Let macOS handle naturally

3. **Mixed Windows**
   - App might have both ignored and managed windows
   - Solution: Window-level ignore provides granular control

## Why This Isn't Already a Feature

Based on the code comments and GitHub issues:

1. **Design Philosophy**: AeroSpace aims to manage ALL windows for consistency
2. **Edge Cases**: Partially managed apps create complexity
3. **Alternative Solution**: Current floating window support handles many use cases
4. **Limited Demand**: Most users adapt to the tiling paradigm

However, the use case for professional apps like DaVinci Resolve is valid and the implementation is straightforward enough to justify.

## Recommendation

Implement **Option 1 (App-Level Ignore)** first as it's:
- Simplest to implement (~30 lines of code)
- Most efficient performance-wise
- Solves the immediate DaVinci Resolve use case
- Can be extended later with Option 2 if needed

The feature would:
1. Add an `ignored-apps` config array
2. Skip registration for apps in this list
3. Let them operate completely independently of AeroSpace

This gives you the "best of both worlds" - AeroSpace for regular apps, native behavior for specialized apps like DaVinci Resolve.

## Next Steps

1. Fork the AeroSpace repository
2. Implement Option 1 (app-level ignore)
3. Test with DaVinci Resolve
4. Submit PR or maintain personal fork
5. Consider Option 2 if more flexibility needed

The implementation is definitely achievable and would be a valuable addition to AeroSpace!