# Module 3 — Error Pattern Analyst subagent

## What it does
After a Genau tutoring session ends, this subagent reads the full conversation
transcript and the persistent error-pattern history, decides which mistakes
the learner actually made (as opposed to mistakes the tutor caught before they
happened, or gaps the tutor filled in for them), classifies each one against
the existing pattern taxonomy or proposes a new category when a mistake is
genuinely distinct, updates the pattern-history file, and recommends what the
next session should focus on. This is the process behind the "Patterns" and
opening-screen moments in the original Genau prototype — it's what has to run
for "What keeps slipping" and "Let's work on the dative today" to be true
statements rather than guesses.

## Why this needs to be a subagent, not a skill or an inline prompt
- **Independent reasoning.** Deciding whether "ich habe mein Geldbeutel
  vergessen" is the same underlying pattern as an existing dative-preposition
  category, a brand-new accusative-case category, or a one-off slip not worth
  tracking yet is a judgment call made against the specifics of that sentence
  — not a fixed, rule-following process like the Module 2 design-system skill.
- **Dedicated, bounded context.** It needs the entire session transcript plus
  the full pattern history, and nothing else — not the live tutoring
  conversation, not unrelated app state. Keeping that analysis isolated means
  it can't accidentally leak session-specific noise back into the live tutor's
  context, and the live tutor doesn't have to carry the weight of a full
  classification pass mid-conversation.
- **Independent operation.** It runs *after* the conversation is already over,
  asynchronously, and hands back a finished artifact (the updated file) — it
  is never invoked mid-sentence the way the live tutoring agent is.

## When it's called
Once per completed tutoring session, immediately after the session ends —
never mid-conversation, and never on a partial transcript. In the real app
this would be triggered automatically when a session closes; triggering it
manually mid-conversation is explicitly against its calling conditions
(see the `description` field in the agent file, which Claude Code uses to
decide when to delegate to it).

## What context it receives
Exactly two inputs, nothing more:
1. The full transcript of the session that just ended (speaker-labeled).
2. The current `patterns.json` — the running history of error categories,
   counts, and examples.

It does not receive the live tutoring conversation, the app's UI state, or
any other session's transcript directly — only what's already been persisted
to `patterns.json` from prior runs. This is the "dedicated context" piece:
narrow inputs, narrow responsibility.

## Tools it's given
`Read` and `Write` only. It never needs the network, voice, or any tool
beyond reading the two input files and writing the updated pattern file —
giving it anything broader would violate its own bounded responsibility.

## Test run
`test/transcript-session19.md` is a sample transcript constructed from the
same error types shown in the Module 2 prototype. Running the subagent against
it and `test/patterns.before.json` produces `test/patterns.after.json` (the
updated history — note the new "Accusative case for direct objects" category
it correctly split out rather than folding into dative) and
`test/session19-summary.md` (its plain-language report, including the
assisted-vocab distinction and its next-focus recommendation).

## Constraints worth calling out
- Never deletes or silently merges a category — ambiguity is reported, not
  resolved by guessing.
- Never invents a German correction it isn't confident is actually correct.
- Explicitly keeps "assisted" vocabulary (words the tutor had to supply)
  separate from error patterns — a gap in production isn't the same thing as
  a mistake, and conflating them would make the Patterns screen dishonest
  about what the learner can and can't yet do unassisted.
