# VaultShare Docs

System architecture, design decisions, and implementation history for the VaultShare
file-sharing platform. This repository is the reference for understanding *why* the
system is built the way it is.

## Contents

| File | Purpose |
|---|---|
| [ARCHITECTURE.md](ARCHITECTURE.md) | Full system architecture, data models, API design, and 9-phase implementation plan |
| [DESIGN_DECISIONS.md](DESIGN_DECISIONS.md) | Reasoning behind every significant architectural choice and documented trade-offs |
| [backend-take-home-task_v1.1 (1).md](<backend-take-home-task_v1.1%20(1).md>) | Original SECJUR engineering challenge specification |

## Implementation History

The [previous_implementation_phases/](previous_implementation_phases/) directory contains
the detailed implementation plans used during development:

| File | Phase |
|---|---|
| [phase2_implementation_plan.md](previous_implementation_phases/phase2_implementation_plan.md) | Accounts & Storage Abstraction |
| [phase3_implementation_plan.md](previous_implementation_phases/phase3_implementation_plan.md) | Core Files & Audit Trail |
| [phase4_implementation_plan.md](previous_implementation_phases/phase4_implementation_plan.md) | Sharing & Permissions |
| [manual_testing_phase4.md](previous_implementation_phases/manual_testing_phase4.md) | Manual testing notes for Phase 4 |

## Related

- Agent playbooks for AI-assisted development are in [vaultshare-api/agents/](https://github.com/secjur-vault-share/vaultshare-api/tree/main/agents)
- Infrastructure and quick start instructions are in [vaultshare-infra](https://github.com/secjur-vault-share/vaultshare-infra)
