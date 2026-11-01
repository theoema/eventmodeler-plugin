---
name: eventmodeling-checking-completeness
description: "Step 8 of Event Modeling - Completeness Check. Verify every field has origin and destination. Ensure complete event model before implementation. Use after all scenarios defined. Do not use for: architectural validation against event modeling principles (use eventmodeling-validating-event-models) or elaborating Given-When-Then specs (use eventmodeling-elaborating-scenarios)."
---

# Checking Completeness

This step applies the shared element rules in **`eventmodeling-core-rules`** — read it once per session if you haven't already; it defines what a COMMAND/EVENT/READMODEL/SCREEN/AUTOMATION is, how each is named, and the anti-patterns to reject, so this step doesn't restate them. Tool names and conventions are in `eventmodeling-core-rules` → `../eventmodeling-core-rules/references/tools.md`. The rule for where every field comes from is in `eventmodeling-information-completeness`.

You are in **Critic Mode**: apply every rule strictly, surface every gap, and record findings rather than quietly adding the missing structure.

## Model Context

Before starting, read the current model state to drive the analysis from what is actually there rather than relying solely on conversation context.

1. **Run `check_model` first** (once per sheet, or without `sheet` for the whole model). It already checks, mechanically, much of what this step asks for; don't re-derive those by hand. See "What `check_model` covers" below.
2. `get_model_view` lists the sheets and their slices. For each sheet, `get_slice_view` per slice returns the working detail this step needs: every element with its fields, the flows with their field mappings, and the scenarios. `get_sheet_view` shows the layout, and `render_sheet` an image of it.

Use these results as the source of truth for the completeness check.

**Before treating any elements as duplicates**: check whether they are linked copies. An element with `linkedCopy: true` (and an `originalStickyId`) is an intentional **linked copy** of an original, placed elsewhere for readability, not a modeling defect. When two or more elements share a name and type:
- If any of them is a linked copy, do not report a duplicate. This is expected, not a gap.
- Never propose deleting, suppressing, or "cleaning up" either element in a linked pair. Specifically, never target the original (the one that is not a copy) for removal — copies reference it, and deleting it removes every copy too.
- Only flag same-named elements as an actual duplicate gap when **none** of them is a linked copy — i.e., they are genuinely two independent, unlinked elements describing the same concept. Same-named screens are the exception: they are the instances of one page split by component (`eventmodeling-identifying-outputs` Step 5c), each with its own component marked. `check_model` does not compare names, so look for same-named elements in `get_sheet_view` yourself.

### What `check_model` covers

| Workflow item below | `check_model` reports | Left to you (judgment) |
|---|---|---|
| 1 Field origin & destination, 8 Field traceability | `untraceable-field` (a field no inbound flow can supply), `unsourced-fields` (an element with fields, nothing flowing in and no stated origin) | Whether the *stated* origin is true; destinations that consume a field |
| 2 Command inputs | `command-without-event` (a command that produces nothing), `command-without-issuer`, `command-multiple-issuers` | Whether every input is captured in the resulting event |
| 3 Read model sources | `read-model-without-source`, `read-model-unconsumed` | Whether each displayed field comes from a connected event |
| 4 Slice coverage | `empty-slice`, `slice-without-scenario`, `slice-mixed-shape` | Slice names and statuses (nothing checks them), the rest of slice shape, see `eventmodeling-slicing-event-models` |
| 5 Event stream completeness | `event-disconnected`, `event-without-command` | Missing events, alternative paths |
| 6 System boundaries | — | All of it |
| 7 Workflow step contracts | — | All of it |
| 9 Gaps | `dangling-flow`, `scenario-missing-ref`, `orphan-element` | Duplicates (same name, no link), whether the data already exists elsewhere |

What the table leaves to you, do by hand.

## Recording Findings

Critic Mode: do not fix what you find; record it.
- A finding → `set_note` on the cell that holds the element (a note attaches to a cell, a slice (column) or a lane (row), never to an element itself; a cell holds one element, so the cell is the element). `set_note` replaces the whole note: read the current one first (`get_cell_view`, `get_slice_view`) and append. Start each finding with its kind — `TASK:` for a required fix, `COMMENT:` for a gap that needs clarification — and name the element.
- A gap that needs a decision only the human can make → `create_pin` on the cells concerned, worded as a question. It stays open until answered. A decided failure is never a question — it is a scenario with an error outcome (`eventmodeling-elaborating-scenarios`).
- Findings about a whole slice go on the slice (`set_note` with the column letter or slice name); about a lane, on the row number.

## Workflow

Perform comprehensive completeness check:

### 1. Field Origin & Destination Matrix
For every field in every event, verify its origin (a command, a calculation, or the system) and every destination that consumes it (other events, read models, external systems), marking each field's status as complete or a gap. A full worked example (Order domain) is in `references/examples.md`.

### 2. Check All Commands
Verify every command input is captured somewhere in the resulting event, whether directly or implicitly. A full worked example is in `references/examples.md`.

### 3. Check All Read Models
Verify every field a read model displays is sourced from a connected event. A full worked example is in `references/examples.md`.

### 4. Check Slice Coverage

In this tool every column already belongs to a slice, so a column with no slice cannot occur. The question this check asks instead is whether the slices are *usable*: every slice has a real name (not the placeholder a split leaves behind), a status, and holds exactly one command, one read model or one automation — never combined. Linked copies are exempt: a column whose only COMMAND/READMODEL is a linked copy implies no slice of its own (see `eventmodeling-core-rules`), so it must not be named as one.

Read the slices from `get_sheet_view` (`slices[]`: name, status, columns) and flag any slice that is empty, unnamed, without a status, or mixed. The full shape check, and how to correct it, is `eventmodeling-slicing-event-models`. A full worked example is in `references/examples.md`.

### 5. Check Event Stream Completeness
Verify no "missing" events:

Walk the full sheet end-to-end (creation through terminal state) checking for missing events, and separately verify that alternative/branch paths (cancellations, failures, compensations) are also represented. A full worked example is in `references/examples.md`.

### 6. Check System Boundaries
Verify each system owns events:

For each system/bounded context, verify it owns a coherent, non-overlapping set of events, and that any processor it runs only reacts to events it's entitled to. A full worked example is in `references/examples.md`.

### 7. Define Workflow Step Contracts
Each workflow step is a contract between the previous step and the next. Document preconditions and postconditions:

Treat each workflow step as a contract: document its preconditions (what must already be true, usually the previous step's postcondition) and its postconditions (what event now exists and which fields it carries), so a later step — or a different team — can build against the contract without waiting on the implementation. A full worked example (a three-step Order contract chain) is in `references/examples.md`.

Record a step's contract in the note of its slice (`set_note` on the slice), in its own `CONTRACT:` section, read and appended like any note.

**Why contracts matter for parallel development**: a team can start building the next step immediately against the previous step's postcondition (mocking the event it depends on), instead of waiting for that step's implementation to finish. See `eventmodeling-orchestrating-event-modeling` for how contracts feed project planning.

### 8. Check Field Traceability
Matrix of all fields origin → destination:

Build a matrix of every field crossing Event / Command / Read Model / Processor, confirming each field's presence is accounted for across the whole model. A full worked example is in `references/examples.md`.

### 9. Identify Gaps
Document any missing pieces:

For each candidate gap, check whether the data already exists elsewhere in the model before treating it as missing, and record which gaps were fixed versus which remain outstanding. A full worked example is in `references/examples.md`.

## Output Format

The output of this step is findings on the model (notes and pins, see "Recording Findings") plus a short report in your reply: what was checked, which gaps were found, which remain outstanding, and the readiness verdict. No separate document is produced.

## Quality Checklist

- [ ] Every field has clear origin
- [ ] Every field has identified destinations
- [ ] All command inputs are captured
- [ ] All read models have sources
- [ ] **Every slice has a real name, a status, and exactly one command, read model or automation (linked copies exempt)**
- [ ] Event flow is complete
- [ ] No events are missing
- [ ] System boundaries are clear
- [ ] Alternative paths covered
- [ ] Error paths documented
- [ ] Processors are identified
- [ ] No circular dependencies
- [ ] All scenarios have data sources
- [ ] **Workflow step contracts defined for each step**
- [ ] **Each contract has explicit preconditions**
- [ ] **Each contract has explicit postconditions**
- [ ] **Dependencies between steps documented**
- [ ] **Teams can work in parallel based on contracts**

### CRITICAL: Event vs Read Model Validation
- [ ] **Reviewed every "calculated event"**: Is it a domain fact or pure calculation?
- [ ] **No aggregation events**: Totals, averages, counts are read models, NOT events
- [ ] **Recalculated state identified**: If a value changes multiple times, it's a read model
- [ ] **Processor outputs categorized**:
  - [ ] Facts → Events (e.g., Payment Authorized)
  - [ ] Calculations → Read Models (e.g., Seller Rating)
  - [ ] Notifications → No event/model (info-only)
- [ ] **History tracking correct**: Read models track history in `history[]`, not as separate events

## Completeness Criteria

The model is **complete** when:
- Every field on every element has a source per `eventmodeling-information-completeness`
- Every command input becomes event/state data
- Every read model field has event source
- All state transitions are covered
- Alternative flows are documented
- Error conditions are handled
- System boundaries are clear
- No "magic" data appears without source
- Data flows logically end-to-end
- All stakeholder needs are met
- **Events are facts (immutable domain actions)**; **read models are projections (derived/calculated state)**; **no calculated events exist** (aggregations/totals are read models)
- **Every slice has a name, a status and one shape**, linked copies excepted

## Common Incompleteness Issues

| Issue | Example | Fix |
|-------|---------|-----|
| Missing event | "No event for failure" | Add failure event |
| Orphaned data | "Field in view, not in event" | Add field to event |
| Circular flow | "A needs B, B needs A" | Redesign boundary |
| Missing field | "View needs date, event has none" | Add field to event |
| Unclear origin | "Where does this come from?" | Trace back to source |
| **Calculated event** | Seller Rating Calculated, Inventory Total | **Move to read model** (recalculated state is projection) |
| **False duplicate** | Two elements share a name | Check whether one is a linked copy before reporting — a linked copy is intentional, not a gap |
| **Unusable slice** | A slice with the placeholder name, no status, or a command and a read model together | Name it, give it a status, correct its shape (`eventmodeling-slicing-event-models`) |

## Next Steps

If completeness check passes:
→ Proceed to Step 9 (`eventmodeling-validating-event-models`) — completeness is not the
  final gate; validation, slicing, and documentation (Steps 9–11) still follow before
  the model is ready for implementation.

If gaps found:
→ Return to appropriate step to fix
→ Re-run completeness check
