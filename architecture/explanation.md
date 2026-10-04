# Module 4 — Genau Multi-Agent Architecture

## The work being decomposed
Genau's post-session processing was, after Module 3, a single subagent
(`error-pattern-analyst`) doing one job well. But the actual app needs more
done after every session ends: grammar mistakes need classifying (existing),
vocabulary production strength needs tracking (new, separate evidence and
separate rules), and something needs to decide what the *next* session should
open with, combining both. These are three genuinely different kinds of
judgment, over different (if overlapping) inputs, so they're decomposed into
three specialized pieces rather than one subagent trying to do all of it.

## Architecture chosen: orchestrator-worker + parallel + sequential (combined)
A pure "parallel agents" design doesn't fit, because `session-planner`
genuinely cannot do its job until both other subagents have finished — it
needs their combined output. A pure "sequential handoff" design doesn't fit
either, because `error-pattern-analyst` and `vocabulary-tracker` have zero
data dependency on each other and running them one after another would just
double the latency for no benefit. So the architecture is a **combination**,
coordinated by an orchestrator:

```
                 Session Transcript
                        │
                        ▼
          SESSION WRAP-UP ORCHESTRATOR (skill)
                 /              \
                /                \
   error-pattern-analyst   vocabulary-tracker        ← parallel, independent
      (subagent)              (subagent)
           │                      │
   patterns.json (upd.)     vocab.json (upd.)
                 \                /
                  \              /
                SESSION PLANNER (subagent)            ← sequential, depends on both
                        │
                        ▼
            next-session-brief.json
                        │
                        ▼
       (read by Live Tutor Agent at next session start)
```

See `genau-architecture.excalidraw` (open at excalidraw.com) for the full
diagram, including the live-session tier and the separate build-time tier;
`preview.svg` is a quick-look image of the same thing.

## Every piece, fully specified

### Live Tutor Agent (existing concept, not built this module)
- **Responsibility**: conducts the real-time voice conversation — asks
  questions, recasts mistakes inline, decides when to supply a word the
  learner doesn't know.
- **Inputs**: the learner's live speech (via STT) and `learner-state.json`
  (today's `next_focus` and `vocab_to_reinforce`) read once at session start.
- **Context**: only the current session's conversation so far — it does not
  need the full pattern or vocabulary history, just today's target.
- **Tools**: STT/TTS, a grammar-rule lookup tool, a vocabulary lookup tool.
- **Outputs**: the spoken conversation itself, plus the finished transcript
  once the session ends (the artifact everything downstream consumes).
- **Dependencies**: depends on the *previous* session's `next-session-brief.
  json` having been written; produces the input every downstream agent needs.
- **Why not a subagent**: this needs to respond within a live conversation
  turn — subagent dispatch overhead and independent context would add
  latency a real-time voice interaction can't afford. It's the main agent.

### error-pattern-analyst (subagent, built Module 3)
Unchanged from Module 3 — see that module's explanation for full detail.
Responsibility: classify grammar mistakes into pattern categories. Inputs:
transcript + `patterns.json`. Tools: Read, Write. Output: updated
`patterns.json`. Dependency: none on the other workers — reads only the
transcript and its own state file.

### vocabulary-tracker (subagent, new this module)
- **Responsibility**: update word-level production-strength ratings from
  what the learner actually produced unprompted this session, keeping
  tutor-supplied words clearly separate from independently-produced ones.
- **Inputs**: the same session transcript + `vocab.json`.
- **Context**: this session's transcript and the running vocabulary history
  only — no grammar-pattern data, since it doesn't need it.
- **Tools**: Read, Write (nothing else — same least-privilege reasoning as
  `error-pattern-analyst`).
- **Outputs**: updated `vocab.json`, plus a short plain-language summary.
- **Dependencies**: none on `error-pattern-analyst` — they read the same
  transcript independently and write to different files, which is exactly
  what makes them safe to run in parallel.

### session-planner (subagent, new this module)
- **Responsibility**: decide the next session's focus and draft its opening
  line, by combining both workers' results — not by reclassifying anything
  itself.
- **Inputs**: the *updated* `patterns.json` and `vocab.json` — it never reads
  the raw transcript directly.
- **Context**: only the two merged state files; narrower than either worker's
  context, since by this point the transcript has already been distilled.
- **Tools**: Read, Write.
- **Outputs**: `next-session-brief.json` (next_focus, vocab_to_reinforce,
  drafted opening line).
- **Dependencies**: hard dependency on both `error-pattern-analyst` and
  `vocabulary-tracker` having finished — this is the one genuine sequencing
  constraint in the system.

### session-wrapup (skill — the orchestrator)
- **Responsibility**: sequencing and failure-handling only. Decides *when*
  to call what; does none of the actual classification or planning judgment.
- **Inputs**: knows where the transcript and the two state files live.
- **Context**: no domain context of its own — it's pure control flow.
- **Tools**: the Task tool, to dispatch subagents (two calls in one message
  for the parallel stage, one call afterward for the sequential stage).
- **Outputs**: a short status report (what ran, what failed, what's next).
- **Dependencies**: triggered by the Live Tutor Agent's session ending.
- **Why a skill, not a subagent**: subagents in this architecture don't call
  other subagents directly — dispatching is something the main thread does.
  A skill is the right artifact for "instructions the main thread follows,"
  which is exactly what orchestration is here.

### Design System Skill (built Module 2 — separate, build-time tier)
Not part of this runtime loop at all. It's used by a developer (or a coding
agent working on Genau's actual UI) when building a new screen, so it sits in
its own tier on the diagram rather than connected to the session pipeline.

## Handoffs in detail
1. Live Tutor Agent's session ends → its transcript becomes the shared input
   for the wrap-up stage.
2. `session-wrapup` skill dispatches `error-pattern-analyst` and
   `vocabulary-tracker` **in the same message** (true parallelism, not just
   back-to-back calls) since neither needs the other's result.
3. Each worker writes its own file independently; no merge step is needed at
   this stage because they don't touch the same data.
4. Once both are confirmed written, `session-wrapup` dispatches
   `session-planner` with both updated files as its explicit inputs — this
   is the one sequential handoff in the system, and it's sequential because
   the data dependency is real, not a design preference.
5. `session-planner`'s output (`next-session-brief.json`) is what the Live
   Tutor Agent reads at the start of the *next* session — closing the loop
   back to the top of the diagram.

## What happens when an agent fails
- **One worker fails** (`error-pattern-analyst` or `vocabulary-tracker`):
  the orchestrator lets the other's result stand, leaves the failed worker's
  file exactly as it was, and logs which one failed. The pipeline still
  completes with partial data rather than stalling.
- **Both workers fail** (e.g. a malformed transcript): the orchestrator skips
  `session-planner` for this run entirely — per `session-planner`'s own
  constraints, invoking it on stale/absent data isn't allowed. Instead, the
  previous `next-session-brief.json` is carried forward unchanged, and the
  next session opens on the same focus as before rather than guessing.
- **`session-planner` itself fails**: same carry-forward rule — the previous
  brief stands, so the Live Tutor Agent never has no brief to read from.

## Implemented this module
- `.claude/agents/vocabulary-tracker.md` and `.claude/agents/session-planner.
  md` — real, working subagent definitions, same format as Module 3's.
- `.claude/skills/session-wrapup/SKILL.md` — the orchestrator logic.
- A full test run (`test/`) chaining all three: the Module 3 transcript and
  updated `patterns.json` feed `vocabulary-tracker` (producing `vocab.after.
  json`) and then `session-planner` (producing `next-session-brief.json`),
  demonstrating the parallel-then-sequential handoff end to end.
