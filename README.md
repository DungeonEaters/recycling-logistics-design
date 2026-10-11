# Recycling Logistics Design

Source of truth for architecture, product scope, delivery decisions, and operational
requirements for Recycling Logistics. Implementation lives in the repositories under
the `DungeonEaters` organization; this repository records why and what is built.

## Repository layout

| Directory | Purpose |
|---|---|
| [`docs/architecture/`](docs/architecture/) | System architecture and Architecture Decision Records (ADRs). |
| [`docs/delivery/`](docs/delivery/) | Backlog, implementation roadmap, and CI/CD strategy. |
| [`docs/infrastructure/`](docs/infrastructure/) | Oracle Cloud provisioning, credentials, and deployment design. |
| [`docs/research/`](docs/research/) | Gemini conversation and source material used to establish the design. |

## Canonical documents

- [Architecture](docs/architecture/architecture.md)
- [Decisions (ADRs)](docs/architecture/decisions.md)
- [Implementation plan](docs/delivery/implementation-plan.md)
- [Backlog](docs/delivery/backlog.md)
- [CI/CD strategy](docs/delivery/cicd-strategy.md)
- [Oracle Cloud provisioning](docs/infrastructure/oracle-cloud-provisioning.md)
- [Oracle administrator setup](docs/infrastructure/oracle-credentials-setup.md)

## Implementation repositories

- [recycling-backend](https://github.com/DungeonEaters/recycling-backend)
- [recycling-web](https://github.com/DungeonEaters/recycling-web)
- [recycling-mobile](https://github.com/DungeonEaters/recycling-mobile)
- [recycling-infra](https://github.com/DungeonEaters/recycling-infra)

## Contribution rules

- `main` requires a pull request and one approval. Do not bypass branch protection.
- Update the relevant canonical document and ADR when a design decision changes.
- Link to code, workflows, and deployment details in implementation repositories; do
  not duplicate their runtime or code-level documentation here.
- Never add credentials, private keys, Terraform state, tokens, or production secrets.
