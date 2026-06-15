# Base state-machine schema

The loop coordinates entirely through one Lark Base table. **One row = one
change's full lifecycle.** Create a table with the fields below, then put its
`app_token` and `table_id` into `loop.config.yaml`.

> Column names are up to you — the skills resolve them through `base.fields` in
> `loop.config.yaml`. The names below are the defaults.

## Fields

| Field         | Type           | Who writes        | Purpose                                                                 |
|---------------|----------------|-------------------|-------------------------------------------------------------------------|
| `task`        | Text           | human (kickoff)   | The requirement / task description.                                      |
| `change_id`   | Text           | coder             | OpenSpec change identifier, filled after the spec is written.           |
| `status`      | Single select  | coder / reviewer / human | Lifecycle stage — see options below. Agents only write `done` from `integrating`; the human writes it on kickoff and at acceptance. |
| `owner`       | Single select  | coder / reviewer / human | The relay baton: `coder` \| `reviewer` \| `human`. Cleared (empty) on `done`. |
| `spec_ref`    | Text           | coder             | Spec content or a link to it.                                            |
| `review`      | Text (multi)   | reviewer          | Structured, severity-tagged feedback. Append per round; don't overwrite.|
| `resolution`  | Text (multi)   | coder             | Per-item judgement of feedback + what was changed.                      |
| `round`       | Number         | reviewer / coder / human | Review iteration counter (spec + code reviews). Drives `max_rounds`; the reviewer increments it, and it resets to 0 when the spec is approved (reviewer), when a fresh code loop starts, and when a human bounces a bug back (human) — so each loop gets a full budget. |
| `test_report` | Text (multi)   | coder / human     | Raw machine test/lint output + the human's acceptance notes.            |
| `bug`         | Text (multi)   | human             | Repro steps logged when bouncing a change back from `testing`.          |
| `claimed_by`  | Text           | coder / reviewer  | Optimistic-lock token: an agent writes a unique run token before working a row and re-reads to confirm it won the claim. Cleared on hand-off. |
| `updated_at`  | Auto timestamp | (auto)            | Used by pollers to pick the oldest pending record.                      |

## `status` single-select options

| Option         | Meaning                                              | Typical `owner` |
|----------------|------------------------------------------------------|-----------------|
| `new`          | Fresh task, spec not written yet                     | coder           |
| `spec`         | Spec written, awaiting spec review                   | reviewer        |
| `implementing` | Spec approved, being implemented (or a human bug fixed) | coder        |
| `reviewing`    | Implementation under code review                     | reviewer        |
| `fixing`       | Review feedback being applied                        | coder           |
| `testing`      | Clean — awaiting human acceptance test               | human           |
| `integrating`  | Human accepted — coder commits & archives            | coder           |
| `blocked`      | Circuit breaker tripped — needs a human decision (NOT acceptance) | human |
| `done`         | Committed & archived (written only from `integrating`) | —             |

## `owner` single-select options

`coder` · `reviewer` · `human`

## Recommended views

- **🔴 Needs me** — filter `owner = human`. This is your inbox. Two kinds of row
  land here: `status = testing` (a finished change awaiting your acceptance test)
  and `status = blocked` (the loop couldn't converge and parked it for your
  decision — there may be no working implementation to test yet, so don't treat
  it as acceptance work).
- **🔄 In flight** — filter `owner != human AND status != done`. The loop's
  live activity.
- **✅ Done** — filter `status = done`.

A notification on the "Needs me" view (Base automation → send a Lark
message when `owner` becomes `human`) gives you a clean ping without any
polling on your side.
