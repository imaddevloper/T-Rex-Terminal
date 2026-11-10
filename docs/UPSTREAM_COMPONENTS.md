# T-Rex Terminal — Upstream Component Inventory (Phase 0)

Baseline: `termux/termux-app`, commit `8629e632fcb95da272221be327db653fb24befe9` (from ZIP comment), app version 0.118.0.
Archive SHA-256: `a025715aa1f7d0076be5b75c61346890330c4fa69e6d5e2e41b36d0c66d81dc6`.
**Local changes so far: none.** Every row below is currently **UPSTREAM — UNMODIFIED** unless noted.

Classes: UPSTREAM-UNMOD · UPSTREAM-PATCHED · T-REX-NEW · THIRD-PARTY · GENERATED · VENDOR/BINARY.
Verification: **[V]** read in archive · **[U]** not yet verified.

## 1. First-party modules (in this repository)

| Component | Class | Path | Upstream URL | Revision | License | Local changes | Reason for reuse | Update strategy | Risk |
|---|---|---|---|---|---|---|---|---|---|
| App shell | UPSTREAM-UNMOD (will become PATCHED in Phase 2) | `app/` | github.com/termux/termux-app | `8629e632…` | GPLv3 only [V] | none | Activity, service, installer, settings, file receiver | Track upstream tags; keep patches small | App ID, `sharedUserId` and permission strings live here |
| Terminal emulator | UPSTREAM-UNMOD | `terminal-emulator/` | same | same | Apache 2.0 [V] | none | VT parsing, screen buffer, PTY JNI | Rebase on upstream | `-Werror` native build; JNI names tied to `com.termux.terminal` |
| Terminal view | UPSTREAM-UNMOD | `terminal-view/` | same | same | Apache 2.0 [V] | none | Rendering, gestures, selection | Rebase on upstream | One file (`PopupWindowCompatGingerbread.java`) carries its own Apache header |
| Shared library | UPSTREAM-UNMOD | `termux-shared/` | same | same | Mixed, see §2 [V] | none | Constants, shell runners, file/socket utils | Rebase on upstream | `TermuxConstants.java` is the central identity file |

## 2. License map inside `termux-shared` [V]

| Path | License |
|---|---|
| Default | MIT |
| `src/main/java/com/termux/shared/termux/*` | GPLv3 only |
| `…/shared/termux/TermuxConstants.java`, `…/settings/properties/TermuxPropertyConstants.java` | MIT (exceptions within the GPL folder) |
| `src/main/java/com/termux/shared/file/filesystem/*` | GPLv2 only + Classpath exception (OpenJDK libcore code) |
| `src/main/java/com/termux/shared/shell/StreamGobbler.java` | Apache 2.0 (libsuperuser) |

## 3. Native code

| Component | Class | Path | License | Notes |
|---|---|---|---|---|
| PTY / subprocess JNI | UPSTREAM-UNMOD | `terminal-emulator/src/main/jni/termux.c`, `Android.mk` | Apache 2.0 (module default) [V] | Builds `libtermux` |
| Local socket JNI | UPSTREAM-UNMOD | `termux-shared/src/main/cpp/local-socket.cpp`, `Android.mk`, `Application.mk` | MIT (module default) [V] | |
| Bootstrap embed | UPSTREAM-UNMOD | `app/src/main/cpp/termux-bootstrap.c`, `termux-bootstrap-zip.S`, `Android.mk` | GPLv3 only (app default) [V] | `.incbin` of the per-ABI bootstrap ZIP; JNI name tied to `com.termux.app.TermuxInstaller` |

## 4. Vendor / binary items

| Item | Class | Where | Provenance | Status |
|---|---|---|---|---|
| Bootstrap ZIPs (aarch64, arm, i686, x86_64) | VENDOR/BINARY | **Not in the repo.** Downloaded during build into `app/src/main/cpp/bootstrap-<arch>.zip` | `github.com/termux/termux-packages/releases/download/bootstrap-2026.02.12-r1+apt.android-7/…`; SHA-256 pinned in `app/build.gradle` [V] | Checksums not re-verified [U]. Contents are built for prefix `/data/data/com.termux/files/usr` |
| Alternate bootstrap `apt-android-5` | VENDOR/BINARY | same mechanism, version `2022.04.28-r6+apt-android-5` | pinned in `app/build.gradle` [V] | Legacy variant; Phase 1 can ignore it |
| Debug keystore | VENDOR/BINARY | `app/testkey_untrusted.jks` | Upstream "untrusted" test key; passwords in `app/build.gradle` [V] | **Debug only. Never a release key.** |
| Gradle wrapper JAR | VENDOR/BINARY | `gradle/wrapper/gradle-wrapper.jar` | Validated by upstream workflow `gradle-wrapper-validation.yml` [V] | Re-validate checksum when importing |
| Screenshots, icon, sponsor logos | VENDOR/BINARY (branding) | `fastlane/metadata/.../images`, `art/`, `site/assets/` | Termux project assets | Branding and trademark review needed before release |

No `.so`, `.apk` or other prebuilt native binaries are checked in. [V]

## 5. Third-party dependencies (declared in Gradle) [V]

| Dependency | Version | Used by | Source | License |
|---|---|---|---|---|
| androidx.annotation | 1.9.0 | all modules | google() | [U] (expected Apache 2.0) |
| androidx.core | 1.13.1 | app, shared | google() | [U] |
| androidx.drawerlayout / preference / viewpager | 1.2.0 / 1.2.1 / 1.0.0 | app | google() | [U] |
| androidx.appcompat | 1.6.1 | shared | google() | [U] |
| androidx.window | 1.1.0 | shared | google() | [U] |
| com.google.android.material | 1.12.0 | app, shared | google() | [U] |
| com.google.guava:guava | 24.1-jre | app, shared | mavenCentral() | [U] |
| com.google.guava:listenablefuture | 9999.0-empty-to-avoid-conflict-with-guava | app | mavenCentral() | [U] |
| io.noties.markwon (core, ext-strikethrough, linkify, recycler) | 4.6.2 | app, shared | mavenCentral() | [U] |
| org.lsposed.hiddenapibypass | 6.1 | shared | mavenCentral() | [U] |
| commons-io | 2.5 (must stay ≤ 2.5 per upstream comment for Android < 8) | shared | mavenCentral() | [U] |
| com.termux:termux-am-library | v2.0.0 | shared | jitpack.io | [U] |
| com.android.tools:desugar_jdk_libs | 1.1.5 | app, shared | google() | [U] |
| junit / robolectric / androidx.test.ext:junit | 4.13.2 / 4.10 / 1.1.5 | tests only | mavenCentral() / google() | [U] |

All licenses above are expected to be permissive but are **not yet verified**; the verification is a Phase 0 follow-up that needs network access.

## 6. CI / tooling references (upstream)

| Item | Class | Path | Notes |
|---|---|---|---|
| GitHub Actions workflows (6) | UPSTREAM-UNMOD | `.github/workflows/` | Actions used: `actions/checkout@v6`, `setup-java@v5`, `upload-artifact@v6`. Pin by commit SHA before any release pipeline [U] |
| JitPack config | UPSTREAM-UNMOD | `jitpack.yml` | Only relevant if publishing the libraries; T-Rex does not need it initially |

## 7. Update strategy (proposed, to be confirmed)

1. Keep an `upstream` branch that tracks `termux/termux-app` exactly; never commit T-Rex changes there.
2. Keep T-Rex work on separate branches; record each local patch in `docs/PATCHES.md` with file, reason, and upstream reference.
3. Merge or rebase upstream tags only after the T-Rex build and tests pass on a scratch branch.
4. Re-run the license inventory when dependency versions change.
