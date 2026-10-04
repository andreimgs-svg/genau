---
name: session-wrapup
description: Run this whenever a Genau tutoring session has just ended and its full transcript is ready, to update the learner's grammar-pattern and vocabulary history and decide the next session's focus. This is the orchestrator step — it coordinates the three specialized subagents rather than doing their classification work itself. Not for mid-conversation use.
---

# Session Wrap-Up Orchestrator

This is not a worker — it is the coordination logic the main thread follows
after a Genau session ends. It owns sequencing and failure-handling; the
actual judgment calls (classifying mistakes, tracking vocabulary, picking the
next focus) belong entirely to the three subagents it calls.

## When this runs
Immediately after a tutoring session ends and its transcript is finalized.
Triggered once per completed session — never mid-conversation, never on a
partial transcript.

## Step by step

1. **Locate inputs**: this session's full transcript, the current
   `patterns.json`, and the current `vocab.json`.

2. **Dispatch the two independent workers in parallel.** `error-pattern-
   analyst` and `vocabulary-tracker` both only read the transcript and their
   own state file, and each writes to a different file — neither depends on
   the other's output, so call both in the same message (two Task-tool calls
   together) rather than one after another. Running them sequentially would
   only add latency with no benefit, since they can't conflict.

3. **Handle a single-worker failure.** If one subagent errors out or produces
   no output, do not block the other: let the other worker's result stand,
   leave the failed worker's file exactly as it was before this run, and
   record in the session's wrap-up log which worker failed and why. A
   transcript-formatting issue in one subagent's input has no bearing on the
   other's ability to do its job.

4. **Handle a double failure.** If both workers fail (e.g. the transcript
   itself is unreadable or malformed), skip directly to step 6 with no
   update — do not invoke `session-planner` on stale or absent data per its
   own constraints; it will carry the previous brief forward unchanged.

5. **Dispatch `session-planner` sequentially**, once both (or whichever
   succeeded) worker files are confirmed written. This step is sequential by
   necessity — the planner's whole job depends on having both updated files
   to read, which is exactly why it cannot run in parallel with the workers
   that produce them.

6. **Report**: a short summary back to whoever triggered this (the app
   backend, or a human reviewing it) — what changed, which workers succeeded
   or failed, and what the next session will open with. Don't let any of the
   three subagents' verbose internal reasoning leak into this summary; keep
   it to the outcome.

## Why this is an orchestrator-worker + parallel + sequential combination
- **Orchestrator-worker**: this skill never classifies a mistake or tracks a
  word itself — it only decides calling order and what to do when something
  fails. All the actual reasoning is delegated.
- **Parallel**: `error-pattern-analyst` and `vocabulary-tracker` have no data
  dependency on each other, so they run concurrently to cut latency in half
  on that stage.
- **Sequential**: `session-planner` has a hard data dependency on both
  parallel workers' output, so it cannot start until they finish — this is a
  genuine ordering constraint, not an arbitrary choice.

## Failure philosophy
A partial result (one worker succeeded) is always preferred over no result.
A missing or corrupted upstream file is never silently papered over with a
guess — every subagent in this chain is built to say "no change" or "carry
forward the previous state" rather than invent data it doesn't have.
