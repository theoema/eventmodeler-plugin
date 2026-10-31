---
name: eventmodeling-interview-protocol
description: Shared procedure for deciding whether to ask the user clarifying questions before an event-modeling step proceeds, and how to record what was decided. Not a step of its own — referenced by other eventmodeling-* skills' Interview Phase section. Do not use standalone.
---

# Interview Protocol

Shared by every event-modeling step that has an "Interview Phase" section. A step's own SKILL.md defines only what is specific to it: when to interview, and its Critical Questions. Everything below is common to all of them.

## When to ask

Missing information, or something ambiguous that this step needs? Ask the user, using the step's own Critical Questions as a guide. Put the questions in the conversation, together, with a short set of options where the answer space is small. Follow any listed follow-up trigger based on the answer.

Already have everything the step needs? Skip straight to the step's own workflow. Don't ask questions whose answers you already have, and don't ask what the model itself already answers (read it first: `get_model_view`, then `get_sheet_view` on the sheets in scope).

**Unless told not to ask** (an autonomous or unattended run, or the user has said not to stop for questions): don't block. Proceed with the most reasonable assumption for anything missing, and say so plainly and visibly, so it can be corrected later. Leave a pin (`create_pin`) on the area the assumption touches, worded as the question you would have asked and the assumption you took. Never silently guess.

## Recording the outcome

Record what was asked and decided, or, if nothing was asked, what was assumed and why, **in the model**, where the person who opens it next will find it. A file outside the model is not read by the next session.

- A decision that applies to one place goes in a note on that place: `set_note` on the cell, slice (column) or lane it concerns.
- A decision that spans the whole sheet (or the whole model, on the first sheet) goes in the note on the sheet's first slice, under a heading for this step. Until that slice exists, keep the record in the conversation and write it into the note the moment the slice does:

```markdown
## <Step Number>. <Step Name> (<skill-name>)
[What was asked and decided — or, if nothing was asked, what was assumed and why]
```

  A note holds one text and a new write replaces it, so read the slice first (`get_slice_view`), keep what is there, and write it back with this step's section added or updated.
- Keep an `## Interview Trail` table at the top of that same note, one row per step, added or updated when the step is done (create it if this is the first step to run):

```markdown
| <Step Number> | <skill-name> | Done | <one-line summary of key outputs> |
```
- Anything still unanswered is a pin, not a note. A note records something settled; a pin waits for an answer.
- Before relying on the trail, check it against the model itself: the model is the source of truth for what exists, the trail for why.
