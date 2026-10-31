---
name: eventmodeling-slicing-event-models
description: "Verify and correct the slices of a completed event model — each slice must be exactly one state-change (a command), one state-view (a read model) or one automation, never combined; spans and splits must be right; every slice named after its element and given a status. Use after completing event modeling to settle slice boundaries and note event dependencies between slices. Do not use for: organizational team structure based on Conway's Law (use eventmodeling-applying-conways-law), or planning before the event model is complete (complete the full model first using eventmodeling-orchestrating-event-modeling). This skill only makes existing elements' slices correct — it never invents new model content."
---

# Slicing Event Models

This step applies the shared element and slice rules in **`eventmodeling-core-rules`** — read it once per session if you haven't already; its "Slices" section defines the state-change/state-view/automation slice types this step verifies, so this step doesn't restate them. Tool names and conventions are in `eventmodeling-core-rules` → `../eventmodeling-core-rules/references/tools.md`.

**Purpose**: Make a completed event model's slices correct and explicit, and note the event dependencies between them.

**When to Use**:
- After completing the full event model (commands, events, read models are in place)
- To settle slice boundaries before implementation
- To see which slices' events other slices depend on

**How slices work here**: in this tool a slice is a first-class part of the sheet's structure — one column or a span of columns — so every column already belongs to a slice. Nothing derives slices from a finished model and nothing "defines" one on a column. Slicing therefore means *verifying and correcting the structure that already exists*: each slice is exactly one state-change, state-view or automation shape; spans and splits are right; slices are named and their status is set.

---

## Core Concept: A Slice Is One Command, One Read Model, or One Automation — Never Combined

See `eventmodeling-core-rules`'s "Slices" section for the definition (state-change / state-view / automation, never combined) and why a feature needing both a command and a read model is two slices, not one.

---

## Slices Already Exist in the Structure

By the time an event model is complete, the slices are already implied by the sheet's structure. This skill's job is to check that the structure says what the model means — not to invent broader groupings.

Walk the sheet slice by slice. Each slice's **subject** is its one:
- **COMMAND** → a `state-change` slice, named after the command
- **READMODEL** → a `state-view` slice, named after the read model
- **AUTOMATION** (a processor and the command it issues) → an `automation` slice, named after the automation (or the command it issues)

**Linked copies are never a slice's subject.** A COMMAND/READMODEL that is a linked copy (`linkedCopy: true`) is a copy of an element elsewhere (see `eventmodeling-core-rules`'s Linked Copies and Slices sections), not a second independently-deployable thing. Judge each slice by its canonical elements only. A column whose only COMMAND/READMODEL is a linked copy must not be a slice of its own; it is covered by the slice it serves (see Step 3).

---

## Step 1: Resolve the Sheet

Slices live on a sheet. If it wasn't given up front, resolve it before doing anything else: `get_model_view` lists the sheets with their slices.

- **Exactly one sheet** → use it automatically, tell the user which one was selected.
- **Multiple sheets** → list them by name and ask which one to slice.
- **No sheets** → stop and tell the user to create a sheet first (see `eventmodeling-orchestrating-event-modeling`).

## Step 2: Enumerate the Slices and What Is in Them

`get_sheet_view` returns the slices (name, status, note, column letters), the columns and lanes, and every element by cell address. For each slice, list what its columns hold:

- its **subjects** — canonical COMMANDs, READMODELs and AUTOMATIONs (processors)
- its **linked copies** (events, read models, screens, external events)
- its screens, events and external events
- its scenarios

Record `{ slice, columns, name, status, commands, readModels, processors, copies }` for every slice. `get_slice_view` gives one slice in full (fields, flows, scenarios) when you need to look closer, and `render_sheet` a picture.

## Step 3: Verify Each Slice's Shape

A slice is correct when it has **exactly one subject** and the shape matches it:

| Slice type | Its subject | Shape across the slice's columns |
|---|---|---|
| state-change | one COMMAND | SCREEN/AUTOMATION → COMMAND → EVENT(s) |
| state-view | one READMODEL | EVENT(s) → READMODEL → SCREEN/AUTOMATION |
| automation | one AUTOMATION and the COMMAND it issues | EVENT(s) → AUTOMATION → COMMAND → EVENT(s) |

(`check_model` reports `slice-mixed-shape` (an error) for a slice that holds more than one subject: a command together with a read model, two commands, two read models, or a screen-issued command beside a free-standing automation. An automation together with the command it issues is one subject, and linked copies are not counted. It reports `empty-slice` for a slice with no elements. Whether the elements across the slice's columns have the shape in the table, and whether a column of copies belongs with the slice it serves, is yours to read.)

Correct what is wrong with the slice tools, never by adding model content:

- **Two subjects in one slice** (a command and a read model, two commands, two read models) → `split_slice` at the column where the second subject begins; each half is its own slice. If the two subjects share one column, `insert_slice` a new slice next to it and `move_element` the second subject (and what belongs to it) there.
- **A slice too narrow for its shape** (its event, or the read model's source events, sit in the neighbouring slice and belong with this subject) → `span_slice` to absorb the column.
- **A column in the wrong slice** → `move_column` (with `toSlice`) to reparent it; `move_slice` to put a whole slice in its right place in the timeline.
- **A column with no subject of its own** (only events, a screen, or linked copies) must still sit inside a slice: let the slice it feeds or serves absorb it — events that feed a read model go in that read model's slice; a screen goes with the command it triggers or the read model it shows; a column whose only COMMAND/READMODEL is a linked copy goes with the slice it serves. Do not name it a slice of its own.
- **A slice that is empty** → it has no element to make a slice of: report it, and remove its column (`remove_target`) only when it is plainly a leftover. Don't leave it unmentioned.

**The todo-list read model of an automation sits in the column before its automation** (see `eventmodeling-designing-automation-chains`), so it is its own state-view slice and the automation + its command is the automation slice. They are never one slice.

## Step 4: Name the Slices

- COMMAND slice → name = the command's name exactly (state-change slice)
- READMODEL slice → name = the read model's name exactly (state-view slice)
- AUTOMATION slice → name = the automation's name, or the command it issues (automation slice)

Rename a slice with `update_slice` (`name`). A split leaves the new slice with a placeholder name — always rename it. Never derive a broader "feature" name that spans several elements; if a slice seems to need one, it is two slices. `check_model` does not check names: read them from `get_sheet_view` and look for a leftover placeholder such as "Slice", or a name that differs from its subject's.

## Step 5: Set the Status

Every slice carries a status (`update_slice`, `status`: draft, created, planned, in-progress, blocked or done). Leave a status a person has set. Where it is unset, set it from what the checks above found: a slice that passes them (one shape, named, scenarios in place) is `planned`; one that doesn't is `draft` until it does. `check_model` does not report an unset status: read each slice's `status` from `get_sheet_view`.

## Step 6 (Optional): Note Dependencies Between Slices

Slices depend on each other only through events — never directly:

- **Event dependency** — slice B's command or read model needs an event that slice A produces.
- **No dependency** — slices work off entirely separate events.

This is useful to surface back to the user (e.g. "`Order Detail` depends on `Place Order`'s `Order Placed` event"). Read it from the flows (`get_slice_view` lists every flow touching a slice, and a linked copy points at its original). Record it in the slice's note (`set_note` on the slice, appended to what is there). Slicing itself does not require planning team allocation, sprint sizing, or effort estimates — that's a separate concern from defining the slices.

**If Steps 3–5 find nothing to correct**, this skill's job is done — there is nothing left to make right. Do not invent new model content here; that is out of scope for a skill whose whole design assumes the model is already complete. Missing content belongs to the modeling steps (see `eventmodeling-orchestrating-event-modeling`).

---

## Quality Checklist

- [ ] Every slice contains exactly one COMMAND (state-change), one READMODEL (state-view), or one AUTOMATION — never a COMMAND and a READMODEL together
- [ ] Linked copies were not counted as subjects, and no slice exists for a linked copy alone
- [ ] Spans and splits are right: no column sits in a slice it doesn't belong to, and no subject is cut in two
- [ ] Slice name matches the command/read model/automation name exactly, and no placeholder name is left
- [ ] Every slice has a status
- [ ] No slice is empty
- [ ] Dependencies are expressed as events only, never as direct slice-to-slice calls or shared state

---

## Reference Documentation

- **[patterns.md](references/patterns.md)** — naming, boundaries, cross-slice communication patterns
- **[examples.md](references/examples.md)** — worked example of checking and correcting the slices of a sheet
