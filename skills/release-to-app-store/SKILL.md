---
name: release-to-app-store
description: Use when the user asks to release, ship, or publish a new version of the app, bump the version for a release, or upload a build to App Store Connect or TestFlight. Picks the next SemVer version from the commits since the last tag, commits the bump on main, tags and pushes it, archives the App Store build with xcodegen and xcodebuild on the CLI, and uploads it to App Store Connect.
---

# Release to App Store

Release a new version of the app and upload it to App Store Connect.

Run every command from the root of the repository: the directory that `git rev-parse --show-toplevel` prints.

Use only the CLI. Never open the project in Xcode for any step: Xcode changes the generated project.

## Flow

```
preflight ─▶ pick version ─▶ bump on main ─▶ tag ─▶ push ─▶ xcodegen ─▶ archive ─▶ upload ─▶ report
```

## Steps

### 1. Preflight

```bash
git branch --show-current
git status --porcelain
git fetch origin --tags
git pull --ff-only origin main
```

- Stop if the current branch is not `main`, or if `git status --porcelain` prints anything.
- Stop if the fetch or pull fails. Report the blocked host per the sandbox rule.

### 2. Pick the version

```bash
last=$(git tag --sort=-v:refname | head -1)
grep -n 'MARKETING_VERSION' project.yml
git log --format='%h %s%n%b' "$last"..HEAD
```

- Stop if there are no commits since `$last`.
- Stop if `MARKETING_VERSION` does not equal `$last`.
- Pick the next version from the commits, using the highest change found:

  | Commits since the last tag | Before 1.0.0 | From 1.0.0 |
  |---|---|---|
  | Any breaking change (`!` after the type, or a `BREAKING CHANGE:` footer) | MINOR | MAJOR |
  | Any `feat` | MINOR | MINOR |
  | Only `fix` and others | PATCH | PATCH |

- Reset the lower parts to 0 (for example `0.17.3` with a `feat` becomes `0.18.0`).
- Call the result `<version>`. Tags are the version alone, with no `v` prefix.

### 3. Bump the version on main

The version bump is committed directly on `main`, without a branch.

```bash
sed -i '' 's/MARKETING_VERSION: <last>/MARKETING_VERSION: <version>/' project.yml
xcodegen generate
git diff
```

- `git diff` must show only the `MARKETING_VERSION` line in `project.yml`. Stop if it shows anything else.
- Never set the version anywhere else.

```bash
git commit -am "chore: Version <version>" -m "Co-Authored-By: <the attribution line this session requires>"
```

### 4. Tag

```bash
git tag <version>
```

### 5. Push main and the tag

```bash
git push origin main <version>
```

If the push fails, stop and report. Do not archive.

### 6. Archive

```bash
xcodegen generate
git status --short
```

Stop if `git status --short` prints anything.

Run the archive in the background. It takes several minutes:

```bash
xcodebuild archive \
  -project SpotGolf.xcodeproj \
  -scheme "SpotGolf App Store" \
  -destination "generic/platform=iOS" \
  -archivePath build/SpotGolf-<version>.xcarchive \
  -allowProvisioningUpdates \
  > build/archive-<version>.log 2>&1
```

- Always use the `SpotGolf App Store` scheme. Its `AppStore` configuration leaves Developer Mode out.
- Never skip the "Set the build instant" build script.
- Stop if the log does not end with `** ARCHIVE SUCCEEDED **`. Report the errors from the log.

Check the archive:

```bash
A=build/SpotGolf-<version>.xcarchive
/usr/libexec/PlistBuddy -c 'Print :ApplicationProperties' "$A/Info.plist"
/usr/libexec/PlistBuddy -c 'Print :SpotGolfBuildInstant' "$A/Products/Applications/SpotGolf.app/Info.plist"
ls "$A/Products/Applications/SpotGolf.app/Watch/SpotGolfWatch.app"
```

Stop if any of these is wrong:

| Check | Expected |
|---|---|
| `CFBundleShortVersionString` and `CFBundleVersion` | `<version>` |
| `SpotGolfBuildInstant` | The time of this build |
| Watch app | Present |

Remember the archive's `Team` value as `<team>`.

### 7. Upload to App Store Connect

Each version can be uploaded only once. If an upload for this version already succeeded, stop. To upload again, release a new PATCH version.

Write the export options to the session's scratchpad directory, not the repository:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>method</key>
	<string>app-store-connect</string>
	<key>destination</key>
	<string>upload</string>
	<key>teamID</key>
	<string><team></string>
</dict>
</plist>
```

Run the upload in the background:

```bash
xcodebuild -exportArchive \
  -archivePath build/SpotGolf-<version>.xcarchive \
  -exportOptionsPlist <scratchpad>/ExportOptions-AppStore.plist \
  -exportPath build/export-<version> \
  -allowProvisioningUpdates \
  > build/upload-<version>.log 2>&1
```

- Stop if the log does not end with `** EXPORT SUCCEEDED **`. Report the errors from the log.
- If the upload is blocked by the sandbox, report the blocked host per the sandbox rule.

### 8. Report

| Item | Value |
|---|---|
| Version and tag | `<version>` |
| Version commit | hash |
| Build instant | from step 6 |
| Archive | `build/SpotGolf-<version>.xcarchive` |
| Upload | Succeeded or failed |

Nothing else.
