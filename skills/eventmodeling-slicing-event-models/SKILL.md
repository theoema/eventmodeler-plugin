---
name: eventmodeling-slicing-event-models
description: "Identify feature slices directly from a completed event model's sheet and make each slice explicit on it — each COMMAND becomes a state-change slice, each READMODEL becomes a state-view slice, each AUTOMATION becomes an automation slice, spanned and split correctly, named after its element and given a status. Use after completing event modeling to define slice boundaries and note event dependencies between them. Do not use for: organizational team structure based on Conway's Law (use eventmodeling-applying-conways-law), planning before the event model is complete (complete the full model first using eventmodeling-orchestrating-event-modeling), or 'add the next slice' when every existing element already has one (use eventmodeling-adding-next-slice — this skill only makes existing elements explicit, it never invents new ones)."
---

# Slicing Event Models

> Every tool call takes the `modelId` (from `list_models`), and a `sheet` when the model has more than one. Consult [tools.md](../eventmodeling-core-rules/references/tools.md) (in particular its **Structure** section) only if you need to look up a specific tool or field this file doesn't cover — don't load it eagerly.

This step applies the shared element and slice rules in **`eventmodeling-core-rules`** — read it once per session if you haven't already; its "Slices" section defines the state-change/state-view/automation slice types this step derives, so this step doesn't restate them.

**Purpose**: Turn a completed event model's sheet into explicit slice definitions, and note the event dependencies between them.

**When to Use**:
- After completing the full event model (commands, events, read models are in place)
- To make slice boundaries explicit on the sheet before implementation
- To see which slices' events other slices depend on

---

## Core Concept: A Slice Is One Command, One Read Model, or One Automation — Never Combined

See `eventmodeling-core-rules`'s "Slices" section for the definition (state-change / state-view / automation, never combined) and why a feature needing both a command and a read model is two slices, not one.

---

## Slices Already Exist on the Sheet

By the time an event model is complete, the slices are already implied by the sheet's structure. This skill's job is to make them explicit on the sheet — not to invent broader groupings.

In EventModeler a slice is a first-class part of the sheet (one column, or a contiguous span of columns, with a name and a status), so every column already belongs to one: "defining" a slice here means making the slice that holds an element match that element — its span, its split and its name.

Walk the sheet column by column:
- Every **COMMAND** → one `state-change` slice, named after the command
- Every **READMODEL** → one `state-view` slice, named after the read model
- Every **AUTOMATION** → one `automation` slice, named after the automation (or the command it issues)

**Skip any column whose only COMMAND/READMODEL is a linked copy** (`copy_element`) — it's a copy of an element elsewhere (see `eventmodeling-core-rules`'s Linked Copies and Slices sections), not a second independently-deployable thing. Only the original element's own column gets a slice. Every column here belongs to some slice, so a copy-only column can't be left without one: absorb it into the slice it serves (see Step 3) rather than naming it a slice of its own.

---

## Step 1: Resolve the Sheet

The sheet is required for every call below. If it wasn't given up front, resolve it before doing anything else — only names are needed to pick a sheet, and `get_model_view` lists the sheets with their slices:

- **Exactly one sheet** → use it automatically, tell the user which one was selected.
- **Multiple sheets** → list them by name and ask the user which one to slice.
- **No sheets** → stop and tell the user to create a sheet first (`create_sheet`, see `eventmodeling-orchestrating-event-modeling`).

## Step 2: Enumerate Commands, Read Models, and Automations

Use `get_sheet_view` (or existing model knowledge) to list every COMMAND, READMODEL and AUTOMATION (a `processor`) across the resolved sheet. It returns the slices (name, status, note, column letters), the columns and lanes, and every element by cell address, so the column of each element is the letter of its address (`C2.1` sits in column C); `get_slice_view` gives one slice in full when you need to look closer.

Record `{ elementType, name, column }` for every COMMAND, READMODEL, and AUTOMATION, and for each slice its columns, name and status and what its columns hold: its subjects (canonical COMMANDs, READMODELs and AUTOMATIONs), its linked copies, its screens, events and external events, and its scenarios. `render_sheet` gives a picture of the sheet.

Check which columns already have a slice, so you don't create duplicates: every column already belongs to a slice, so compare each element's name with the name of the slice that holds its column.

A column already has a slice if its element's name matches the name of the slice that holds it.

## Step 3: Define slices

For every column from Step 2 that doesn't already have a matching slice, make the slice that holds it match its element, on the slice that already exists:

- COMMAND column → name = command name (state-change slice)
- READMODEL column → name = read model name (state-view slice)
- AUTOMATION column → name = automation name, or the command it issues (automation slice)

Rename a slice with `update_slice` (`name`). A split leaves the new slice with a placeholder name — always rename it, and look in `get_sheet_view` for a leftover placeholder such as "Slice", or a name that differs from its element's. Make one call per slice (`apply_operations` only if the unit must land all-or-nothing).

A slice is correct when it has **exactly one subject**: one COMMAND, one READMODEL, or one AUTOMATION and the command it issues. Linked copies aren't counted. `check_model` reports `slice-mixed-shape` (an error) for a slice that holds more than one subject — a command together with a read model, two commands, two read models, or a screen-issued command beside a free-standing automation — and `empty-slice` for a slice with no elements. It does not check names or status, whether the elements across the slice's columns have the right shape, or whether a column of copies belongs with the slice it serves: read those from `get_sheet_view`. Correct what is wrong with the slice tools, never by adding model content:

- **Two subjects in one slice** → `split_slice` at the column where the second subject begins; each half is its own slice. If the two subjects share one column, `insert_slice` a new slice next to it and `move_element` the second subject (and what belongs to it) there.
- **A slice too narrow for its shape** (its event, or the read model's source events, sit in the neighbouring slice and belong with this subject) → `span_slice` to absorb the column.
- **A column in the wrong slice** → `move_column` (with `toSlice`) to reparent it; `move_slice` to put a whole slice in its right place in the timeline.
- **A column with no subject of its own** (only events, a screen, or linked copies) must still sit inside a slice: let the slice it feeds or serves absorb it — events that feed a read model go in that read model's slice; a screen goes with the command it triggers or the read model it shows; a column whose only COMMAND/READMODEL is a linked copy goes with the slice it serves. Do not name it a slice of its own.
- **A slice that is empty** → it has no element to make a slice of: report it, and remove its column (`remove_target`) only when it is plainly a leftover. Don't leave it unmentioned.

Never use `insert_slice` to give an element that already sits on the sheet its slice — `insert_slice` creates a brand-new empty column with a slice of its own, which would leave the element in its old slice (the shared-column correction above is the one exception, and it moves the element). The name always comes from you — it is never derived automatically from the command/read model/automation.

**If Step 2 finds nothing to slice** (every COMMAND/READMODEL/AUTOMATION on the sheet already has a matching slice), this skill's job is done once the status is set (Step 4) — there is no existing element left to make explicit. Do not invent new model content here; that is out of scope for a skill whose whole design assumes the model is already complete. Invoke `eventmodeling-adding-next-slice` instead — it owns deciding on and creating a genuinely new slice from scratch.

## Step 4: Set the Status

Every slice carries a status (`update_slice`, `status`: draft, created, planned, in-progress, blocked or done). Leave a status a person has set. Where it is unset, set it from what the checks above found: a slice that passes them (one shape, named, scenarios in place) is `planned`; one that doesn't is `draft` until it does. `check_model` does not report an unset status: read each slice's `status` from `get_sheet_view`. It reports a state-change slice with no Given/When/Then scenario as `slice-without-scenario` (info; read-model and automation slices are not flagged); `eventmodeling-assessing-scenario-coverage` judges how complete a slice's scenarios are.

## Step 5 (Optional): Note Dependencies Between Slices

Slices depend on each other only through events — never directly:

- **Event dependency** — slice B's command or read model needs an event that slice A produces.
- **No dependency** — slices work off entirely separate events.

This is useful to surface back to the user (e.g. "`Order Detail View` depends on `Place Order`'s `Order Placed` event"). Slicing itself does not require planning team allocation, sprint sizing, or effort estimates — that's a separate concern from defining the slices. Read the dependencies from the flows (`get_slice_view` lists every flow touching a slice, and a linked copy points at its original). Record them in the slice's note (`set_note` on the slice; it replaces the note, so read it first and write it back with the dependencies appended).

---

## Quality Checklist

- [ ] Every slice contains exactly one COMMAND (state-change), one READMODEL (state-view), or one AUTOMATION — never a COMMAND and a READMODEL together
- [ ] Linked copies were not counted as subjects, and no slice exists for a linked copy alone
- [ ] Spans and splits are right: no column sits in a slice it doesn't belong to, and no subject is cut in two
- [ ] Slice name matches the command/read model/automation title exactly, and no placeholder name is left
- [ ] Every slice has a status
- [ ] No slice is empty
- [ ] No slice was created for an element that already has one
- [ ] Dependencies are expressed as events only, never as direct slice-to-slice calls or shared state

---

## Reference Documentation

- **[patterns.md](references/patterns.md)** — naming, boundaries, cross-slice communication patterns
- **[examples.md](references/examples.md)** — worked example of deriving slices from a sheet
