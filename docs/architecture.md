# Architecture — where to change what

> A navigation map for contributors. This file does **not** restate the
> contract; it points you at the file that owns each concern. For the rules
> themselves, follow the links. Start from [`AGENTS.md`](../AGENTS.md).

## The mental model in one paragraph

`loop-vibe-coding` is a **contract, not a runtime.** There is no orchestrator
process. A Lark Base table holds one row per change; the `owner` field is a relay
baton (`coder` | `reviewer` | `human`). Each agent polls for rows it owns, does
its job, and hands the baton on. The "program" is the state machine, and the
state machine is described — redundantly, on purpose — across the files below.
Your job as a contributor is to keep those descriptions byte-consistent.

## The pieces

| Layer | File(s) | Owns |
|-------|---------|------|
| Config template + machine source | [`loop.config.yaml`](../loop.config.yaml) | `roles` mapping, `loop.transitions`, `loop.breaker`, `quality_gate`, Base IDs |
| Role behaviour | [`skills/loop-coder/SKILL.md`](../skills/loop-coder/SKILL.md), [`skills/loop-reviewer/SKILL.md`](../skills/loop-reviewer/SKILL.md) | per-status instructions each agent follows; **project-agnostic** |
| Setup | [`skills/loop-init/SKILL.md`](../skills/loop-init/SKILL.md) | one-shot AI initializer (creates table, writes config, patches target `AGENTS.md`) |
| State-table spec | [`docs/base.schema.json`](base.schema.json), [`docs/base-schema.md`](base-schema.md) | fields, `status`/`owner` options, `round` writers |
| Contract prose | [`docs/state-machine.md`](state-machine.md) | transition table, flow diagram, invariants (authoritative prose) |
| Implementer restatement | [`docs/executor-contract.md`](executor-contract.md) | zero-ambiguity atomic hand-off, `next_owner: none`, claim & integrating pseudocode |
| Target-repo snippet | [`templates/AGENTS.section.md`](../templates/AGENTS.section.md) | the `## Loop Commands` block users paste into **their** repo |
| Onboarding / verification | [`docs/getting-started.md`](getting-started.md), [`docs/smoke-test.md`](smoke-test.md) | walkthrough + real two-agent check |

## "Where do I change X?"

- **A transition, severity gate, `round` rule, or owner hand-off** → it lives in
  **seven** files at once. See the sync table and source-of-truth order in
  [`AGENTS.md`](../AGENTS.md#the-single-most-important-rule-keep-the-state-machine-in-sync).
  Update all of them in one change, then grep the repo for the status/field name
  to confirm no hit was missed.
- **A new `status` or `owner` value** → `loop.config.yaml` →
  `docs/base.schema.json` + `docs/base-schema.md` → `docs/state-machine.md` →
  `docs/executor-contract.md` → both `skills/*` → the summary tables in
  `README.md`.
- **What an agent does in a given status** → the matching `skills/*/SKILL.md`.
  Keep it project-agnostic: no tech stack, no concrete commands.
- **A test/lint/build command** → never here. Those live in the **target repo's**
  `AGENTS.md`, resolved at runtime; the template is
  `templates/AGENTS.section.md`.
- **Role assignment (who plays coder/reviewer)** → `roles:` in
  `loop.config.yaml`. Nothing else should reference a vendor by name.
- **The pitch / summary tables** → `README.md` (never a source of truth — it must
  follow the seven files, not lead them).

## Invariants

The load-bearing rules (only `integrating` writes `done`; the acceptance gate is
always human; the breaker can't be bypassed; `round` has exactly three writers;
`blocked` ≠ `testing`; `next_owner: none` clears the cell) are listed in
[`AGENTS.md`](../AGENTS.md#invariants-you-must-never-break). A change that
violates one is wrong even if every file agrees with it.

## Before you commit

- `loop.config.yaml` must parse as YAML; `docs/base.schema.json` must parse as
  JSON.
- Product naming: **"Lark Base"** / **"Base"**, never "Bitable".
- Append-only audit fields (`review`, `resolution`, `test_report`) are never
  overwritten.
- Never commit secrets — credentials live in each agent's environment, not the
  repo.
