# CI Build Optimizations for Android AArch64

This document outlines the modifications made to the GitHub Actions workflows to optimize build times by restricting the CI pipeline to only build the Android AArch64 (ARM64) APK.

## Objectives
1. **Save Build Time:** Disable all unnecessary OS builds (Windows, macOS, Linux, iOS, AppImage, Flatpak, Web, etc.).
2. **Save Runner Resources:** Optimize the Android build matrix to strictly target only the `aarch64` architecture (ignoring `armv7` and `x86_64`).
3. **Manual Trigger Only:** Prevent builds from running automatically on every push or pull request to save CI minutes.
4. **Fix 403 Forbidden Errors:** Prevent the workflow from crashing at the very end when trying to automatically create GitHub Releases.

## Modifications Made

### 1. Disabling Automatic Triggers
To prevent workflows from firing on every commit, the `push` and `pull_request` triggers were removed. Only the `workflow_dispatch` trigger (manual run) remains active.

**Affected Files:**
- `.github/workflows/flutter-ci.yml`
- `.github/workflows/ci.yml`
- `.github/workflows/wf-cliprdr-ci.yml`

*Example change:*
```yaml
on:
  workflow_dispatch:
  # push and pull_request triggers were removed
```

### 2. Disabling Unnecessary Jobs
In `.github/workflows/flutter-build.yml` and `.github/workflows/ci.yml`, the `if: false` condition was injected into all jobs that are not relevant to the Android AArch64 build.

**Jobs Disabled (`if: false` added):**
- `build-RustDeskTempTopMostWindow`
- `build-for-windows-flutter`
- `build-for-windows-sciter`
- `build-rustdesk-ios`
- `build-for-macOS`
- `publish_unsigned`
- `build-rustdesk-android-universal`
- `build-rustdesk-linux`
- `build-rustdesk-linux-sciter`
- `build-appimage`
- `build-flatpak`
- `build-rustdesk-web`
- Primary `build` job in `ci.yml`

### 3. Trimming the Android Build Matrix
In `.github/workflows/flutter-build.yml`, the `build-rustdesk-android` job's matrix was modified. The `armv7` and `x86_64` targets were removed, leaving only `aarch64-linux-android`.

*Modified Matrix:*
```yaml
strategy:
  fail-fast: false
  matrix:
    job:
      - {
          arch: aarch64,
          target: aarch64-linux-android,
          os: ubuntu-24.04,
          reltype: release,
          suffix: "",
        }
```

### 4. Bypassing 403 Release Errors
Forks frequently encounter `403 Forbidden` errors during the final steps of a GitHub Action if the workflow attempts to use `softprops/action-gh-release` to create a GitHub Release, as standard `GITHUB_TOKEN` permissions for forks do not permit release creation.

To fix this, the automated "Publish signed apk package" and "Publish unsigned apk package" release steps in `.github/workflows/flutter-build.yml` were commented out. 

*Note: The built APK is still retained! It is uploaded and available for download as a Workflow Artifact via the `actions/upload-artifact` step.*

## How to Run the Build
Since automatic triggers are disabled, you must trigger the workflow manually:
1. Go to the **Actions** tab in the GitHub repository.
2. Select **Full Flutter CI** from the left sidebar.
3. Click the **Run workflow** dropdown on the right side.
4. Select the `master` branch and click **Run workflow**.

Alternatively, via the GitHub CLI:
```bash
gh workflow run flutter-ci.yml --ref master
```

## Git Remote Setup (Local)
To streamline working with the fork, the local Git remotes were remapped so that the fork acts as the default target.

```bash
git remote rename origin upstream   # Original rustdesk/rustdesk becomes upstream
git remote rename fork origin       # git-hub-tig/rustdesk becomes origin
git branch --set-upstream-to=origin/master master
```
