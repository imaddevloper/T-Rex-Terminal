# T-Rex Terminal — Phone Workflow (Phase 0)

Goal: build and test T-Rex Terminal while working from an Android phone only.

**Principle:** the phone edits, reviews and starts builds. GitHub's servers compile.

## 1. What fits where

| Task | On the phone (Termux or browser) | Needs cloud (GitHub Actions) | Needs a desktop |
|---|---|---|---|
| Read, search and edit source | Yes | | |
| Git: clone, branch, commit, push | Yes | | |
| Run terminal-emulator unit logic | Not practical | Yes (`./gradlew test`) | |
| Compile APK (Gradle, SDK 36, NDK 29, JDK 17) | **Not recommended.** The toolchain is large and upstream does not target on-device builds. **[I]** | **Yes** | Optional |
| Download built APK and install it | Yes | | |
| Verify APK checksum | Yes (`sha256sum`) | | |
| Generate a release signing key | Possible, but back it up carefully | | |
| Build a custom bootstrap and package repo (only if Phase 2 chooses option A) | No | Needs a large build host | Likely |
| Android Studio / emulator | No | | Yes |

## 2. What you need

1. A **GitHub account** (free is enough for public repos; private repos have limited free Actions minutes). *Not yet confirmed that you have one.*
2. Roughly **1.5 GB free storage** on the phone for a clone, APKs and tools (estimate). **[I]**
3. One working **Termux** (or any terminal with git) if you want to use the command line. The GitHub website and mobile app are enough for the first build.
4. A stable connection. The CI run downloads Gradle, dependencies and the bootstrap ZIPs itself.

Never paste passwords, tokens or signing keys into chat. Use GitHub's own login and its **Secrets** screen for anything sensitive.

## 3. Phase 1 build plan (unmodified baseline)

Nothing in the source is changed. The aim is a debug APK from the exact upstream code.

**Step A — Create the repository (browser or GitHub app).**
Create an empty repository, for example `T-Rex-Terminal`. Keep the default branch `main` or `master` (upstream workflows trigger on `master`, so using `master` avoids touching them).

**Step B — Get the prepared repository onto GitHub.**
The file `T-Rex-Terminal.bundle` (provided in chat) is a Git bundle that already contains the three branches: `upstream` (untouched import), `master` (same commit, the reference baseline) and `trex` (development branch with the docs). In Termux:
```bash
pkg install git
cd ~/storage/downloads   # where you saved the bundle; run termux-setup-storage first if this folder is missing
git clone T-Rex-Terminal.bundle T-Rex-Terminal
cd T-Rex-Terminal
git branch upstream origin/upstream
git branch master origin/master
git remote set-url origin https://github.com/<your-user>/T-Rex-Terminal.git
git push origin upstream master trex
```
Expected: three branches appear on GitHub. If the push asks for credentials, use your GitHub username and a personal access token as the password (create it in GitHub settings; never share it). To verify the import, `git log --oneline upstream` should show one commit starting with `467d3ac`.

**Step C — Run the existing workflow.**
Pushing `master` triggers **Build** (`debug_build.yml`) automatically; pushing `upstream` or `trex` does not. Open the repository's **Actions** tab and watch it. Expected duration: several minutes per variant (estimate). **[I]**

**Step D — Download and verify.**
From the finished run, download the artifact for your phone's ABI (most modern phones: `arm64-v8a`) and the `sha256sums` artifact. In Termux:
```bash
cd ~/storage/downloads
sha256sum -c *_sha256sums --ignore-missing
```
Expected output: `OK` for each APK you downloaded.

**Step E — Install and smoke-test.**
Install the APK, open it, wait for the first-run bootstrap to finish, then run the smoke tests in the master prompt §6.2 (shell starts, `pkg update`, `echo $PREFIX`, resize/rotate, multiple sessions, copy/paste, `exit`).

> Caution: this baseline still uses the `com.termux` ID. If you have a normal Termux installed, Android will refuse the install because the signatures differ. **Do not uninstall Termux until you have backed up `~/` and `$PREFIX` customizations** (uninstalling deletes its private data). Use a second device or an emulator if you need both.

**Step F — Record the result.**
Write the outcome (run link, APK checksum, pass/fail per smoke test) into `docs/BASELINE_BUILD_REPORT.md`. Only then move to Phase 2.

### Common failure causes (Phase 1)

| Symptom | Likely cause | Safe response |
|---|---|---|
| Actions run does not start | Workflow files missing (hidden `.github` not uploaded) or Actions disabled | Check `.github/workflows` exists in the repo; enable Actions in repository settings |
| Build fails while downloading bootstrap | Network or release asset unavailable, or checksum mismatch | Re-run once; if the checksum error repeats, stop and report the exact log line |
| `Could not resolve com.termux:termux-am-library` | JitPack outage | Re-run later; do not edit dependencies |
| `-Werror` native compile error | Different NDK than pinned | Confirm the runner uses the pinned NDK; do not upgrade it |
| APK will not install | Same ID as installed Termux, or wrong ABI | See caution above; try the universal APK |

## 4. Git workflow (proposed)

- `upstream` branch: the unmodified import only. Never edit it.
- `main` / `master`: working branch for T-Rex.
- Small topic branches per task (`phase1-baseline`, `phase2-identity`, …), merged after CI is green.
- Every commit message states what changed and why. Local patches are logged in `docs/PATCHES.md`.

## 5. What is not decided yet

- Which identity option in `BASELINE_AUDIT.md` §13 (A, B or C) T-Rex will follow. **This is blocking only for Phase 2, not for Phase 1.**
- Final application ID, signing approach, and package-repository hosting.
- Whether to add T-Rex-specific CI (a release workflow) or keep upstream's.

## 6. Verification status of this document

Written without network access or a working Android toolchain. The CI behavior comes from reading the workflow files; **no build has been run by me**, and time and storage numbers are estimates.
