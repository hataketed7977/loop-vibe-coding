# State machine in detail

The loop has no orchestrator process. Each agent, on each scheduled tick, asks
one question — *"is there a row where `owner == me`?"* — does the job that the
row's `status` calls for, then flips `status` + `owner` to hand off. The table's
transitions, defined in `loop.config.yaml`, **are** the orchestration.

## Flow

```
            ┌──────────────────────────── you create row ───────────────────┐
            ▼                                                                │
        ┌───────┐  spec written   ┌──────┐  spec ok    ┌──────────────┐     │
   ─────│  new  │────────────────▶│ spec │────────────▶│ implementing │     │
        └───────┘   owner=coder   └──────┘ owner=coder └──────────────┘     │
            ▲                          │                       │            │
            │ spec needs work          │                       │ built      │
            └──────────────────────────┘                       ▼            │
                                                        ┌─────────────┐     │
                            ┌──────────────────────────▶│  reviewing  │     │
                            │   re-review after fix      └─────────────┘     │
                            │                          │            │        │
                        ┌────────┐  P0/P1 issues       │            │ clean  │
                        │ fixing │◀────────────────────┘            ▼        │
                        └────────┘                           ┌───────────┐   │
                            owner=coder                      │  testing  │   │
                                                             └───────────┘   │
                                                            owner=human      │
                                                             │       │       │
                                                       pass  │       │ bug   │
                                                             ▼       └───────┘
                                                    ┌──────────────┐ (back to implementing,
                                                    │ integrating  │  round reset to 0)
                                                    └──────────────┘
                                                       owner=coder
                                                    commit + archive
                                                             │
                                                             ▼
                                                        ┌──────┐
                                                        │ done │
                                                        └──────┘
```

## Transition table

| From `status` | Owner acts | Condition           | To `status`    | New `owner` |
|---------------|------------|---------------------|----------------|-------------|
| `new`         | coder      | spec written        | `spec`         | reviewer    |
| `spec`        | reviewer   | spec has gaps       | `new`          | coder       |
| `spec`        | reviewer   | spec sound          | `implementing` | coder       |  (round reset to 0)
| `implementing`| coder      | implemented         | `reviewing`    | reviewer    |
| `reviewing`   | reviewer   | P0/P1 present       | `fixing`       | coder       |
| `reviewing`   | reviewer   | no blocking issues  | `testing`      | human       |
| `fixing`      | coder      | fixes applied       | `reviewing`    | reviewer    |
| `testing`     | human      | acceptance pass     | `integrating`  | coder       |
| `testing`     | human      | bug found (round=0) | `implementing` | coder       |
| `integrating` | coder      | commit + archive    | `done`         | —           |

## The two invariants

1. **Agents never *decide* `done`.** The acceptance judgement always belongs to a
   person. An agent only writes the literal `done` value in `integrating` — the
   mechanical commit/archive step that runs *after* a human has already passed
   acceptance. No agent can reach `done` from any other status.

2. **The loop cannot spin forever.** `round` increments on every review — both
   spec reviews (`new ↔ spec`) and code reviews (`reviewing ↔ fixing`). When
   `round >= loop.max_rounds`, whichever agent holds the baton stops the
   ping-pong and routes to `owner=human, status=testing` with a note explaining
   it didn't converge. An unattended loop that can't converge escalates; it does
   not quietly burn tokens. The two loops each get their own full budget:
   `round` resets to 0 when the spec is approved (`spec → implementing`), and
   again when a human bounces a change back with a `bug` (`testing →
   implementing`) — so a hard-fought spec debate never starves the later code
   review/fix loop, and a bugfix never inherits an exhausted counter.

## Why severities gate the flow

Only `quality_gate.block_levels` (P0/P1 by default) send a change back to
`fixing`. P2/P3 are recorded as advice but do not block — otherwise the loop
would never converge on style nits. Tighten or loosen this in
`loop.config.yaml` without touching the skills.

## Customising the flow

Everything above is data, not code. To change the process — add a stage, change
who reviews the spec, make P2 blocking — edit `loop.transitions` and
`quality_gate` in `loop.config.yaml`. The skills follow whatever the config
says.
