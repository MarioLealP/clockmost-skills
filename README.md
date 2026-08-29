# clockmost-skills

Claude Code plugin marketplace for the clockmost homelab.

## Install

```
claude plugin marketplace add https://git.clockmost.com/Benylin/clockmost-skills.git
claude plugin install clockmost-package@clockmost
```

Stay current:

```
claude plugin update clockmost-package
```

## Plugins

| plugin | what it does |
|---|---|
| **clockmost-package** | Scaffold / build / release Arch packages via `git.clockmost.com/pkgs` — hardened supply-chain CI (cargo-deny/audit/vet, semgrep, trivy), the cargo-vet triage loop, and the pacman client setup. |

New skills get added here; `claude plugin update` picks them up.
