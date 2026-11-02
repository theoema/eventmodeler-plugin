---
name: eventmodeling-plotting-events
description: "Step 2 of Event Modeling - Arrange events chronologically in logical narrative sequence. Create timeline showing event flow and dependencies. Use after brainstorming events. Do not use for: brainstorming new events (use eventmodeling-brainstorming-events) or designing command/read model architecture (use eventmodeling-designing-event-models)."
---

# Plotting Events

This step applies the shared element rules in **`eventmodeling-core-rules`** — read it once per session if you haven't already; it defines what a COMMAND/EVENT/READMODEL/SCREEN/AUTOMATION is, how each is named, and the anti-patterns to reject, so this step doesn't restate them.

Every tool call carries `modelId` (from `list_models`); `sheet` may be omitted only when the model has exactly one sheet. Positions are cell addresses (`B2`: column letter, lane number), never coordinates. The tool for each concept below is in [tools](../eventmodeling-core-rules/references/tools.md).

Arrange all brainstormed events chronologically to create a logical sequence that makes sense as a narrative timeline. Show how events flow and depend on each other.

## Workflow

Given a list of brainstormed events, create the chronological plot:

### 1. Sequence Events Chronologically
Order events in time-based narrative:
- What happens first?
- What depends on what?
- What's the causal chain?

Format:
```
Timeline: Order Processing

1. Customer initiates → Order Created
   (Event: Order Created)

2. Order confirmed → Order Confirmed
   Depends on: Order Created happened
   (Event: Order Confirmed)

3. Payment processed → Payment Authorized
   Depends on: Order Confirmed happened
   (Event: Payment Authorized)

4. Inventory reserved → Inventory Reserved
   Depends on: Payment Authorized happened
   (Event: Inventory Reserved)

5. Order shipped → Order Shipped
   Depends on: Inventory Reserved happened
   (Event: Order Shipped)

6. Delivery confirmed → Delivery Confirmed
   Depends on: Order Shipped happened
   (Event: Delivery Confirmed)
```

### 2. Show Dependencies and Causality
Document what triggers each event:
```
Event: Order Confirmed
Can only happen after: Order Created
Triggered by: Customer confirms order
Precondition: Order in Draft state

Event: Payment Authorized
Can only happen after: Order Confirmed
Triggered by: Payment gateway authorizes
Precondition: Order confirmed and payment submitted
```

### 3. Identify Alternative Paths
Show events that can diverge:
```
After Order Created:
Path A: Customer confirms → Order Confirmed
Path B: Customer cancels → Order Cancelled

After Payment Authorized:
Path A: Payment succeeds → Payment Processed
Path B: Payment fails → Payment Failed → Order Cancelled
```

**Every path here is a decision point, not a divergent journey** — by the time plotting runs, sheet structure is already fixed (Step 1), so a path in this step is always one trigger resolving to one of several outcomes within *this* sheet's story, shown as branching columns. If a "path" you're documenting actually starts a different story with its own screens/commands rather than just deciding how this one ends, it was mis-grouped in Step 1 — stop and send it back to `eventmodeling-brainstorming-events`'s sheet grouping (see its "Divergent journey vs. a decision point" note) rather than modeling it as a branch here.

### 4. Create Timeline Diagram
Visual representation of event flow:

```

 Time →                                           

 Order Created                                     
    ↓                                             
 Order Confirmed                                   
    → Payment Authorized                          
        → Inventory Reserved                     
            → Order Shipped                     
                 → Delivery Confirmed           
        → Payment Failed → Order Cancelled        
    → Order Cancelled (rejected before payment)   

```

## Sheets and Timelines

Timelines (sheets) are **created and assigned during Step 1 (Brainstorming)**. This step does not create new sheets. Each plotting pass operates on a **single sheet** — the one the user is currently focusing on.

Before placing events, resolve the target sheet. `get_model_view` lists the sheets with their slices and counts:

```
get_model_view { "modelId": "<modelId>" }
```

If multiple sheets exist, ask the user which one to work on now. Never reorder events across sheets in a single pass.

## Output Format

Instead of writing a markdown document, **place every event on the sheet** with `place_element`.

Work within the chosen sheet only. For each event in the final chronological order, call `place_element` with:

| Parameter | Value |
|-----------|-------|
| `sheet` | the sheet this event belongs to |
| `cell` | the event-lane cell of the event's column (e.g. `B3`) |
| `element` | `{ "scope": "event", "name": "<Event Name>" }` |

Reuse existing empty columns, do not blindly create new ones. "Empty" means no EVENT anywhere in that column — **never place two EVENT elements in the same column, even across different swimlanes**; each column is a single moment in the timeline. If the column already holds an EVENT in another swimlane, place this one in a new column instead (see `eventmodeling-core-rules`, Lanes).

**Do not resolve that conflict by checkerboarding.** When a sheet has more than one swimlane (e.g. a second system's events, see `eventmodeling-brainstorming-events`'s Swimlane Rules), the sheet's own swimlane carries the continuous story throughout — do not alternate "our event, their event, our event, their event" column by column just because each needs its own column, and do not give the other swimlane a multi-column run of its own either. An event in the other swimlane is a single-column handover: it marks one moment where control passes to the other system, and the very next column returns to this sheet's own swimlane to continue the story. Two swimlanes trading events every column with no genuine trigger behind each handover is a sign of forcing visual symmetry rather than following the actual causal order — re-sequence instead. If the other swimlane ends up needing a long run of its own consecutive events, that's a sign those events belong in a sheet of their own, not this one.

Events brainstorming already placed sit one per column in the order they were thought of; plotting puts them in chronological order and completes the set. Read the sheet first (`get_sheet_view`: the event lane's row, the columns in order, the events in them). Then, for each event in the final chronological order, one at a time:

| The event is... | Do |
|---|---|
| on the sheet, in the wrong position | `move_slice { slice, to }`: the column moves with its contents; `to` is the 1-based position among the slices |
| not on the sheet yet | `insert_slice { name, at }` to add its column (a column always comes with its own slice) at the right position, then `place_element` it into the event lane's cell in that column |
| in the right position | leave it |

Column letters shift whenever you insert or move, so re-read the sheet before using another address. A cell holds at most one element, so an occupied cell is refused (`CELL_OCCUPIED`). Alternative outcomes of one decision point each get their own column right after the trigger, one event per column; `check_model` reports two events in one column as `column-multiple-events`.

Process events in order, one at a time. Do not skip any event.

After all events are placed, summarise to the user:
- Sheet(s) with their names
- Numbered list of events placed per sheet
- Key insights: critical path, decision points, terminal events
- Any errors

## Quality Checklist

- [ ] **Working on a single named sheet** — not mixing events from different sheets
- [ ] **No new sheets created here** — sheet structure was fixed in Step 1 (Brainstorming)
- [ ] Every event has clear predecessor
- [ ] Dependencies are explicitly documented
- [ ] Alternative paths are shown
- [ ] Flow forms a coherent narrative
- [ ] No events without trigger
- [ ] Terminal states are clear
- [ ] Compensation/cancellation flows are complete
- [ ] Timeline makes business sense

## Principles

1. **Narrative Coherence**: Events tell a story
2. **Dependency Clarity**: What must come before what
3. **Alternative Paths**: Show all possible flows (happy path + errors)
4. **Natural Sequence**: Order matches business domain logic
5. **Completeness**: Every brainstormed event appears
