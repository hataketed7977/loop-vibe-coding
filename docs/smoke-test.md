# Smoke test — the real two-agent loop

`docs/state-machine.md` describes the design. This file is the checklist for
proving it works with the **real** Codex + Claude Code CLIs, on **your** Base.

The state machine itself (transitions, round counting, circuit breaker, bug
bounce, the `integrating → done` path) is already verified by walking one record
through all transitions. What a single-process walkthrough **cannot** prove, and
what this checklist targets, is the three things that only emerge with two live
agents:

1. each agent can authenticate and read/write the same table on its own;
2. concurrent ticks do not double-claim the same row;
3. scheduled polling keeps the loop moving with no human in between.

## Prerequisites

- One Base table created from `docs/base.schema.json`, its `app_token` /
  `table_id` filled into the `loop.config.yaml` of **both** agents' repos. These
  two values are only identifiers.
- **Each agent's own Base API credential** in its own environment (an app/tenant
  access token, or an MCP/Base skill that holds one). Never commit it.
- `roles.coder` and `roles.reviewer` set — e.g. `coder: codex`,
  `reviewer: claude-code`.

## Checklist

### 1. Each agent authenticates independently

Run `loop-coder` once in Codex and `loop-reviewer` once in Claude Code. Each must
be able to read the table (a dry `record-get` is enough). If one can read and the
other cannot, the failure is that agent's credential — not the config.

### 2. One change runs end to end, unattended

Create a row with `status = new`, `owner = coder`. Schedule both agents to poll
(Codex Automation / Claude Code `/loop`; interval per `loop.poll_hint`, ~300s).
Walk away. Expect the row to move spec → review → implement → review on its own
and come to rest in **🔴 Needs me** (`owner = human`, `status = testing`).

### 3. Concurrent ticks do not double-claim

Schedule both agents to tick at (almost) the same instant to force a race. Watch
`updated_at` and the `review` / `resolution` history: a single row must never be
rewritten by both agents in the same window. This exercises the `claimed_by`
optimistic lock (write a unique run token → re-read → only proceed if it's still
yours) in the claim step.

### 4. The human's two exits both work

- **Accept** → set `status = integrating`, `owner = coder`. Confirm the coder
  runs its final check, commits, `openspec archive`s the change, then sets
  `status = done` and clears `owner`. This is the only path to `done`.
- **Bug** → write repro steps in `bug`, set `status = implementing`,
  `owner = coder`, `round = 0`. Confirm the coder re-enters, reads the `bug`
  field first, and fixes rather than re-implementing from scratch.

### 5. The circuit breaker actually stops

Temporarily set `loop.max_rounds = 2` and feed a change that keeps bouncing.
Confirm that once `round` hits the cap the agent stops and parks the row in
`status = blocked`, `owner = human` (the circuit-breaker lane — NOT `testing`)
instead of looping forever and burning tokens.

## What "pass" means

Steps 1, 3 and 5 are the ones a single-process walkthrough can't give you —
treat them as the real signal. If all five pass, the loop is wired correctly and
safe to leave running on a relaxed cadence.
