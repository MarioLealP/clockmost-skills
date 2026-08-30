---
name: project-conventions
description: >
  Apply when starting a new repo, or tidying an existing one into "a proper
  project". Triggers: "set up a new project", "organize this repo", "make this a
  proper project", "add conventional commits", "branch strategy", "protect main",
  "write a README", "set up CI / linting / formatting", "how do we version this",
  "add a CHANGELOG", "release process". Covers the language-agnostic core plus a
  per-language (Rust / Python / JS / shell) lint+format+CI baseline.
---

# Project conventions

One consistent shape for every repo, so anyone (or any Claude session) can land
in a project and know where things are, how to change them, and how a change
ships. Apply the whole thing to a new repo; for an existing one, work the
checklist and fix what's missing.

## 1. Repository layout

Every repo has, at the root:

| file | purpose |
|---|---|
| `README.md` | §4 — what it is + **how to set it up and run it** |
| `LICENSE` | an SPDX-named license (`MIT`, `Apache-2.0`, `GPL-3.0-only`, …). No license = nobody can legally use it. |
| `.gitignore` | build output, `.env`, editor dirs, OS cruft. Never `.gitignore` the lockfile. |
| `.editorconfig` | indentation + final-newline + charset, so editors agree |
| `CHANGELOG.md` | §5 — human-readable, newest first, or generated from commits |
| `CONTRIBUTING.md` | (once >1 person) build, test, lint, branch, commit, PR — a pointer to the relevant §s here |
| CI config | `.github/workflows/` or `.gitea/workflows/` — §6 |
| lockfile | `Cargo.lock`, `uv.lock`, `package-lock.json` — **committed** for apps and tools |

Source lives under `src/` (or the language's convention). Tests mirror the
source tree. A `docs/` or an `AGENT.md` / `DESIGN.md` for anything non-obvious.
Keep the root uncluttered — config files are fine, stray scripts and notes are not.

## 2. Branching — trunk + short-lived PR branches

- **`main` is the trunk.** It is always releasable and always green. Nobody
  pushes to it directly.
- **Work on a short-lived branch** off `main`, named `<type>/<slug>`:
  `feat/live-thumbnails`, `fix/grey-output`, `chore/bump-deps`,
  `docs/readme-setup`. One coherent change per branch.
- **Open a PR** into `main`. CI runs; another person (or, for a solo project,
  the author after a self-review pass) approves; **squash-merge**; **delete the
  branch**. The PR description says what changed and why, and links the issue.
- **No long-lived `dev` / `develop` branch.** It just accumulates drift and
  merge pain. If you need a staging environment, deploy `main` to it.
- **Release branches only if you support old majors** (`release/1.x`) — most
  projects never need one.

### Protect `main`

On the forge (GitHub/Gitea → repo → Settings → Branches):
- require the PR's CI checks to pass before merge
- require ≥1 approving review (solo: require the author's own review + green CI)
- require the branch to be up to date with `main`
- disallow force-push and deletion
- (optional) require **linear history** so `main` reads as a clean list of squashes

## 3. Conventional Commits

Every commit subject: **`<type>(<scope>)!: <summary>`**

- `<type>` — one of:
  `feat` (a feature) · `fix` (a bug fix) · `docs` · `refactor` (no behaviour
  change) · `perf` · `test` · `build` (build system / deps) · `ci` · `chore`
  (everything else) · `style` (formatting only)
- `<scope>` — optional, a noun for the area: `feat(auth):`, `fix(parser):`
- `!` before the colon **or** a `BREAKING CHANGE: <what>` footer — signals a
  breaking change (→ major version bump)
- `<summary>` — imperative mood, lowercase, no trailing period, ≤72 chars
  ("add", not "added" / "adds")
- Body (optional, after a blank line): *why*, not *what* — the diff shows what.
  Wrap at 72.
- Footers: `Fixes #123`, `Refs #45`, `Co-Authored-By: …`

```
feat(switcher): software-render the chrome by default

wgpu's AMD-tiled dmabufs can't be imported by cosmic-comp's GLES renderer
on RDNA2, so the output goes grey. The chrome is a static thumbnail strip —
no GPU path needed.

Fixes #12
```

```
fix(build): add libinput to the link deps

BREAKING CHANGE: the minimum toolchain is now 1.93 (edition 2024)
```

Why bother: the history becomes a changelog, tools can compute the next SemVer
bump, and `git log --oneline` is actually readable. Enforce it with
`commitlint` (a `commit-msg` hook) or a CI check on PR titles when you squash.

## 4. README — lead with setup

A reader should get from "just cloned this" to "it runs" without asking anyone.

```markdown
# <name>

One or two sentences: what it is, who it's for, what problem it solves.

## Install / setup

Prerequisites (with versions), then the exact commands. Copy-pasteable.
For an app: how to run it. For a library: how to add it as a dependency.
For a service: the env vars it needs (point at `.env.example`).

## Usage

The 2–3 most common things someone does with it, with real commands/output.

## Development

    <clone>
    <install toolchain>
    <format>  <lint>  <test>  <build>
    <run locally>

Branch + commit conventions: link to CONTRIBUTING.md or this skill.

## License

<SPDX id> — see LICENSE.
```

Keep it current — a README that lies is worse than none. If setup changes, the
same PR updates the README.

## 5. Versioning & releases

- **SemVer** `MAJOR.MINOR.PATCH`. Breaking → MAJOR, feature → MINOR, fix → PATCH.
  Pre-1.0: MINOR is your "breaking", PATCH is everything else.
- **Tag releases** `vX.Y.Z` (annotated: `git tag -a v1.2.0 -m v1.2.0`). CI builds
  and publishes off the tag.
- **`CHANGELOG.md`** — [Keep a Changelog](https://keepachangelog.com) format, or
  generate it from Conventional Commits (`git-cliff`, `release-please`). An
  `## [Unreleased]` section at the top that you move down on release.
- One command to cut a release (`./release.sh X.Y.Z`): bump the version file(s),
  update the changelog, commit `release: vX.Y.Z`, tag, push.

## 6. Lint, format, test, CI

**Format is not a choice** — one formatter, run in CI in check mode, and ideally
a pre-commit hook so nobody argues in review. Lint catches real bugs; treat lint
warnings as errors in CI.

CI on every push and PR, blocking merge:

1. **format check** (fast, fails first)
2. **lint** (`-D warnings` / `--max-warnings 0`)
3. **test**
4. **build**
5. supply-chain / security scans where they apply (see the `clockmost-package`
   skill for the Rust/Arch pipeline)

### Per-language baseline

| language | format | lint | types | test | deps |
|---|---|---|---|---|---|
| **Rust** | `cargo fmt --check` | `cargo clippy -- -D warnings` | (the compiler) | `cargo test` | `cargo deny check`, `cargo machete` |
| **Python** | `uv run ruff format --check` | `uv run ruff check` | `uv run ty check` (or `mypy`) | `uv run pytest` | `uv lock --check`, `uv run pip-audit` |
| **JS/TS** | `prettier --check` | `eslint --max-warnings 0` | `tsc --noEmit` | `vitest` / `jest` | `npm audit`, `npm ci` |
| **Shell** | `shfmt -d` | `shellcheck` | — | `bats` | — |

Python specifics (this ecosystem uses `uv` for everything):
- `uv init`, deps via `uv add` / `uv add --dev`, run via `uv run`, never a bare
  `pip`. Commit `uv.lock`.
- `pyproject.toml` holds ruff + ty + pytest config — one file.
- `ruff` replaces black + isort + flake8 + most plugins; `ty` (Astral's type
  checker) or `mypy` for types.

### Pre-commit (optional but nice)

A `.pre-commit-config.yaml` (or a plain `.git/hooks/pre-commit`) running the
format + lint locally so CI rarely catches style. Keep it fast — anything slow
(full test suite, full build) stays in CI only.

## 7. Secrets & config

- **Never commit secrets.** Ship a `.env.example` with every key present and
  fake/empty values; `.gitignore` the real `.env`.
- If secrets must live in the repo, encrypt them (`sops` + `age`) and gitignore
  the decrypted form.
- CI secrets go in the forge's secret store, never in the workflow YAML.
- Rotate anything that ever touched a commit, a log, or a chat.

## Applying this to an existing repo

Go top to bottom; open one PR per section (`chore/repo-hygiene`,
`ci/add-lint-gate`, `docs/readme-setup`). Don't do it all in one giant commit —
that defeats the point.
