# T-Rex Terminal — Baseline Audit (Phase 0)

**Status:** Phase 0 (discovery only). No source file was modified. The ZIP was extracted to a scratch directory and read.
**Legend:** **[V]** verified from a file in the archive · **[I]** inferred · **[U]** unverified (needs network or a real build).

---

## 1. Upstream source and revision

| Item | Value |
|---|---|
| Archive | `termux-app-master.zip` |
| Size | 1,224,861 bytes; 346 files in 162 directories (508 archive entries) **[V]** |
| SHA-256 | `a025715aa1f7d0076be5b75c61346890330c4fa69e6d5e2e41b36d0c66d81dc6` **[V]** |
| Embedded commit (ZIP comment) | `8629e632fcb95da272221be327db653fb24befe9` **[V]** (GitHub writes this into archive comments) |
| Entry timestamps | 2026-09-26 05:11 (likely the commit date) **[I]** |
| Git metadata | None. No `.git` directory, so `git log`, `git remote` and `git describe` cannot run. **[V]** |
| App version in source | `versionName "0.118.0"`, `versionCode 118` **[V]** |
| Canonical upstream | `https://github.com/termux/termux-app` (named in `README.md` and `docs/en/index.md`) **[V]**. Not checked against the live site. **[U]** |
| Branch | Folder name says `master`. Not proven, and "master" is not claimed to be the latest stable release. |

**Open item:** confirm that `8629e632…` exists on `termux/termux-app` and note what commit upstream is at today. This needs network access (see `PHONE_WORKFLOW.md`).

## 2. Repository structure

Four Gradle modules, declared in `settings.gradle` **[V]**: `:app`, `:termux-shared`, `:terminal-emulator`, `:terminal-view`.

| Module | Type | Java files | Native | Role |
|---|---|---|---|---|
| `app` | application | 35 | `src/main/cpp` (bootstrap embed) | `TermuxActivity`, `TermuxService`, `TermuxInstaller`, `RunCommandService`, settings fragments, file receiver, documents provider |
| `termux-shared` | library | 121 | `src/main/cpp/local-socket.cpp` | Constants, shell/session runners, file utils, reporting, local sockets |
| `terminal-emulator` | library | 37 | `src/main/jni/termux.c` | `TerminalEmulator`, `TerminalBuffer`, `TerminalSession`, PTY JNI |
| `terminal-view` | library | 8 | none | `TerminalView`, `TerminalRenderer`, gestures, text selection |

No Kotlin files exist in the tree. **[V]** Other top-level items: `art/`, `docs/en/index.md` (a stub), `fastlane/metadata/android/en-US` (store text and screenshots), `site/assets` (sponsor logos), `.github/` (workflows and issue templates).

## 3. Build environment requirements

| Item | Value | Source |
|---|---|---|
| Gradle (wrapper) | 9.2.1 | `gradle-wrapper.properties` **[V]** |
| Android Gradle Plugin | 8.13.2 | root `build.gradle` **[V]** |
| JDK | 17 (CI uses Temurin 17, `jitpack.yml` uses openjdk17) | workflows **[V]** |
| compileSdk / targetSdk / minSdk | 36 / **28** / 21 | `gradle.properties` **[V]** |
| NDK | 29.0.14206865 (`ndk-build`, **no CMake**) | `gradle.properties`, `Android.mk` **[V]** |
| Java source/target | 1.8, with core-library desugaring (`desugar_jdk_libs:1.1.5`) | module build files **[V]** |
| Gradle JVM heap | `-Xmx2048M` | `gradle.properties` **[V]** |
| Extra repositories | `google()`, `mavenCentral()`, `https://jitpack.io` | root `build.gradle` **[V]** |

Do not upgrade any of these in Phase 1. Build the combination upstream CI uses.

**Network is required at build time.** The `downloadBootstraps` task (hooked before Java compile) fetches bootstrap ZIPs from `github.com/termux/termux-packages/releases` and checks each against a pinned SHA-256 **[V]**. The Gradle wrapper itself is fetched from `services.gradle.org`, and `com.termux:termux-am-library:v2.0.0` comes via JitPack **[V]**.

## 4. Build variants

- **No product flavors.** The "package variant" is chosen by the environment variable `TERMUX_PACKAGE_VARIANT`: `apt-android-7` (default) or `apt-android-5` **[V]**. The variant is baked into `BuildConfig` and must match `TermuxBootstrap.PackageVariant`, or the app crashes at startup (comment in `app/build.gradle`).
- **Build types:** `debug` (signed with the repo's test key) and `release` (R8 minify on, `-dontobfuscate`, shrinkResources off, **no signing config**, so it produces an unsigned APK) **[V]**.
- **ABI splits:** x86, x86_64, armeabi-v7a, arm64-v8a, plus a universal APK. On by default for debug, off for release **[V]**.
- **Output names:** `termux-app_<tag>_<abi|universal>.apk` **[V]**.
- Pinned bootstrap for `apt-android-7` is `2026.02.12-r1+apt.android-7` (four architectures, SHA-256s in `app/build.gradle`) **[V]**. The checksums were not independently re-verified. **[U]**

## 5. Application ID and package identifiers

| Identifier | Value | Where |
|---|---|---|
| `namespace` | `com.termux` | `app/build.gradle` **[V]** |
| `applicationId` | not set explicitly, so AGP defaults to the namespace, `com.termux` | `app/build.gradle` **[V]** |
| `TERMUX_PACKAGE_NAME` | `"com.termux"` | `TermuxConstants.java` **[V]** |
| Manifest placeholder | `TERMUX_PACKAGE_NAME = com.termux` | `app/build.gradle` **[V]** |
| `sharedUserId` | `${TERMUX_PACKAGE_NAME}` | `AndroidManifest.xml` **[V]** |
| Custom permission | `${TERMUX_PACKAGE_NAME}.permission.RUN_COMMAND` (dangerous) | manifest **[V]** |
| Provider authorities | `${…}.documents`, `${…}.files` | manifest **[V]** |
| Data dir / prefix | `/data/data/com.termux`, `…/files/usr`, `…/files/home` | `TermuxConstants.java` **[V]** |
| JNI symbols | `Java_com_termux_terminal_JNI_*`, `Java_com_termux_app_TermuxInstaller_getZip`, `Java_com_termux_shared_net_socket_local_LocalSocketManager_*` | C/C++ sources **[V]** |

Rough scale of a rename: **218 files** contain the string `com.termux`, and 26 lines in `app/src/main/res` plus the manifest. Most are Java `package` lines, which do not have to change (see §13).

## 6. Native ABI support

x86, x86_64, armeabi-v7a, arm64-v8a **[V]**. Native pieces:
1. `terminal-emulator/src/main/jni/termux.c` builds `libtermux` (PTY and subprocess: `createSubprocess`, `setPtyWindowSize`, `waitFor`, `close`).
2. `termux-shared/src/main/cpp/local-socket.cpp` (Unix local sockets).
3. `app/src/main/cpp` builds `libtermux-bootstrap`, which embeds the bootstrap ZIP into the library with `.incbin` per architecture.

C flags: `-std=c11 -Wall -Wextra -Werror -Os`. `-Werror` means a newer NDK can fail the build on a new warning, which is one more reason not to upgrade casually. **[I]**

## 7. Terminal implementation overview

- **Emulator:** `terminal-emulator` is pure Java, with state in `TerminalEmulator` and `TerminalBuffer` and one JNI file for the PTY. It also contains `TerminalSixel`/`TerminalBitmap`/`ITermImage` (Sixel graphics), `KeyHandler`, and `WcWidth` (character widths). **[V]**
- **View:** `terminal-view` provides `TerminalView` (an Android `View`, not Compose), `TerminalRenderer`, gesture and selection code. **[V]**
- **Sessions:** `TerminalSession` runs the child process on a PTY. `TermuxService` (foreground service) keeps sessions alive. `TermuxActivity` hosts the UI. `RunCommandService` and `TermuxSession` live in `app` and `termux-shared`. **[V]**
- The UI is classic Views plus AndroidX Preference and Material, with `supportsRtl="false"` in the manifest. **[V]**

## 8. Bootstrap and package system overview

- The app ships a minimal **bootstrap** ZIP (shell, apt, coreutils and so on), unpacked on first launch by `TermuxInstaller` into the private prefix `…/files/usr`. **[V]** (installer class present; unpack logic not traced in this phase.)
- Package management is **apt**, using repositories hosted by the Termux project. No package manager code lives in this repo. **[V]** (README: package recipes are in `termux/termux-packages`.)
- Binaries in the bootstrap and in Termux repositories are compiled for the prefix `/data/data/com.termux/files/usr`. The upstream README (Forking section) states that a different package name requires **recompiling the bootstrap** for that name. **[V]**

## 9. Permission and storage model

Declared permissions **[V]**: INTERNET, ACCESS_NETWORK_STATE, READ/WRITE_EXTERNAL_STORAGE, MANAGE_EXTERNAL_STORAGE, WAKE_LOCK, VIBRATE, FOREGROUND_SERVICE, REQUEST_IGNORE_BATTERY_OPTIMIZATIONS, SYSTEM_ALERT_WINDOW, READ_LOGS, DUMP, WRITE_SECURE_SETTINGS, REQUEST_INSTALL_PACKAGES, RECEIVE_BOOT_COMPLETED, PACKAGE_USAGE_STATS, SET_ALARM.

Notes:
- Several are privileged or special-access permissions (READ_LOGS, DUMP, WRITE_SECURE_SETTINGS, PACKAGE_USAGE_STATS) that a normal install cannot hold unless granted via adb. They exist for power-user and plugin workflows. Each must be justified or removed in a later security phase.
- `android:allowBackup="false"` and `installLocation="internalOnly"`; `requestLegacyExternalStorage="true"`. **[V]**
- **Exported components:** `TermuxActivity`, `SettingsActivity`, `FileShareReceiverActivity` / `FileViewReceiverActivity` (aliases), `TermuxDocumentsProvider` (guarded by `MANAGE_DOCUMENTS`), the `…files` content provider and `RunCommandService` (both guarded by the `RUN_COMMAND` permission). **[V]**
- `targetSdkVersion=28` is a deliberate upstream choice. The reason is not written in this repo; I believe it relates to executing binaries from app data on Android 10+ and to Play Store target-level policy. **[I]** Confirm in Termux documentation before relying on it. A low target level very likely rules out Google Play distribution. **[I]**

## 10. Existing tests and CI

- **Unit tests:** 19 files in `terminal-emulator/src/test` (parser, buffer, history, key handling, Unicode, resize and so on) and 2 in `app/src/test` (Robolectric 4.10). One further test-path file exists in `termux-shared`; not examined. **[V]**
- **Not run.** This environment has no Android SDK and no network, so no test or build was executed. **[U]**
- **Workflows:** `debug_build.yml` (matrix over both package variants; `assembleDebug`; checks that all five APKs exist; writes `sha256sums`; uploads artifacts), `run_tests.yml` (`./gradlew test`), `gradle-wrapper-validation.yml`, `dependency-submission.yml`, `attach_debug_apks_to_release.yml`, `trigger_library_builds_on_jitpack.yml`. **[V]**
- The debug workflow builds on push and PR to `master`. A fork gets this CI for free once pushed to GitHub, which suits the phone-first plan.

## 11. Licenses and third-party notices

| Scope | License | Source |
|---|---|---|
| Repository default | GPLv3 only | `LICENSE.md` |
| `terminal-emulator`, `terminal-view` | Apache 2.0 (code from jackpal Android-Terminal-Emulator) | `LICENSE.md` |
| `termux-shared` default | MIT | `termux-shared/LICENSE.md` |
| `termux-shared/.../shared/termux/*` | GPLv3 only (except `TermuxConstants.java` and `TermuxPropertyConstants.java`, MIT) | same |
| `termux-shared/.../shared/file/filesystem/*` | GPLv2 only with Classpath exception (OpenJDK code) | same |
| `termux-shared/.../shell/StreamGobbler.java` | Apache 2.0 (libsuperuser) | same |

All **[V]**. Gaps: only two `LICENSE.md` files exist, and they **link** to licenses rather than including full texts. There is no `NOTICE` or third-party list. Declared dependencies (androidx, material, guava 24.1-jre, markwon 4.6.2, commons-io 2.5, hiddenapibypass 6.1, termux-am-library v2.0.0, desugar_jdk_libs 1.1.5, junit, robolectric) have **not** had their licenses checked. **[U]** The bootstrap binaries and apt packages are separate works with their own licenses. **[U]** Redistribution implies GPLv3 obligations (corresponding source, license text, notices).

Trademark and branding: `art/`, `fastlane/…/icon.png`, screenshots and `site/assets` carry Termux branding and sponsor logos. Reuse of the name "Termux", its icon and sponsor material in a fork needs review before any release. **[I]**

## 12. Known risks and blockers

1. **Package-name change vs. bootstrap/package ecosystem (critical).** Official Termux bootstrap and apt packages are built for `/data/data/com.termux/...`. A new application ID needs a rebuilt bootstrap and, realistically, a self-hosted package repository, which is heavy infrastructure and not phone-feasible. Keeping `com.termux` avoids that but collides with a normal Termux install (same ID, different signature). **This decision is the owner's and is not made here.**
2. **`sharedUserId`.** Set in the manifest. It cannot be added or removed without a clean install, and it is deprecated by Android. **[I]** Decide before the first T-Rex release.
3. **targetSdk 28** limits store distribution and brings Android-version compatibility questions. **[I]**
4. **Build-time downloads** (bootstrap, JitPack, Gradle). Supply-chain and availability exposure. **[V]**
5. **Debug signing key is committed** (`app/testkey_untrusted.jks`, passwords in `app/build.gradle`). Upstream names it "untrusted", so it is fine for debug builds only. It must never be used for T-Rex releases. Release has no signing config. **[V]**
6. **Broad permission set and exported components** (§9). **[V]**
7. **No RTL support** (`supportsRtl="false"`) and the stock UI is English-only in `docs`/`fastlane`. Relevant to the localization requirements. **[V]**
8. **Unverified:** upstream commit currency, checksums, dependency licenses, and any successful build. **[U]**
9. The README says upstream is **looking for maintainers** and notes instability on Android 12+ (phantom-process killing). Plan for a fork that carries its own maintenance. **[V]**

## 13. Recommended minimal fork plan (decision-gated)

**Phase 1 — unmodified baseline (no identity changes).**
1. Owner creates a GitHub repository (suggested `T-Rex-Terminal`) and uploads this ZIP's contents as the first commit, noting the upstream commit `8629e632…` in the commit message and in a new `UPSTREAM.md`.
2. GitHub Actions runs the **existing** `debug_build.yml` unchanged.
3. Download the debug APK (arm64-v8a for most phones), verify it against the generated `sha256sums`, install it, and run the smoke tests from the master prompt §6.2.
4. Success is a working, unmodified Termux debug build you built yourself. Nothing is renamed.

**Phase 2 — decide identity, then fork.** Choose among:
- **A.** New application ID plus own bootstrap/repo (full independence; large effort).
- **B.** Keep `com.termux` identity internally for now, rebrand only user-visible text (cannot coexist with an installed Termux; needs your own signing key).
- **C.** Hybrid: new ID with a PRoot userland as the "environment", and the host prefix kept minimal.

Library namespaces (`com.termux.terminal`, `com.termux.view`, `com.termux.shared`) can stay as they are in every option. What must change is the app ID, the manifest placeholders, `TermuxConstants`, the JNI symbol names tied to `com.termux.app`, and the bootstrap.

**Files not to touch before the Phase 2 decision:** `TermuxConstants.java`, `app/build.gradle` (identity and signing), `AndroidManifest.xml`, `app/src/main/cpp/*`, `terminal-emulator/src/main/jni/*`.

## 14. Items not completed in this phase

- Live check of upstream's current commit and release state.
- Any build, test or install.
- Dependency license review and bootstrap checksum verification.
- Tracing `TermuxInstaller` unpack logic and `TermuxConstants` fork instructions in detail.
- Whether the owner already has a GitHub account and repository.
