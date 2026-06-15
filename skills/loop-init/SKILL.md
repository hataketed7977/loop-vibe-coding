---
name: loop-init
description: >-
  One-shot initializer for loop-vibe-coding. Run this once per project to set up
  the whole loop with zero manual steps: it creates the Feishu Bitable
  state-machine table (fields, single-select options, and views) from
  docs/bitable.schema.json, writes the resulting app_token / table_id back into
  loop.config.yaml, fills in the roles mapping, and adds the `## Loop Commands`
  section to the repo's AGENTS.md. Designed to be AI-friendly: an agent can run
  it end to end from a single instruction like "initialize loop-vibe-coding".
  Use when a user wants to bootstrap loop-vibe-coding in a repository and would
  rather have the AI do the setup than do it by hand.
---

# loop-init

You are setting up `loop-vibe-coding` in a project so the user can immediately
start using the coder↔reviewer loop. Do the whole setup yourself; only ask the
user for things you genuinely cannot determine. Be friendly and report what you
did at the end.

## What you will produce

1. A Feishu Bitable table that acts as the loop's state machine.
2. A filled-in `loop.config.yaml` in the repo root.
3. A `## Loop Commands` section in the repo's `AGENTS.md`.

## Step 0 — Gather inputs (ask only if missing)

Determine, in order of preference (config file > repo files > ask the user):

- **Where the table should live**: a Bitable app (base) the user can write to.
  If the user has an existing base, ask for its `app_token`; otherwise create a
  new base named `loop-vibe-coding`.
- **Role mapping**: which tool plays `coder` and which plays `reviewer`. Default
  to `coder: codex`, `reviewer: claude-code` if the user has no preference.
- **Project commands**: the test / lint / build commands. Try to infer them
  from the repo (package.json scripts, Makefile, pyproject, go.mod, CI config).
  Only ask if you cannot find them.

## Step 1 — Create the state table

Read `docs/bitable.schema.json` (the machine-readable spec). Using whatever
Feishu/Lark Bitable capability you have (an MCP server, a Bitable skill, or the
OpenAPI directly), create the table EXACTLY as specified:

- Create each field with the given type. For `single_select` fields (`status`,
  `owner`) create every option listed, in order.
- `updated_at` must be a last-modified-time field (auto).
- `round` is a number field.
- Create the three views with their filters (`Needs me`, `In flight`, `Done`).

Capture the resulting **base app_token** and **table_id**.

> If you have no Bitable API access at all, STOP and tell the user you need it
> (a Bitable MCP/skill or API token), and show them `docs/bitable-schema.md` so
> they can create the table manually. Do not fake success.

## Step 2 — Write loop.config.yaml

If `loop.config.yaml` does not exist in the repo root, copy the template from
this repository. Then set:

- `base.app_token` and `base.table_id` to the values from Step 1.
- `base.fields` — only change entries whose actual column name differs from the
  default; otherwise leave them.
- `roles.coder` and `roles.reviewer` to the chosen tools.
- Leave `loop.max_rounds`, `quality_gate`, and `openspec.dir` at defaults unless
  the user asked otherwise.

## Step 3 — Update AGENTS.md

Ensure the repo's `AGENTS.md` contains a `## Loop Commands` section (use
`templates/AGENTS.section.md` as the shape). Fill in the real test / lint /
build commands you inferred in Step 0. If `AGENTS.md` does not exist, create it
with at least this section. If the section already exists, leave the user's
values intact unless they are empty placeholders.

## Step 4 — Verify

- Re-read `loop.config.yaml` and confirm `app_token`, `table_id`, `roles`, and
  the field map are all populated (no remaining `XXXX` placeholders).
- Confirm the table has all 11 fields and 3 views.
- Do a dry read of the table to confirm the agent's credentials can access it.

## Step 5 — Report

Tell the user, concisely:
- the base/table that was created (with a link if available);
- the role mapping;
- the commands written into AGENTS.md;
- and the exact next action: *"Create a row, set status=new, owner=coder, then
  schedule each agent to run its loop skill (loop-coder / loop-reviewer)."*

Point them to `docs/getting-started.md` for the scheduling step (Codex
Automation / Claude Code /loop).

## Guardrails

- Never invent an `app_token` or `table_id`. If you couldn't create the table,
  say so.
- Never put secrets (tokens) into committed files. Config holds only the base
  app_token and table_id, which are identifiers, not credentials.
- Be idempotent: if a table or config already exists, reuse it rather than
  creating duplicates; report what you reused.
