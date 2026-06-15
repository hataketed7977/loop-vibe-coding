# loop-vibe-coding

> A portable **loop harness** that turns two AI coding agents into an autonomous
> **coder ↔ reviewer** pair. You kick off the task and sign off the result —
> the agents handle everything in between, coordinating through a shared
> **state machine** (a Lark Base).

Built on the idea of **loop engineering**: instead of prompting a coding agent
turn by turn, you design a loop that prompts the agents for you. See Addy
Osmani's [_Loop Engineering_](https://addyosmani.com/blog/loop-engineering/) for
the underlying philosophy.

---

## Why

Most AI pair-coding today is a **manual relay**: you copy the reviewer's
comments, paste them to the coder, wait, copy the next round, paste again. That
copy-paste *is* the bottleneck — you've become a wire between two agents.

`loop-vibe-coding` removes the human from the **middle** of the loop while
keeping the human at the **two ends that actually matter**: the kickoff and the
acceptance test.

```
Before:  you ⇄ coder ⇄ (you copy/paste) ⇄ reviewer ⇄ (you copy/paste) ⇄ coder ...
After:   you ──start──▶  [ coder ⇄ reviewer loop ]  ──▶ you (accept)
```

---

## How it works

```
You ──start──▶ [State Machine: new task, owner=coder]
                       │
         ┌─────────────┴──────────────┐
         ▼                             ▲
   coder agent polls            reviewer agent polls
   (claims owner==coder)        (claims owner==reviewer)
   spec / implement / fix       review → structured feedback
   → hand off to reviewer       → hand back, or pass through
         │                             │
         └──────────► owner=human ◀────┘
                       │
You ──acceptance test──▶ pass = integrating → coder commits & archives → done
                         bug  = back into the loop
```

Three principles:

- **No central orchestrator.** Each agent only ever asks one question:
  *"is there a record with `owner == me`?"* If yes, it does its job, then hands
  the baton to the next role. **The state machine *is* the orchestration** —
  there is no driver process to maintain.
- **State lives outside the conversation.** Agents forget everything between
  runs; the Base does not. It is the loop's memory and its dashboard.
- **The human owns the acceptance gate.** Agents advance a task up to
  `testing` and then stop — they never *decide* a change is done. Only after you
  pass the acceptance test (`status = integrating`) does the coder do the
  mechanical commit + archive and write `done`. "It works" is a claim, not a
  proof — the judgement stays with you.

---

## Roles, not tools

Skills are named by **role**, not by vendor — so you can swap who plays each
part without touching the loop, the skills, or the state machine.

| Role     | Skill           | Job                                       |
|----------|-----------------|-------------------------------------------|
| coder    | `loop-coder`    | write spec, self-check, implement, fix    |
| reviewer | `loop-reviewer` | review spec & code, produce structured feedback |
| human    | (you)           | kickoff + final acceptance                |

Who plays which role is a **one-line config change**:

```yaml
# loop.config.yaml
roles:
  coder:    codex          # swap freely — skills & state machine don't care
  reviewer: claude-code
```

Want Claude Code to write and Codex to review? Just swap the two values.

---

## Project-agnostic by design

The loop skills contain **zero project knowledge**. Tech stack, and the
test / lint / build commands all come from the **target repo's own
`AGENTS.md`** — they are never hard-coded into the skills. Drop the same skills
into any [OpenSpec](https://github.com/Fission-AI/OpenSpec) project and they
just work.

```
┌─ Per-repo (changes per project) ──────────────────────┐
│  AGENTS.md          → stack, test/lint/build commands  │
│  openspec/          → change specs                     │
│  loop.config.yaml   → state-table IDs + role mapping   │
└────────────────────────────────────────────────────────┘
┌─ Reusable (this repo — never changes) ────────────────┐
│  skills/loop-coder/      pure state-machine logic      │
│  skills/loop-reviewer/   pure state-machine logic      │
└────────────────────────────────────────────────────────┘
```

---

## The state machine

One Base record = one change's full lifecycle. The `owner` field is the
relay baton that drives everything.

| status         | owner    | action                              | next                                  |
|----------------|----------|-------------------------------------|---------------------------------------|
| `new`          | coder    | write OpenSpec change spec          | `spec` / reviewer                     |
| `spec`         | reviewer | review the spec for soundness       | back to coder, or `implementing`/coder|
| `implementing` | coder    | self-check + implement (or fix a logged bug) | `reviewing` / reviewer       |
| `reviewing`    | reviewer | code review, write structured notes | `fixing`/coder (issues) · `testing`/human (clean) |
| `fixing`       | coder    | judge feedback, fix, re-apply       | `reviewing` / reviewer                |
| `testing`      | human    | **real acceptance test**            | `integrating`/coder (pass) · `implementing`/coder (bug) |
| `integrating`  | coder    | commit + `openspec archive`         | `done`                                |
| `done`         | —        | committed & archived                | —                                     |

**Safety rails.** A `max_rounds` circuit breaker guards **both** the spec
(`new ↔ spec`) and code (`reviewing ↔ fixing`) loops — each gets its own full
budget (`round` resets to 0 when the spec is approved and on a human bug bounce)
— and forces a hand-off to the human if the agents fail to converge, so an
unattended loop can't sit there burning tokens arguing with itself. Agents only
write `status=done` from the `integrating` step, and only after a human has
already accepted the change.

---

## Quick start

### The easy way — let an AI set it up

Install the [`loop-init`](skills/loop-init/SKILL.md) skill and tell any capable
agent (one that has Lark Base access):

> "Initialize loop-vibe-coding in this repo."

`loop-init` creates the state table from the machine-readable
[`docs/base.schema.json`](docs/base.schema.json), writes the resulting
`app_token` / `table_id` back into `loop.config.yaml`, fills the `roles` mapping,
and adds the `## Loop Commands` section to your `AGENTS.md` — end to end, no
manual steps. This is the recommended path if you'd rather have the AI do the
boring setup.

### The manual way

1. **Create the state table.** In Lark Base, create a table from the spec
   in [`docs/base-schema.md`](docs/base-schema.md) (human-readable) or
   [`docs/base.schema.json`](docs/base.schema.json) (machine-readable).
2. **Configure the project.** Copy [`loop.config.yaml`](loop.config.yaml) into
   your target repo and fill in `base.app_token` / `base.table_id` and the
   `roles` mapping.
3. **Declare your commands.** Add the `## Loop Commands` section from
   [`templates/AGENTS.section.md`](templates/AGENTS.section.md) to your repo's
   `AGENTS.md` (test / lint / build).
4. **Install the skills.** Install `skills/loop-coder` into the agent playing
   *coder* and `skills/loop-reviewer` into the agent playing *reviewer*.
5. **Schedule polling.** Have each agent poll on an interval:
   - Codex → an **Automation** that calls the skill.
   - Claude Code → `/loop` or a cron/scheduled task that calls the skill.
6. **Go.** Create a task row, set `status=new, owner=coder`, and walk away.
   Come back when a row reaches `owner=human`.

See [`docs/getting-started.md`](docs/getting-started.md) for the full walkthrough.

---

## Repository layout

```
loop-vibe-coding/
├── README.md
├── LICENSE
├── loop.config.yaml                 # per-project config template
├── skills/
│   ├── loop-init/SKILL.md           # one-shot AI initializer (creates table + config)
│   ├── loop-coder/SKILL.md          # coder-role loop skill (tool-agnostic)
│   └── loop-reviewer/SKILL.md       # reviewer-role loop skill (tool-agnostic)
├── templates/
│   └── AGENTS.section.md            # the `## Loop Commands` snippet
└── docs/
    ├── getting-started.md           # step-by-step onboarding
    ├── base-schema.md               # state-table field spec (human-readable)
    ├── base.schema.json             # state-table field spec (machine-readable)
    └── state-machine.md             # transitions & safety rails in detail
```

---

## Status

🚧 **Early / experimental.** Loop design is *harder* than prompt engineering,
not easier — the leverage point just moved. Two people can build the exact same
loop and get opposite results: one uses it to move faster on work they
understand deeply, the other to avoid understanding the work at all. The loop
doesn't know the difference. You do.

**Build the loop. But stay the engineer.**

---

## License

[MIT](LICENSE)
