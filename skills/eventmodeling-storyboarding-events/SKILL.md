---
name: eventmodeling-storyboarding-events
description: "Step 3 of Event Modeling - Create UI storyboards/mockups showing what users see at each step. Capture all data fields needed from user perspective. Use after sequencing events. Do not use for: identifying commands or processor actions (use eventmodeling-identifying-inputs) or designing read models (use eventmodeling-identifying-outputs)."
---

# Storyboarding Events

> Every tool call takes the `modelId` (from `list_models`), and a `sheet` when the model has more than one. Consult [tools.md](../eventmodeling-core-rules/references/tools.md) only if you need to look up a specific tool or field this file doesn't cover — don't load it eagerly. Element kinds are written as `screen` and `processor` (an AUTOMATION), among others. Positions are cell addresses (`B2`: column letter, row number), never coordinates. Addresses are positional: after inserting or moving a column or lane, re-read the sheet before using one.

This step applies the shared element rules in **`eventmodeling-core-rules`** — read it once per session if you haven't already; it defines what a COMMAND/EVENT/READMODEL/SCREEN/AUTOMATION is, how each is named, and the anti-patterns to reject, so this step doesn't restate them.

## Interview Phase (Optional)

**When to Interview**: Skip if the user has already specified: existing UI patterns or mockups to reference, critical data fields, and UI/UX preferences. Interview when these details haven't been discussed or when the user wants guidance on storyboarding depth.

### Critical Questions

1. **Current UI State** (Impact: Determines if you're designing from scratch or enhancing existing)
   - Question: "Do you have: (A) Existing UI/wireframes to reference, (B) Rough sketches, (C) Starting from scratch?"
   - Why it matters: Existing UI provides constraints and patterns; starting fresh allows more design freedom
   - Follow-up triggers: If (A) → ask to share; if (C) → ask about platform/technology

2. **Most Critical Data Fields** (Impact: Determines storyboard focus and detail level)
   - Question: "Which data fields are most important for users to see? (e.g., order status, payment confirmation, tracking info)"
   - Why it matters: Knowing priorities helps avoid over-designing; users need to see what matters most first
   - Follow-up triggers: For each critical field → ask "What decisions do users make based on this data?"

3. **UI/UX Preferences & Constraints** (Impact: Shapes storyboard style and interaction patterns)
   - Question: "Any UI preferences? (A) Web, (B) Mobile, (C) Both. And design style: (A) Minimal wireframes, (B) Detailed mockups, (C) Interact prototypes?"
   - Why it matters: Platform and fidelity affect storyboard detail; mobile has different constraints than web
   - Follow-up triggers: If (C) → ask about prototype tool; if minimal → discuss what level of detail is enough

Follow **`eventmodeling-interview-protocol`** to run this interview and record its findings — label this step "**3. Storyboarding** (`eventmodeling-storyboarding-events`)". Findings should cover: existing UI context, most critical data fields (priority order), platform and detail-level preferences, key UI interactions, and storyboard focus.

---

## Workflow

Given the event timeline, create UI storyboards:

### 1. Identify UI Screens/Views
Create a mockup for each state of the system: for each screen, note the trigger action, the command it produces, the resulting event, and the data fields the screen captures. A full worked example (Order Creation Form) is in `references/examples.md`.

### 2. Show State Transitions Between Screens
Document what changes when events occur: after each event, the next screen shows the fields that were just set by that event, alongside the next command the user can trigger. A full worked example (Order Confirmation, after Order Created) is in `references/examples.md`.

### 3. Document All Data Fields
For each screen, list what data is displayed, and the specific event each field's value originated from. A full worked example (Order Status View) is in `references/examples.md`.

### 4. Show Data Flow Through Screens
Map how data enters/exits UI: user input flows into a command, the command produces an event, and the event's data flows back out into the next screen that displays it. A full worked example is in `references/examples.md`.

### 5. Organize Screens by Swimlane (Actor/System)

In this section a "swimlane" is the narrative grouping of screens by who uses them. Its physical form on the sheet is an actor lane (see below). It is not the event-lane kind called `swimlane`.

**MANDATORY**: Use the **Role Catalog** from Step 1 (eventmodeling-brainstorming-events) as the source of swimlanes. Every human role in the catalog MUST have its own swimlane. Every system actor that has a UI or todo-list view gets a swimlane too — but this swimlane is narrative-only (see "Model Integration" below): system actors never get a physical actor lane of their own on the sheet, only human roles do.

Group screens by who interacts with them: one swimlane per human role (Customer, Seller, Support Agent, ...) listing that role's screens, plus one narrative swimlane per system actor (Payment Processor, Inventory System, ...) listing the screens/views it interacts with. A full worked example (Order domain swimlane grouping) is in `references/examples.md`.

**Validation**: If a role from the catalog has zero screens, either:
- The role is missing screens (add them), or
- The role doesn't belong in the catalog (remove it in Step 1)

This shows which actors interact with which screens and helps visualize system boundaries.

**This grouping is not just narrative for human roles** — "Model Integration" below turns each *human role's* swimlane in this catalog into its own physical actor lane on the sheet, so a screen's role determines which lane it is actually placed in, not just how it is described in the report. System actor swimlanes stay narrative-only: their automations are placed in the sheet's one shared automation lane, never a lane fabricated to mimic a human role's lane (see "Placing Automations" below).

### 6. Show Processor "Todo List" Pattern
For automated processors, show the todo list metaphor: each received triggering event adds a todo item, the processor checks each item's condition, and success or failure produces a corresponding event while marking the item done or failed. A full worked example (Inventory Reserver's todo list) is in `references/examples.md`.

**When it comes time to elaborate scenarios for this todo list (`eventmodeling-elaborating-scenarios`), reach for a storyline rather than plain GWT scenarios.** A todo list is exactly the shape a storyline is built for: the *same* read model (the todo list itself) walked through multiple states — empty → item added → item marked done/failed — which is one narrated walkthrough, not a set of isolated before/after pairs. See that skill's storyline sections for the data shape and `set_scenario` mechanics.

### 7. Identify Missing Data
Highlight where data doesn't have a clear source:

```
Problem: Order Status screen needs "expectedShip" date
Current state: Not in any event
Solution: Add expectedShip to Inventory Reserved event

Problem: Order status needs "last updated" timestamp
Current state: No tracking of when last change occurred
Solution: Every event includes timestamp
```

`trace_field` follows one field to where its value starts or is used (`eventmodeling-tracing-data`).

## Model Integration

Before starting the analysis, read existing screens from the sheet to avoid designing screens that already exist. `get_sheet_view` lists every element by name and kind per cell, and for each screen its `designMode`: screens created by this skill carry an HTML design (`designMode: html`), but a sheet may also hold hand-drawn sketch screens (`designMode: excalidraw`) — check both types. Titles and field names are all this dedupe needs (`get_slice_view` adds fields); read a screen's design with `get_screen_design` only if you must inspect it. A screen whose `designMode` is `excalidraw` is a sketch a person drew: read it as a screen, but leave it exactly as it is.

After completing the screen analysis, flag every screen node where data fields are unclear or missing sources are identified: if a human has to decide, leave a `create_pin` on the screen's cell worded as the question; if it is a finding that needs no answer, record it with `set_note` on that cell (a note attaches to a cell, a column or a lane, never to an element).

## Resolve One Actor Lane Per Human Role (do this once, before placing any screens)

**Each human role gets its own physical lane — never place two different human roles' screens in the same actor row.** **System actors and processors are not part of this map** — they are never given a labeled lane of their own; every automation renders in the sheet's one shared automation lane instead (see "Placing Automations" below). A sheet can hold several lanes of kind `actor` at once. Build a role→lane map covering human roles only, once per sheet, before the screen-placement loop, instead of resolving it screen-by-screen:

1. Read the sheet (`get_sheet_view`) and collect every row whose lane kind is `actor`, keyed by its name.

2. For every **human role only** in the Role Catalog (Step 1's swimlane list above), check the map for a name that matches the role name (case-insensitive). If found, reuse that row. **Skip system actors/processors entirely** — do not create or look up a lane for them here; they never get an entry in this map.

3. **If no matching lane exists, create one** — named with the role name, so the lane is visibly identifiable on the sheet:

   ```
   insert_lane { "kind": "actor", "name": "<Role Name>", "at": <row> }
   ```

   Pass `at` so the lane sits above the interaction lane. Add the new lane's row number to the map under that role's name. Do this once per role, not once per screen. A lane that exists under the wrong name is renamed with `update_lane`, never duplicated.

4. **Leave the sheet's shared automation lane alone** — don't rename it or force the first role into it. It is fine for it to stay unused by human roles; every role, including the first one, gets a freshly named lane from Step 3.

The result is a `{ roleName: row number }` map, covering human roles only, used by every SCREEN placed on this sheet (Step A below) — resolve it once, not per screen, and re-read the sheet and extend it only if a new human role appears mid-session that wasn't in the original catalog (inserting a lane shifts the row numbers below it). **AUTOMATION nodes never consult this map** — they always target the sheet's shared automation lane (see "Placing Automations" below).

## Mandatory Field Definitions on Screen Nodes

> **CRITICAL: Every screen node MUST include `fields`, and every field must have a known source.** A screen without fields cannot show data lineage — it becomes impossible to verify that all displayed data has a source event or command. This applies regardless of how the screen is designed (an HTML design by default, or a hand-drawn sketch).

Apply `eventmodeling-information-completeness` to every screen field as you add it. In short: input the user enters is `isUserInput: true`; session values and display-only calculations are `isGenerated: true`. Displayed data names its read model, even when that read model isn't designed yet, which Step 5 then groups by. A field can't name a read model that isn't on the sheet yet, so record it as a line `Planned read model "<Read Model Title>": <field>, <field>` in the screen cell's note (`set_note` replaces the note: read it first with `get_cell_view` and write old plus new). A screen field never maps to the command it submits. A displayed field is never marked `isGenerated` just because its read model is missing.

A field is exactly `{name, fieldType, isList?, isGenerated?, isOptional?, isUserInput?, isPii?, example?}` (plus `subfields` for a Custom field); any other key is refused. Lineage between elements is not stored on the field: it lives in flows. Once the read model is on the sheet and flows into the screen, a same-named field needs nothing; a differently named one is declared with `source: { from, field }` on the field when you write it, or with `set_flow_mapping` afterwards.

### Example: command screen fields

```json
{
  "scope": "screen",
  "name": "Reserve a Bike",
  "fields": [
    {"name": "bikeId",      "fieldType": "String",   "example": "bike-17",               "isUserInput": true},
    {"name": "stationId",   "fieldType": "String",   "example": "stn-03",                "isUserInput": true},
    {"name": "startTime",   "fieldType": "DateTime", "example": "2026-06-01T09:00:00Z",  "isUserInput": true},
    {"name": "returnTime",  "fieldType": "DateTime", "example": "2026-06-01T17:00:00Z",  "isUserInput": true},
    {"name": "bikeCategory","fieldType": "String",   "example": "City Bike"},
    {"name": "dailyRate",   "fieldType": "Decimal",  "example": "0.10"}
  ]
}
```

`bikeCategory` and `dailyRate` are displayed data from a read model that isn't on the sheet yet, so the screen's cell note names it:

```
set_note { "target": "E2", "text": "Planned read model \"Available Bike View\": bikeCategory, dailyRate (from ratePerMinute)" }
```

### Example: view screen fields (Read Model as source)

```json
{
  "scope": "screen",
  "name": "Reservation Confirmed",
  "fields": [
    {"name": "reservationId",  "fieldType": "String",   "example": "res-001"},
    {"name": "bikeName",       "fieldType": "String",   "example": "City Bike — Gazelle"},
    {"name": "stationName",    "fieldType": "String",   "example": "Central Park East"},
    {"name": "startTime",      "fieldType": "DateTime", "example": "2026-06-01T09:00:00Z"},
    {"name": "expiresAt",      "fieldType": "DateTime", "example": "2026-06-01T09:30:00Z"},
    {"name": "estimatedCost",  "fieldType": "Decimal",  "example": "48.00",                 "isGenerated": true}
  ]
}
```

```
set_note { "target": "F2", "text": "Planned read model \"Active Reservation View\": reservationId, bikeName, stationName, startTime, expiresAt" }
```

A read model named in a planned-read-model line but not on the sheet (or not connected to this screen) is a **gap** for Step 5 (Identifying Outputs) to close. `check_model` reports a screen's fields with no stated origin as `unsourced-fields` (info) until it lands. Flag it in the completeness notes, and don't invent connections that don't exist.

A field that genuinely displays/collects a list of values (e.g. a repeated line-item row) is `isList: true`; leave it off for a single value, which is the default when unsure. Only add the fields the screen actually shows or captures — do not add speculative fields; enrich later via `eventmodeling-adding-attributes` (`update_element` with `fields` replaces the list: resend every field, and unchanged names keep their ids).

A screen that only has a title and no fields is an empty placeholder — place the fields before moving on.

## Mandatory Screen Design

Every screen node requires a design. **An HTML design is the default for every screen** — a real HTML/CSS mockup, placed with `place_element` or set with `set_screen_design`, not a wireframe sketch. If the user asks for a "sketch", "wireframe", or "low-fidelity mockup", place the screen with a deliberately plain HTML design; an Excalidraw sketch someone has drawn is left untouched. The correct order for every screen is:

**Step A — Find the cell.** This applies to SCREEN nodes (human roles only) — AUTOMATION nodes follow "Placing Automations" below instead. Screens go in **that screen's own role's actor lane** in their target column — look up the row in the role→lane map built above, keyed by the screen's role (e.g. "Admin", "User"). Never fall back to "the" actor lane as if there were only one.

1. Determine the target column (same column as the event/command for a command/input screen, OR the same column as the read model for a view/output screen — one column to the right only if that shared column isn't available).
2. Row = the role's entry in the role→lane map — the map was already resolved once for the whole sheet; do not re-read the sheet per screen. If this screen's role is genuinely new (wasn't in the original Role Catalog), resolve/create its lane now the same way (see above) and add it to the map before continuing.
3. Cell address = column letter + row number (column E, this role's lane in row 2 → `E2`). A cell holds one element: an occupied cell fails with `CELL_OCCUPIED`.

**Step B — Place the screen with its design, in one atomic call.** Use `place_element` with `design` — this creates the screen, places it in that cell and sets its design together, so there is no window where the screen exists without content:

```
place_element {
  "cell": "E2",
  "element": { "scope": "screen", "name": "<Screen Title>", "fields": [ /* per "Mandatory Field Definitions" above — set in this same call */ ] },
  "design": "<div>...</div>"
}
```

The `place_element` call above already sets the screen's name and `fields` in the same call — no separate follow-up needed.

A storyboard screen is placed at a *provisional* position — Steps 4 and 5 wire it to its COMMAND / READMODEL once those exist, and may move it first (`move_element`). Placing a screen creates no flows, so the real `SCREEN → COMMAND` and `READMODEL → SCREEN` edges are created deliberately in Steps 4 and 5.

Design the screen as real HTML/CSS: write full-size markup (readable body text, generous padding — the canvas scales it down, don't shrink it yourself), one complete self-contained fragment (no `<html>`/`<head>`/`<body>` wrapper — the canvas adds those), no `<script>`/inline handlers (stripped server-side), and the model's theme classes: define the look once with `set_screen_theme` (design tokens plus component classes such as `.btn`, `.card`) so every screen stays a few lines of semantic markup. Scripts, inline event handlers, iframes, forms, links and non-data-URI images are stripped; `<style>` blocks, inline styles, form controls (`input`, `button`, `select`, `textarea`, `label`) and data-URI images survive, and the result reports what was `stripped`. `get_screen_design` on an existing HTML screen returns the model's `themeCss`; when no HTML screen exists yet (nothing to read it from), call `set_screen_theme` before designing the first one. Every design MUST include real field labels matching the actual event/command fields this screen captures or displays, and at least one primary action (submit/confirm button) for command screens.

> **CRITICAL: NEVER pass an empty `design`.** An empty design produces a blank placeholder and is always wrong. You MUST design and include actual markup in the same call that places the screen. To change a design later, use `set_screen_design`. Check the result with `render_screen` (one screen) or `render_sheet` (the sheet as the canvas draws it).

A screen node without a design is an empty placeholder — a bare screen is an unfinished thought. It must not be left that way.

## Timeline Placement Rules

When placing screens on the sheet, follow these alignment rules:

| Screen type | Where it goes on the sheet |
|-------------|---------------------------|
| **Input/command screen** (triggers a command) | **The role's own actor lane, same column as the COMMAND and EVENT** it produces. The screen and command share a column — the screen sits in that role's actor lane, the command in the interaction row, the event in the swimlane row. |
| **View/output screen** (displays a read model) | **The role's own actor lane, the SAME column as the READ MODEL** it displays (READMODEL in the interaction row, screen in the actor row — a downward connection, not a backward one). Only bumps one column to the right if that column's interaction row is already taken by something else. If the screen displays more than one read model, only the primary one shares its column — every additional read model goes further left. This column is finalised in Step 5 (Identifying Outputs) — during storyboarding, just document which read model each view screen will query. |

> **Do not create standalone screen columns that are disconnected from commands or read models.** Every screen must either share its column with the command it submits, or share its column with the (primary) read model it displays.

### Multi-component screens are broken apart in Step 5, not here

Storyboarding renders **one plain screen per screen state** — do not pre-split a screen into per-component copies here. Deciding how many components a view screen actually has, and breaking it apart into one highlighted copy per component, is `eventmodeling-identifying-outputs`'s job (its "Step 5a — Enumerate consumers and identify components" and "Step 5c — Break apart multi-component screens into copies"), because a component is defined by its read model and read models aren't designed until Step 5. During storyboarding, just place the single screen in the same column where its read model will end up (per the table above); document which read model it will query even before that read model exists.

### Placing Automations

When a processor or system actor reacts to events automatically (no human interaction), place an **AUTOMATION** node (`scope: "processor"`) in the sheet's **one shared automation actor lane** instead of a SCREEN — never create, reuse, or look up a per-system-actor lane for it, and never resolve it through the human role→lane map above. If the sheet has no such lane yet, create it once (`insert_lane` with `kind: "actor"`, named for what it is, e.g. "Automations", not for any one system actor) and reuse it for every automation on the sheet. Automations go in the same column as the COMMAND they trigger. Unlike a view screen, an automation's READMODEL is never in that same column — the automation's own column already holds the COMMAND it issues (interaction row), so the read model that feeds it always goes one column to the left.

**Do not design automation actor lanes to mimic human ones.** A "Payment Processor" or "Inventory System" swimlane in the narrative report (Step 5 above) is a documentation grouping only — it must never be materialized as its own labeled `actor`-type lane on the sheet. Only human roles get a physical lane; every automation, regardless of which system actor it narratively belongs to, renders in the same shared automation lane.

A column is an automation column (not a screen column) when:
- The action is triggered by the system, not a user gesture
- No human sees or interacts with the UI at this step
- The pattern is `READMODEL → AUTOMATION → COMMAND` (the processor reads state, decides, issues a command)

Examples:
- Payment gateway webhook arrives → AUTOMATION "Authorize Payment"
- Inventory check fires after payment authorized → AUTOMATION "Reserve Inventory"
- Notification service sends an email → AUTOMATION "Send Confirmation Email"

Human roles get SCREEN nodes placed in their own labeled actor lane. System actors and processors get AUTOMATION nodes placed in the shared automation lane. Place both types during storyboarding — do not defer automations to a later step.

### Maintain a Consistent Actor Perspective Across Columns

Tell the story from **one actor's perspective as much as possible**. When walking the timeline column by column, keep placing screens in the same role's lane as the previous screen unless the process genuinely hands the story off to a different actor (e.g. a Member requests a reservation, then the Librarian must act on that request — that's a real handoff, place the Librarian's screen next). Do not alternate actors column-to-column just because a different role's read model also changed at that point — a state change only earns its own screen when someone would actually look at it as the next beat in the story, not for symmetry between lanes.

Before placing a screen, ask "whose turn is it in the story?" If the answer is still the same actor as the last screen, keep the story there — it's fine, and expected, for a stretch of consecutive columns to belong entirely to one actor while another actor's lane sits empty. It's also fine to occasionally cut away to show what another actor sees, but that should read as a deliberate narrative beat, not a mechanical alternation.

```
✅ Consistent: Member screens in cols 1, 2, 3 (their whole request flow), then Librarian screens in cols 4, 5 (their whole fulfillment flow)
❌ Checkerboard: Librarian, Member, Librarian, Member, Librarian across cols 1-5 with no real handoff driving each switch
```

### One Screen Per Column (Hard Rule)

**Never place more than one SCREEN node in the same column**, even across different actor lanes. Each column represents a single moment in the timeline. Two screens in the same column means two different interactions at the same moment — that breaks the visual narrative and always signals a design error.

```
✅ Correct: Member "Reserve Book" in col 8  |  Librarian "Confirm Checkout" in col 11
❌ Wrong:   Member "My Loans" AND Librarian "Confirm Checkout" both in col 11
```

If a second role also needs a screen related to the same event, insert a new column immediately after (`insert_slice` with `at` = the position just after that column, `cols: 1`, named for what the column holds) and place the second screen there. No `check_model` code flags two screens in one column: check it by hand in `get_sheet_view`.

---

## Output Format

Older versions of this skill wrote the storyboard as a markdown document (swimlane organization, one section per screen, processor todo lists, a field traceability matrix) rather than placing designed screens on the sheet — that legacy template is kept in `references/examples.md` for reference only; it is not the actual output mechanism. The actual output is the designed screen and AUTOMATION nodes placed per "Mandatory Screen Design" and "Timeline Placement Rules" above.

## Quality Checklist

- [ ] **Every screen has a design** — an HTML design by default (`place_element` with `design`, or `set_screen_design` since) — no exceptions
- [ ] **No column contains more than one SCREEN node** across all actor lanes
- [ ] **Screens follow one actor's perspective across consecutive columns** — no mechanical alternation between two actors' lanes column-to-column; a different actor's screen appears only where the story genuinely hands off to them
- [ ] Every screen's wireframe shows real field labels matching the event/command fields
- [ ] Every displayed field has a source event
- [ ] Every user action maps to a command
- [ ] Commands map to events
- [ ] Data flows make sense
- [ ] No missing data sources
- [ ] State transitions are clear
- [ ] Alternative states are shown
- [ ] Error states are shown
- [ ] **Every human role from the Role Catalog has at least one swimlane**
- [ ] **Every human-role swimlane is labeled with the role name from the catalog**
- [ ] **Swimlanes organized by actor/system in the narrative report**
- [ ] **Every human role's swimlane is a real, distinct `actor`-type lane on the sheet, not just a grouping in the markdown report** — no two different human roles share the same lane
- [ ] **No system actor / processor has been given its own labeled actor lane** — every AUTOMATION node sits in the sheet's one shared automation lane, never a lane fabricated to mimic a human role's
- [ ] **Human role screens clearly separated from processor screens**
- [ ] **Processor todo list pattern shown for automated systems**
- [ ] **System boundaries visible through swimlane organization**

## Key Principles

1. **User-Centric**: Design from what users see and do
2. **Data Traceability**: Every field has origin and destination
3. **Completeness**: All needed data is visible
4. **Clarity**: UI clearly shows system state
5. **Consistency**: Same data presented consistently across screens
6. **Narrative Continuity**: Tell the story from one actor's perspective across consecutive columns; switch actors only for a genuine handoff, not for symmetry between lanes

## Common Patterns

### Input Screen Pattern
```
User fills form (captures command data)
  ↓
Submit button (issues command)
  ↓
Event created with form data
  ↓
Confirmation screen displayed
```

### Status Screen Pattern
```
System displays current state (from read model)
  ↓
Based on latest events
  ↓
Shows all relevant information
  ↓
Available actions based on state
```

### Error State Pattern
```
User action fails (command rejected)
  ↓
No event created
  ↓
Error message displayed
  ↓
UI allows retry or alternative action
```
