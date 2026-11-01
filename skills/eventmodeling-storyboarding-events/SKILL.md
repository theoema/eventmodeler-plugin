---
name: eventmodeling-storyboarding-events
description: "Step 3 of Event Modeling - Create UI storyboards/mockups showing what users see at each step. Capture all data fields needed from user perspective. Use after sequencing events. Do not use for: identifying commands or processor actions (use eventmodeling-identifying-inputs) or designing read models (use eventmodeling-identifying-outputs)."
---

# Storyboarding Events

This step applies the shared element rules in **`eventmodeling-core-rules`**. Read it once per session if you haven't already. It defines what a COMMAND/EVENT/READMODEL/SCREEN/AUTOMATION is, how each is named, and the anti-patterns to reject, so this step doesn't restate them. Its `../eventmodeling-core-rules/references/tools.md` maps the vocabulary below to this server's tools.

Every tool call carries `modelId` (from `list_models`). `sheet` may be omitted only when the model has exactly one sheet. Element kinds are written as `screen` and `processor` (an AUTOMATION), among others. Positions are cell addresses (`B2`: column letter, row number), never coordinates. Addresses are positional: after inserting or moving a column or lane, re-read the sheet before using one.

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

Follow **`eventmodeling-interview-protocol`** to run this interview and record its findings. Label this step "**3. Storyboarding** (`eventmodeling-storyboarding-events`)". Findings should cover: existing UI context, most critical data fields (priority order), platform and detail-level preferences, key UI interactions, and storyboard focus.

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

In this section a "role swimlane" is the narrative grouping of screens by who uses them. Its physical form on the sheet is an actor lane (see below). It is not the event-lane kind called `swimlane`.

**MANDATORY**: Use the **Role Catalog** from Step 1 (eventmodeling-brainstorming-events) as the source of role swimlanes. Every human role in the catalog MUST have its own role swimlane. Every system actor that has a UI or todo-list view gets one too, but this swimlane is narrative-only (see "Model Integration" below): system actors never get a physical actor lane of their own on the sheet, only human roles do.

Group screens by who interacts with them: one role swimlane per human role (Customer, Seller, Support Agent, ...) listing that role's screens, plus one narrative swimlane per system actor (Payment Processor, Inventory System, ...) listing the screens/views it interacts with. A full worked example (Order domain swimlane grouping) is in `references/examples.md`.

**Validation**: If a role from the catalog has zero screens, either:
- The role is missing screens (add them), or
- The role doesn't belong in the catalog (remove it in Step 1)

This shows which actors interact with which screens and helps visualize system boundaries.

**This grouping is not just narrative for human roles.** "Model Integration" below turns each *human role's* swimlane in this catalog into its own physical actor lane on the sheet, so a screen's role determines which lane it is actually placed in, not just how it is described in the report. System actor swimlanes stay narrative-only: their automations are placed in the sheet's one shared automation actor lane, never a lane fabricated to mimic a human role's lane (see "Placing Automations" below).

### 6. Show Processor "Todo List" Pattern
For automated processors, show the todo list metaphor: each received triggering event adds a todo item, the processor checks each item's condition, and success or failure produces a corresponding event while marking the item done or failed. A full worked example (InventoryReserver's todo list) is in `references/examples.md`.

**When it comes time to elaborate scenarios for this todo list (`eventmodeling-elaborating-scenarios`), reach for a storyline rather than plain GWT scenarios.** A todo list is exactly the shape a storyline is built for: the *same* read model (the todo list itself) walked through multiple states (empty → item added → item marked done/failed), which is one narrated walkthrough, not a set of isolated before/after pairs. See that skill's "Storylines" section for the data shape and posting mechanics.

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

## Model Integration

Before starting the analysis, read the existing screens from the sheet to avoid designing screens that already exist. `get_sheet_view` lists every element by name and kind per cell, and for each screen its `designMode`. `get_slice_view` adds fields when you need them. A screen whose `designMode` is `excalidraw` is a hand-drawn sketch, which a human keeps editing by hand: read it as a screen, but leave it as it is unless the user asks for a replacement design. Screens you create here carry an HTML design (`designMode: html`).

After completing the screen analysis, flag every screen where data fields are unclear or missing sources are identified. If a human has to decide, leave a `create_pin` on the screen's cell worded as the question. If it is a finding that needs no answer, record it with `set_note` on that cell (notes attach to cells, columns and lanes, never to an element).

## Resolve One Actor Lane Per Human Role (do this once, before placing any screens)

**Each human role gets its own physical lane. Never place two different human roles' screens in the same actor lane.** **System actors and processors are not part of this map.** They are never given a labeled lane of their own; every automation renders in the sheet's one shared automation lane instead (see "Placing Automations" below). A sheet can hold several lanes of kind `actor`. Build a role→lane map covering human roles only, once per sheet, before the screen-placement loop, instead of resolving it screen-by-screen:

1. Read the sheet (`get_sheet_view`) and collect every row whose `laneKind` is `actor`, keyed by its `name`.

2. For every **human role only** in the Role Catalog (Step 1's role swimlane list above), check the map for a name that matches the role name (case-insensitive). If found, reuse that row. **Skip system actors/processors entirely.** Do not create or look up a lane for them here; they never get an entry in this map.

3. **If no matching lane exists, create one**, named with the role name, so the lane is visibly identifiable on the sheet:

   ```
   insert_lane { sheet, kind: "actor", name: "<Role Name>", at: <row> }
   ```

   Keep every actor lane above the interaction lane (lanes read top to bottom: actor, interaction, swimlane(s), specification). Do this once per role, not once per screen. A lane that exists under the wrong name is renamed with `update_lane`, never duplicated.

The result is a `{ roleName: row number }` map, covering human roles only, used by every SCREEN placed on this sheet. Resolve it once, not per screen. Re-read the sheet and extend it only if a new human role appears mid-session that wasn't in the original catalog, because inserting a lane shifts the row numbers below it. **AUTOMATION elements never consult this map** (see "Placing Automations" below).

## Mandatory Field Definitions on Screens

> **CRITICAL: Every screen MUST include `fields`, and every field must have a known source.** A screen without fields cannot show data lineage. It becomes impossible to verify that all displayed data has a source event or command.

Apply `eventmodeling-information-completeness` to every screen field as you add it. In short: input the user enters is `isUserInput: true`; session values and display-only calculations are `isGenerated: true`. Displayed data comes from the read model it is shown from, even when that read model isn't designed yet, which Step 5 then groups by. A screen field never maps to the command it submits.

A field is exactly `{name, fieldType, isList?, isGenerated?, isOptional?, isUserInput?, isPii?, example?}`. Any other key is silently dropped. Lineage between elements is not stored on the field: it lives in flows. Once the read model is on the sheet and flows into the screen, a same-named field needs nothing; a differently named one is declared with `source: { from, field }` on the field when you write it, or with `set_flow_mapping` afterwards.

### Example: command screen fields

```json
{
  "scope": "screen",
  "name": "Reserve a Bike",
  "fields": [
    {"name": "bikeId",       "fieldType": "String",   "example": "bike-17",              "isUserInput": true},
    {"name": "stationId",    "fieldType": "String",   "example": "stn-03",               "isUserInput": true},
    {"name": "startTime",    "fieldType": "DateTime", "example": "2026-06-01T09:00:00Z", "isUserInput": true},
    {"name": "returnTime",   "fieldType": "DateTime", "example": "2026-06-01T17:00:00Z", "isUserInput": true},
    {"name": "bikeCategory", "fieldType": "String",   "example": "City Bike"},
    {"name": "dailyRate",    "fieldType": "Decimal",  "example": "0.10"}
  ]
}
```

`bikeCategory` and `dailyRate` are displayed data. They come from a read model, "Available Bikes", that isn't on the sheet yet, so they carry no flag. Record the planned read model on the screen's cell so Step 5 can group by it:

```
set_note { sheet, target: "E2", text: "Planned read model \"Available Bikes\": bikeCategory, dailyRate" }
```

### Example: view screen fields (read model as source)

```json
{
  "scope": "screen",
  "name": "Reservation Confirmed",
  "fields": [
    {"name": "reservationId", "fieldType": "String",   "example": "res-001"},
    {"name": "bikeName",      "fieldType": "String",   "example": "City Bike - Gazelle"},
    {"name": "stationName",   "fieldType": "String",   "example": "Central Park East"},
    {"name": "startTime",     "fieldType": "DateTime", "example": "2026-06-01T09:00:00Z"},
    {"name": "expiresAt",     "fieldType": "DateTime", "example": "2026-06-01T09:30:00Z"},
    {"name": "estimatedCost", "fieldType": "Decimal",  "example": "48.00", "isGenerated": true}
  ]
}
```

Every field except `estimatedCost` is displayed from the planned read model "Active Reservation"; the cell note records that. `estimatedCost` is a display-only calculation, so it is `isGenerated`. A displayed field is never marked generated just because its read model is missing.

A read model named in a cell note but not on the sheet (or not connected to this screen) is a **gap** for Step 5 (Identifying Outputs) to close. `check_model` reports a screen's fields with no stated origin as `unsourced-fields` (info) until it lands. Flag it in the completeness notes, and don't invent flows that don't exist.

Set `"isList": true` only when the field genuinely displays/collects a list of values (e.g. a repeated line-item row). Omit it otherwise. Only add the fields the screen actually shows or captures. Do not add speculative fields; enrich later with `update_element` (passing `fields` replaces the list: resend every field, and unchanged names keep their ids).

A screen that only has a name and no fields is an empty placeholder. Place the fields before moving on.

## Mandatory Screen Design

Every screen requires a design. A screen is HTML: a real HTML/CSS mockup, not a wireframe sketch. Place a screen together with its design. A bare screen is an unfinished thought.

**Step A: Find the cell.** This applies to SCREEN elements (human roles only). AUTOMATION elements follow "Placing Automations" below instead. Screens go in **that screen's own role's actor lane**, in their target column.

1. Determine the target column (same column as the event/command for a command/input screen, OR the same column as the read model for a view/output screen; one column to the right only if that shared column isn't available).
2. Row = the role's entry in the role→lane map built above. Do not re-read the sheet per screen. If this screen's role is genuinely new (wasn't in the original Role Catalog), resolve/create its lane now the same way (see above) and add it to the map before continuing.
3. Cell address = column letter + row number (column A, role lane in row 2 → `A2`).

**Step B: Place the screen with its design, in one call.**

```
place_element {
  sheet,
  cell: "E2",
  element: { "scope": "screen", "name": "<Screen Name>", "fields": [ /* per "Mandatory Field Definitions" above, set in this same call */ ] },
  design: "<div class=\"card\">...</div>"
}
```

A storyboard screen is placed at a *provisional* position. Steps 4 and 5 create its `SCREEN → COMMAND` and `READMODEL → SCREEN` flows deliberately once those elements exist, and may move it first (`move_element`). Placing a screen creates no flows.

Design the screen as real HTML/CSS:

- Define the model's look **once** with `set_screen_theme` (design tokens plus component classes such as `.btn`, `.card`, `.nav`). Every screen design renders with it, so each screen stays a few lines of semantic markup. `get_screen_design` returns the current `themeCss`; if it is empty, set a theme before designing the first screen.
- Write full-size markup: generous padding, readable body text. The canvas scales it down; don't shrink it yourself.
- Scripts, inline event handlers, iframes, forms, links and non-data-URI images are stripped. `<style>` blocks, inline styles, form controls (`input`, `button`, `select`, `textarea`, `label`) and data-URI images survive. The result reports what was `stripped`.
- Every design MUST include real field labels matching the actual event/command fields this screen captures or displays, and at least one primary action (submit/confirm button) for command screens.
- Check the result with `render_screen { screen }` (one screen) or `render_sheet` (the sheet as the canvas draws it).

> **CRITICAL: NEVER pass an empty `design`.** An empty design produces a blank placeholder and is always wrong. You MUST design and include actual markup in the same call that places the screen. To change a design later, use `set_screen_design`.

A screen without a design is an empty placeholder. It must not be left that way.

## Timeline Placement Rules

When placing screens on the sheet, follow these alignment rules:

| Screen type | Where it goes on the sheet |
|-------------|---------------------------|
| **Input/command screen** (triggers a command) | **The role's own actor lane, same column as the COMMAND and EVENT** it produces. The screen and command share a column: the screen sits in that role's actor lane, the command in the interaction lane, the event in the event lane. |
| **View/output screen** (displays a read model) | **The role's own actor lane, the SAME column as the READ MODEL** it displays (READMODEL in the interaction lane, screen in the actor lane: a same-column connection, not a backward one). Only bumps one column to the right if that column's interaction cell is already taken by something else. If the screen displays more than one read model, only the primary one shares its column. Every additional read model goes further left. This column is finalised in Step 5 (Identifying Outputs). During storyboarding, just document which read model each view screen will query. |

> **Do not create standalone screen columns that are disconnected from commands or read models.** Every screen must either share its column with the command it submits, or share its column with the (primary) read model it displays.

### Multi-component screens are broken apart in Step 5, not here

Storyboarding places **one plain screen per screen state**. Do not pre-split a screen into per-component copies here. Deciding how many components a view screen actually has, and breaking it apart into one highlighted copy per component, is `eventmodeling-identifying-outputs`'s job (its "Step 5a: Enumerate consumers and identify components" and "Step 5c: Break apart multi-component screens into copies"), because a component is defined by its read model and read models aren't designed until Step 5. During storyboarding, just place the single screen in the same column where its read model will end up (per the table above); document which read model it will query even before that read model exists.

### Placing Automations

When a processor or system actor reacts to events automatically (no human interaction), place an **AUTOMATION** (a `processor` element) in the sheet's **one shared automation actor lane** instead of a SCREEN. Never create, reuse, or look up a per-system-actor lane for it, and never resolve it through the human role→lane map above. A new sheet has no such lane yet: create it once (`insert_lane { kind: "actor", name: "Automations" }`, named for what it is, not for any one system actor) and reuse it for every automation on the sheet. Automations go in the same column as the COMMAND they trigger. Unlike a view screen, an automation's READMODEL is never in that same column. The automation's own column already holds the COMMAND it issues (interaction lane), so the read model that feeds it always goes one column to the left.

**Do not design automation actor lanes to mimic human ones.** A "Payment Processor" or "Inventory System" swimlane in the narrative report (Step 5 above) is a documentation grouping only. It must never be materialized as its own labeled lane on the sheet. Only human roles get a physical lane; every automation, regardless of which system actor it narratively belongs to, renders in the same shared automation lane.

A column is an automation column (not a screen column) when:
- The action is triggered by the system, not a user gesture
- No human sees or interacts with the UI at this step
- The pattern is `READMODEL → AUTOMATION → COMMAND` (the processor reads state, decides, issues a command)

Examples:
- Payment gateway webhook arrives → AUTOMATION "Authorize Payment"
- Inventory check fires after payment authorized → AUTOMATION "Reserve Inventory"
- Notification service sends an email → AUTOMATION "Send Confirmation Email"

Human roles get SCREEN elements placed in their own labeled actor lane. System actors and processors get AUTOMATION elements placed in the shared automation lane. Place both kinds during storyboarding. Do not defer automations to a later step.

### Maintain a Consistent Actor Perspective Across Columns

Tell the story from **one actor's perspective as much as possible**. When walking the timeline column by column, keep placing screens in the same role's lane as the previous screen unless the process genuinely hands the story off to a different actor (e.g. a Member requests a reservation, then the Librarian must act on that request. That's a real handoff, place the Librarian's screen next). Do not alternate actors column-to-column just because a different role's read model also changed at that point. A state change only earns its own screen when someone would actually look at it as the next beat in the story, not for symmetry between lanes.

Before placing a screen, ask "whose turn is it in the story?" If the answer is still the same actor as the last screen, keep the story there. It's fine, and expected, for a stretch of consecutive columns to belong entirely to one actor while another actor's lane sits empty. It's also fine to occasionally cut away to show what another actor sees, but that should read as a deliberate narrative beat, not a mechanical alternation.

```
Consistent:   Member screens in cols 1, 2, 3 (their whole request flow), then Librarian screens in cols 4, 5 (their whole fulfillment flow)
Checkerboard: Librarian, Member, Librarian, Member, Librarian across cols 1-5 with no real handoff driving each switch
```

### One Screen Per Column (Hard Rule)

**Never place more than one SCREEN in the same column**, even across different actor lanes. Each column represents a single moment in the timeline. Two screens in the same column means two different interactions at the same moment. That breaks the visual narrative and always signals a design error.

```
Correct: Member "Reserve Book" in col 8  |  Librarian "Confirm Checkout" in col 11
Wrong:   Member "My Loans" AND Librarian "Confirm Checkout" both in col 11
```

If a second role also needs a screen related to the same event, insert a new column immediately after (`insert_slice { name, at: N }`) and place the second screen there.

---

## Output Format

The actual output is the designed SCREEN and AUTOMATION elements placed per "Mandatory Screen Design" and "Timeline Placement Rules" above, not a document. The full storyboard-as-document template (swimlane organization, one section per screen, processor todo lists, a field traceability matrix) is kept in `references/examples.md` for reference only, as the shape of the information a complete storyboard covers.

## Quality Checklist

- [ ] **Every screen has a design**: placed with its HTML in the same call (or `set_screen_design` since), no exceptions
- [ ] **No column contains more than one SCREEN** across all actor lanes
- [ ] **Screens follow one actor's perspective across consecutive columns.** No mechanical alternation between two actors' lanes column-to-column; a different actor's screen appears only where the story genuinely hands off to them
- [ ] Every screen's design shows real field labels matching the event/command fields
- [ ] Every displayed field has a source event
- [ ] Every user action maps to a command
- [ ] Commands map to events
- [ ] Data flows make sense
- [ ] No missing data sources
- [ ] State transitions are clear
- [ ] Alternative states are shown
- [ ] Error states are shown
- [ ] **Every human role from the Role Catalog has at least one role swimlane**
- [ ] **Every human-role swimlane is labeled with the role name from the catalog**
- [ ] **Swimlanes organized by actor/system in the narrative report**
- [ ] **Every human role's swimlane is a real, distinct lane of kind `actor` on the sheet, not just a grouping in the report.** No two different human roles share the same lane
- [ ] **No system actor / processor has been given its own labeled actor lane.** Every AUTOMATION sits in the one shared automation lane, never a lane fabricated to mimic a human role's
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
