---
name: eventmodeling-adding-next-slice
description: "Decide on and add a genuinely new slice when no existing COMMAND, READMODEL or AUTOMATION is left to slice: the model does not yet suggest an obvious next capability, or every element already sits in a correct slice. Reads the story so far, picks the step that most naturally comes next, and builds it as one slice (one command, one read model or one automation). Use when asked to add the next slice, or when eventmodeling-slicing-event-models finds nothing left to make explicit. Do not use for: making an already-modeled element's slice correct (use eventmodeling-slicing-event-models, which only ever fixes existing elements and never invents new ones), or modeling a whole process from requirements (use eventmodeling-orchestrating-event-modeling)."
---

# Adding the Next Slice

This step applies the shared element and slice rules in **`eventmodeling-core-rules`** (read it once per session if you haven't): it defines the elements, how each is named, the slice shapes and the anti-patterns to reject, so none of that is restated here. Tool names and conventions are in `eventmodeling-core-rules` → `../eventmodeling-core-rules/references/tools.md`.

**Purpose**: given a sheet where every existing COMMAND, READMODEL and AUTOMATION already sits in a correct slice (or where there is no content yet), decide on a plausible next capability and build it as a brand-new slice, with the elements that slice needs.

This is the counterpart to `eventmodeling-slicing-event-models`, which only ever makes *existing* elements' slices correct and rightly refuses to invent new content. Something has to own deciding what comes next. That is this skill, and `eventmodeling-slicing-event-models` points here whenever it finds nothing left to slice.

---

## Core Concept: One Command, One Read Model or One Automation, Never Combined

The slice you add obeys the same rule as every slice (`eventmodeling-core-rules`, Slices, and `eventmodeling-slicing-event-models`). If the capability needs both a command and a read model ("place an order" needs the Place Order command *and* an Order Detail read model), that is **two slices**; build them one at a time, each complete before the next. An automation is likewise two: its todo-list read model is a state-view slice in the column before it, and the automation with the command it issues is the automation slice (`eventmodeling-designing-automation-chains`).

---

## Never Stop at a Question

"I don't know what the next capability is" is not the same as "any guess risks doing the wrong thing." A working domain always has a plausible next step (the next lifecycle stage, an unaddressed affordance on an existing screen, a natural gap in what an entity can do), and a wrong guess costs almost nothing: it is one more slice, easy to rename, rework or remove.

**Recording a question and then adding nothing is not an acceptable outcome of this skill.** It only hands the same empty decision to the next request. A pin or note about the assumption flags it for a person to correct later; it does not replace building the slice. Decide, record the assumption, and build, in the same turn.

---

## Step 1 — Decide on the next capability

**If the person's request names the next capability** (or points at a pin or note that does), use that and go to Step 2.

**If it gives no instruction** (a bare "add the next slice"), do not ask for one; read the story and derive it:

1. `get_model_view` for the sheets and slices, `get_sheet_view` for the sheet in question, and `get_slice_view` on the slices nearest the end to see the elements' fields. Titles and field names are enough; read the slices in order, left to right, because they tell a story (Reserve Table, Cancel Reservation, Check In, ...).
2. `list_pins` with `status: "waiting"` and the slice and lane notes: a waiting question or a note may already name what is wanted next. Read `get_history` for the sheet before adding something that looks missing: a person may have removed it on purpose, and then you ask instead of restoring it.
3. Find what the story is missing next. Candidates, roughly in this order of preference:
   - the natural following **lifecycle stage** of the thing the story is about (reserved, then checked in, then completed);
   - an **unaddressed affordance** on an existing screen: a button or link with nothing behind it, or a screen that shows data no read model yet feeds;
   - a **gap in what the entity can do**: created but never changed, cancelled or looked at in detail; an action with no confirmation view; an outcome nobody is told about;
   - a **role** that never acts or never sees anything (`eventmodeling-core-rules`, Role Catalog: each human role needs at least one command and one read model).
4. Prefer the option that most directly **continues the existing narrative** over one that opens an unrelated thread. The goal is the slice that makes the most sense as *next*, not any plausible slice.
5. Check it against `eventmodeling-core-rules`, Offline-First Thinking and Concrete Over Generic: the capability is a real step in the business, with real data, not a step that exists only because of the software.

If after this several equally reasonable next steps remain, pick the one that continues the story most closely, and write down what you assumed (Step 5). Do not stop.

**An empty sheet** has no story to continue. A sheet starts with a state-view (a read model feeding a screen) or an automation reacting to an existing event, never with an unmotivated command (`sheet-starts-with-command`). If the model has no roles yet or no sheet at all, there is nothing to continue: start with `eventmodeling-orchestrating-event-modeling` instead.

---

## Step 2 — Place the slice

A slice is a column or a span of columns. Name it after its one subject (the command's, read model's or automation's name, exactly) and put it where the story needs it, normally after the last slice on the sheet. Arrows stay in their column or run forward, never back, so the new slice goes **after** the events it builds on.

```
insert_slice { name: "Cancel Reservation", at: <column position after the last slice>, cols: 1 }
```

Use one column unless a second element of the same lane must sit in the same slice; one cell holds one element, so widen the slice (`span_slice`, or `cols`) rather than stacking.

Pick the subject by the type you decided on:

| Slice type | Subject | What goes in it |
|---|---|---|
| state-change | a new COMMAND | the SCREEN (or AUTOMATION) that issues it, the COMMAND, the EVENT it produces |
| state-view | a new READMODEL | the EVENT(s) it is built from, the READMODEL, the SCREEN that shows it |
| automation | a new AUTOMATION | its todo-list READMODEL (in the column before, its own slice), the AUTOMATION, its COMMAND, the EVENT |

If the sheet has several lanes of one kind (one actor lane per human role, say), place each element in the lane of the **specific role** that acts or sees. Never a generic "User". If the role has no lane yet, add one (`insert_lane`) and describe the role in the lane's note (`set_note` on the lane).

---

## Step 3 — Fill the slice, one element at a time

Place each element by cell address and wire it as you go.

- `place_element` for each element in its lane's cell: the COMMAND named as an imperative business intent, the EVENT in the past tense with captured data only, the READMODEL named for the data it shows (`isList: true` if the whole read model is several records), the SCREEN with its design (Step 4).
- `create_flow` between them: SCREEN or AUTOMATION to COMMAND, COMMAND to EVENT, EVENT to READMODEL, READMODEL to SCREEN. Placing never connects anything.
- A fact that lives elsewhere on the model is reused, not repeated: `copy_element` places a linked copy of the earlier event or read model in this slice's column, wired forward. Place a copy only to wire it.
- Every field has a source (`eventmodeling-information-completeness`): a same-named field on a connected upstream element, a mapping (`set_flow_mapping`, or `source` on the field), or `isUserInput` / `isGenerated` where the information starts. A field you cannot source is a gap, not a detail to fill in later: do not invent one. Use real example values (`example`) so the slice says what it is about.
- A read model whose later event changes data an earlier read model already shows is a new read model after that event, as a linked copy of the earlier one; never a backward arrow (`eventmodeling-core-rules`, Updating a Read Model That Already Feeds a Screen).

---

## Step 4 — Give the screen a real design, matching the model

A new SCREEN is never left as a placeholder ("Untitled screen"). Design it in the same call as placing it, or with `set_screen_design`:

1. Look at the other screens on the sheet (`get_screen_design`, or `render_screen` for a picture) and note the established style: layout, tone, recurring components, how buttons are treated. If the model has a theme (`set_screen_theme`), the design stays a few lines of semantic markup that picks it up.
2. Design the new screen from the slice's fields: what the person sees and enters, in the screen's own words, matching that style.
3. Check it with `render_screen`.

The rules for what a screen shows and the fields it carries are in `eventmodeling-storyboarding-events`.

---

## Step 5 — Mark it as a proposal, record the assumption

The capability is your inference, not a decision a person made. Say so on the model:

- set the new slice's status to draft (`update_slice`), so nobody mistakes it for agreed work. A person can promote it;
- when you chose between several reasonable next steps, or assumed something a person might reasonably answer differently, record it: a note on the slice (`set_note`; read the existing note first, since it replaces the text) for what you assumed and why, and a pin (`create_pin`, anchored on the new slice's cells, worded as a question) only where an answer would change the slice. The pin does not replace the slice, which is built either way.

Scenarios are the next step, not part of this one (`eventmodeling-elaborating-scenarios`). Build one only if the person asks. For a command slice `check_model` reports it as having none (`slice-without-scenario`, info); a read-model or automation slice without one is not flagged, so say it is unspecified in your reply.

---

## Step 6 — Check, then report

Run `check_model` on the sheet and fix every error that comes from the new slice (a command with no issuer, a backward flow, a mixed slice, an automation with no todo list) before calling the work done. It checks structure, not whether the capability is the right one; say that if it comes back clean.

Report in the reply:

- the capability you chose and why it is the natural next step;
- the slice's name, type and columns, and the elements in it;
- what you assumed, and any pin left for a person;
- the `check_model` result, and the fact that the slice has no scenarios yet.

---

## Quality Checklist

- [ ] A capability was chosen and a slice was built: the turn did not end on a question alone
- [ ] The slice is exactly one subject (one command, one read model or one automation), named after it exactly
- [ ] It sits after the events it builds on; every connection stays in its column or runs forward
- [ ] Commands name a specific role, never "User"; events are past-tense business facts
- [ ] Every field has a source or is marked where the information starts
- [ ] The screen has a real design matching the model's other screens, not a placeholder
- [ ] The slice is a draft, and the assumption is recorded where a person will see it
- [ ] `check_model` shows no errors from the new slice
