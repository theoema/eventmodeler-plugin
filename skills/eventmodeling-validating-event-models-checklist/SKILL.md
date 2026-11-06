---
name: eventmodeling-validating-event-models-checklist
description: "Validate an event model against 14 structural checks across 7 phases. Identifies notation anti-patterns and confirms the model is internally consistent. Use when reviewing an event model for readiness or after completing event modeling steps. Do not use for: reviewing incomplete or in-progress models (use eventmodeling-validating-event-models), or for elaborating new scenarios (use eventmodeling-elaborating-scenarios)."
---

# Event Model Validation Checklist Skill

This step applies the shared element rules in **`eventmodeling-core-rules`** — read it once per session if you haven't already; it defines what a COMMAND/EVENT/READMODEL/SCREEN/AUTOMATION is, how each is named, and the anti-patterns to reject, so this step doesn't restate them.

Tool names and conventions are in [tools.md](../eventmodeling-core-rules/references/tools.md).

**Purpose**: Validate any event model against 14 structural checks across 7 phases, plus two final questions. Identifies notation anti-patterns and confirms the model is internally consistent.

**Applies To**: Any domain - e-commerce, banking, SaaS, marketplace, healthcare, etc.

**When to Use**:
- After completing Step 2 (Event Plot) of the event modeling workflow, as an early structural check
- Alongside Step 9 (Validate), as an optional second pass before declaring the model complete
- When reviewing an existing event model
- When suspicious of structural issues in the model

**What It Does**:
1. Reads the current model (EVENT, COMMAND, READMODEL elements) as input
2. Runs `check_model` first, then systematically applies the 14 validation checks across 7 phases, plus the two final questions
3. Identifies notation anti-patterns (calculations modeled as events, events mixed into the wrong entity's timeline, etc.)
4. Verifies read model/event distinction
5. Confirms every event and command traces cleanly
6. Returns pass/fail verdict with evidence

---

## Model Context

Read the current model before running the checklist:

- **Name/type checks** (Phase 1.1 timeline ownership by name, Phase 2.3 past tense, duplicate names) — `get_sheet_view` once per sheet: every element's name and type by cell, with the lanes, flows and scenario titles. `get_model_view` lists the sheets.

- **Field-level checks** (Phase 2.1/2.2 computed or mutable event data, Phase 3 read-model sources, Phase 5 traceability, Phase 7 structural shapes via flows) — `get_slice_view` per slice: elements with fields and flows, no scenario bodies needed.

- **Business rules / scenarios** (Phase 4) — `get_slice_view` also returns the scenario bodies.

Slice names come from `get_model_view`.

An entity's timeline in this checklist is its own event stream: the events in one swimlane lane (the lane named for the entity).

Also run `check_model` with `severity: "info"` once per sheet up front (or for the whole model; the default lists only warnings and errors, and several codes below are info) — several checklist items (Command With Multiple Issuers in Phase 1: `command-multiple-issuers`, backward-arrow / event-flow in Phase 6: `flow-backward`, read-model-source distinctions in Phase 3: `read-model-without-source`) have a purely structural half the tool answers directly, so start each of those from its `findings` and spend the manual effort on the semantic judgement the tool can't make. A row with no code in the table below is all by hand.

## What `check_model` Covers

| Check | `check_model` reports | Left to you (judgment) |
|---|---|---|
| 1.1 Each event belongs to one entity/timeline | — (nothing compares names: look for the same fact modeled twice, unlinked) | Whether the lane really is one entity's story |
| 1.2 Each command is issued by exactly one thing | `command-multiple-issuers`, `command-without-issuer` | — |
| 2.1 Events are facts, not calculations | — (nothing checks names for UI, machinery or calculation words) | Is it a fact or a recalculated value? Are any fields computed? |
| 2.2 Event data is immutable | — | All of it |
| 2.3 Past-tense names | `event-name-not-past-tense` | Is the chosen name a true domain fact? |
| 3.1 A read model is not an event timeline | (a read model cannot be placed in a swimlane lane) | Whether something modeled as an event is really a projection |
| 3.2 A read model has a natural query | `read-model-unconsumed`, `read-model-without-source` | What question it answers |
| 4.1 Command preconditions documented | `slice-without-scenario`, `no-error-scenario`, `scenario-when-shape`, `scenario-then-shape` | Whether the scenarios state the preconditions explicitly (`eventmodeling-assessing-scenario-coverage` rates how much of a slice they cover) |
| 4.2 Event preconditions explicit | — | All of it |
| 5.1 Input → Event → Read Model traceability | `untraceable-field`, `unsourced-fields`, `read-model-without-source`, `command-without-event` | Whether the stated origins are true (`eventmodeling-tracing-data` follows one field through the model) |
| 6.1 No impossible event sequences | `sheet-starts-with-command`, `state-change-without-trigger`, `flow-backward`, `column-multiple-events`, `event-without-command` | Whether the sequence can happen in this business |
| 6.2 No event in two entities' timelines | — (look for the same fact modeled twice, unlinked) | Naming/boundary mistakes behind a "shared" event |
| 7.1 No SCREEN wired to more than one COMMAND | `screen-multiple-commands` | — |
| 7.2 Fan-out/fan-in outliers reasoned about | `command-fan-out`, `read-model-fan-in`, `slice-scenario-outlier` (candidates only: more than two resulting events, more than three source events, a slice well above its sheet's median scenario count) | Whether the outlier is justified by the business |
| Final questions 1–2 | — | All of it |

## Recording Findings

After the checklist, record what you found rather than quietly fixing it.
- A finding → `set_note` on the cell that holds the element (a note attaches to a cell, a slice (column) or a lane (row), never to an element itself; a cell holds one element, so the cell is the element). `set_note` replaces the whole note: read the current one first (`get_cell_view`, `get_slice_view`) and append. Start each finding with its kind — `TASK:` for a failed check that must be fixed, `COMMENT:` for a warning or recommendation — and name the element.
- A question only the human can answer → `create_pin` on the cells concerned, worded as a question. It stays open until answered. A decided failure is never a question — it is a scenario with an error outcome (`eventmodeling-elaborating-scenarios`).
- Findings about a whole slice go on the slice (`set_note` with the column letter or slice name); about a lane, on the row number.

## Validation Phases (Domain-Agnostic)

### Phase 1: Event & Command Ownership Validation (2 checks)
- Check 1.1: Each event belongs to exactly one entity/timeline
- Check 1.2: Each command is issued by exactly one thing — no COMMAND has more than one inbound SCREEN/AUTOMATION flow

**Anti-pattern to catch**: a command wired from two issuers — `check_model` reports it as `command-multiple-issuers`, and `create_flow` does not guard against it, so a double-issuer command found on the sheet is most likely a flow added by hand later or left over from earlier wiring

### Phase 2: Event Quality Validation (3 checks)
- Check 2.1: Events represent domain facts, not calculations
- Check 2.2: Event data is immutable after creation
- Check 2.3: Event names use past tense (what actually happened)

**Anti-pattern to catch**: storing computed/aggregated data as an event

### Phase 3: Read Model vs Event Distinction (2 checks)
- Check 3.1: Each read model is NOT an event timeline
- Check 3.2: Read model has a natural query pattern

**Anti-pattern to catch**: confusing projections/calculations with domain facts

### Phase 4: Business Rules Validation (2 checks)
- Check 4.1: Every command's preconditions are documented explicitly
- Check 4.2: Event preconditions are explicit (what must already be true before a command is valid)

**Anti-pattern to catch**: business rules scattered across the model or left implicit ("obviously can't ship an unconfirmed order")

### Phase 5: Data Traceability (1 check)
- Check 5.1: Input → Event → Read Model traceability is complete

**Anti-pattern to catch**: command inputs that disappear, or read model fields without a source

### Phase 6: Event Flow Validation (2 checks)
- Check 6.1: No impossible event sequences (the slice-transition flow is sound)
- Check 6.2: No event belongs to more than one entity's timeline — an event that seems to span two entities usually means a naming or boundary mistake, not a shared fact

**Anti-pattern to catch**: events that can occur in invalid combinations; a "shared" event quietly coupling two entities together

### Phase 7: Structural Shape Validation (2 checks)
- Check 7.1: No SCREEN is wired to more than one COMMAND
- Check 7.2: Fan-out/fan-in outliers (a command with more than two resulting events, a read model built from more than three events, a slice with markedly more scenarios than its neighbors) have been reasoned about against the business context, not flagged on count alone

See `eventmodeling-core-rules`'s **Structural Shapes** section for the full definitions ("the bed", "left chair", "right chair", "shelf") and their thresholds — this phase doesn't restate them. Those names are internal shorthand: never write them in a note, pin or report; describe the concern in plain terms.

**Anti-pattern to catch**: Check 7.1 catches "the bed" — a screen fanning into multiple commands hides where the user's actual decision is made; this is a real anti-pattern, always fail it. Check 7.2 catches "the left chair" / "the right chair" / "the shelf" — these are candidates, not automatic fails: only report an outlier if the specific events/fields/scenarios involved still look like they're doing more than one job once you've checked them against the domain.

### Final Questions (2 checks)
- Question 1: Could a modeler unfamiliar with this domain understand the model in 15 minutes?
- Question 2: Could the business rule/calculation behind a read model change without rewriting event history?

---

## Output Format

The skill returns a validation report (in your reply, with the findings recorded as notes and pins) with:

### For Each Check
```
 Check 1.2: Command Issued by Exactly One Thing
Status: PASS
Evidence: [Specific examples from your model]
```

### Anti-Patterns Identified (if any)
```
CRITICAL: [Anti-pattern description]
Problem: [Why it violates event modeling principles]
Violates: [Which checks fail]
Fix: [Recommended action]
```

### Final Verdict
```
Status:  PASS (or  PASS WITH WARNINGS or  FAIL)
Confidence: [percentage]
```

---

## Common Anti-Patterns (Domain-Agnostic)

### 1. Calculation Events
```
 ANTI-PATTERN:
Calculation Performed {
  metric: 4.5  ← Mutable/recalculated
  timestamp: T
}

 CORRECT:
- Event: Some Action Occurred (immutable fact)
- ReadModel: Metric View (recalculated from events)
```

**Why**: Calculations change as source data changes. Events are immutable.

### 2. Command With Multiple Issuers

A command is never issued by more than one thing — a COMMAND with 2+ inbound SCREEN/AUTOMATION flows hides which actor is actually responsible. Decide which issuer really triggers it, keep that flow and remove the rest (`remove_flow`); record it on the command's cell, naming the issuer to keep and the flow(s) to remove.

**Why**: Each command represents one specific trigger's decision to act — collapsing two triggers onto one command usually means either a naming/slice-boundary mistake or a stray manual/pre-existing flow.

### 3. An Event Shared Across Entities
```
 ANTI-PATTERN:
Entity A → Event A → also placed on Entity B's timeline

 CORRECT:
Entity A → Event A (on Entity A's timeline only)
Entity B reacts to Event A via a read model/automation, it doesn't own it
```

**Why**: An event belongs to the story of exactly one entity. If a second entity needs to react to it, that's a read model or automation reading it — not the same event living on two timelines.

### 4. A Screen Fanning Into Multiple Commands ("the bed")
```
 ANTI-PATTERN:
Screen → Command A
       → Command B
       → Command C

 CORRECT:
Screen → Command A only (one screen, one decision)
```

**Why**: A screen represents the moment a user has already committed to one decision. Wiring it to several commands hides where that choice actually gets made — see `eventmodeling-core-rules`'s **Structural Shapes** section for this and the three related fan-out/fan-in candidates ("left chair", "right chair", "shelf").

---

## Questions to Ask During Validation

**For each entity's timeline**:
1. "Do all events on this timeline belong to the same entity?"
2. "Could these events occur in any order, or does sequence matter?"
3. "Is every event on this timeline an immutable fact that actually happened?"

**For each command**:
1. "Are its preconditions written down, not just assumed?"
2. "Does it produce an event, or document why it's rejected?"

**For each read model**:
1. "Is this calculated from events via a projection?"
2. "Does it answer a specific query need?"
3. "Could its data change due to new events?"

---

## Success Criteria

 **Model is validated when**:
- All 14 checks pass (or have documented workarounds)
- No critical anti-patterns identified
- Both final questions answer YES
- Event modeling principles are clearly upheld

 **Model needs fixes when**:
- Any check fails with clear evidence
- Anti-patterns identified with specific violations
- A final question has a NO answer
- Fixes are straightforward and targeted

 **Model should be redesigned when**:
- Multiple phases fail
- Structural assumptions are fundamentally flawed
- Anti-patterns are systemic and pervasive
- Would require rewriting the core event timeline

---

## Example Validation Patterns

### Pattern: Calculation vs Event
When you see something like "Calculation Done" or "Review Rating Updated":
- Ask: "Is this immutable and caused by a user/system action?"
- If NO → It's a read model, not an event
- Fix: Remove from events, create a read model projection instead

### Pattern: Data That Doesn't Trace
When a read model field appears without source:
- Ask: "Where did this come from?"
- If no event source → Add the event or remove the field
- If sourced from a calculation → Verify it's in the read model, not the event

---

## Integration with Event Modeling Process

**Recommended timing**:
```
Step 1: Brainstorm Events
Step 2: The Plot (Sequence)
  ↓
→ RUN eventmodeling-validating-event-models-checklist (catch structural issues early)
  ↓
Fix any violations
  ↓
Step 3-8: Storyboard, Inputs, Outputs, Conway's Law, Scenarios, Completeness
  ↓
Step 9: Validate (eventmodeling-validating-event-models)
  ↓
→ RUN eventmodeling-validating-event-models-checklist again (final structural pass)
  ↓
PASS → Step 10: Slice, Step 11: Document Reasoning
FAIL → Fix identified issues
```

Running the checklist after Step 2 prevents wasting time on later steps if the core events are flawed.

---

## Checklist Questions by Domain

The skill applies the same 14 checks regardless of domain. Here's how to think about it in different contexts:

**E-commerce domain**:
- Events: Order Created, Order Confirmed, Payment Authorized, Order Shipped
- NOT events: Order Total, Inventory Level, Shipping Cost (these are read models)

**Banking domain**:
- Events: Account Opened, Deposit Received, Withdrawal Processed, Funds Transferred
- NOT events: Account Balance, Interest Calculated (these are read models)

**SaaS domain**:
- Events: Subscription Created, Payment Processed, Plan Upgraded, Subscription Cancelled
- NOT events: Monthly Recurring Revenue, Churn Rate (these are read models)

**Healthcare domain**:
- Events: Patient Registered, Appointment Scheduled, Procedure Completed, Bill Generated
- NOT events: Patient Age, Average Cost (these are read models)

The principle is the same across all domains: **immutable facts as events, calculated results as read models**.

---

## Tips for Best Results

1. **Be specific**: List actual event and command names from your model
2. **Reference your documentation**: Link to or quote from your step 1-7 work
3. **Provide context**: Explain what your domain is
4. **Ask follow-ups**: If a check flags an issue, ask "How do I fix this specifically?"
5. **Iterate**: Run again after making fixes to confirm all checks pass

## Quality Checklist

- [ ] All 14 checks evaluated — no check skipped without documented justification
- [ ] Every FAIL result includes the specific event, command, or entity that violated the check
- [ ] Anti-patterns identified by name with the exact model element that triggered the flag
- [ ] Final verdict is one of: PASS / PASS WITH WARNINGS / FAIL — no ambiguous outcomes
- [ ] Both final questions answered YES before declaring the model ready
- [ ] Any FAIL result has a recommended fix, not just a problem statement

---

## Related Skills

- **eventmodeling-orchestrating-event-modeling**: Main skill coordinating the full event modeling process
- **eventmodeling-brainstorming-events**: Extract events from requirements (Step 1)
- **eventmodeling-plotting-events**: Sequence events chronologically (Step 2)
- **eventmodeling-designing-event-models**: Design your complete event model
- **eventmodeling-validating-event-models**: Detailed validator with deep analysis

---

## Validation Checklist Reference

The 14-point checklist is defined in the **Validation Phases** section above.
Each check includes the anti-pattern to catch and questions to ask when evaluating your model.
