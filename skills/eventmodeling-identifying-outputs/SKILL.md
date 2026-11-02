---
name: eventmodeling-identifying-outputs
description: "Step 5 of Event Modeling - Identify Outputs/Read Models from events. Show what data flows back to UI and Processors. Use after defining inputs. Do not use for: identifying commands or inputs (use eventmodeling-identifying-inputs) or verifying field completeness (use eventmodeling-checking-completeness)."
---

# Identifying Outputs

> Every tool call takes the `modelId` (from `list_models`), and a `sheet` when the model has more than one. Consult [tools.md](../eventmodeling-core-rules/references/tools.md) only if you need to look up a specific tool or field this file doesn't cover — don't load it eagerly.

This step applies the shared element rules in **`eventmodeling-core-rules`** — read it once per session if you haven't already; it defines what a COMMAND/EVENT/READMODEL/SCREEN/AUTOMATION is, how each is named, and the anti-patterns to reject, so this step doesn't restate them.

## Interview Phase (Optional)

**When to Interview**: Skip if the user has clearly identified: read model queries needed by UI, processor needs, and refresh patterns. Interview when unclear which data queries are critical or how frequently they're accessed.

### Critical Questions

1. **Query Patterns** (Impact: Determines which read models are needed and their update frequency)
   - Question: "What data do users/processors need to query? (A) Real-time (sub-second), (B) Near-real-time (seconds), (C) Periodic (minutes/hours)?"
   - Why it matters: Query frequency drives read model design and caching strategy
   - Follow-up triggers: If (A) → ask which specific screens or processors require sub-second reads; these need dedicated, highly optimized read models

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
- Inventory Level (projects available stock)

**Characteristics**:
- Calculated/aggregated state
- Recalculates when source events change
- Derived from other events
- Query optimization
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

Instead of writing a markdown document, **place each READMODEL (and any missing AUTOMATION) on the sheet** with `place_element`. Screens are typically already placed from Step 3 (storyboarding) as one plain screen per screen state — Step 3 does not pre-split anything, so working out how many components a screen actually has is this step's job (Step 5a below), not something to re-derive from storyboarding's output. Do not re-place an existing screen unless one is clearly missing. Automations are placed here only in the rare case where analysis reveals a processor that reads state and issues commands but was missed entirely in Step 4 — if that happens, immediately apply `eventmodeling-designing-automation-chains`'s rules to it (todo-list read model, translation chain if externally triggered) before continuing, rather than leaving it for Step 5i's defensive re-check to catch.

This step proceeds as a sequence of lettered sub-steps: identify each screen's components (5a), break multi-component screens apart into copies (5c), then design and place one read model per component (5d onward). Every automation's todo-list read model and any translation chain is designed in **Step 4b** (`eventmodeling-designing-automation-chains`), immediately after Step 4 — not here. This step only ever designs screen-facing read models; Step 5i below re-checks automations defensively, not as this step's primary job.

> **CRITICAL: Every READMODEL node MUST have `fields`, and every field must trace back to a connected source event (or be `isGenerated`).** A read model without fields, or with fields of unknown origin, has no data lineage and cannot be traced back to its source events.

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

**Before designing any read model, enumerate every SCREEN and AUTOMATION already placed on the sheet** (from Step 3 — Storyboarding). Storyboarding hands off one plain screen per screen state and does not pre-split anything — deciding how many components a screen actually has is this step's decision, because a component is defined by its read model: **one component in a screen resembles one read model.**

Read models exist to serve the elements already on the sheet:
- Every **view screen** (output/read model screen) needs at least one read model to supply its data.
- Every **automation** needs at least one read model to read from — its todo list. This applies to every automation without exception, not only ones that visibly "decide" something; this was already handled in Step 4b (`eventmodeling-designing-automation-chains`) for every automation placed in Step 4.
- Every **command/input screen** needs a read model unless it is a blank creation form with no prior state to display (this is the rare exception, not the rule).

For each SCREEN node, look at its design (`get_screen_design`, or `render_screen` for the picture) and its fields (`get_cell_view`) and identify its components — groups of fields/UI elements a user would perceive as one area: a stats tile, a list below it, a summary card, a detail panel, a status column in a table, etc. **Most screens genuinely have exactly one component — do not force a split.** A screen has more than one component only when a user would point at two separate areas and describe them as different things.

A single homogeneous list/table is still one component, even when its rows draw on many different event types (e.g. a catalog whose per-row status is set by many different lifecycle events scattered across the timeline) — don't split a table by row or by source event. **But do not use this as a reason to also fold in other fields that don't share that same wide fan-in.** A book's title/author (set by 1-2 events, updated rarely) and a copy's live availability status (derived across its full lifecycle, many events) are two different kinds of computation even when they render on the same visible page — a user pointing at "the book's info" versus "whether it's available right now" is describing two different things, whatever the layout looks like. If a candidate component mixes a field with irreducible wide fan-in and a field that only needs 1-2 events, that is **always** two components — split it now, at this step, before any read model is built from it, rather than discovering the mismatch after the fact.

Once a component's fields are genuinely homogeneous (every field needs the same wide-lifecycle fan-in, none of them are cheap identity/fact fields riding along), its read model will legitimately need wide fan-in. This is expected, not a modeling error — document it with a note (`set_note` on the read model's cell, per the orchestrating skill's "Documenting decisions inline, at any step") naming *which field(s)* have that irreducible fan-in and why, e.g. "Copy Availability View.copyStatus is one per-copy status derived across that copy's full lifecycle (repair, loss, reservation, return)." That note documents a specific field's fan-in, not a blanket exemption for the node — re-verify every field on the node against the ">3-events heuristic" (`eventmodeling-orchestrating-event-modeling`) at every later step that touches it, including Step 5i's mandatory per-node verification below, rather than treating a prior note as settled once and for all.

After this step is done, **every SCREEN and every AUTOMATION on the sheet must be connected to at least one read model** via a `READMODEL → SCREEN` or `READMODEL → AUTOMATION` connection, and every screen identified above as having 2+ components must have been broken apart per Step 5c before any read model is placed. If a screen or automation has no incoming read model connection, it is a gap — either a read model is missing or the connection arrow is missing.

> **Placement rule**: A read model must be placed immediately upstream of the SCREEN or AUTOMATION it serves — sharing that column when possible (a SCREEN with a free interaction row), or one column to the left when not (any AUTOMATION; a SCREEN whose column is already occupied). So a read model's consumer sits either in the read model's own column or in the very next one — both are correct, and a consumer one column to the right is **not** a gap and must never be reported as one. What *is* a gap is a read model with no SCREEN or AUTOMATION in either of those two places: an orphan that will never be displayed or acted on (`check_model` reports it as `read-model-unconsumed`).

> **Automation todo-list read models are designed in Step 4b, not here.** The full pattern — every automation's todo-list read model, the "no invisible signal" rule, and the two-chained-automation translation requirement for externally-triggered automations — now lives in `eventmodeling-designing-automation-chains` (Step 4b), which runs immediately after Step 4, before this step. If Step 4b ran, every automation already on the sheet has its todo-list read model wired; Step 5i below only re-checks this defensively. The rare exception is an automation discovered only now, during output analysis (see "Output Format" above) — if that happens, apply `eventmodeling-designing-automation-chains`'s rules to it directly rather than re-deriving them here.

### Step 5c — Break apart multi-component screens into copies

**Do not default to a single monolithic read model that supplies an entire screen.** For every screen identified in Step 5a as having 2+ components, break it apart into one screen copy per component before designing its read models. A copy here is a plain `screen` placed with `place_element`, not a `copy_element` linked copy: a linked screen copy shows its original's design and marks exactly, so it cannot call out a different component. Address each copy by its cell address (`C1.1`), since they all share one name.

- Each copy is the **same page**, so it **keeps the same screen name/title** — do not rename copies after their component (e.g. don't title one copy `"Librarian Dashboard — Statistics"` and another `"Librarian Dashboard — Recently Added"`; both stay `"Librarian Dashboard"`).
- Distinguish copies visually, not by name: **mark/highlight the one component each copy is about**, using the native Marks feature (`marks` on `place_element` / `set_screen_design`). In short: the component of interest gets `data-em-mark-id="<id>"` baked onto it in the page HTML, paired with a matching `marks` entry (`{id, color?, blurOutside: true}`) that blurs everything else on the page. Never hand-roll this with inline `filter`/`outline`/`opacity` CSS — that does not match how the app itself renders a mark, and `check_model` and board files cannot see it.
- Produce each copy from the original screen's markup (`get_screen_design`), marking the one component to keep crisp and blurring the rest: pass the marked markup as `design` and the `marks` entry to `place_element`.
- Mark the original's own component the same way (`set_screen_design` on the original, with `marks`). `whiteOutside` whites the rest out instead of blurring it; leave both off for an outline only. Check each copy with `render_screen`.
- Give each copy the fields of its own component, the ones its read model will feed (Step 5d groups them), and leave the original only its own component's fields, so each field shows on exactly one copy.
- The command a screen issues stays issued by exactly one screen copy — the one sitting in the command's own column. The other copies only ever receive `READMODEL → SCREEN` flows (`check_model` reports a command with several issuers as `command-multiple-issuers` and a screen wired to several commands as `screen-multiple-commands`).
- Place each copy in its own column, the **same column** as the read model that will feed it (insert a new column immediately before if that copy's column is already occupied) — this is normally a different column per copy, since each component typically has a different natural source event.

Why this matters, beyond tidiness:
- **It prevents backward arrows.** A single screen-wide read model is forced to aggregate from whatever events each of its components needs, which often means reaching back across many columns to events scattered throughout the timeline — and the read model can only sit in one column, so some of those connections end up spanning a wide gap or, worse, pushing the read model's column later than some of its screen's other consumers require. Splitting by component lets each narrower read model sit close to its own natural source event(s), keeping every `EVENT → READMODEL` arrow short and forward.
- **Components evolve independently.** A stats tile and a "recently added" list are driven by different events, change at different rates, and are typically owned by different slices in implementation. Bundling them into one read model couples their release/change cadence for no reason.

Before finalizing any read model, ask: "does this screen contain more than one component?" If yes and it wasn't already broken apart, do it now — don't ask whether it's worth the extra columns, it always is at this scale, since the alternative is a hidden coupling and a higher chance of a backward-arrow layout error.

### Step 5d — Pull field mappings from Step 3 — they are the spec, not a guess

**Do not re-derive read model needs from a screen's title or description alone, and do not rely on the orchestrator's phase-summary handoff for this** — if you arrived here via `eventmodeling-orchestrating-event-modeling`, the handoff after Step 3 is a short hand-written prose summary (the interview record, see `eventmodeling-interview-protocol`), not the actual field data. It will not reliably carry the per-field mappings forward. Go back to the sheet itself:

For every SCREEN node, read it directly (`get_slice_view` for the slice's elements with their fields, or `get_cell_view` for one cell together with its note — never from memory). Step 3 already required every view field to name its read model, as a line `Planned read model "<Read Model Title>": <field>, <field>` in the screen cell's note (a field cannot name a read model that isn't on the sheet yet). It was recorded so this step doesn't have to guess it again.

- **Group the screen's fields by the `<Read Model Title>` already named in those lines.** That grouping — not a fresh read of the screen's visuals — is the read model's title and field list. Build the READMODEL node from it directly, and remove each line from the note once its read model is placed and connected.
- A field named under a title that isn't on the sheet yet is a read-model reference. Treat it as a requirement, not a suggestion. (Fields marked `isUserInput` or `isGenerated` are input or display-only values. They need no read model.)
- A screen with no fields, or with fields that name no read model, is **not** evidence that it needs no read model. Re-check it against the three rules above (view screen / automation / command screen showing prior state) before concluding it's the rare blank-form exception — and say explicitly why it qualifies.

### Step 5e — Field data lineage — the `source` of READMODEL fields

Apply `eventmodeling-information-completeness` to every READMODEL field as you add it. A read model's upstream is the EVENTs connected to it by `EVENT → READMODEL` arrows. A value the projection calculates itself (a count, a sum, a duration) is `isGenerated: true`. A field no connected event supplies is a missing event field or a missing connection: add it before proceeding.

Lineage lives in the flows, not on the field: a field may carry `source: { from, field? }` — `from` is the source event's address or name on this sheet, `field` the field on it that feeds this one (defaults to the same name). `place_element` turns every `source` into the flow and its field mapping. A field with the same name as a field on a connected event needs no `source`; where the names differ and there is no `source`, state the correspondence with `set_flow_mapping` once the flow exists (Step 5h). If the source event isn't on the sheet yet, leave `source` off and declare the mapping once it lands: `create_flow`, then `set_flow_mapping` (by field name). `trace_field` follows any field back to where its value starts (`eventmodeling-tracing-data`).

A field that genuinely holds a list of values (e.g. line items, a collection projected from multiple events) is `isList: true`; leave it off for a single value, which is the default when unsure. Include only the fields the consuming SCREEN or AUTOMATION actually displays or needs — do not add speculative fields; enrich later via `eventmodeling-adding-attributes`. Give each field an `example` value.

If the READMODEL itself represents a set of rows rather than a single record — e.g. `Invoice List`, `Product List` — also set `isList: true` on the read model itself (`place_element` / `update_element`; see `eventmodeling-core-rules`'s READMODEL section). This is separate from a field's `isList`: `isList` on the read model says the whole read model is a list of elements shaped by its defined fields, `isList` on a field says one field on an otherwise-single-record read model holds multiple values.

```json
{
  "cell": "C2",
  "element": {
    "scope": "readmodel",
    "name": "Active Reservation View",
    "fields": [
      {"name": "reservationId",  "fieldType": "String",   "example": "res-001"},
      {"name": "customerId",     "fieldType": "String",   "example": "cust-42"},
      {"name": "bikeId",         "fieldType": "String",   "example": "bike-17"},
      {"name": "stationId",      "fieldType": "String",   "example": "stn-03"},
      {"name": "expiresAt",      "fieldType": "DateTime", "example": "2026-06-01T09:30:00Z",  "source": {"from": "Reservation Confirmed"}},
      {"name": "status",         "fieldType": "String",   "example": "confirmed",             "source": {"from": "Reservation Confirmed", "field": "toStatus"}},
      {"name": "minutesLeft",    "fieldType": "Int",      "example": "25",                    "isGenerated": true}
    ]
  }
}
```

The field shape is exactly `{name, fieldType, isList?, isGenerated?, isOptional?, isUserInput?, isPii?, example?}` (plus `subfields` for a Custom field) — any other key is refused. Field types: `UUID`, `Boolean`, `Double`, `Decimal`, `Date`, `DateTime`, `Long`, `Int`, `String`, `Custom`.

Read models go in the `interaction` lane. For a **SCREEN**, the primary read model shares the SCREEN's column (READMODEL in interaction row, SCREEN in actor row of the same column) — unless that column's interaction row is already occupied (e.g. a command/input screen, where the COMMAND already sits there), in which case the read model goes one column to the left instead. For an **AUTOMATION**, the read model always goes one column to the left, never the same column — an automation's own column already holds the COMMAND it issues. If a consumer needs more than one read model, only the primary one gets this placement; every additional read model goes further left still.

> **Timeline alignment rule**: Place the read model in the same column as its consumer screen when that column is free, or one column before its consumer (screen or automation) when it isn't. Insert the new column immediately **before** the consumer's column (`insert_slice` with `at` = the consumer's 1-based column position and `cols: 1`, shifting the consumer right) rather than after — the read model must sit upstream of (to the left of) the element it feeds. Do not append read model columns to the end of the timeline — doing so severs the visual left→right flow from data projection to UI consumption.

### Step 5f — Placing READMODEL and AUTOMATION elements by cell address (Mandatory)

**Every `place_element` call MUST include a `cell`** — the address of the column and lane the element goes in (`C2`). Placement is always by cell address, never by coordinates. A cell holds exactly one element, so first find a free interaction-lane cell in the right column.

**For a READMODEL** (interaction lane):

`place_element` collapses finding the empty interaction cell and creating the element into one call, but it does **not** resolve an occupied target cell for you: placing into an occupied cell fails outright (`CELL_OCCUPIED`) rather than making room. **Target column depends on the consumer type**:
- **SCREEN**: target the screen's own column. First check whether that column's interaction cell is already occupied (e.g. a command/input screen) — `get_sheet_view`. If it is, insert a new column immediately before the screen first (`insert_slice` with `at` = the screen's 1-based column position and `cols: 1` — this shifts the screen and everything after it one column right), then place the read model in that new column's interaction cell.
- **AUTOMATION**: target the column one to the *left* of the automation — never the automation's own column, since it already holds the COMMAND the automation issues.
```
place_element {
  "cell": "<interaction cell of the consumer screen's column>",   // or of the column left of an AUTOMATION consumer
  "element": {
    "scope": "readmodel",
    "name": "Active Reservation View",
    "fields": [...]
  }
}
```
Set `fields` in the same call (see Step 5e) — and `isList: true` on the read model if it is list-shaped. To find the consumer's column beforehand, or to check whether a specific interaction cell is already occupied, read the sheet (`get_sheet_view`, or `get_cell_view` for one cell).

**Addresses shift after every column insertion** — re-read the sheet before reusing one. A column cannot exist outside a slice: `insert_slice` gives the new column its own slice, so name it for the read model it will hold.

**Placement never connects anything in EventModeler.** Wire every `EVENT → READMODEL` / `READMODEL → SCREEN` / `READMODEL → AUTOMATION` flow explicitly in Step 5h. After wiring, run `check_model` — its `flow-backward` and `read-model-without-source` findings confirm the wiring is what you intended, and there are no stray edges to clean up (apart from the flows a field `source` declared in Step 5e).

**For an AUTOMATION** (actor lane) — its READMODEL always goes **one column to its left**, never the same column: the automation's own column already holds the COMMAND it issues (interaction row), so the read model can't also live there. Place the automation with `place_element` and `scope: "processor"` in the actor lane, and its read model as above.

### Step 5g — Preventing backward arrows (mandatory pre-placement check)

The timeline must always progress left-to-right, or downward within the same column. A `READMODEL → SCREEN` connection going right-to-left is a layout error.

The correct layout is: **READMODEL and its (primary) view SCREEN share the same column** — READMODEL in the interaction row, SCREEN in the actor row, a downward connection. The read model only sits in a separate column, immediately **before** the screen's, when the screen's column is already unavailable to it (its interaction row is occupied by something else, e.g. a COMMAND on a command/input screen). If a screen displays more than one read model, only the primary read model shares the screen's column — every additional read model goes further left, never to the right. Before placing each read model, find the view screen it serves and verify the column order:

```
For each view screen S that queries this read model as its primary read model:
  If column(S)'s interaction row is free:
    → Place the READMODEL in column(S) itself (same column as S). No adjustment needed.
  If column(S)'s interaction row is already occupied:
    → Insert a new column immediately before column(S) and place the READMODEL there instead.
    insert_slice { "name": "<read model name>", "at": <position of column(S)>, "cols": 1 }
  If the read model ends up more than one column away from S with nothing in between:
    → Gap between read model and screen. Close it: move the read model (`move_element`) into column(S) (if free) or column(S) - 1.
```

**View screens normally share the column of the (primary) read model they display** — either because they were placed there in Step 3, or because the read model was placed into the screen's own column just now (per the check above). A screen's own position is never moved to resolve a read model placement conflict — when the shared column isn't available, the **read model** gets a new column immediately before the screen's, not the other way around (this preserves the screen's narrative order from Step 3).

**Checking a flow's direction.** `check_model` reports every flow whose target column is left of its source column as `flow-backward`, with both element addresses, and exempts only the one case below. No tool rejects a backward flow, so run `check_model` after every flow this step creates. (`get_sheet_view` returns every flow with its endpoint addresses if you want to read the columns yourself.)

**The same rule applies to `EVENT → READMODEL`** (shared statement: `eventmodeling-core-rules`'s "Updating a Read Model That Already Feeds a Screen"). If a later event needs to update data a read model already feeds to a SCREEN, do not connect that later event back into the existing read model — `check_model` reports a backward `EVENT → READMODEL` flow as `flow-backward`, and exempts it only when the read model already has a `READMODEL → AUTOMATION` flow (the todo-list pattern from Step 4b, `eventmodeling-designing-automation-chains`). For any read model feeding a SCREEN, resolve the update the same way Step 5c resolves multi-component screens: place a **new READMODEL node** in a new column immediately after the later event's column, then link it to the earlier read model instance (`copy_element` — it's the same read model carried forward, not an unrelated new one; see `eventmodeling-core-rules`'s Linked Copies section). Connect the later event forward into the linked copy, then update the field(s) that event actually changes (a linked copy has no fields of its own here, so it cannot show a changed field: state what the event changes with `set_flow_mapping` on the flow into the copy). Place a matching SCREEN in that same new column — same title, updated data, optionally re-marked/highlighted with Marks (as in Step 5c); place it as a plain `screen` with `place_element`, not as a `copy_element` copy: a screen copy shows its original's design exactly. Give it the fields the updated read model feeds; what the later event changes is carried by the flow mapping and the screen design.

### Step 5h — Wire connections after placing each READMODEL (and its SCREEN)

After `place_element` returns the READMODEL's address, create the arrows that complete the slice (`create_flow` infers the flow type from the two elements):

1. **EVENT → READMODEL** — find the primary source EVENT node in the swimlane row of the same column (`get_sheet_view`), then connect it:
   ```
   create_flow { "from": "<event address>", "to": "<read model address>" }
   ```
   A flow a field `source` already created in Step 5e exists — don't repeat it (`create_flow` accepts a duplicate).

2. **READMODEL → SCREEN** — connect to the existing SCREEN node in the actor row of the same column (or the column immediately after, only when that column wasn't available to the read model — screens are typically already placed from Step 3). Several instances of one screen share a name, so address each by its cell address.
   ```
   create_flow { "from": "<read model address>", "to": "<screen address>" }
   ```

3. **READMODEL → AUTOMATION** — if the read model is consumed by an automatic process (scheduler, background job, external trigger), place the AUTOMATION node and connect it:
   - Place the AUTOMATION in the actor lane, in the column immediately to the right of its read model — unlike a view screen, this is unconditional for automations: the automation's own column always holds the COMMAND it issues, so its read model never shares that column.
   - Then connect:
   ```
   create_flow { "from": "<read model address>", "to": "<automation address>" }
   ```

Skip a connection silently if the target cell is empty, and create it when that element lands. Log each created arrow: `→ connected EVENT→READMODEL "Order Placed"→"Order Status View"`, `→ connected READMODEL→SCREEN "Order Status View"→"Order Status Screen"`, or `→ connected READMODEL→AUTOMATION "Order Status View"→"Fulfillment Processor"`.

Where a source and target field have different names, state the correspondence with `set_flow_mapping` (by field name). Make one `create_flow` call per pair; use `apply_operations` only if the unit must land all-or-nothing.

4. **Document the reasoning for each connected event** — for every EVENT → READMODEL edge wired in step 1 (including any added later, e.g. via Step 5g's backward-connection exemption or a Step 5c/copy pattern), record why that event feeds this read model: which field(s) it sets or updates, and why. Use one note per read model, on that read model's own cell (`set_note`, per `eventmodeling-orchestrating-event-modeling`'s "Documenting decisions inline, at any step"). Extend the existing note (don't create a second one) when the read model later gains another connected event: `set_note` replaces the note, so read it first (`get_cell_view`) and write it back extended.

   Example body:
   ```markdown
   ## Event → field reasoning — Order Status View

   - **Order Placed** → sets `orderId`, `status: "placed"`, `items[]` — the read model's creation event; the order doesn't exist before it.
   - **Order Shipped** → updates `status: "shipped"`, `trackingNumber` — the only event carrying a tracking number.
   - **Order Cancelled** → updates `status: "cancelled"` — terminal state, no further events expected after this.
   ```

### Step 5i — Mandatory per-node verification (run before declaring this step done)

Do not declare Step 5 complete on the strength of the read models you happened to design. Instead, **re-read every SCREEN and AUTOMATION node on the sheet** (don't rely on the list built earlier in this step, the sheet may have moved on) and check each one individually. Questions 1–3 need only element types and flows: `get_sheet_view` returns every element per cell plus the flow list. Read an element in full (`get_slice_view` or `get_cell_view`) only when you fix a gap and need its fields:

1. Does it now have an incoming `READMODEL → SCREEN` or `READMODEL → AUTOMATION` connection?
2. If it's a SCREEN and not connected — is it a provably blank creation form with no prior state? State the reason in one line (e.g. `"Register Account" screen: blank form, no prior state — exempt`). This exemption applies to screens only.
3. If it's an AUTOMATION and not connected, it is **never** exempt — this should already be resolved by Step 4b (`eventmodeling-designing-automation-chains`); if it isn't, apply that skill's rules now to identify its todo-list read model.
4. If a SCREEN is neither connected nor exempt, it is an **unresolved gap**. Fix it now: design the missing read model (pulling from its fields and planned-read-model lines as above) and wire the connection. Do not move to Step 6 with an unresolved gap silently carried forward — either fix it or explicitly flag it to the user as accepted debt (a `create_pin` on the area keeps it visible on the sheet).
5. Does any screen still carry more than one component undivided (a Step 5a/5c miss)? If so, break it apart now per Step 5c before counting it as resolved.
6. **Re-read every READMODEL too** and run the >3-events heuristic (`eventmodeling-orchestrating-event-modeling`) on each one field by field — including read models a MARKDOWN note already justified as a roll-up. A prior note documents one field's irreducible fan-in; it does not exempt the rest of that node's fields from this check. The failure mode this catches: a wide-fan-in field (e.g. live per-copy availability) bundled together with a cheap, low-fan-in identity/fact field (e.g. a title set by 1-2 events) that has nothing to do with the roll-up — that pairing is always two read models, never one, no matter how the note reads.
7. **Every READMODEL has its event-reasoning note (Step 5h.4)**, and that note accounts for *every* inbound `EVENT → READMODEL` edge on the node — not just the one from when it was first placed. If a read model gained a connected event later and the note wasn't extended, fix it now rather than carrying the gap forward.
8. **No backward flow** other than the todo-list exception (Step 5g): `check_model` reports none as `flow-backward`.

Run `check_model` once as the mechanical half of this pass — its `read-model-without-source` findings list every READMODEL with no inbound EVENT, `read-model-unconsumed` every READMODEL nothing reads, `screen-disconnected` every screen with no flows, `untraceable-field` every field no inbound flow supplies, and `flow-backward` catches a `READMODEL → SCREEN` that points left. No `check_model` code flags a column that ended up with two screens: check that by hand in `get_sheet_view`. Then do the judgement checks above (blank-form exemptions, the >3-events heuristic, note completeness) that the tool can't make. It does flag a read model built from more than three events as `read-model-fan-in` (info), which is the prompt for the heuristic, not its verdict.

List the result of this pass (connected / exempt / fixed) for every screen and automation checked — this list is the evidence the orchestrator's Step 5 gate ("every screen data need is satisfied by a read model") actually holds, not just an assumption.

After all read models, screens, automations, and connections are in place, present the Read Model Catalog summary as text to the user.

---

Older versions of this skill wrote the read model catalog as a markdown document rather than placing elements on the sheet — that legacy template is kept in `references/examples.md` for reference only; it is not the actual output mechanism.

## Quality Checklist

### Read Model Design
- [ ] **Typical pattern applied**: most screens follow `READ MODEL → SCREEN → COMMAND → EVENT`
- [ ] **Every SCREEN from storyboarding is connected to at least one read model** (via `READMODEL → SCREEN`); only blank creation forms may be exempt — verified via the mandatory per-node pass above, not assumed
- [ ] **Every AUTOMATION from storyboarding is connected to at least one todo-list read model** (via `READMODEL → AUTOMATION`, Step 4b) — no exemption for automations, unlike screens; even a simple relay automation gets one
- [ ] **No automation's todo list is opened directly by another system's (second-swimlane) EVENT unless that automation is itself the translation automation** (Step 4b) — an automation doing the actual domain work is only ever opened by an internal event; a second-swimlane EVENT feeding straight into a work automation's todo list is a missing translation automation
- [ ] **No read model is placed without a connected SCREEN or AUTOMATION consumer**
- [ ] **No read model spans more than one component** — a screen with N distinct components gets N read models and N highlighted screen copies (same screen name, one component crisp per copy), not one screen-wide read model
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
- [ ] **History tracking is clear**: Derived state keeps history in read model `history[]`, not as separate events

## Key Principles

1. **Event-Driven**: All data comes from events
2. **Projection-Based**: Read models are projections, not persistent
3. **UI-Focused**: Optimized for UI display needs
4. **Processor-Friendly**: Enough data for processor decisions
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
ReadModel: Summary of each item
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
Events: State-changing events
ReadModel: Current state only
Processor reads to decide next action
```
