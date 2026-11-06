---
name: eventmodeling-analyzing-existing-models
description: "Read-only analysis of an existing event model: which sheets and contexts it has, how many elements of each kind, the slice status breakdown, scenario coverage, open questions, and the structural gaps and shapes worth a look. Reads check_model, the views and each slice's coverage, and reports in the reply without changing the model. Use when asked to summarise, assess, audit or get oriented in an existing model, before extending or reviewing it. Do not use for: judging the model against the method rule by rule or correcting it (use eventmodeling-validating-event-models or eventmodeling-validating-event-models-checklist), or tracing every field to its origin (use eventmodeling-checking-completeness)."
---

# Analyzing Existing Models

This skill applies the shared rules in **`eventmodeling-core-rules`** (read it once per session if you haven't): it defines the elements, the slice shapes and the structural shapes this analysis looks for, so none of that is restated here. Tool names and conventions are in `eventmodeling-core-rules` → `../eventmodeling-core-rules/references/tools.md`.

Read the whole event model and produce a structured analysis: what sheets and contexts exist, how many slices are in each state, which slices have scenarios and how well they cover the slice, where the visible gaps are. **This skill is read-only.** It never edits the model, never writes a note and never leaves a pin. The report goes in the reply to the person. If they want a finding to stay on the model afterwards, that is a separate, explicit step (`set_note` for something observed, `create_pin` for a question).

---

## Step 1 — Scope

From the person's request, work out:

| What | How to find it | Default |
|---|---|---|
| The model | the one they name; otherwise `list_models` | the only model, or ask which |
| A context | a named box of sheets (`set_context`) | every context |
| A sheet | a sheet they name | every sheet in scope |

If they named a context or sheet that does not exist, say so and list the ones that do. Do not guess a near match.

---

## Step 2 — Orient

`get_model_view` is the cheapest full picture: the sheets (with their neighbours in `layout`), the contexts and which sheets they hold, every slice with its name, status, columns and coverage percentage (when assessed), and per sheet the number of elements and scenarios. The per-kind inventory, the coverage's `missing` cases and whether it is stale come from `get_sheet_view`. Save it; most of the report comes from here.

**Contexts.** A context is a named box of sheets (`set_context`). A sheet that sits in no box is its own context, named after the sheet. Count slices per context on that basis. Sheets are one journey each and read top to bottom in story order; note a model whose sheets are not in story order, or that has sheets lying on top of each other (`sheets-overlap`).

---

## Step 3 — Run the mechanical checks first

Run `check_model` for the model, or for the one sheet in scope, **before** reading anything by hand. It is the structural pass: a finding is a fact about the grid, and most of the gaps this analysis looks for are already in it. Group the findings by `check` code and severity; keep the target and message of each, because the report names the slice or element, not the code alone.

`check_model` judges nothing. A clean result means the structure is sound, not that the model is right or complete, so say that in the report whenever it comes back clean.

---

## Step 4 — Read what the checks cannot see

Read in tiers, narrowest that answers the question.

- `get_sheet_view` per sheet in scope: the grid with element names and types per cell, flows and scenario titles. Enough for the inventory, the slice shapes and the structural shapes.
- `get_slice_view` only for a slice you need to look closer at: fields, flows with mappings, scenario bodies, notes. Do not read every slice in full on a large model; read the ones the checks or the counts point at.
- `list_pins` with `status: "waiting"`: the questions still waiting on a person. Count them and name the areas they sit on.
- `render_sheet` when a picture helps you see a layout problem within a sheet, `render_board` for how the sheets and contexts sit on the canvas; neither is needed for the counts.

Count **canonical elements only**. A linked copy is the same fact reused, intentional and never a duplicate (only a copy wired to nothing is worth a mention, as `linked-copy-unwired`), so report copies on their own line and do not add them to the inventory.

---

## Step 5 — Analyse

For each context (and the model as a whole), work out the following.

### 5a — Element inventory

Count canonical elements by kind: EVENT (and `external-event`), COMMAND, READMODEL (and how many are list-shaped, `isList: true`), SCREEN, AUTOMATION, SCENARIO. Add the number of linked copies and the number of actor lanes (the roles).

### 5b — Slice status breakdown

Group the slices by status: draft, created, planned, in-progress, blocked, done. Report the count per status. Flag a status that suggests stalled work: a slice that is `blocked`, and a slice that is still draft or created while slices after it are further along. A slice with no status is unset, not draft: report it as unset.

### 5c — Scenario coverage

Coverage has two levels. Report both.

**Presence.** Which slices have at least one scenario, and which have none. `check_model` with `severity: "info"` reports only the command slices without one (`slice-without-scenario`); count the read-model and automation slices without one from the scenario titles in `get_sheet_view`. Of the command slices, which have no rejection scenario at all (`no-error-scenario`); that is only a gap where the command can be refused for a business reason, so read the command before reporting it. Of the todo-list read models, which have no storyline (`todo-list-without-storyline`). Scenarios whose steps carry no example values (`scenario-without-examples`) count as present but weak: say so.

**Assessed coverage.** Each slice may carry a `coverage` assessment in the views: `percentage` (the share of the slice's meaningful use cases its scenarios specify) and `missing` (up to ten meaningful cases still unspecified). Report it where it exists:

- the percentage per slice, and the lowest ones first (below 50 is poor, below 70 needs work);
- for the weakest slices, the `missing` cases in the assessment's own words;
- `stale: true` means a scenario of the slice was added, removed or edited since it was assessed, so call that number old rather than current;
- a slice with no assessment is **not assessed**. Say so. Do not turn presence into a percentage and do not invent one. This skill is read-only and cannot write an assessment; the person can ask for one separately.

### 5d — Structural gaps per slice

`check_model` already reports the shape errors; this step is reading them with judgment. A slice is exactly one subject, and its shape follows (see `eventmodeling-core-rules`, Slices):

| Slice type | Expect | Flag if |
|---|---|---|
| state-change | SCREEN or AUTOMATION, then COMMAND, then EVENT | any of the three is absent (`command-without-issuer`, `command-without-event`) |
| state-view | EVENT(s), then READMODEL, then SCREEN or AUTOMATION | the read model has no source (`read-model-without-source`) or nothing reads it (`read-model-unconsumed`) |
| automation | the todo-list READMODEL before it, the AUTOMATION, its COMMAND, an EVENT | no todo list (`automation-without-todo-list`) |

Run `check_model` once with `severity: "info"`: its default hides the info findings this analysis counts (`slice-without-scenario`, `no-error-scenario`, `todo-list-without-storyline`, `empty-slice`, `unsourced-fields`, the candidates). Also report a slice that holds more than one subject (`slice-mixed-shape`), an empty slice (`empty-slice`), a connection that runs right to left (`flow-backward`), two events in one column (`column-multiple-events`), a sheet that opens with an unmotivated command (`sheet-starts-with-command`) and a state change that follows another with no trigger in between (`state-change-without-trigger`).

Do not report a gap that is plainly intentional: an internal slice with no screen, or the one blank creation form that may start a sheet. Use judgment; report only what looks unintentional given the slice's name. Say what you left out and why in one line if it is a close call.

### 5e — Orphans and loose ends

Elements and connections that lead nowhere: `orphan-element`, `event-disconnected`, `screen-disconnected`, `linked-copy-unwired`, `dangling-flow`, `scenario-missing-ref`. List them by name and sheet. An EVENT with no COMMAND (`event-without-command`) is expected only where it is another system's own timeline in a second swimlane lane.

### 5f — Structural shapes

Check the connection shapes defined in `eventmodeling-core-rules`, Structural Shapes: the bed, the left chair, the right chair and the shelf.

- **The bed** is a real anti-pattern. Report every instance (`screen-multiple-commands`): the screen's name and the commands it fans into.
- **The left chair, the right chair and the shelf** are candidates only (`command-fan-out`, `read-model-fan-in`, `slice-scenario-outlier`, all info). Report one only if, after reading the events, fields and scenarios involved, the count still looks like it is doing more than one job in this domain. A count over a threshold is not a finding on its own.

### 5g — Where the information comes from

`untraceable-field`, `unsourced-fields` and `pii-mismatch` are the field findings; report how many there are and on which elements. For the one or two that matter most (a field a screen shows, a PII field), use `trace_field` to see where its value really starts and say whether the origin is marked or a genuine gap. This is a sample, not a full lineage check; the full pass belongs to `eventmodeling-checking-completeness`.

### 5h — The roles

Read the actor lanes: each human role needs a lane, at least one command and at least one read model. A role that never acts or never sees anything is decorative. A command attributed to a generic "User" is a gap. Say which lanes are thin; do not fix them.

---

## Step 6 — Report

Answer in the reply. Lead with the outcome, then the evidence. No file, no note, no pin. A report in this shape works well; drop a section that has nothing in it, and say "none" for a section that was checked and came out clean.

```
## Model analysis: <model name>
Scope: <model / context / sheet>

### Summary
<2-4 sentences: how mature the model is, the most important gap or risk, one concrete next step>

### Contexts and sheets
- <context or sheet>: <n> slices

### Slice status
| Status | Count |
| draft | x | ... | Total | n |

### Element inventory (per context)
| Kind | Count |
EVENT, COMMAND, READMODEL (n list-shaped), SCREEN, AUTOMATION, SCENARIO; copies: n; roles: n

### Scenario coverage
- <n> of <total> slices have at least one scenario
- Without scenarios: <slice names, max 10, then "and N more">
- Assessed coverage: <lowest slices with percentage and what is missing>; <n> slices not assessed; <n> assessments stale

### Structural gaps
<slice or element, and what is missing or wrong> / none

### Structural shapes
<every bed; a chair or shelf only if it held up> / none

### Information sources
<counts, the traced samples> / none

### Open questions
<n> waiting pins: <areas>
```

Close with what you checked and what you did not (for example, "assessed coverage exists for 4 of 31 slices; the rest is not assessed"), so the person knows how far to trust the numbers. Offer the full list of anything you cut short.

---

## Example — a full model analysis

```
list_models                                  → the model
get_model_view                               → sheets, contexts, slices, statuses, counts, coverage %
check_model                                  → findings, grouped by code
get_sheet_view { sheet: "Order a Table" }    → grid, flows, scenario titles (once per sheet in scope)
get_slice_view { slice: "Cancel Booking" }   → only where a finding or a count points there
list_pins { status: "waiting" }              → open questions
trace_field { element: "...", field: "..." } → origin of the one or two fields that matter
```

If a context was asked for and not found, say so and list the contexts that exist.

---

## Quality Checklist

- [ ] The model was not changed: no note, no pin, no edit
- [ ] `check_model` ran before any hand-checking, and a clean result was reported as "structure sound", not "model right"
- [ ] Copies were counted apart from canonical elements
- [ ] Assessed coverage was quoted from the views, with stale and not-assessed slices called out, never estimated
- [ ] Every gap names a slice or element; an intentional shape was not reported as a gap
- [ ] The bed was reported wherever it occurs; the other shapes only where they held up
- [ ] The report says what was and was not checked
