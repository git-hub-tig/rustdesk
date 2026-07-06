# Custom Android Client for Family Remote Support

This document outlines the specific modifications made to this fork of RustDesk. The goal of this fork is to create a highly streamlined, zero-setup Android client tailored for remotely supporting family members (specifically in mainland China), bypassing public server mobile restrictions.

## 1. Static ID & Password
To eliminate the need for the remote user to read out their connection credentials, the Android client has been hardcoded to use a static ID and temporary password.

* **Files Modified:** 
  * `libs/hbb_common/src/config.rs`
  * `libs/hbb_common/src/password_security.rs`
* **Details:** 
  * `get_auto_id()` and `Config::load()` conditionally force the ID to `44445555` on Android.
  * `get_auto_password()` conditionally forces the temporary session password to `666666` on Android.
  * *Note:* Desktop builds remain unaffected and generate standard random IDs/passwords to prevent network conflicts.

## 2. Geoblock / Mobile Restriction Bypass
The official RustDesk public rendezvous servers (`hbbs`) restrict connections to mobile devices originating from mainland China IPs. The server identifies mobile devices by checking if their ID is a 10-digit number (1B-2B range) and if their UUID is a cryptographic public key. 

* **Files Modified:**
  * `libs/hbb_common/src/lib.rs`
* **Details:**
  * Since our static ID (`44445555`) already looks like a PC ID, we only needed to spoof the UUID.
  * Modified `get_uuid()` for Android to return a static, desktop-formatted ASCII string (`"fake-desktop-uuid-44445555"`). The rendezvous server now classifies the Android phone as a standard PC, bypassing the "Access to mobile devices is restricted" error entirely.

## 3. UI Simplification & Auto-Start
The Android client UI was dramatically simplified to prevent confusion for non-technical users, and the service initialization was automated.

* **Files Modified:**
  * `flutter/lib/mobile/pages/home_page.dart`
  * `flutter/lib/models/server_model.dart`
  * `flutter/lib/mobile/pages/server_page.dart`
* **Details:**
  * **Removed Tabs:** The "Connection" (remote control) and "Chat" tabs were removed from the bottom navigation bar. Only "Share screen" and "Settings" remain.
  * **Auto-Open:** Because "Share screen" is now the first tab, it is the default view when the app opens.
  * **Auto-Start:** Added `checkAndroidPermission()` to the `HomePage`'s `initState`. When the app launches, it immediately attempts to start the background service.
  * **Streamlined Permissions:** 
    * Removed the custom "Scam Warning" popup that interrupted the service start.
    * When the app opens, it instantly prompts for "Screen Capture" (MediaProjection).
    * If "Input Control" is not granted, it immediately opens the Android Accessibility Settings so the user can easily toggle "RustDesk Input" to ON.

## 4. GitHub Actions CI/CD Pipeline Fixes
To ensure the custom APK builds successfully in the cloud without requiring a local Flutter/Android NDK environment.

* **Files Modified:**
  * `.github/workflows/flutter-build.yml`
  * `.github/workflows/flutter-ci.yml`
  * `.gitmodules`
* **Details:**
  * Inlined the `hbb_common` submodule directly into the repository tree so GitHub Actions picks up the Rust-level ID/Password/UUID modifications.
  * Granted `permissions: contents: write` to the workflow so it can publish releases.
  * Forced `upload-artifact: true` and configured the workflow to output and publish the `*unsigned.apk` directly to GitHub Releases, removing the strict dependency on private Android signing keys.

## How to Build
To generate a new APK, navigate to the **Actions** tab on GitHub, select the **Full Flutter CI** workflow, and click **Run workflow** on the `master` branch. The resulting `rustdesk-*-aarch64-unsigned.apk` will be available in the workflow's Artifacts and on the Releases page.