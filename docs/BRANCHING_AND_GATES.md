# T-Rex Terminal — Branches and Phase Gates

## 1. Branches

| Branch | Contents | Rules |
|---|---|---|
| `upstream` | Exact import of `termux/termux-app` @ `8629e632fcb95da272221be327db653fb24befe9` | **Never edited.** Only fast-forwarded from upstream by a dedicated, reviewed update. |
| `master` | Same commit as `upstream` for now | The **reference baseline**. Upstream's CI triggers on `master`, so this is the branch that produces the unmodified reference APK. Stays unmodified through Phase 1. |
| `trex` | `master` + T-Rex documentation (`docs/`) | **Development branch.** All T-Rex work happens here, in small commits. |
| `spike/*` | Experiments (for example `spike/custom-prefix`) | Throw-away; merged only after the gate in `IDENTITY_PLAN.md`. |

Upstream's workflows build on pushes to `master` and `github-releases/**`, and on pull requests to `master`. A push to `trex` does **not** build by itself; open a pull request from `trex` into `master` to trigger CI, or add a `trex` trigger in a separate, reviewed CI change (Phase 2).

## 2. Phase gates

A phase is finished only when every line below is true and written down in `docs/PHASE_REPORTS/phase-N.md`.

1. The change set is committed on `trex` (or merged from a spike) with a clear message.
2. **A CI build ran and succeeded** (link recorded). No claim of success without a run.
3. **The APK was installed on the owner's phone** and the smoke tests were run (shell opens, input/output, resize, multiple sessions, `pkg update`, copy/paste, `exit`). Results recorded as pass/fail per test.
4. Regressions against the baseline APK are listed (compare behavior with the `master` build).
5. Known issues and limitations are listed.
6. The owner approves moving on.

If any step fails, the next phase does not start.

## 3. Phase list (condensed)

| Phase | Outcome | Identity |
|---|---|---|
| 1 | Unmodified baseline APK built by CI, installed, smoke-tested (branch `master`) | `com.termux` (unchanged) |
| 2 | T-Rex branding and settings scaffolding on `trex`, CI trigger for `trex` | `com.termux` (unchanged) |
| 3 | Core reliability work (sessions, input, lifecycle, tests) | unchanged |
| 4 | New T-Rex interface and UX | unchanged |
| 5 | Developer tools and package UX on top of the real APT | unchanged |
| Spike | Custom prefix/bootstrap experiment (`IDENTITY_PLAN.md` §3) | separate branch |
| 6+ | Identity switch if the spike passes; optional PRoot environments; security and release hardening | decided by the gate |

## 4. Coding rules that protect the later identity switch

- No literal `com.termux` or `/data/data/com.termux` in new code or strings. Use `TermuxConstants`, `BuildConfig`, manifest placeholders or `Context`.
- Do not rewrite the terminal engine (`terminal-emulator`, PTY JNI) or the installer/bootstrap flow to implement UI features. Extend around them.
- Keep patches to upstream files small and log each one in `docs/PATCHES.md` (file, reason, upstream reference).
- No signing keys, tokens or passwords in Git.
