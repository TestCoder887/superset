# Automation Observability Log

This document describes the event-logging convention used to track issue remediation work in
this fork — both work done manually (in a live Cloud Agent session) and work done via
event-driven Automations.

## Why this exists

Individually, every PR, issue, and automation run in this repo already has a record somewhere
in GitHub's UI. But nothing connects them into a single timeline, and nothing distinguishes
"the agent said it worked" from "a human confirmed it worked." This log exists to answer three
questions that GitHub's native views don't answer well on their own:

1. **Status** — what state is every issue in right now (open / in progress / blocked / merged)?
2. **Success / failure** — of the work that ran, what merged cleanly, what needed human
   correction, and what failed outright?
3. **Progress / throughput** — how long does work take to move through the pipeline, and how
   is that trending over time?

## The unit of the log: state transitions, not issues

Each line in `.github/automation-log.jsonl` is **one event** — one state transition for one
issue — not a single summary row per issue. This is deliberate: a snapshot ("issue #4: merged")
can always be derived from a transition log, but a transition log can never be reconstructed
from a snapshot. Keeping every transition preserves the ability to measure latency between
steps and to see exactly where in the pipeline a problem was introduced or caught.

## Where data comes from — manual today, partially automatable later

Every event in this log today is written manually, by a human, after directly observing what
happened (reading a diff, reading an agent's report, confirming a merge). This is intentional:
the most important event type in this log — `review_finding` — is a human judgment that no
automated step can honestly self-report. Purely mechanical, independently-verifiable events
(e.g. `trigger_fired`, `pr_opened`) are reasonable future candidates for automated logging
(e.g. via a GitHub Action reacting to PR/issue webhooks), but `review_finding` and any
verification-of-claim event should remain human-authored indefinitely, to avoid the log
becoming a place where automation grades its own homework.

## Schema

Each line is a single JSON object with these fields:

| Field | Type | Description |
|---|---|---|
| `timestamp` | ISO 8601 string | When the event occurred |
| `issue_num` | integer | The issue number this event relates to (in this fork) |
| `event_type` | string | See Event Types below |
| `category` | string | `bug` \| `bug/cosmetic` \| `dependency` \| `code-quality` \| `security` \| `test-migration` \| `infra` |
| `trigger_type` | string | `human` \| `agent_manual_session` \| `agent_automation` |
| `detail` | string | Free-text description of what happened at this event |
| `pr_num` | integer or null | Associated PR number, once one exists |

## Event types

Events generally occur in this order, though not every issue passes through every event
(e.g. a manually-fixed issue has no `trigger_fired` event; an issue caught at pre-flight and
halted has no `diagnosis_stated` event).

| Event | Meaning |
|---|---|
| `issue_created` | Issue filed in this fork (may reference an upstream issue) |
| `trigger_fired` | An automation's trigger condition was met (label added, comment posted, etc.) — automation-driven work only |
| `preflight_started` | Step 1 pre-flight check began: checking for an existing linked PR and any unresolved maintainer objection to its approach |
| `preflight_result` | Outcome of the pre-flight check — e.g. no conflict found, or a stale/already-fixed target was discovered, or a disputed upstream approach was found and work was halted |
| `diagnosis_stated` | Root-cause hypothesis stated, with file/line citations, before implementation |
| `pr_opened` | A PR was opened (draft or otherwise) |
| `human_review_started` | A human began reviewing the PR/diff |
| `review_finding` | A human found something during review — could be confirming correctness, or flagging a defect (e.g. a regression, a stale description, an unverified claim) |
| `merged` | PR merged cleanly, no corrective action needed first |
| `merged_after_correction` | PR merged, but only after a human-driven correction (e.g. cherry-pick, rebase, manual fix) following a `review_finding` |
| `closed_unmerged` | PR closed without merging (e.g. superseded by a corrected PR, or abandoned) |
| `blocked_pending_human` | Automation halted per its own instructions (e.g. Step 1's escape hatch) and is waiting on a human decision |

## Why pre-flight events matter most

Most event types represent mechanical execution — work getting done. `preflight_result` is
different: it represents the system's judgment about *whether to proceed at all*, and it is
the step most directly responsible for the safety properties this pipeline depends on (not
reproducing a disputed upstream fix; not redoing already-completed work). The rate at which
pre-flight actually changes downstream behavior (versus passing through as a no-op) is the
single most informative metric in this log for evaluating whether the system's judgment layer
is doing real work.

## Adding an entry

Append one JSON line per event to `.github/automation-log.jsonl`. Do not rewrite or reorder
existing lines. Example:

{"timestamp": "2026-08-14T11:15:00Z", "issue_num": 4, "event_type": "merged", "category": "bug/cosmetic", "trigger_type": "human", "detail": "merged after correction via cherry-pick onto clean branch", "pr_num": 6}
