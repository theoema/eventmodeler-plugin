---
name: eventmodeling-validating-event-models
description: "Step 9 of Event Modeling - Validate the model for completeness and consistency. Ensures events are immutable facts, read models are deterministic projections, and commands are traceable decisions. Identifies gaps before the model is declared done. Use when reviewing a model before it's considered final. Do not use for: the structured 14-check checklist (use eventmodeling-validating-event-models-checklist) or field-level completeness verification (use eventmodeling-checking-completeness)."
---

# Validating Event Models

This step applies the shared element rules in **`eventmodeling-core-rules`** — read it once per session if you haven't already; it defines what a COMMAND/EVENT/READMODEL/SCREEN/AUTOMATION is, how each is named, and the anti-patterns to reject, so this step doesn't restate them. Its `../eventmodeling-core-rules/references/tools.md` maps the vocabulary below to this server's tools.

For validation you treat the Event Model as read only. The only things you are allowed to change are notes and pins.
For critical questions, leave a pin (`create_pin`) on the cells concerned.

For every field in the command, it must be clear where it is coming from.
Either it´s defined in a transitively connected Read Model, or it is marked as "generated" (`isGenerated`; a value typed on a screen is `isUserInput`) in either a screen or an automation.
There should not be any fields without a defined source.

The source can be also determined by looking at the defined Scenarios. Are all Scenarios covered?

## Model Context

Before starting, read the current model state to validate what is actually on the model.

Pick the read by what each check needs:

- **Naming, past tense, duplicates** — only names and types are needed, so read each sheet once with `get_sheet_view` (element names and types per cell, with the lanes and flows). **Attribute names** need `get_slice_view`, which returns the elements with their fields. `get_model_view` lists the sheets and slices.

- **Field-source traceability** (every command field comes from a connected read model or is marked generated) — needs field bodies and dependencies, so use `get_slice_view` per slice (elements with fields, flows with field mappings). `trace_field` follows one field upstream to where it starts and reports whether its origin is user input, generated, or `unsourced`.

- **Scenario coverage** — `get_slice_view` also returns the scenario bodies (given/when/then) of the slice, and the slice's recorded scenario coverage.

- **One cell, or one element, in full** — `get_cell_view` (including the cell's note and its slice and lane notes). **The layout, when it matters** — `render_sheet` gives an image of the sheet.

After validation, post the findings on the relevant elements — `TASK` for critical violations that must be fixed, `COMMENT` for warnings and recommendations. A finding is a note: `set_note` on the cell that holds the element (a note attaches to a cell, a slice or a lane, never to an element itself). `set_note` replaces the whole note, so read the current one first (`get_cell_view`, `get_slice_view`) and append. Start each finding with `TASK:` or `COMMENT:` and name the element. Findings about a whole slice go on the slice (`set_note` with the column letter or slice name). A question only the human can answer is a pin (`create_pin`) on the cells concerned, worded as a question. A decided failure is never a question — it is a scenario with an error outcome (`eventmodeling-elaborating-scenarios`).

## Purpose

Ensures the event model is complete, correct, and internally consistent.

## Workflow

When given an event model, perform comprehensive validation:

### 0. Automated structural pass — run `check_model` first

Before the manual checks below, run the structural checklist once per sheet (pass `sheet`, or omit it for the whole model):

```
check_model { sheet: "<sheet name>" }
```

It returns a compact `findings` list (no element dumps; errors first, each with a stable `check` code, the sheet and the target) covering: backward arrows (`flow-backward`, with the todo-list `EVENT → READMODEL` exception already applied), COMMANDs with zero or multiple issuers (`command-without-issuer`, `command-multiple-issuers`), READMODELs with no inbound EVENT (`read-model-without-source`), and command slices with no Given/When/Then scenario (`slice-without-scenario`; read-model slices are not flagged). Unplaced nodes cannot exist here — every element lives in a cell — and columns with more than one screen are not reported, so check those by hand from `get_sheet_view`. This replaces the per-type scans and the edge spot-checks those manual checks would otherwise need — start from its `findings`, then use the sections below for the semantic checks it can't make (naming, immutability, field-source traceability, scenario coverage depth). An empty `findings` list from `check_model` is necessary but not sufficient — still do the semantic pass.

The full set it reports:

| Area | `check_model` reports |
|---|---|
| Wiring | `command-without-event`, `command-without-issuer`, `command-multiple-issuers`, `screen-multiple-commands`, `automation-without-todo-list`, `screen-disconnected`, `screen-mark-unanchored`, `event-disconnected`, `event-without-command`, `orphan-element`, `linked-copy-unwired`, `dangling-flow` |
| Direction and layout | `flow-backward`, `column-multiple-events`, `sheet-starts-with-command`, `state-change-without-trigger`, `external-event-direct`, `sheets-overlap` |
| Read models | `read-model-without-source`, `read-model-unconsumed` |
| Naming | `event-name-not-past-tense`, `command-name-past-tense` |
| Field sources | `untraceable-field`, `unsourced-fields`, `pii-mismatch` |
| Scenarios | `slice-without-scenario`, `no-error-scenario`, `scenario-missing-ref`, `scenario-stale-value`, `scenario-when-shape`, `scenario-then-shape`, `scenario-on-linked-copy`, `scenario-without-examples`, `storyline-shape`, `todo-list-without-storyline` |
| Slices | `empty-slice`, `slice-mixed-shape` |
| Candidates, worth a look rather than violations | `command-fan-out`, `read-model-fan-in`, `slice-scenario-outlier` (info) |

Errors are the method's hard rules (one issuer per command, flows that read forward, one event per column, one subject per slice, a todo list per automation); warnings are almost certainly a mistake; info is a prompt to look. `check_model` lists warnings and errors by default: pass `severity: "info"` to list the info findings too (`counts` always counts them), and `checks: [codes]` to list only some codes. Linked copies resolve to their originals, so a copy is never reported as a duplicate, a second issuer or a slice of its own.

Not checked by `check_model`; do these by hand from `get_sheet_view`: a command named like a query (`Load Orders`) and an event named for the UI, machinery or a calculation (`Button Clicked`, `Total Calculated`), a generic "User" actor lane, same-named elements that are not linked copies (except screens: same-named screens are the instances of one page, Step 5c of `eventmodeling-identifying-outputs`, so judge them by whether each marks a different component), a read model with no view scenario, whether a scenario's example values make business sense and the slice it sits in, and slice names and statuses. An event with no producing command (`event-without-command`) is expected when it is another system's fact in a second swimlane lane, so judge it by its lane.

### 1. Entity/Timeline Completeness Check

An entity's timeline here is its own event stream: the events in one swimlane lane (the lane named for the entity).

Verify each entity's timeline has:
- Clear name (identity)
- At least one event type
- An initial event (what starts the entity's story)
- State transitions documented

**For each event:**
- Uses past tense (Created, Confirmed, etc.) — `check_model` lints tense mechanically; this check is about whether the chosen name is actually a true domain fact
- Contains **only facts** (no computed fields)
- All data is **immutable**
- Unique semantics (no duplicates)

**For each read model:**
- Can be deterministically derived from events
- No side effects in how it's built

**For each command:**
- Clear input parameters
- Preconditions defined in scenarios
- Resulting events specified (or rejection reason)

### 2. Consistency Checks

- [ ] **Event-Entity Mapping**: Every event belongs to exactly one lane/entity (`check_model` does not compare names: look for the same fact modeled twice without a link)
- [ ] **Single Command Issuer**: Every command is issued by exactly one SCREEN or AUTOMATION — never two. Check each COMMAND's inbound flows; more than one SCREEN/AUTOMATION wired into the same command is a CRITICAL violation (commonly a wiring artifact; `check_model` reports it as `command-multiple-issuers`, and a command with no issuer at all as `command-without-issuer`)
- [ ] **Command Outcomes**: Every command produces events OR documents a rejection (`command-without-event` for the former; the rejection is a scenario with an error outcome)
- [ ] **Deterministic Read Models**: A read model can only be derived one way from its events
- [ ] **Event Immutability**: No event data is ever modified
- [ ] **Naming Consistency**: Are naming patterns consistent?
  - Commands: present-tense verb (Create Order, Confirm Payment)
  - Events: past-tense verb (Order Created, Payment Confirmed)

### 3. Event Modeling Principles Compliance

- [ ] **Events are Facts**: describe what happened, not potential futures
  - "Order May Be Confirmed" →  "Order Confirmed"
  - "Payment Pending" (as an event) →  "Payment Initiated", "Payment Authorized"

- [ ] **Events are Immutable**: no modification of event data
  - "Update Order Created event with new total" →  "Append Order Total Corrected event"

- [ ] **Complete Event Data**: events contain all facts a read model needs to project them
  - Event "Order Confirmed" missing paymentId →  Event includes paymentId

- [ ] **No Computed Fields in Events**: only raw captured facts
  - Order Created includes "totalTax" (computed) →  Includes items + amounts; tax computed in the read model

- [ ] **Deterministic Read Models**: replaying the same events always produces the same read model

### 4. Event Flow Validation

- [ ] **Command → Event Mapping**: clear what each command produces
- [ ] **No Zombie Commands**: commands that never produce events (read-only commands are fine if documented as such) — `check_model` reports `command-without-event`

### 5. Role & Actor Attribution Validation

Verify that every command has explicit actor attribution from the Role Catalog (on a sheet, the Role Catalog is the actor lanes and their lane notes; a command's attribution is the actor lane of the screen or automation that issues it):

- [ ] **Role Catalog exists**: a Role Catalog was defined in Step 1 (eventmodeling-brainstorming-events)
  - CRITICAL: no Role Catalog found — commands have no actor attribution
  - PASS: Role Catalog with human roles and system actors defined

- [ ] **Every command has actor attribution**: no command uses generic "User" (`check_model` does not look at lane names: read them from `get_sheet_view`)
  - CRITICAL: `Create Order` attributed to "User" (which user? Customer? Admin? Seller?)
  - PASS: `Create Order` attributed to "Customer" (specific role from catalog)

### 6. Command Validation

- [ ] **Preconditions Clear**: when can each command execute?
  - "Can only confirm if state is Draft"
  - "Can sometimes confirm"
- [ ] **Rejection Handling**: what happens if a precondition fails?
  - "Reject, no events appended" (a scenario with an error outcome)
  - "Append a rejection/failure event and continue" (if that's the modeled outcome)
- [ ] **Valid State Transitions**: document what state changes are allowed
```text
Draft → Confirmed (Confirm Order)
Draft → Cancelled (Cancel Order)
Confirmed → Shipped (Ship Order)
Confirmed ↛ Draft (invalid)
```

### 7. Read Model Validation

- [ ] **Read Models**: are rich projections shaped for their query
- [ ] **Read Models Optional**: are they needed, or just convenience? (`check_model` reports `read-model-unconsumed`)
- [ ] **Regenerable**: could be rebuilt from events at any time

### 8. Issues & Recommendations Report

Format findings as notes (`set_note`, as described under "Model Context") and give the summary in your reply:

```markdown
## Validation Summary

**Overall Status**:  Ready with recommendations

**Blockers**: 0 critical issues

**Recommended Fixes**:
1. Add missing Order Cancelled event
2. Document all implicit preconditions explicitly

## Next Steps
1. Review recommendations with domain expert
2. Update model with critical fixes
3. Beyond structure: `eventmodeling-reviewing-business-fit` (what the business would question), `eventmodeling-analyzing-stressors` (what could break the design), `eventmodeling-assessing-scenario-coverage` (how much behaviour the scenarios specify)
```

## Common Issues to Flag

| Issue | Pattern | Fix |
|-------|---------|-----|
| Missing cancellation flows | No "Cancelled" events | Add the missing outcome slices |
| Implicit preconditions | "Obviously can't do X" | Make preconditions explicit |
| Orphaned events | Events no one reads | Link to a read model or command |
| No read models | Commands validated against raw event replay with no documented read model | Add a read model documenting what the command actually reads |
| Command issued by multiple things | COMMAND has 2+ inbound SCREEN/AUTOMATION flows | Keep the deliberate same-column issuer, remove the rest via `remove_flow` — as a finding on the cell, not a silent fix |

## Key Principles

See `eventmodeling-core-rules` for the element definitions this validation checks against (events as immutable facts, read models as optional projections, commands as decisions against documented preconditions).

## Success Criteria

Your event model validation is successful when:

- All requirements are captured in events
- Commands clearly trigger events
- Business rules are explicit preconditions (not hidden assumptions)
- Read models serve actual query needs
- Events are immutable facts (past tense, no computed fields)
- All command-to-event mappings are documented
- Critical issues are resolved or documented as known limitations
- A Role Catalog exists with all human roles and system actors, and every command has explicit actor attribution

## Quality Checklist

- [ ] All events are immutable facts (past tense)
- [ ] No computed fields stored in events
- [ ] Read models are derived deterministically from events
- [ ] Commands are checked against documented preconditions
- [ ] Each command either produces events or rejects (no silent failures)
- [ ] **No command has more than one inbound SCREEN/AUTOMATION flow (a command is never issued by more than one thing)**
- [ ] Event causality/command-event mapping is clear
- [ ] State transitions are documented
- [ ] Read models serve specific query needs (or are removed)
- [ ] **Role Catalog exists with human roles and system actors**
- [ ] **Every command attributed to a specific role/actor (no generic "User")**
- [ ] **Every human role has at least one command and one read model**
- [ ] **Permission boundaries from Role Catalog are respected**
