---
name: install-on-phone
description: Use when the user wants to build and install the app on their physical iPhone or Apple Watch, create an .ipa file, or deploy to a connected device
---

# Install on Phone

Build the current project's iOS app as an .ipa and install it on a connected iPhone.

## Prerequisites

- iPhone connected by USB or Wi-Fi (Xcode > Window > Devices and Simulators > "Connect via network"), unlocked
- The app target uses automatic signing with a development team

## Flow

```
find root, scheme, team ──▶ ExportOptions ──▶ archive ──▶ export .ipa ──▶ install on iPhone
```

## Steps

### 1. Find the project

Run every command from the repository root:

```bash
cd "$(git rev-parse --show-toplevel)"
```

- **Project or workspace:** use the `.xcworkspace` if there is one, otherwise the `.xcodeproj`. If the project is generated (for example from an XcodeGen `project.yml`), generate it first as the project's instructions say.
- **Scheme:** use the scheme the project's instructions (`CLAUDE.md`) name for the iOS app. Otherwise list the schemes with `xcodebuild -list` and pick the one that builds the iOS app. Ask the user if more than one could be it.
- **Team:** read the app target's development team:

  ```bash
  xcodebuild -showBuildSettings -scheme <scheme> -destination 'generic/platform=iOS' 2>/dev/null \
    | sed -nE 's/^ *DEVELOPMENT_TEAM = (.+)$/\1/p' | head -1
  ```

Every later step uses these as `<scheme>` and `<team>`.

### 2. ExportOptions.plist

Skip this step if `ExportOptions.plist` already exists in the repository root.

- If the project has an `ExportOptions.template.plist`, render it with the team:

  ```bash
  sed "s/YOUR_TEAM_ID/<team>/" ExportOptions.template.plist > ExportOptions.plist
  ```

- Otherwise write a development one to `build/ExportOptions.plist` and use that path in step 4:

  ```bash
  mkdir -p build
  /usr/libexec/PlistBuddy -c "Add :method string development" -c "Add :teamID string <team>" build/ExportOptions.plist
  ```

In a git worktree, a git-ignored `ExportOptions.plist` from the main checkout is not there. Render it again in the worktree.

### 3. Archive

```bash
xcodebuild archive \
  -scheme <scheme> \
  -archivePath build/<scheme>.xcarchive \
  -destination 'generic/platform=iOS' \
  -allowProvisioningUpdates
```

- Add `-workspace <name>.xcworkspace` or `-project <name>.xcodeproj` when the root holds more than one.
- `-allowProvisioningUpdates` lets Xcode make a new team provisioning profile. Without it, a capability added to the app since the last profile (for example iCloud) fails with "Provisioning profile … doesn't include the … capability", even when the developer portal is set up correctly.

### 4. Export the .ipa

```bash
rm -rf build/ipa
xcodebuild -exportArchive \
  -archivePath build/<scheme>.xcarchive \
  -exportPath build/ipa \
  -exportOptionsPlist ExportOptions.plist \
  -allowProvisioningUpdates
```

The export names the .ipa after the app, which may differ from the scheme. Use whichever single `.ipa` is in `build/ipa`.

### 5. Install on the iPhone

```bash
xcrun devicectl list devices
xcrun devicectl device install app --device <iPhone UDID> build/ipa/<app>.ipa
```

- Pick the physical iPhone (type `physical`, state `available (paired)`). Ask the user if more than one is connected.
- Confirm the install:

  ```bash
  xcrun devicectl device info apps --device <iPhone UDID> | grep <bundle id>
  ```

### 6. Watch app

Skip this step if the app has no watch app.

A watch app embedded in the iOS app installs on the paired Apple Watch through the iPhone. If it does not appear, the user opens the Watch app on the iPhone, goes to My Watch > Installed on Apple Watch, and turns the app on.

### 7. Report

State the device the app was installed on and its version. Nothing else.

## Troubleshooting

- **Device not found:** unlock the iPhone, check it is connected, and run `xcrun devicectl list devices` again.
- **Signing or profile errors:** check that the archive used `-allowProvisioningUpdates`. If it still fails, the app's capabilities may be missing in the developer portal (Certificates, Identifiers & Profiles > Identifiers > the app's ID). Tell the user which capability the error names.
