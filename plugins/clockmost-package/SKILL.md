---
name: clockmost-package
description: >
  Use when working with the clockmost Arch package system at git.clockmost.com/pkgs.
  Triggers: "add a package to clockmost", "publish/ship <tool> as an Arch package",
  "release a new version of <clockmost package>", "cut a clockmost release",
  "cargo-vet is failing" / "unvetted dependencies" in a pkgs/* repo, "set up the
  clockmost pacman repo" / "install clockmost packages on this machine".
---

# clockmost package pipeline

`git.clockmost.com/pkgs` is a Gitea org that builds Arch packages with a hardened
supply-chain CI and serves them from one pacman repo (`[clockmost]`). Push a repo
under `pkgs/`; a `v*` tag triggers a build that scans, packages and publishes.
Every machine with the `[clockmost]` repo gets updates via `pacman -Syu`.

- **Your Gitea login** is your username (`kuri`, `mira`, `Benylin`, …).
- **The repo MUST live under `pkgs/`** (not your personal namespace) — the publish
  step reads `PACKAGE_TOKEN`, which only exists as an org secret on `pkgs/*`.
- **The reusable CI** is `pkgs/packaging/.gitea/workflows/rust-pkg.yml` — never
  fork it. **Templates + a "first-build failures" table** are in
  `git.clockmost.com/pkgs/packaging` (`templates/` and `README.md`, both public).

Pick the task that matches the request.

---

## Task A — add a new package

1. **Create the repo under `pkgs`.** If you have push-to-create:
   ```sh
   cd <the project>
   git remote add gitea https://git.clockmost.com/pkgs/<name>.git
   git push gitea main
   ```
   (`origin`, e.g. GitHub, is left alone.) If you accidentally pushed to
   `<you>/<name>`, transfer it: repo Settings → *Transfer ownership* → `pkgs`.

2. **Fetch the templates** from `pkgs/packaging/templates/` and adapt:

   | file | edit |
   |---|---|
   | `PKGBUILD` | `pkgname`, `pkgdesc`, `license`, `depends=()` (rough — CI's `namcap` prints corrections). Builds from `$startdir`; keep `options=('!lto' '!debug')`. |
   | `.gitea/workflows/build.yml` | set `pkgname:`; for Cargo keep `rust-pkg.yml`, else switch `uses:` to `generic-pkg.yml`; set `extra_pacman_deps` to the C libs the build links |
   | `deny.toml` | usually unchanged. For a `git =` dep, add its GitHub org to `[sources].allow-org`. libcosmic → merge the commented COSMIC block. |
   | `.semgrep.yml` | unchanged |
   | `supply-chain/config.toml` (Cargo) | paste its `[imports.*]` into what `cargo vet init` generates (step 3) |
   | `renovate.json` | unchanged |

3. **Cargo projects — initialise cargo-vet** (required; CI fails without it):
   ```sh
   cargo install cargo-vet --locked      # if missing
   cargo vet init                        # grandfathers the whole current tree
   git add supply-chain/ && git commit -m "cargo-vet: init"
   ```

4. **Guard the GitHub workflow** — in every `.github/workflows/*.yml` job add:
   ```yaml
       if: ${{ github.server_url == 'https://github.com' }}
   ```

5. `makepkg --printsrcinfo > .SRCINFO`, commit everything, `git push gitea main`.
   Watch `git.clockmost.com/pkgs/<name>/actions`. The `scan` job will almost
   certainly fail 1–3 times on the first real build — **this is normal**. For
   each failure, look it up in the `pkgs/packaging` README "first-build failures"
   table. The frequent ones:
   - `cargo build` → `mold: fatal: library not found: X` → add `libX` to
     `extra_pacman_deps`, push again.
   - `semgrep` ERROR on your `unsafe` / `Command::new` → add
     `// nosemgrep: <rule-id>  <why>` on the line, **only after** confirming it's fine.
   - `cargo-deny` license/source → `[[licenses.clarify]]` or `[sources].allow-org`.
   - `cargo vet` unvetted → **Task C**.

6. When `scan` is green → cut the first release (**Task B**).

7. If you have the clockmost host, add the package to the inventory in
   `~/runbooks/clockmost-packages.md`.

---

## Task B — cut a release

From a clean `main` in the package repo:

```sh
./release.sh 0.2.0          # or 0.2.0-2 for a packaging-only rebuild (bumps pkgrel)
```

(`release.sh` is in `pkgs/packaging/templates/` — copy it in if missing.) It bumps
`Cargo.toml` + `PKGBUILD`, regenerates `.SRCINFO`, commits, tags `v0.2.0`, pushes.
The **tag push** runs `scan` again then `package`: build → `trivy rootfs` →
`namcap` → publish to the registry (main **and** `-debug` package if present).

Watch `.../actions`. On success it's live — clients get it on the next
`pacman -Syu`. `namcap` output in the log tells you which `depends` to tighten;
fold that into a `-2`.

`409 already in the registry` → that exact `pkgver-pkgrel` was already published;
bump `pkgrel` (or `pkgver`) and re-tag.

---

## Task C — cargo-vet flagged unvetted dependencies

`cargo vet init` (Task A step 3) grandfathers the initial tree. After that, CI's
`cargo vet` only fails on the **delta** — a new crate, or a version bump.
**You (Claude) review it — do not ask the user to read Rust.**

1. In the repo:
   ```sh
   cargo vet suggest        # exact crates + versions (also printed in the CI log)
   ```
2. Per crate:
   ```sh
   cargo vet inspect <crate> <version>     # new crate — read the whole source
   cargo vet diff <crate> <old> <new>      # bump — read only the delta
   ```
   Look for: `build.rs` / proc-macro behaviour, `unsafe`, process spawning,
   network or filesystem access, obfuscation, base64/hex-then-exec, anything
   off-purpose. Small utility crates should be small and boring.
3. Record what you judged safe:
   ```sh
   cargo vet certify <crate> <version>                    # --criteria safe-to-deploy (default)
   cargo vet certify <crate> <version> --criteria safe-to-run   # build-only / dev dep
   ```
   Add a one-line `notes` saying what you checked.
4. `git add supply-chain/audits.toml && git commit -m "vet: <crate> <version>" && git push`.

If something looks **genuinely suspicious**: do NOT certify. Stop and tell the
user the exact crate/version and what you found; suggest pinning to the last-good
version or dropping the dependency.

---

## Task D — set up the pacman client (any Arch/CachyOS machine)

The registry is **public — no token**. As root:

```sh
# 1. trust the registry signing key
curl -fsSL https://git.clockmost.com/api/packages/pkgs/arch/repository.key \
  | pacman-key --add -
pacman-key --lsign-key BC27FC1EB71B365E28B7FDC0F93259A911AB1037

# 2. append to /etc/pacman.conf
cat >> /etc/pacman.conf <<'EOF'

[clockmost]
SigLevel = Required
Server = https://git.clockmost.com/api/packages/pkgs/arch/clockmost/x86_64
EOF

# 3.
pacman -Syu
```

Pin `x86_64` in the Server line, **not `$arch`** — CachyOS multi-arch would try
`x86_64_v3` and 404. `pacman -Syy` (double y) forces a DB refresh if a stale one
got cached. Key fingerprint should be `BC27 FC1E B71B 365E 28B7 FDC0 F932 59A9
11AB 1037`.

### Replacing a hand-built install with the package

If the tool was previously `make install`ed to `~/.local/bin` (or similar):
1. stop the running process(es), `rm` the old binary + its `~/.config/autostart/*`
   entry, `systemctl --user reset-failed` any stale generated unit;
2. `pacman -S <name>`;
3. **grep configs for the old path** — e.g. COSMIC shortcuts in
   `~/.config/cosmic/com.system76.CosmicSettings.Shortcuts/v1/system_actions` —
   and `sed` `~/.local/bin/<name>` → `/usr/bin/<name>`;
4. start `/usr/bin/<name>` (the package ships `/etc/xdg/autostart/` for next login).

---

## Notes

- The runner builds on the clockmost host (limited RAM). A cold libcosmic-scale
  build is 15–40 min; `CARGO_BUILD_JOBS` is already capped at 4.
- `scan` runs on every push + PR and **blocks merge**; `package` runs only on
  `v*` tags and **blocks publish**. Neither can be weakened from a package repo.
- The pipeline runs `cargo-deny` (advisories + licenses + bans + **sources
  allow-list**), `cargo-vet`, `semgrep`, `trivy` — **not** a separate
  `cargo audit` (cargo-deny's `[advisories]` is the same RUSTSEC DB, one ignore list).
- Renovate opens bump PRs (7-day release age, never auto-merged); the full scan
  runs on each so a poisoned bump fails checks.
