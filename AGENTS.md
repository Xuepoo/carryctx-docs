# CarryCtx Documentation Repository Instructions

## Scope

This repository is the source of truth for the CarryCtx product, CLI contracts, storage model, engineering rules, delivery plans, workflows, roadmaps, and verification reports.

## Authoring rules

- Keep requirements, CLI behavior, configuration, architecture, and implementation aligned.
- Resolve contradictions instead of documenting both variants as valid.
- Use `config.toml` and `config.local.toml`; runtime project state belongs in the Git common directory.
- Treat JSON envelopes, exit codes, command names, and configuration keys as public contracts.
- Label features as v0.1, P1, P2, experimental, or future consistently across documents.
- Prefer concrete examples, explicit invariants, and testable acceptance criteria.
- Do not store temporary logs or generated package artifacts here; place them in `../recording/`.
- Test reports committed under `reports/` must include commands, dates, tool versions, outcomes, and any known gaps.

## Change coupling

When CLI implementation changes documented behavior, update the affected document in the same workstream. Architecture decisions and implementation plans belong in this repository even though code lives in `../carryctx-cli/`.
