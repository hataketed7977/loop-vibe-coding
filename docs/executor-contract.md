# Executor contract (reference runtime)

This repo is a **contract**, not a runtime. The skills, `loop.config.yaml`, and
the Base schema describe *what* each role must do; nothing here executes them.
That is by design — any capable agent can play a role — but it means an
implementer (or a reviewer auditing the repo) needs one place that pins down
*how* the contract should be consumed, so two correct readings can't diverge.

This document is that place. It adds **no new rules**; it restates the ones
already in the skills / config / schema in the precise, low-ambiguity form an
executor needs. If anything here ever disagrees with `state-machine.md`,
`base-schema.md`, or the skills, those are the source of truth and this file is
the bug.

## 1. The minimal atomic hand-off

A hand-off is **one Base update** that flips the baton. To avoid a half-written
row another tick could observe, every hand-off MUST write these fields in the
**same update**:

| Field        | Always | When                                                            |
|--------------|--------|-----------------------------------------------------------------|
| `status`     | ✅     | the transition's target status                                  |
| `owner`      | ✅     | the transition's target owner (see §2 for the `none` sentinel)  |
| `claimed_by` | ✅     | **cleared** — you are releasing the row to the next owner        |
| `round`      | ▢      | only when this transition increments or resets it (see §3)      |
| `bug`        | ▢      | cleared by the coder once fixed — and only after it is archived into `resolution` |
| `resolution` | ▢      | coder: append your per-item judgement + what you ran/changed     |
| `review`     | ▢      | reviewer: append structured feedback — never overwrite history    |
| `test_report`| ▢      | coder/human: append raw machine output / acceptance notes        |

Append-only fields (`review`, `resolution`, `test_report`) are appended, never
overwritten — they are the loop's audit trail.

## 2. The `next_owner: none` sentinel

`transitions.integrating.next_owner: none` means **clear the `owner` field**
(leave the cell empty). It does **not** mean write the three-character string
`"none"`.

```
target_owner = transition.next_owner
if target_owner == "none":
    owner = EMPTY          # clear the cell
else:
    owner = target_owner
```

A `done` row has no owner. Writing the literal `"none"` would leave the row
falsely owned and break the `owner = human` / `owner != human` views.

## 3. Who writes `round` — the only writes that exist

`round` drives the `max_rounds` circuit breaker. It has exactly three writers,
each with one job. **No role writes `round` outside this table.**

| Role     | Action            | When                                                       |
|----------|-------------------|------------------------------------------------------------|
| reviewer | **increment +1**  | on **every** review (spec review AND code review)           |
| reviewer | **reset to 0**    | on spec approval (`spec → implementing`, `on_clean`)        |
| human    | **reset to 0**    | on a bug bounce (`testing → implementing`, `on_bug`)        |
| coder    | **reset to 0 only** | self-heal: a bug bounce arrived with a stale non-zero `round` (human/executor forgot the reset). The coder **never increments.** |

The two resets give the spec loop (`new ↔ spec`) and the code loop
(`reviewing ↔ fixing`) each its own full `max_rounds` budget.

## 4. Claim / optimistic-lock pseudocode

Lark Base has no compare-and-swap, so the claim is **last-writer-wins on
re-read**: write a unique token, re-read, proceed only if it is still yours.
Both roles use the identical shape (`me` is `"coder"` or `"reviewer"`).

```
row = oldest_by(updated_at, where owner == me)
if row is None:
    exit            # nothing to do this tick

token = me + ":" + uuid()
write(row, claimed_by = token)
row = reread(row)                       # last-writer-wins settles here

if row.claimed_by != token:
    exit            # a parallel tick won the claim — back off

if row.owner != me or row.status != status_at_claim:
    exit            # a human retook the row in the claim window — drop it

# ... the token is yours and the row is unchanged: do the job for row.status ...
```

`claimed_by` is cleared on every hand-off (§1), so a released row is immediately
claimable by the next owner.

## 5. `integrating` final re-check pseudocode

`integrating` is a long window (test + commit + `openspec archive`); a human can
change their mind inside it. The coder MUST re-read immediately before writing
`done` and bail if the row moved. This is the ONLY status from which `done` is
written, and only because a human already accepted the change.

```
# status == integrating, owner == coder, human has already passed acceptance
run_final_checks()                      # test + lint from AGENTS.md → test_report
commit()
openspec_archive()

row = reread(row)                       # human may have pulled it back
if row.status != "integrating" or row.owner != "coder":
    exit            # do NOT write done — leave the human's new state intact

handoff(row, status = "done", owner = EMPTY, claimed_by = CLEAR)   # see §1, §2
```

The circuit breaker does **not** apply here: `integrating` is post-acceptance
bookkeeping, so it always completes regardless of `round` and is never diverted
to `blocked`.

## 6. Circuit breaker, restated

Before starting the job for the current `status`, whoever holds the baton checks
`round`:

```
if status in {new, spec, reviewing, fixing, implementing-after-review}
   and round >= loop.max_rounds
   and the row is still bouncing (blocking issues remain):
       handoff(status = loop.breaker.next_status,   # blocked
               owner  = loop.breaker.next_owner,    # human
               claimed_by = CLEAR)
       note the non-convergence in review/resolution
       exit
```

`blocked` is the parking lane for non-convergence and is **distinct** from
`testing` (the human acceptance lane). `integrating` is exempt (§5). A fresh
`implementing` arriving from spec approval or a bug bounce starts at `round = 0`,
so it never trips this on entry.
