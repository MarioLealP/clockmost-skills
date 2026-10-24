# clockmost-skills

Claude Code plugin marketplace for the clockmost homelab.

## Install

```
claude plugin marketplace add https://git.clockmost.com/MarioGLealP/clockmost-skills.git
claude plugin install clockmost-package@clockmost
claude plugin install project-conventions@clockmost
```

Stay current:

```
claude plugin update            # all installed plugins
```

## Plugins

| plugin | what it does |
|---|---|
| **clockmost-package** | Scaffold / build / release Arch packages via `git.clockmost.com/pkgs` — hardened supply-chain CI (cargo-deny, cargo-vet, semgrep, trivy), the cargo-vet triage loop, and the pacman client setup. |
| **project-conventions** | The one shape for every repo: layout, trunk + short-lived PR branches, Conventional Commits, SemVer + CHANGELOG, a README that leads with setup, and the per-language (Rust / Python / JS / shell) lint+format+CI baseline. |

Complements from Anthropic's official marketplace (`claude-plugins-official`):
`commit-commands` (the `/commit` `/push` `/pr` tooling), `code-review`,
`pr-review-toolkit`, `claude-md-management`.

New skills get added here; `claude plugin update` picks them up.
