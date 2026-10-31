---
name: eventmodeling-identifying-outputs
description: "Step 5 of Event Modeling - Identify Outputs/Read Models from events. Show what data flows back to UI and Processors. Use after defining inputs. Do not use for: identifying commands or inputs (use eventmodeling-identifying-inputs) or verifying field completeness (use eventmodeling-checking-completeness)."
---

# Identifying Outputs

This step applies the shared element rules in **`eventmodeling-core-rules`** — read it once per session if you haven't already; it defines what a COMMAND/EVENT/READMODEL/SCREEN/AUTOMATION is, how each is named, and the anti-patterns to reject, so this step doesn't restate them. The tool mechanics every step shares (addressing, flows, fields) are in `eventmodeling-core-rules`' `../eventmodeling-core-rules/references/tools.md`.

Every tool call takes the `modelId`, and a `sheet` when the model has more than one. Placement is always by cell address (`C2`) — never by coordinates.

## Interview Phase (Optional)

**When to Interview**: Skip if the user has clearly identified: read model queries needed by UI, processor needs, and refresh patterns. Interview when unclear which data queries are critical or how frequently they're accessed.

### Critical Questions

1. **Query Patterns** (Impact: Determines which read models are needed and how fresh each must be)
   - Question: "What data do users/processors need to query? (A) Real-time (sub-second), (B) Near-real-time (seconds), (C) Periodic (minutes/hours)?"
   - Why it matters: How fresh each consumer needs its data decides which consumers need a read model of their own
   - Follow-up triggers: If (A) → ask which specific screens or processors require sub-second reads; these get dedicated read models

2. **Event vs Read Model Clarification** (Impact: Ensures we don't model calculations as events)
   - Question: "Are there calculated/aggregated fields? (e.g., average rating, total sales, inventory count) - These are read models, not events."
   - Why it matters: Common mistake to model calculations as events; identifying them upfront prevents architecture errors
   - Follow-up triggers: For each calculated field mentioned → confirm "This recalculates as source data changes, so it belongs in a read model projection — does that match your expectation?"

Follow **`eventmodeling-interview-protocol`** to run this interview and record its findings — label this step "**5. Identifying Outputs** (`eventmodeling-identifying-outputs`)". Findings should cover: query patterns (which consumers need what freshness), calculated/aggregated fields identified (and confirmed as read models, not events), and a read model summary by freshness tier.

---

## CRITICAL: Events vs Read Models

**This is the most important distinction in event sourcing.** Many architectures fail because this line gets blurred.

### Events = Immutable Domain Facts
Things that actually happened in the domain. Once created, they never change:
- Customer Created (a customer actually signed up)
- Order Placed (someone actually placed an order)
- Payment Authorized (payment gateway actually authorized)
- Order Shipped (fulfillment actually shipped the order)

**Characteristics**:
- Represents an action someone took
- Immutable once recorded
- Can be replayed to rebuild state
- Provides audit trail
- Independent of other events

### Read Models = Derived Projections
Optimized views calculated FROM events. They recalculate multiple times:
- Customer Dashboard (projects current customer data)
- Order Status View (projects order state)
- Inventory Level View (projects available stock from receipt/sale events)

**Characteristics**:
- Calculated/aggregated state
- Recalculates when source events change
- Derived from other events
- Shaped for one query
- Can be regenerated from events

### The Test: Is It an Event or Read Model?

Ask these questions in order:

| Question | Answer | Type | Example |
|----------|--------|------|---------|
| Did an actor perform an action? | YES | EVENT | Customer confirmed the order |
| Is this pure calculation? | YES | READ MODEL | Inventory level total |
| Is it immutable once created? | YES | EVENT | Payment Authorized |
| Does it recalculate multiple times? | YES | READ MODEL | Total sales (updates as orders change) |
| Is it independent (causes no other events)? | YES | EVENT | Order Flagged (flagged for manual review) |
| Is it derived FROM other events? | YES | READ MODEL | Order Status (derived from multiple events) |

### Common Anti-Patterns

**DON'T model these as EVENTS**:
- Inventory level totals (calculation from stock events)
- Inventory totals (sum of transactions)
- Account balances (calculation from transactions)
- Search indexes (derived from documents)
- Aggregated metrics (sums, counts, averages)
- Scheduled calculations (processor outputs that are pure calculation)

**DO model them as READ MODELS**:

WRONG: Modeling as Event
```
Inventory Level Recalculated
  productId: product-456
  currentStock: 84          (This recalculates!)
  reservedStock: 12         (Derived, not a fact)
```

CORRECT: Model as Read Model
```
Inventory Level View
  productId: product-456
  totalReceived: 200
  totalSold: 116
  currentStock: 84
  lastUpdated: 2025-01-24T10:30:00Z
  history:
    - 2024-12-01: stock 150 (200 received)
    - 2024-12-15: stock 110 (40 sold)
    - 2025-01-24: stock 84 (26 sold)
```

**WHY**:
- Events should capture facts (what happened)
- Calculations should be projections (how we view the facts)
- Otherwise you end up with circular dependencies and replay issues

---

## Workflow

Given commands and events, identify all outputs: for each screen, trace which events its data comes from; define a read model per component; document the event→data mapping; catalog the resulting read models; check for UI needs with no event source; and identify what processors consume. A full worked example of this reasoning (Order/Payment domain) is in `references/examples.md`.

## Output Format

Instead of writing a document, **place each READMODEL (and any missing AUTOMATION) on the sheet** with `place_element`. Screens are typically already placed from Step 3 (storyboarding) as one plain screen per screen state — Step 3 does not pre-split anything, so working out how many components a screen actually has is this step's job (Step 5a below), not something to re-derive from storyboarding's output. Do not re-place an existing screen unless one is clearly missing. Automations are placed here only in the rare case where analysis reveals a processor that reads state and issues commands but was missed entirely in Step 4 — if that happens, immediately apply `eventmodeling-designing-automation-chains`'s rules to it (todo-list read model, translation chain if externally triggered) before continuing, rather than leaving it for Step 5i's defensive re-check to catch.

This step proceeds as a sequence of lettered sub-steps: identify each screen's components (5a), break multi-component screens apart into copies (5c), then design and place one read model per component (5d onward). Every automation's todo-list read model and any translation chain is designed in **Step 4b** (`eventmodeling-designing-automation-chains`), immediately after Step 4 — not here. This step only ever designs screen-facing read models; Step 5i below re-checks automations defensively, not as this step's primary job.

> **CRITICAL: Every READMODEL element MUST have fields, and every field must trace back to a connected source event (or be `isGenerated`).** A read model without fields, or with fields of unknown origin, has no data lineage and cannot be traced back to its source events.

### The typical slice pattern

The standard pattern for a screen in an event model is:

```
READ MODEL → SCREEN → COMMAND → EVENT
```

**Most screens need a read model on their left** — not only view/status screens, but also command/input screens that show current state before the user acts (e.g., a booking form that displays a bike's current availability, a checkout screen that shows the cart). The read model feeds the screen; the screen triggers a command; the command produces an event.

Pure view screens (no outgoing command) follow a shorter pattern:
```
READ MODEL → SCREEN
```

Automations follow:
```
READ MODEL → AUTOMATION → COMMAND → EVENT
```

Treat any screen or automation without an incoming read model as a gap. A screen may be exempt if it provably needs no prior state at all (e.g., a blank registration form) — an **automation is never exempt**: every automation gets a todo-list read model, even a trivial one that opens and closes within the same slice. For automations, that read model was already designed in **Step 4b** (`eventmodeling-designing-automation-chains`); this step's job is limited to the defensive re-check in Step 5i below.

### Step 5a — Enumerate consumers and identify components

**Before designing any read model, enumerate every SCREEN and AUTOMATION already placed on the sheet** (from Step 3 — Storyboarding): `get_sheet_view` lists every element by cell address. Then read each screen cell's note (`get_cell_view`): storyboarding records the read models a screen needs as lines of the form `Planned read model "<Read Model Name>": <field>, <field>`. Use those lines as the starting point for the read models below, and remove each line from the note once its read model is placed and connected. Storyboarding hands off one plain screen per screen state and does not pre-split anything — deciding how many components a screen actually has is this step's decision, because a component is defined by its read model: **one component in a screen resembles one read model.**

Read models exist to serve the elements already on the sheet:
- Every **view screen** (output/read model screen) needs at least one read model to supply its data.
- Every **automation** needs at least one read model to read from — its todo list. This applies to every automation without exception, not only ones that visibly "decide" something; this was already handled in Step 4b (`eventmodeling-designing-automation-chains`) for every automation placed in Step 4.
- Every **command/input screen** needs a read model unless it is a blank creation form with no prior state to display (this is the rare exception, not the rule).

For each SCREEN, look at its design (`get_screen_design`, or `render_screen` for the picture) and its fields (`get_cell_view`) and identify its components — groups of fields/UI elements a user would perceive as one area: a stats tile, a list below it, a summary card, a detail panel, a status column in a table, etc. **Most screens genuinely have exactly one component — do not force a split.** A screen has more than one component only when a user would point at two separate areas and describe them as different things.

A single homogeneous list/table is still one component, even when its rows draw on many different event types (e.g. a catalog whose per-row status is set by many different lifecycle events scattered across the timeline) — don't split a table by row or by source event. **But do not use this as a reason to also fold in other fields that don't share that same wide fan-in.** A book's title/author (set by 1-2 events, updated rarely) and a copy's live availability status (derived across its full lifecycle, many events) are two different kinds of computation even when they render on the same visible page — a user pointing at "the book's info" versus "whether it's available right now" is describing two different things, whatever the layout looks like. If a candidate component mixes a field with irreducible wide fan-in and a field that only needs 1-2 events, that is **always** two components — split it now, at this step, before any read model is built from it, rather than discovering the mismatch after the fact.

Once a component's fields are genuinely homogeneous (every field needs the same wide-lifecycle fan-in, none of them are cheap identity/fact fields riding along), its read model will legitimately need wide fan-in. This is expected, not a modeling error — document it with a note (per the orchestrating skill's "Documenting decisions inline, at any step"; here `set_note` on the read model's cell) naming *which field(s)* have that irreducible fan-in and why, e.g. "Copy Availability View.copyStatus is one per-copy status derived across that copy's full lifecycle (repair, loss, reservation, return)." That note documents a specific field's fan-in, not a blanket exemption for the node — re-verify every field on the node against the ">3-events heuristic" (`eventmodeling-orchestrating-event-modeling`) at every later step that touches it, including Step 5i's mandatory per-node verification below, rather than treating a prior note as settled once and for all.

After this step is done, **every SCREEN and every AUTOMATION on the sheet must be connected to at least one read model** via a `ReadModelToScreen` or `ReadModelToProcessor` flow, and every screen identified above as having 2+ components must have been broken apart per Step 5c before any read model is placed. If a screen or automation has no incoming read model flow, it is a gap — either a read model is missing or the flow is missing.

> **Placement rule**: A read model must be placed immediately upstream of the SCREEN or AUTOMATION it serves — sharing that column when possible (a SCREEN with a free interaction-lane cell), or one column to the left when not (any AUTOMATION; a SCREEN whose column is already occupied). So a read model's consumer sits either in the read model's own column or in the very next one — both are correct, and a consumer one column to the right is **not** a gap and must never be reported as one. What *is* a gap is a read model with no SCREEN or AUTOMATION in either of those two places: an orphan that will never be displayed or acted on (`check_model` reports it as `read-model-unconsumed`).

> **Automation todo-list read models are designed in Step 4b, not here.** The full pattern — every automation's todo-list read model, the "no invisible signal" rule, and the translation chain required for externally-triggered automations — lives in `eventmodeling-designing-automation-chains` (Step 4b), which runs immediately after Step 4, before this step. If Step 4b ran, every automation already on the sheet has its todo-list read model wired; Step 5i below only re-checks this defensively. The rare exception is an automation discovered only now, during output analysis (see "Output Format" above) — if that happens, apply `eventmodeling-designing-automation-chains`'s rules to it directly rather than re-deriving them here.

### Step 5c — Break apart multi-component screens into copies

**Do not default to a single monolithic read model that supplies an entire screen.** For every screen identified in Step 5a as having 2+ components, break it apart into one screen instance per component before designing its read models. The original screen stands for one component; every other component gets a copy made with `copy_element` (a linked copy of the screen — screens may be copied):

- Each copy is the **same page**, so it **keeps the same screen name** — a linked copy borrows its original's name and fields, so this is automatic. Do not rename copies after their component (e.g. don't title one copy "Librarian Dashboard — Statistics" and another "Librarian Dashboard — Recently Added"; both stay "Librarian Dashboard").
- Distinguish copies visually, not by name: **mark/highlight the one component each instance is about** in its HTML design (`set_screen_design`) — the component of interest stays crisp and every other top-level section is dimmed or blurred. Start from the original's markup (`get_screen_design`) so every instance is the same page, and add the marking with plain CSS in a `<style>` block (`opacity`, `filter: blur(...)`), keeping the markup itself identical. A linked copy carries its own HTML design independent of its original's, so each instance gets its own marking. Check the result with `render_screen`.
- Address each instance by its cell address (`C1.1`), not by name — they all share one name, and a name-based reference is ambiguous.
- Place each copy in its own column, the **same column** as the read model that will feed it (insert a new column immediately before if that copy's column is already occupied) — this is normally a different column per copy, since each component typically has a different natural source event.
- The command a screen issues stays issued by exactly one screen instance — the one sitting in the command's own column. The other instances only ever receive `ReadModelToScreen` flows.

Why this matters, beyond tidiness:
- **It prevents backward arrows.** A single screen-wide read model is forced to aggregate from whatever events each of its components needs, which often means reaching back across many columns to events scattered throughout the timeline — and the read model can only sit in one column, so some of those connections end up spanning a wide gap or, worse, pushing the read model's column later than some of its screen's other consumers require. Splitting by component lets each narrower read model sit close to its own natural source event(s), keeping every `EventToReadModel` flow short and forward.
- **Components evolve independently.** A stats tile and a "recently added" list are driven by different events, change at different rates, and are typically owned by different slices in implementation. Bundling them into one read model couples their release/change cadence for no reason.

Before finalizing any read model, ask: "does this screen contain more than one component?" If yes and it wasn't already broken apart, do it now — don't ask whether it's worth the extra columns, it always is at this scale, since the alternative is a hidden coupling and a higher chance of a backward-arrow layout error.

### Step 5d — Pull the screen's field data from Step 3 — it is the spec, not a guess

**Do not re-derive read model needs from a screen's title or description alone, and do not rely on a phase-summary handoff for this** — a handoff is a short hand-written prose summary, not the actual field data. It will not reliably carry the per-field detail forward. Go back to the sheet itself:

For every SCREEN, read it directly (`get_slice_view` for the slice's elements with their fields, or `get_cell_view` for one cell, together with any note on the cell) — never from memory. Step 3 already required every view field to name the read model it is displayed from; it was recorded so this step doesn't have to guess it again.

- **Group the screen's fields by the read model Step 3 named for them.** That grouping — not a fresh read of the screen's visuals — is the read model's name and field list. Build the READMODEL from it directly.
- A field whose read model isn't on the sheet yet is a requirement, not a suggestion. (Fields marked `isUserInput` or `isGenerated` are input or display-only values. They need no read model.)
- A screen with no fields, or with fields that name no read model, is **not** evidence that it needs no read model. Re-check it against the three rules above (view screen / automation / command screen showing prior state) before concluding it's the rare blank-form exception — and say explicitly why it qualifies.

### Step 5e — Field data lineage — every READMODEL field has a source

Apply `eventmodeling-information-completeness` to every READMODEL field as you add it. A read model's upstream is the EVENTs connected to it by `EventToReadModel` flows. A value the projection calculates itself (a count, a sum, a duration) is `isGenerated: true`. A field no connected event supplies is a missing event field or a missing flow: add it before proceeding.

Lineage lives in the flows, not on the field. Declare it as you place the element: a field may carry `source: { from, field? }` — `from` is the source element's address or name on this sheet, `field` the field on it that feeds this one (defaults to the same name). `place_element` turns every `source` into the flow and its field mapping for you, so the lineage is recorded while you know it. If the source event isn't on the sheet yet, leave `source` off and declare the mapping once it lands: `create_flow`, then `set_flow_mapping` (by field name) for fields whose names differ. Same-named fields match without any mapping.

Include only the fields the consuming SCREEN or AUTOMATION actually displays or needs — do not add speculative fields; enrich later with `update_element`. Give each field an `example` value. A field that genuinely holds a list of values (e.g. line items, a collection projected from multiple events) is `isList: true`; leave it off for a single value.

If the READMODEL itself represents a set of rows rather than a single record — e.g. `Invoice List`, `Product List` — it is list-shaped (see `eventmodeling-core-rules`' READMODEL section). This is separate from a field's `isList`: list-shaped says the whole read model is a list of elements shaped by its defined fields, `isList` says one field on an otherwise-single-record read model holds multiple values. Mark it with `isList: true` on the read model itself (`place_element` / `update_element`); a field's own `isList` stays as it is.

```json
{
  "cell": "C2",
  "element": {
    "scope": "readmodel",
    "name": "Active Reservation View",
    "fields": [
      {"name": "reservationId", "fieldType": "String",   "example": "res-001", "source": {"from": "Reservation Confirmed"}},
      {"name": "customerId",    "fieldType": "String",   "example": "cust-42", "source": {"from": "Reservation Confirmed"}},
      {"name": "bikeId",        "fieldType": "String",   "example": "bike-17", "source": {"from": "Reservation Confirmed"}},
      {"name": "stationId",     "fieldType": "String",   "example": "stn-03",  "source": {"from": "Reservation Confirmed"}},
      {"name": "expiresAt",     "fieldType": "DateTime", "example": "2026-06-01T09:30:00Z", "source": {"from": "Reservation Confirmed"}},
      {"name": "status",        "fieldType": "String",   "example": "confirmed", "source": {"from": "Reservation Confirmed", "field": "toStatus"}},
      {"name": "minutesLeft",   "fieldType": "Int",      "example": "25", "isGenerated": true}
    ]
  }
}
```

The field shape is exactly `{name, fieldType, isList?, isGenerated?, isOptional?, isUserInput?, example?}` (plus `subfields` for a Custom field) — anything else is silently dropped. Field types: `UUID, Boolean, Double, Decimal, Date, DateTime, Long, Int, String, Custom`.

Read models go in the **interaction** lane. For a **SCREEN**, the primary read model shares the SCREEN's column (READMODEL in the interaction lane, SCREEN in the actor lane of the same column) — unless that column's interaction-lane cell is already occupied (e.g. a command/input screen, where the COMMAND already sits there), in which case the read model goes one column to the left instead. For an **AUTOMATION**, the read model always goes one column to the left, never the same column — an automation's own column already holds the COMMAND it issues. If a consumer needs more than one read model, only the primary one gets this placement; every additional read model goes further left still.

> **Timeline alignment rule**: Place the read model in the same column as its consumer screen when that column's interaction cell is free, or one column before its consumer (screen or automation) when it isn't. Insert the new column immediately **before** the consumer's column rather than after — the read model must sit upstream of (to the left of) the element it feeds. Do not append read model columns to the end of the timeline — doing so severs the visual left→right flow from data projection to UI consumption.

### Step 5f — Placing READMODEL and AUTOMATION elements by cell address

A cell holds exactly one element, so first find a free interaction-lane cell in the right column. Read the layout with `get_sheet_view` (or `get_cell_view` for one cell) rather than guessing.

**For a READMODEL** (interaction lane), the target column depends on the consumer:
- **SCREEN**: the screen's own column. First check whether that column's interaction cell is occupied (e.g. a command/input screen). If it is, insert a new column immediately before the screen's with `insert_slice` (`at` = the screen's 1-based column position, `cols: 1`; this shifts the screen and everything after it one column right), then place the read model in that new column's interaction cell.
- **AUTOMATION**: the column one to the *left* of the automation — never the automation's own column, since it already holds the COMMAND the automation issues.

```
place_element { "cell": "B2", "element": { "scope": "readmodel", "name": "Active Reservation View", "fields": [ … ] } }
```

Fields go in the same call (see Step 5e). Placement never wires anything: no flow exists until you create it (apart from the ones a field `source` declares), so there are no stray edges to clean up.

**Addresses shift after every column insertion.** Re-read the sheet (`get_sheet_view`) before using another address.

**A column cannot exist outside a slice.** Every column you insert comes with its own slice (`insert_slice`, one column): name it for what the column holds — here, the read model. A slice is one command, one read model or one automation, never a command and a read model together (see `eventmodeling-core-rules`, "Slices"); `eventmodeling-slicing-event-models` reviews the final boundaries.

**For an AUTOMATION** (actor lane) — its READMODEL always goes **one column to its left**, never the same column: the automation's own column already holds the COMMAND it issues (interaction lane), so the read model can't also live there. Place the automation with `place_element` (`scope: "processor"`) in the actor lane and its read model as above.

### Step 5g — Preventing backward arrows (mandatory pre-placement check)

The timeline must always progress left-to-right. A flow either stays within its own column or goes to a later column — a `ReadModelToScreen` flow going right-to-left is a layout error.

The correct layout is: **READMODEL and its (primary) view SCREEN share the same column** — READMODEL in the interaction lane, SCREEN in the actor lane of the same column. The read model only sits in a separate column, immediately **before** the screen's, when the screen's column is already unavailable to it (its interaction cell is occupied by something else, e.g. a COMMAND on a command/input screen). If a screen displays more than one read model, only the primary read model shares the screen's column — every additional read model goes further left, never to the right. Before placing each read model, find the view screen it serves and verify the column order:

```
For each view screen S that queries this read model as its primary read model:
  If column(S)'s interaction cell is free:
    → Place the READMODEL in column(S) itself (same column as S). No adjustment needed.
  If column(S)'s interaction cell is already occupied:
    → Insert a new column immediately before column(S) (insert_slice, at = column(S)'s position)
      and place the READMODEL there instead.
  If the read model ends up more than one column away from S with nothing in between:
    → Gap between read model and screen. Close it: move the read model (move_element) into column(S) (if free) or column(S) - 1.
```

**View screens normally share the column of the (primary) read model they display** — either because they were placed there in Step 3, or because the read model was placed into the screen's own column just now (per the check above). A screen's own position is never moved to resolve a read model placement conflict — when the shared column isn't available, the **read model** gets a new column immediately before the screen's, not the other way around (this preserves the screen's narrative order from Step 3).

**Checking a flow's direction.** `check_model` reports every flow whose target column is left of its source column as `flow-backward`, with both element addresses, and exempts the one case below. No tool rejects a backward flow, so run `check_model` after every flow this step creates. (`get_sheet_view` returns every flow with its endpoint addresses if you want to read the columns yourself.)

**`EVENT → READMODEL` has exactly one backward exception, and it is narrow.** A read model that already carries a `ReadModelToProcessor` flow — a todo-list read model feeding an automation, per `eventmodeling-designing-automation-chains` — may also be fed by a later-column event closing an item it opened earlier. Outside that one case, a backward `EventToReadModel` flow is a layout error (see `eventmodeling-core-rules`, "Updating a Read Model That Already Feeds a Screen"): without the rule, a read model would become a moving target for whatever screen or scenario later reaches back into it.

If a later event needs to update data a read model already feeds to a SCREEN, do not connect that later event back into the existing read model. Resolve the update the same way Step 5c resolves multi-component screens: place a **new READMODEL** in a new column immediately after the later event's column — a linked copy of the earlier read model (`copy_element`; it's the same read model carried forward, not an unrelated new one; see `eventmodeling-core-rules`, "Linked Copies"). Connect the later event forward into the copy (`create_flow`), and declare the field(s) that event actually changes with `set_flow_mapping` (to the copy's address). Place a matching SCREEN in that same new column — a linked copy of the earlier screen (`copy_element`), same name, with an HTML design showing the updated data (`set_screen_design`), optionally marking what changed as in Step 5c. A linked copy shares its original's name and fields, so the copy shows the same field list; what the later event changes is carried by the flow mapping and the screen design. Leave the earlier READMODEL/SCREEN instances exactly as they are — they're still what that earlier point in the timeline correctly showed. Never delete them.

### Step 5h — Wire flows after placing each READMODEL (and its SCREEN)

After `place_element` returns the READMODEL's address, create the flows that complete the slice (`create_flow` infers the type from the two elements):

1. **EVENT → READMODEL** (`EventToReadModel`) — find the primary source EVENT in the swimlane lane of the same column, and connect it: `create_flow { "from": "B3.1", "to": "C2.1" }`. Any flow a field `source` already created in Step 5e exists — don't repeat it.
2. **READMODEL → SCREEN** (`ReadModelToScreen`) — connect to the existing SCREEN in the actor lane of the same column (or the column immediately after, only when that column wasn't available to the read model — screens are typically already placed from Step 3).
3. **READMODEL → AUTOMATION** (`ReadModelToProcessor`) — if the read model is consumed by an automatic process (scheduler, background job, external trigger), place the AUTOMATION in the actor lane of the column immediately to the right of its read model — unlike a view screen, this is unconditional for automations: the automation's own column always holds the COMMAND it issues, so its read model never shares that column. Then connect them.

If a flow's target isn't placed yet, skip the flow and create it when that element lands. Where a source and target field have different names, state the correspondence with `set_flow_mapping` (by field name). Log each created flow: `→ connected EVENT→READMODEL "Order Placed"→"Order Status View"`, `→ connected READMODEL→SCREEN "Order Status View"→"Order Status Screen"`, or `→ connected READMODEL→AUTOMATION "Order Status View"→"Fulfillment Processor"`.

4. **Document the reasoning for each connected event** — for every EVENT → READMODEL flow wired in step 1 (including any added later, e.g. via Step 5g's backward exception or a Step 5c/copy pattern), record why that event feeds this read model: which field(s) it sets or updates, and why. Use one note per read model, on that read model's own cell (`set_note`; per `eventmodeling-orchestrating-event-modeling`'s "Documenting decisions inline, at any step"). `set_note` replaces the note, so when the read model later gains another connected event, read the existing note (`get_cell_view`) and write it back extended — don't lose what it said, and don't keep a second note elsewhere.

   Example body:
   ```markdown
   ## Event → field reasoning — Order Status View

   - **Order Placed** → sets `orderId`, `status: "placed"`, `items[]` — the read model's creation event; the order doesn't exist before it.
   - **Order Shipped** → updates `status: "shipped"`, `trackingNumber` — the only event carrying a tracking number.
   - **Order Cancelled** → updates `status: "cancelled"` — terminal state, no further events expected after this.
   ```

### Step 5i — Mandatory per-node verification (run before declaring this step done)

Do not declare Step 5 complete on the strength of the read models you happened to design. Run `check_model` first as the mechanical half of this pass: `read-model-without-source` lists every READMODEL with no inbound event flow, `read-model-unconsumed` every READMODEL nothing reads, `screen-disconnected` every screen with no flows, and `untraceable-field` every field no inbound flow supplies. Then **re-read every SCREEN and AUTOMATION on the sheet** (`get_sheet_view` — don't rely on the list built earlier in this step, the sheet may have moved on) and check each one individually, together with the flow-direction check from Step 5g (`check_model`'s `flow-backward`, which you have already run):

1. Does it now have an incoming `ReadModelToScreen` or `ReadModelToProcessor` flow?
2. If it's a SCREEN and not connected — is it a provably blank creation form with no prior state? State the reason in one line (e.g. `"Register Account" screen: blank form, no prior state — exempt`). This exemption applies to screens only.
3. If it's an AUTOMATION and not connected, it is **never** exempt — this should already be resolved by Step 4b (`eventmodeling-designing-automation-chains`); if it isn't, apply that skill's rules now to identify its todo-list read model.
4. If a SCREEN is neither connected nor exempt, it is an **unresolved gap**. Fix it now: design the missing read model (pulling from the screen's fields as above) and wire the flow. Do not move to Step 6 with an unresolved gap silently carried forward — either fix it or explicitly flag it to the user as accepted debt (`create_pin` on the area, so it stays visible on the sheet).
5. Does any screen still carry more than one component undivided (a Step 5a/5c miss)? If so, break it apart now per Step 5c before counting it as resolved.
6. **Re-read every READMODEL too** and run the >3-events heuristic (`eventmodeling-orchestrating-event-modeling`) on each one field by field — including read models a note already justified as a roll-up. A prior note documents one field's irreducible fan-in; it does not exempt the rest of that node's fields from this check. The failure mode this catches: a wide-fan-in field (e.g. live per-copy availability) bundled together with a cheap, low-fan-in identity/fact field (e.g. a title set by 1-2 events) that has nothing to do with the roll-up — that pairing is always two read models, never one, no matter how the note reads.
7. **Every READMODEL has its event-reasoning note (Step 5h.4)**, and that note accounts for *every* inbound `EventToReadModel` flow on the node — not just the one from when it was first placed. If a read model gained a connected event later and the note wasn't extended, fix it now rather than carrying the gap forward.
8. **No backward flow** other than the todo-list exception (Step 5g): `check_model` reports none as `flow-backward`.

Then do the judgement checks above (blank-form exemptions, the >3-events heuristic, note completeness) that `check_model` can't make. It does flag a read model built from more than three events as `read-model-fan-in` (info), which is the prompt for the heuristic, not its verdict.

List the result of this pass (connected / exempt / fixed) for every screen and automation checked — this list is the evidence the orchestrator's Step 5 gate ("every screen data need is satisfied by a read model") actually holds, not just an assumption.

After all read models, screens, automations, and flows are in place, present the Read Model Catalog summary as text to the user.

---

The shape of a complete read model catalog (purpose, subscribed events, data, update logic, consumers, completeness check) is kept in `references/examples.md`; it is the content the sheet already holds, not a second output.

## Quality Checklist

### Read Model Design
- [ ] **Typical pattern applied**: most screens follow `READ MODEL → SCREEN → COMMAND → EVENT`
- [ ] **Every SCREEN from storyboarding is connected to at least one read model** (via `ReadModelToScreen`); only blank creation forms may be exempt — verified via the mandatory per-node pass above, not assumed
- [ ] **Every AUTOMATION from storyboarding is connected to at least one todo-list read model** (via `ReadModelToProcessor`, Step 4b) — no exemption for automations, unlike screens; even a simple relay automation gets one
- [ ] **No automation's todo list is opened directly by another system's (second-swimlane) EVENT unless that automation is itself the translation automation** (Step 4b) — an automation doing the actual domain work is only ever opened by an internal event; a second-swimlane EVENT feeding straight into a work automation's todo list is a missing translation automation
- [ ] **No read model is placed without a connected SCREEN or AUTOMATION consumer**
- [ ] **No read model spans more than one component** — a screen with N distinct components gets N read models and N screen instances (same screen name, one component crisp per instance), not one screen-wide read model
- [ ] **Every multi-component screen was broken apart in Step 5c** — each copy keeps the original screen's name, differs only in which component is marked/highlighted
- [ ] **Every automation's todo-list read model identifies its opening and closing events** (Step 4b) — including the automation's own resulting event as a closing event where applicable, not just the triggering event
- [ ] **A field's genuinely irreducible wide fan-in is documented per-field** (inline note, per "Documenting decisions inline") — and that note is never treated as clearing every other field on the same read model from the >3-events check; a cheap identity/fact field bundled alongside a wide roll-up field is always split out, never excused by the roll-up's own note
- [ ] **Every read model has an event-reasoning note** (Step 5h.4) covering every connected event and which field(s) it sets or updates
- [ ] Every read model has clear purpose
- [ ] Every data field has event source
- [ ] Update logic for each event is explicit
- [ ] All UI needs are covered
- [ ] Processor reads are identified
- [ ] Read model access patterns clear
- [ ] No undocumented data sources
- [ ] Compensation/cancellation handled
- [ ] Error states shown

### CRITICAL: Event vs Read Model Validation
- [ ] **Reviewed each read model**: "Is this pure calculation or an actual domain fact?"
- [ ] **No aggregations modeled as events**: (totals, averages, counts are read models)
- [ ] **No recalculated state modeled as events**: (if value changes multiple times, it's a read model)
- [ ] **Processor outputs are categorized**:
  - [ ] Produces NEW EVENT = actual domain fact (e.g., Payment Authorized)
  - [ ] Updates READ MODEL = calculation (e.g., Seller Rating Calculated)
  - [ ] Sends NOTIFICATION = info-only (no event or model)
- [ ] **History tracking is clear**: Derived state keeps history in the read model (a list field such as `history`), not as separate events

## Key Principles

1. **Event-Driven**: All data comes from events
2. **Projection-Based**: Read models are projections of events, never a source of truth
3. **UI-Focused**: Shaped for what the UI displays
4. **Processor-Friendly**: Enough data for the automation to see what work is outstanding
5. **Completeness**: All needed data available

## Common Patterns

### Status View Pattern
```
Events: Create, Confirm, Process, Ship
ReadModel: Accumulates data from all events
Displayed: Current state reflecting all events
```

### List View Pattern
```
Events: Create, Update, Delete (Cancel)
ReadModel: Summary of each item (list-shaped)
Used for: Filtering, sorting, searching
```

### Timeline View Pattern
```
Events: Any event with timestamp
ReadModel: Chronological list
Used for: History, audit trail
```

### Processor Decision Pattern
```
Events: Events that open and close outstanding work
ReadModel: The automation's todo list (Step 4b) — outstanding items only
Automation reads each open item to decide its next command
```
