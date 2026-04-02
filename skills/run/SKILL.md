---
name: run
description: "Build and run the app. Default: booted simulator. Use `device` for physical device. Use `--delete` for clean install. Specify simulator name to target a specific simulator (e.g. `iPhone 16 Pro`)."
disable-model-invocation: true
allowed-tools: Bash
---

Build and run the app on simulator (default) or physical device.

Arguments: `$ARGUMENTS`

## Output Privacy

Do NOT display raw command output that contains sensitive identifiers. Specifically:
- **Suppress**: device UDIDs, simulator UDIDs, full file paths containing usernames (e.g. `/Users/john/...`), provisioning profile IDs, signing identities, team IDs
- **Show only**: build result (SUCCEEDED/FAILED), errors/warnings, and a clean status message (e.g. "Built and launched on iPhone 16 Pro simulator")
- If the build fails, show only the relevant error lines — not the entire build log

## Configuration

Before running, resolve the project configuration:

1. Look for values explicitly set in `.claude/skills/run/SKILL.md` or CLAUDE.md
2. If not set, **auto-detect** from the current directory:
   - **Workspace**: find `*.xcworkspace` (exclude `Pods`). If exactly one found, use it. If multiple, ask the user.
   - **Scheme**: run `xcodebuild -workspace "$WORKSPACE" -list` and pick the first non-test scheme. If ambiguous, ask.
   - **Bundle ID**: run `xcodebuild -workspace "$WORKSPACE" -scheme "$SCHEME" -showBuildSettings | grep PRODUCT_BUNDLE_IDENTIFIER` 
3. If no `.xcworkspace` found, check for `.xcodeproj` and use `-project` instead of `-workspace`
4. If no Xcode project found at all, abort with: "No Xcode project found in the current directory."

## Steps

### 1. Determine target

- If `$ARGUMENTS` contains `device` → **device mode**
- Otherwise → **simulator mode**
  - If `$ARGUMENTS` contains a simulator name (e.g. `iPhone 16 Pro`, `iPhone Air`, `iPad Pro`) → use that specific simulator
  - Otherwise → use the currently booted simulator, or boot the first available iPhone

### 2. Simulator mode

1. If a simulator name was specified in `$ARGUMENTS`, find its UDID: `xcrun simctl list devices available | grep "<name>"` and boot it if not already booted: `xcrun simctl boot <UDID>; open -a Simulator`
2. Otherwise, find a booted simulator: `xcrun simctl list devices booted | grep -i "iphone\|ipad"`
3. If none booted, list available simulators and boot the first iPhone: `xcrun simctl list devices available | grep -i iphone` then `xcrun simctl boot <UDID>; open -a Simulator`
4. Use the resolved simulator's UDID as `$SIM_ID`
4. If `$ARGUMENTS` contains `--delete`: `xcrun simctl uninstall $SIM_ID $BUNDLE_ID 2>/dev/null`
5. Build: `xcodebuild -workspace "$WORKSPACE" -scheme "$SCHEME" -destination "id=$SIM_ID"`. Check for BUILD SUCCEEDED.
6. Get app path: `xcodebuild -workspace "$WORKSPACE" -scheme "$SCHEME" -destination "id=$SIM_ID" -showBuildSettings | grep -m1 "BUILT_PRODUCTS_DIR"` then append `/$SCHEME.app`
7. Install: `xcrun simctl install $SIM_ID "<APP_PATH>"`
8. Launch: `xcrun simctl launch $SIM_ID $BUNDLE_ID`

### 3. Device mode

1. Find connected device: `xcrun devicectl list devices 2>/dev/null | grep -i connected`
2. Use the device UDID as `$DEVICE_ID`
3. If `$ARGUMENTS` contains `--delete`: `xcrun devicectl device uninstall app --device $DEVICE_ID $BUNDLE_ID 2>/dev/null`
4. Build: `xcodebuild -workspace "$WORKSPACE" -scheme "$SCHEME" -destination "id=$DEVICE_ID" -allowProvisioningUpdates`. Check for BUILD SUCCEEDED.
5. Get app path: `xcodebuild -workspace "$WORKSPACE" -scheme "$SCHEME" -destination "id=$DEVICE_ID" -showBuildSettings | grep -m1 "BUILT_PRODUCTS_DIR"` then append `/$SCHEME.app`
6. Install: `xcrun devicectl device install app --device $DEVICE_ID "<APP_PATH>"`
7. Launch: `xcrun devicectl device process launch --device $DEVICE_ID $BUNDLE_ID`
