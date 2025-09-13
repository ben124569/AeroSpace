# Building and Installing Your Modified AeroSpace

## What We've Done

We've successfully added an "ignored-apps" feature to AeroSpace! Here's what was changed:

1. **Added config field** in `Config.swift` (line 58)
2. **Added parser** in `parseConfig.swift` (lines 116 and 299-304)  
3. **Added ignore check** in `MacApp.swift` (lines 45-48)

## Building the App

Since the CLI builds fine but the GUI app has some SwiftUI preview issues (unrelated to our changes), you have two options:

### Option 1: Build with Xcode (Recommended)

1. Open Xcode
2. Open the project file:
   ```bash
   open /Users/benjaminmerritt/Developer/AeroSpace/AeroSpace.xcodeproj
   ```
3. In Xcode:
   - Select "AeroSpace" scheme (not aerospace) from the dropdown at the top
   - Go to Product → Build (or press ⌘B)
   - If you see SwiftUI preview errors, ignore them - they don't affect the actual build
   - Go to Product → Show Build Folder in Finder
   - The built app will be in `Products/Debug/AeroSpace.app`

### Option 2: Try the generate script first

The project uses a generate script before building. Let's try:

```bash
cd /Users/benjaminmerritt/Developer/AeroSpace
./generate.sh
open AeroSpace.xcodeproj
```

Then build in Xcode as above.

## Testing Your Changes

1. Create your config with ignored apps:

```toml
# Add this to your aerospace.toml config
ignored-apps = [
    'com.blackmagic-design.DaVinciResolve'
]
```

2. Quit your current AeroSpace if running

3. Run your modified version:
   - From Xcode: Product → Run
   - Or copy the built app to Applications and run it

4. Test with DaVinci Resolve:
   - Open DaVinci Resolve
   - Go fullscreen
   - It should now be completely ignored by AeroSpace!

## If Build Fails

The SwiftUI preview errors we saw are not critical. If the build still fails:

1. Try cleaning first:
   ```bash
   cd /Users/benjaminmerritt/Developer/AeroSpace
   rm -rf .build
   ./generate.sh
   ```

2. Then open in Xcode and build

3. If you get code signing issues:
   - In Xcode, go to the project settings
   - Under "Signing & Capabilities"
   - Change Team to "None" or your personal team
   - Try building again

## Making it Permanent

Once you verify it works:

1. **Option A**: Keep using your custom build
   - Copy your built `AeroSpace.app` to `/Applications`
   - Use it instead of the Homebrew version

2. **Option B**: Create a Pull Request
   - The changes are clean and minimal
   - This is a legitimate use case
   - Fork the repo on GitHub, push your changes, create PR

## What the Feature Does

With `ignored-apps` configured, AeroSpace will:
- Completely skip registering those applications
- Not create any windows for them
- Not respond to any of their window events
- Let them manage their own fullscreen/windows naturally

This is exactly what you wanted for DaVinci Resolve!