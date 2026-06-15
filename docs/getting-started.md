# Getting started

This walks you through wiring `loop-vibe-coding` into one repository so two AI
coding agents run as an autonomous coder↔reviewer pair while you only handle the
kickoff and the acceptance test.

## Prerequisites

- A repo that uses [OpenSpec](https://github.com/Fission-AI/OpenSpec) for change
  specs (or is willing to).
- Two AI coding agents that can run skills on a schedule — e.g. **Codex**
  (Automations) and **Claude Code** (`/loop`, cron, or scheduled tasks). Either
  can play either role.
- A Lark **Base** you can create a table in, plus **API access for each agent**.
  The `app_token` / `table_id` in `loop.config.yaml` are only identifiers — each
  agent additionally needs its own Base API credential (e.g. an app/tenant
  access token, or an MCP/Base skill that holds one) to read and write rows.
  Keep that credential in the agent's own environment; never commit it.

## Step 1 — Create the state table

Create a Base table following [`base-schema.md`](base-schema.md). Add
the three recommended views (especially **🔴 Needs me** = `owner = human`).
Note the base `app_token` and `table_id`.

## Step 2 — Drop in the config

Copy [`../loop.config.yaml`](../loop.config.yaml) to your target repo root and
fill in:

- `base.app_token`, `base.table_id`, and adjust `base.fields` if your column
  names differ from the defaults.
- `roles.coder` / `roles.reviewer` — pick which tool plays which role.
- `loop.max_rounds`, `quality_gate.block_levels`, and `openspec.dir` if the
  defaults don't suit you.

## Step 3 — Declare your commands

Append the snippet from [`../templates/AGENTS.section.md`](../templates/AGENTS.section.md)
to your repo's `AGENTS.md` and fill in the real **test / lint / build** commands.
The skills read these at runtime — they are the single source of truth, never
hard-coded.

## Step 4 — Install the skills

Install `skills/loop-coder` into the agent named in `roles.coder`, and
`skills/loop-reviewer` into the agent named in `roles.reviewer`.

## Step 5 — Schedule polling

Make each agent call its skill on an interval (see `loop.poll_hint` for a
sensible cadence; the loop is not latency-sensitive):

- **Codex** — create an **Automation** whose prompt is just: *"Run the
  `loop-coder` skill."* (or `loop-reviewer`), with the desired cadence.
- **Claude Code** — use `/loop` on an interval, a scheduled/cron task, or push
  to GitHub Actions so it keeps running after you close the laptop. The prompt
  is the same one line.

Each tick the agent claims at most one row, does its job, hands off, and exits.

## Step 6 — Run it

1. Create a row: fill `task`, set `status = new`, `owner = coder`.
2. Walk away. The agents will spec → review → implement → review → fix → … on
   their own.
3. When a row lands in **🔴 Needs me** (`owner = human`, `status = testing`),
   do the real acceptance test:
   - **Pass** → set `status = integrating`, `owner = coder`. The coder runs a
     final check, commits, and `openspec archive`s the change, then sets
     `status = done`. (Agents reach `done` only through this post-acceptance
     step — never on their own judgement.)
   - **Bug** → write repro steps in `bug`, set `status = implementing`,
     `owner = coder`, and reset `round = 0` so the fix gets a full budget. It
     re-enters the loop.

## Verify it works

Before you trust the loop unattended, run the five checks in
[`smoke-test.md`](smoke-test.md) — they prove each agent can authenticate, that
concurrent ticks don't double-claim, and that the circuit breaker stops a
runaway loop.

## Cost & sanity notes

- A two-model loop reviewing each other's work burns tokens **faster** than a
  single agent. Keep `max_rounds` tight and the polling interval relaxed.
- `loop-vibe-coding` is leverage, not autopilot. Read what the loop produces.
  The faster it ships code you didn't write, the larger the gap between what
  exists and what you actually understand. Build the loop — but stay the
  engineer.
