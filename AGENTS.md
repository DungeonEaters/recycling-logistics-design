# Recycling Logistics Design — Agent Instructions

## Purpose

This repository is the source of truth for Recycling Logistics architecture, scope,
ADRs, implementation sequencing, and operational requirements. It is documentation,
not an implementation repository.

## Structure

```text
docs/
├── architecture/     # architecture.md and decisions.md (ADRs)
├── delivery/         # backlog, implementation plan, CI/CD strategy
├── infrastructure/   # OCI provisioning and administrator setup
└── research/         # Gemini context and source documents
```

## Link, do not duplicate

- Record **why**, scope, requirements, trade-offs, and architecture here.
- Link to implementation behavior in `recycling-backend`, `recycling-web`,
  `recycling-mobile`, and `recycling-infra`.
- Do not copy source code, build commands, CI implementation details, or secrets from
  implementation repositories.

## Document ownership

| Document | Use it for |
|---|---|
| `docs/architecture/architecture.md` | Context, requirements, components, non-functional requirements. |
| `docs/architecture/decisions.md` | ADRs, accepted trade-offs, and open questions. |
| `docs/delivery/implementation-plan.md` | Phases, milestones, and completion criteria. |
| `docs/delivery/backlog.md` | Features, priorities, risks, and decisions log. |
| `docs/delivery/cicd-strategy.md` | Versioning, delivery channels, registry, and release policy. |
| `docs/infrastructure/` | OCI topology, bootstrap, access model, and deployment requirements. |
| `docs/research/` | Source context; it does not override accepted ADRs. |

## Change rules

1. Read the current architecture and ADRs before proposing a change.
2. Update every directly affected design document when the implementation reality
   changes.
3. Preserve historical ADRs; supersede them with a new ADR instead of rewriting a
   decision without a trace.
4. Use relative Markdown links within this repository and absolute GitHub links to
   implementation repositories.
5. Never add credentials, private keys, tokens, Terraform state, customer data, or
   environment files.
6. `main` requires a PR and human approval. Never bypass branch protection, including
   to validate CI or create a release.
