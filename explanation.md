# What this skill simplified

**The repeatable task.** Building the original Genau prototype meant creating five
phone-frame screens that all share one visual system — same color tokens, same
phone chrome, same orb component, same caption voice — but with different content
per moment (opening prompt, live correction, English fallback, error patterns,
vocabulary). Every future screen request (a settings screen, an onboarding step,
a milestone screen, a new correction type) would repeat that same setup work:
re-derive the palette, re-build the phone frame, decide how the orb should look,
match the caption tone — all before getting to the actually-new content.

**What the skill captures.** `SKILL.md` fixes the parts that shouldn't change
(the five color tokens and what each one *means* — blue for the system, plum for
anything the tutor supplied rather than the user producing it themselves, amber
for rule explanations, green for positive/listening states — plus the phone
chrome, orb states, and the five established moment types) and defines exactly
what a new request needs to supply: moment type, purpose, content, and tab state.
It also documents nuances that aren't obvious from looking at the file once
(e.g. plum vs. blue is the product's core trust signal and must never be
conflated; list screens never show the orb; long German lines drop to the
smaller size).

**Demonstration.** I used the skill to generate a screen that wasn't in the
original file: what the user sees after getting the same grammar point right
three times in a row (`example-streak-screen.html`). It reuses the exact phone
chrome and CSS tokens from the system, reuses green rather than inventing a
"celebration" color, and treats the milestone as an extension of the existing
pattern-tracking logic (deprioritizing a topic, naming what's next) rather than
adding a gamification layer that wouldn't fit the product's restrained tone.

**Before vs. after.** Without the skill, a new screen request risks visual drift
(a slightly different radius, an ad-hoc new color for "success", a caption that
reads as marketing copy instead of product reasoning). With the skill, a new
screen is fully specified by four inputs and comes out pixel-consistent with the
rest of the prototype on the first pass — which is the actual point of turning
this into a skill rather than re-designing from the reference file each time.
