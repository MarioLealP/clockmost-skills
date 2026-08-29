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
supply-chain CI and serves them from one pacman repo (`[clockmost]`). Producers
push a repo under `pkgs/`; a tag triggers a build that scans, packages, and
publishes. Every machine that adds the `[clockmost]` repo gets updates via
`pacman -Syu`.

- **Gitea logins:** yours is your Gitea username (e.g. `kuri`, `mira`, `Benylin`).
- **Reusable CI** lives in `pkgs/packaging` — do not fork it; package repos call it.
- **Templates** are in this skill's `templates/` directory (and mirrored in
  `pkgs/packaging/templates/`).

Pick the task below that matches the request.

---

## Task A — add a new package

1. **Create the repo** by pushing (push-to-create is on):
   ```sh
   cd <the project>
   git remote add gitea https://git.clockmost.com/pkgs/<name>.git
   git push gitea main
   ```
   `origin` (e.g. GitHub) is left untouched. If push-to-create is disabled, ask
   the user to create `pkgs/<name>` in the Gitea UI first.

2. **Copy the templates** into the repo and edit:

   | file | from | edit |
   |---|---|---|
   | `PKGBUILD` | `templates/PKGBUILD` | pkgname, pkgdesc, `depends=()` (first guess — CI's `namcap` will correct it), `build()`/`package()` |
   | `.gitea/workflows/build.yml` | `templates/build.yml` | set `pkgname:`; for a Cargo project keep `rust-pkg.yml`, else switch the `uses:` to `generic-pkg.yml`; set `extra_pacman_deps` to the system libs the build needs |
   | `deny.toml` | `templates/deny.toml` | usually unchanged — add the exact git orgs of any `git =` deps to `[sources].allow-org` |
   | `.semgrep.yml` | `templates/.semgrep.yml` | unchanged |
   | `supply-chain/config.toml` | `templates/supply-chain-config.toml` (Cargo only) | unchanged |
   | `renovate.json` | `templates/renovate.json` | unchanged |

3. **Cargo projects only** — init cargo-vet locally so CI has a baseline:
   ```sh
   cargo install cargo-vet --locked      # if missing
   cargo vet init                         # writes supply-chain/{config,audits,imports}.toml
   ```
   Merge the template `config.toml`'s `[imports.*]` into the generated one.

4. **Guard the GitHub workflow** so it does not run on Gitea. In
   `.github/workflows/*.yml`, add to each job:
   ```yaml
       if: ${{ github.server_url == 'https://github.com' }}
   ```

5. `makepkg --printsrcinfo > .SRCINFO`, commit everything, `git push gitea main`.
   The `scan` job runs on the push — watch
   `git.clockmost.com/pkgs/<name>/actions`. Fix what it flags (fmt, clippy,
   cargo-deny licenses/sources, semgrep). For cargo-vet failures see **Task C**.

6. When `scan` is green, cut the first release — **Task B**.

7. Tell the user to update the inventory in `~/runbooks/clockmost-packages.md` (if
   they have the clockmost host) — new package name + one line.

---

## Task B — cut a release

From inside the package repo, on a clean `main`:

```sh
./release.sh 0.2.0          # or 0.2.0-2 for a packaging-only rebuild (bumps pkgrel)
```

(`release.sh` is `templates/release.sh` — copy it in if the repo doesn't have it.)
It bumps `Cargo.toml` + `PKGBUILD`, regenerates `.SRCINFO`, commits, tags `v0.2.0`,
and pushes. The **tag push** triggers the `package` job: build → `trivy rootfs` →
`namcap` → publish to the registry.

Watch `.../actions`. On success the package is live; clients get it on their next
`pacman -Syu`. If `namcap` flags missing/excess `depends`, tighten `PKGBUILD` and
cut a `-2`.

`409 already in the registry` = that exact `pkgver-pkgrel` was already published;
bump `pkgrel` (or `pkgver`) and re-tag.

---

## Task C — cargo-vet failed (unvetted dependencies)

CI's `cargo-vet` step blocks when a dependency (new, or a version bump) is not
covered by an imported audit set. **You (Claude) do the review — do not ask the
user to read Rust.**

1. In the package repo:
   ```sh
   cargo vet suggest        # exact crates + versions needing audit (also in the CI log)
   ```

2. For each one:
   ```sh
   cargo vet inspect <crate> <version>     # new crate — read the full source
   cargo vet diff <crate> <old> <new>      # version bump — read only the delta
   ```
   Read for: `build.rs` / proc-macro behaviour, `unsafe`, process spawning,
   network or filesystem access, obfuscation, base64/hex-then-exec, anything that
   does not match the crate's stated purpose. Small utility crates should be
   small and boring.

3. Record the audit for what you judged safe:
   ```sh
   cargo vet certify <crate> <version>                       # --criteria safe-to-deploy (default)
   cargo vet certify <crate> <version> --criteria safe-to-run   # if it's a build-only / dev dep
   ```
   Write a one-line `notes` when prompted saying what you checked.

4. `git add supply-chain/audits.toml && git commit -m "vet: <crate> <version>" && git push`.
   CI re-runs and goes green.

If something looks **genuinely suspicious**, do NOT certify it — stop and tell the
user exactly what you found and which crate/version, and suggest pinning to the
last-known-good version or removing the dependency.

---

## Task D — set up the pacman client (any Arch/CachyOS machine)

The registry is public — **no token needed**. As root:

```sh
# 1. trust the registry signing key
curl -fsSL https://git.clockmost.com/api/packages/pkgs/arch/repository.key \
  | pacman-key --add -
pacman-key --lsign-key BC27FC1EB71B365E28B7FDC0F93259A911AB1037

# 2. add the repo — append to /etc/pacman.conf
cat >> /etc/pacman.conf <<'EOF'

[clockmost]
SigLevel = Required
Server = https://git.clockmost.com/api/packages/pkgs/arch/clockmost/x86_64
EOF

# 3.
pacman -Syu
```

Pin `x86_64` in the Server line (not `$arch`) — CachyOS's multi-arch would try
`x86_64_v3` and miss. Thereafter `pacman -Syu` updates every `[clockmost]`
package alongside the system. Verify the key fingerprint against
`~/runbooks/clockmost-packages.md` if the user has doubts — it should be
`BC27 FC1E B71B 365E 28B7 FDC0 F932 59A9 11AB 1037`.

---

## Notes

- The runner builds on the clockmost host, which has limited RAM. For a heavy
  Cargo build (libcosmic, wgpu, …) keep `CARGO_BUILD_JOBS` at 4 (the template
  already does) and expect the first cold build to take several minutes.
- The `scan` job runs on every push and PR and **blocks merge**; `package` runs
  only on `v*` tags and **blocks publish**. Neither can be bypassed from a
  package repo — the gates live in `pkgs/packaging`.
- Renovate opens dependency-bump PRs (7-day release-age, never auto-merged); the
  full scan runs on each PR so a poisoned bump fails checks.
