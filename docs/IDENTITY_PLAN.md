# T-Rex Terminal — Identity, Bootstrap and Package Plan

**Status:** plan only. `applicationId`, bootstrap, package repositories and signing are **unchanged** in this repository until the gate in §6 is passed.
Written without network access; statements about Termux's build tooling come from the upstream README and from general knowledge and are marked **[I]** (inferred, to be verified in the spike).

## 1. The problem

The app stores everything under its private data directory. In this source:

- `TermuxConstants.TERMUX_PACKAGE_NAME = "com.termux"` and every path derives from it: `/data/data/com.termux/files/usr` (prefix, `$PREFIX`), `…/files/home` (`$HOME`), `…/usr/etc`, `…/usr/tmp` and so on. **[V]**
- The bootstrap ZIP embedded in the APK is downloaded from `termux-packages` releases and was **compiled for that prefix**. The upstream README says a different package name needs a bootstrap recompiled for it. **[V]**
- APT repositories (`packages.termux.dev` and mirrors) serve `.deb` files built for the same prefix. Binaries and scripts carry the path internally (shebangs, RPATH, config files, dpkg/apt state). **[I]**

Consequence: **if T-Rex gets a new `applicationId`, its data directory becomes `/data/data/<new id>/…`, and the official bootstrap and official packages will not work there.** Changing the ID alone gives a broken app.

## 2. Options

| | A. Keep `com.termux` ID | B. New ID + own bootstrap and package repo | C. New ID + official packages via path emulation |
|---|---|---|---|
| Official bootstrap/APT works | Yes | No (own builds only) | Yes, in theory **[I]** |
| Installs beside Termux | **No** (same ID; different signature is rejected) | Yes | Yes |
| Real independent identity | Weak | Strong | Strong |
| Engineering effort | Low | Very high, ongoing | High and risky, unproven |
| Runtime cost | None | None | Slower exec/IO under emulation **[I]** |
| Phone-feasible | Yes | Needs a build host or CI for packages | Needs research on a PC/CI |
| Main risk | User must uninstall Termux; shared name/ID confusion | Package build and hosting upkeep, signing of the apt repo, security updates | May fail under Android exec/ptrace limits; poor reliability |

Notes:
- **A** is how some other Termux forks operate **[I]**. It lets you ship UI, UX and features quickly with the real APT ecosystem.
- **B** needs a fork of `termux-packages` built with the new app package name (that project has a documented way to build for a custom name, per upstream README links **[V]**), a signed APT repository you host, and CI capacity. Expect to maintain only a **curated subset** of packages, not the full Termux catalogue.
- **C** (for example a PRoot-style remap of `/data/data/com.termux` to the new data dir) needs a small custom-prefix PRoot first, so it still requires some option-B build work. Treat it as a research spike, not an MVP path.

## 3. Recommendation (to be confirmed by you)

**Two-stage approach, with an evidence gate in between.**

**Stage 1 — Build the product on top of identity A (development identity).**
- Phases 1–5 of the roadmap (baseline, core reliability, T-Rex interface, developer tools, package UX) are done on the `trex` branch while the app still uses `com.termux` internally.
- This keeps the real engine, PTY, bootstrap and APT working, so every phase can be built, installed and tested.
- **Hard rule for all new T-Rex code:** never write `com.termux` or `/data/data/com.termux` as a literal. Use `TermuxConstants`, `BuildConfig`, manifest placeholders or `Context` paths. This makes the later identity switch mechanical rather than a search-and-pray exercise.
- Visible branding (name, icon, strings) can change in Stage 1 without touching the ID.
- Limitation to state plainly to testers: it cannot be installed beside regular Termux.

**Stage 2 — Identity spike, then decide.**
Run a time-boxed spike on a separate branch (`spike/custom-prefix`):
1. Fork `termux-packages` on GitHub; build a **minimal bootstrap** for the candidate ID (suggested `com.trex.terminal`, provisional).
2. Build a minimal APK with that ID, new constants and the spike bootstrap.
3. Host a tiny APT repo (GitHub Pages or Releases) signed with a **new** key, with only a handful of packages.

Success criteria (all must hold, measured on a real phone):
- App starts, shell opens, `echo $PREFIX` shows the new path.
- `apt update` works against **our** repo with signature verification on.
- At least these install and run: `git`, `python`, `openssh`, `nano`, `curl`.
- No hard-coded `com.termux` remains in the installed tree (checked with `grep -r`).
- A package update can be published and installed end to end.

If it passes, adopt option B for the release. If it fails or the maintenance cost is unacceptable, ship under A and keep B as a later milestone. Either way the decision is recorded here with the evidence.

## 4. Open decisions (yours)

1. Final application ID (provisional `com.trex.terminal`; check availability of the ID and name).
2. Whether T-Rex must coexist with regular Termux at launch (forces B or C) or may require uninstalling it (A is enough).
3. `sharedUserId`: keep it (needed by Termux plugins that share the user ID) or drop it (cleaner, but breaks plugin compatibility). It cannot be changed on an installed app without a reinstall. **[I]**
4. Where the APT repo and signing key live, and who holds the key. Never paste keys into chat.
5. Distribution channel: GitHub Releases (works with targetSdk 28) versus stores that enforce newer target levels **[I]**.

## 5. Things that must change together later (checklist for the identity switch)

- `app/build.gradle`: `namespace`/`applicationId`, `manifestPlaceholders.TERMUX_PACKAGE_NAME`, display names.
- `TermuxConstants.java`: package name and derived paths (follow its javadoc).
- JNI symbol names that embed the app package: `Java_com_termux_app_TermuxInstaller_getZip` and anything else under `com.termux.app` (`termux.c` symbols are in `com.termux.terminal`, a library namespace that need not change).
- Manifest: `sharedUserId`, custom permission, provider authorities.
- Bootstrap ZIPs and the `downloadBootstraps` task URLs/checksums.
- APT sources list, GPG keyring and repo URLs shipped in the bootstrap.
- Signing key (new, kept out of Git).

## 6. Gate

No change to any item in §5 is made until the owner approves this plan (or a revision of it). Phase 1 (unmodified baseline) and Stage 1 feature work do not require that approval, only the rule in §3.
