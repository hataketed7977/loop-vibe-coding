# AGENTS.md

> **For agents and humans working ON this repository.**
> If you were sent here to *run the loop* on some other project, you want that
> project's own `AGENTS.md`, not this one. See the note in
> [Two different AGENTS.md files](#two-different-agentsmd-files) below.

This file is the entry point for anyone — human or AI — modifying
`loop-vibe-coding` itself. Read it before you touch anything.

## What this repo is (and is not)

`loop-vibe-coding` is a **contract / specification repo, not a runtime.** Nothing
here executes the loop. The skills, `loop.config.yaml`, and the Base schema
describe *what* each role must do; any capable agent (Codex, Claude Code, …)
consumes them to play a role. There is no server, no orchestrator process, no
build step.

The practical consequence: **you cannot "run the tests" to know a change is
correct.** Correctness here means *the documents agree with each other.* The
state machine is described in several places at once, and they must stay
byte-consistent. Drift between them is the #1 class of bug in this repo.

## The single most important rule: keep the state machine in sync

The state machine (statuses, owners, transitions, the `round` circuit breaker,
the `integrating → done` path, the `claimed_by` lock) is **specified redundantly
across seven files on purpose** — each audience needs it in a different form. If
you change any rule, you must update **all** of these in the same change, or you
introduce contract drift:

| # | File | The form it states the contract in |
|---|------|-----------------------------------|
| 1 | `loop.config.yaml` | The machine-editable source: `loop.transitions`, `loop.breaker`, `quality_gate`. |
| 2 | `skills/loop-coder/SKILL.md` | The coder's per-status instructions. |
| 3 | `skills/loop-reviewer/SKILL.md` | The reviewer's per-status instructions. |
| 4 | `docs/base.schema.json` | Machine-readable Base table spec (fields, `status` options, `round` writers). |
| 5 | `docs/base-schema.md` | Human-readable version of the same table spec. |
| 6 | `docs/state-machine.md` | The transition table + flow diagram + invariants prose. |
| 7 | `docs/executor-contract.md` | The zero-ambiguity restatement for implementers (atomic hand-off, `next_owner: none`, `round` writers, claim & integrating pseudocode). |

Plus `README.md` carries a summary table that must not contradict the seven.

**Source-of-truth order** when two files disagree (this order is also stated at
the top of `executor-contract.md`):

1. `skills/*` and `loop.config.yaml` and `docs/base*.schema` — the primary spec.
2. `docs/state-machine.md` — authoritative prose.
3. `docs/executor-contract.md` — restatement; if it disagrees with the above,
   *it* is the bug, not them.
4. `README.md` — summary; never the source of truth.

> Before opening a change that touches any transition, severity gate, `round`
> rule, or owner hand-off: grep the whole repo for the status name and the field
> name you are changing, and reconcile every hit.

## Invariants you must never break

These are load-bearing. A change that violates one is wrong even if every file
agrees:

1. **Only `integrating` writes `done`,** and only after a human has accepted.
   No agent reaches `done` from any other status.
2. **The acceptance gate always belongs to a human.** Agents advance a change up
   to `testing` and stop.
3. **The loop cannot spin forever.** `round` drives a `max_rounds` breaker that
   covers both the spec loop (`new ↔ spec`) and the code loop
   (`reviewing ↔ fixing`); non-convergence parks the change in `blocked`
   (owner=human) — **never** in `testing`.
4. **`round` has exactly three writers:** reviewer (increment on every review +
   reset to 0 on spec approval), human (reset on bug bounce), coder (reset-only
   self-heal — never increments).
5. **`blocked` ≠ `testing`.** `testing` = a finished change awaiting acceptance;
   `blocked` = the agents are stuck and need a decision (maybe no working impl).
6. **`next_owner: none` means clear the owner cell,** not write the string
   `"none"`.

## Repository layout

```
loop-vibe-coding/
├── README.md                 # project pitch + summary tables (not source of truth)
├── AGENTS.md                 # ← you are here: how to work on THIS repo
├── CLAUDE.md                 # pointer to this file
├── loop.config.yaml          # per-project config TEMPLATE + the state machine
├── skills/
│   ├── loop-init/SKILL.md    # one-shot AI initializer (creates table + config)
│   ├── loop-coder/SKILL.md   # coder-role loop skill (tool-agnostic)
│   └── loop-reviewer/SKILL.md# reviewer-role loop skill (tool-agnostic)
├── templates/
│   └── AGENTS.section.md      # the `## Loop Commands` snippet for TARGET repos
└── docs/
    ├── architecture.md       # how the pieces fit + where to change what
    ├── getting-started.md     # onboarding walkthrough
    ├── base-schema.md         # state-table field spec (human-readable)
    ├── base.schema.json       # state-table field spec (machine-readable)
    ├── executor-contract.md   # zero-ambiguity contract for implementers
    ├── state-machine.md       # transitions & safety rails in detail
    └── smoke-test.md          # how to verify a real two-agent setup
```

See [`docs/architecture.md`](docs/architecture.md) for the "where do I change
X?" map.

## Two different AGENTS.md files

Do not confuse them:

- **This file (`/AGENTS.md`)** — instructions for working *on* loop-vibe-coding.
- **`templates/AGENTS.section.md`** — a snippet that the *user* pastes into
  *their own* project's `AGENTS.md`. That is where the loop skills read the
  `test` / `lint` / `build` commands at runtime. The skills contain **zero**
  project knowledge; those commands are the single source of truth and must
  never be hard-coded into the skills.

## Conventions for changes

- **Keep the skills project-agnostic.** No tech stack, no concrete test/lint
  commands, no repo-specific paths in `skills/*`. Anything project-specific
  belongs in the target repo's `AGENTS.md`, resolved at runtime.
- **Product naming:** use **"Lark Base"** or **"Base"**. Do not write "Bitable".
- **Append-only audit fields** (`review`, `resolution`, `test_report`) are
  appended across rounds, never overwritten — preserve them as the loop's trail.
- **Validate machine files after editing:** `loop.config.yaml` must parse as YAML
  and `docs/base.schema.json` as JSON before you commit.
- **Never commit secrets.** Base API credentials and any tokens live in each
  agent's own environment, never in the repo. `app_token` / `table_id` are
  identifiers (safe), but credentials are not.
