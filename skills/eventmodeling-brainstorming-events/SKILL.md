---
name: eventmodeling-brainstorming-events
description: "Step 1 of Event Modeling - Brainstorm all domain events from requirements. Extract every state-changing event the system could have. Use when starting event modeling from requirements or a new domain. Do not use for: arranging events in sequence (use eventmodeling-plotting-events), designing commands or read models (use eventmodeling-designing-event-models), or when a complete event list already exists."
---

# Brainstorming Events

This step applies the shared element rules in **`eventmodeling-core-rules`** — read it once per session if you haven't already; it defines what a COMMAND/EVENT/READMODEL/SCREEN/AUTOMATION is, how each is named, and the anti-patterns to reject, so this step doesn't restate them. Its `../eventmodeling-core-rules/references/tools.md` maps the vocabulary below to this server's tools; consult it only if you need to look up a specific tool or field this file doesn't cover.

Every tool call carries `modelId` (from `list_models`). `sheet` may be omitted only when the model has exactly one sheet. Element kinds are written as `event`, `command`, `readmodel`, `screen`, `processor` (an AUTOMATION) and `external-event`. Positions are cell addresses (`B2`: column letter, row number), never coordinates.

## Interview Phase (Optional)

**When to Interview**: Skip if the user has provided detailed, well-documented requirements (written user stories, feature specs, business rules) and named who understands the domain. Interview when requirements are vague, incomplete, or domain expertise is uncertain.

### Critical Questions

1. **Requirements Completeness** (Impact: Determines if brainstorm is likely to be exhaustive)
   - Question: "How complete are your requirements? Do you have: (A) Written user stories/specs, (B) Documented business rules, (C) Rough list, (D) Just verbal descriptions?"
   - Why it matters: Incomplete requirements cause missed events; complete requirements enable comprehensive brainstorm
   - Follow-up triggers: If (C) or (D) → probe for missing scenarios; if rules aren't documented → ask team to state them explicitly

2. **Domain Expertise & Familiarity** (Impact: Shapes who should participate and what guidance is needed)
   - Question: "Who understands this domain best? (A) Product/Domain expert leading brainstorm, (B) Engineering team figuring it out, (C) Mix of roles"
   - Why it matters: Domain expert participation dramatically improves event completeness; solo engineering leads to gaps
   - Follow-up triggers: If (B) → recommend inviting domain expert; if (C) → ask how decisions will be made

3. **Known Complexity Areas** (Impact: Determines where to focus brainstorming effort and depth)
   - Question: "Are there specific areas known to be complex or error-prone? (e.g., payment processing, state transitions, business rules)"
   - Why it matters: Complex areas often have hidden events; identifying them upfront ensures they're covered
   - Follow-up triggers: For each complex area → ask "What are the edge cases? What can go wrong?"

4. **Explicit Business Rules & Constraints** (Impact: Ensures no implicit assumptions; may reveal missing events)
   - Question: "What are critical business rules that govern this domain? (e.g., 'orders can only be cancelled within 24 hours', 'payments must be authorized before confirmation')"
   - Why it matters: Business rules often generate specific events; documenting them prevents overlooking state changes
   - Follow-up triggers: For each rule → ask "When this rule is violated, what event signals that?"

Follow **`eventmodeling-interview-protocol`** to run this interview and record its findings — label this step "**2. Brainstormed Events** (`eventmodeling-brainstorming-events`)". Findings should cover: requirements completeness, domain expertise available, the Role Catalog, entities/event timelines identified, business rules & constraints, and brainstorming focus areas — this feeds directly into plotting and storyboarding.

---

## Model Context

Before brainstorming, check for EVENT elements already on the model to avoid duplicating events from a previous session — names (plus field names) are all a dedupe needs:

```
get_model_view   → every sheet, its slices, counts
get_sheet_view   → one sheet: element names and types per cell
get_slice_view   → one slice with its elements' fields (only when you need the fields)
```

If events already exist, treat them as the starting list and focus on discovering what might be missing. Also check for existing sheets (timelines) so you can reuse them — `get_model_view` lists them with their names.

If the starting point is a document or an image rather than a conversation, use `eventmodeling-modeling-from-documents`; to only assess a model that already exists, use `eventmodeling-analyzing-existing-models`.

## Sheet Discovery (Mandatory Before Placing Any Event)

Brainstorming is where timelines are discovered. **Every event must be placed into its dedicated sheet — never into a generic or unnamed one.** Later steps (storyboarding, commands, scenarios) operate on one sheet at a time; the sheet structure set here drives that entire downstream focus.

### 1. Group events by workflow / bounded context

After completing the analysis, partition the full event list into groups where each group:
- Represents a single, coherent business process (e.g., Catalogue Management, Reservation, Overdue & Payments)
- Can be understood as a standalone narrative on its own
- Would be naturally owned by one team or one domain expert

If all events belong to a single flow, one sheet is correct — do not split artificially.

**A sheet tells a story — a user journey, not an operation.** Read the sheet names aloud in order: each should be a journey someone lives through from start to outcome (*Owner & Pet Care*, *Visit Booking*), never a single action or screen (*Register Owner*, *Find Owner*, *Update Owner*, *View Owner*). Creating, finding, viewing and updating the same thing is **one** story — registering an owner, looking them up and correcting their address belong in one sheet. Signs a sheet is too fine-grained:

- its name is one verb + one noun, or names one form/page;
- it holds one or two slices;
- the next sheet starts with the read model or entity this one just produced (*Owner Registered* → *Find and View an Owner*) — that is the same story continuing; merge them.

When in doubt, merge: a sheet with a dozen slices that reads as one journey is right; five sheets of two slices each is not.

**Divergent journey vs. a decision point — do not confuse the two.** A group of events sometimes contains a branch, and the branch's nature decides whether it gets its own sheet or stays inside this one:

- **Divergent journey → its own sheet.** The actor makes a different choice *before* the process even starts, and everything downstream differs as a result (e.g. "Checkout with saved card" vs. "Checkout as guest" — different screens, different commands, arguably a different Role Catalog entry). Group these as separate workflows in Step 1, not as one group with a fork in it. If the choice itself is made on another sheet (one slice where the actor picks a path), connect that slice to each journey's sheet with `connect_sheets`; otherwise draw no connector (see `eventmodeling-core-rules`, Connecting Sheets).
- **Decision point → stays in this one sheet.** A single trigger — one command's outcome, or one automation's rule — resolves to one of several mutually exclusive results, and the rest of the process is otherwise the same story (e.g. `Payment Authorized` succeeds or fails, `Order Confirmed` vs. `Order Cancelled` after the same confirm action). Keep this as one group; `eventmodeling-plotting-events`'s "Identify Alternative Paths" step is exactly where this branch gets shown, as sibling paths within the same sheet — it never means a second sheet.

If unsure which one you're looking at, ask: "does this branch start a genuinely different story, or does it just decide how *this* story ends?" A different story is a new sheet; a different ending is a branch inside this one.

### 2. Create one sheet per group

For each group, create a sheet **before placing any events**. Reuse an existing sheet if one already matches the workflow name.

```
create_sheet { name: "Reservation & Lending" }
```

The name is the sheet's title (use the workflow / bounded-context name). A new sheet is empty — it has no columns and no lanes. Give it its event lane before placing anything:

```
insert_lane { sheet: "Reservation & Lending", kind: "swimlane", name: "Reservation & Lending" }
```

This first `swimlane` lane holds all of this bounded context's own domain events; name it for the context. Lanes read top to bottom in the order actor, interaction, swimlane(s), specification. The later steps add the actor and interaction lanes above this one and the specification lane below it (`insert_lane` takes `at`; `move_lane` repositions).

**Stack sheets so they do not overlap, and in general place new sheets close to existing ones they relate to.**
Omit any placement on `create_sheet`: it goes under the existing sheets, so creating the sheets in story order lays the model out top to bottom, and sheets never overlap. Prefer placing the new sheet directly below the existing sheet it is most closely related to (e.g. the same bounded context or an adjacent workflow), rather than mechanically appending below the last one — this keeps related sheets visually near each other on the canvas. Pass `below` / `rightOf` (another sheet's name) to `create_sheet` to place it next to a related sheet, or reposition an existing sheet with (see `eventmodeling-core-rules`, Arranging Sheets):

```
move_sheet { sheet: "Reservation & Lending", below: "Catalogue Management" }
```

Everything inside a sheet is placed by cell, never by position. Record the mapping: `workflow name → sheet name`. Every subsequent event placement will reference this sheet.

### 3. Place each event into its sheet

When placing an EVENT, always target a cell on the matching sheet from step 2. No event may be placed without a sheet and a cell.

## Event Fields (Mandatory)

When placing EVENT elements on the model, you **must** include the event's key fields in `fields`. An event without fields is an opaque label — it cannot be used to validate completeness, write scenarios, or generate code.

A field is exactly `{name, fieldType, isList?, isGenerated?, isOptional?, isUserInput?, isPii?, example?}`. Any other key is refused, and the call changes nothing. `fieldType` is one of `UUID`, `Boolean`, `Double`, `Decimal`, `Date`, `DateTime`, `Long`, `Int`, `String`, `Custom` (a `Custom` field nests `subfields`, and its examples go on the subfields). `example` is one illustrative value, always a string.

Fields use this structure inside the element:

```json
{
  "scope": "event",
  "name": "Order Placed",
  "fields": [
    {"name": "orderId",    "fieldType": "String",   "example": "order-abc123"},
    {"name": "customerId", "fieldType": "String",   "example": "cust-456"},
    {"name": "total",      "fieldType": "Decimal",  "example": "149.99"},
    {"name": "status",     "fieldType": "String",   "example": "Draft"},
    {"name": "placedAt",   "fieldType": "DateTime", "example": "2026-05-29T10:00:00Z"}
  ]
}
```

**Rules for fields:**
- Start with **essential fields only** — the identity key(s) plus the one or two facts that make this event meaningful. Do not try to enumerate every field a consumer might eventually want; that enrichment happens later via `eventmodeling-adding-attributes`.
- Use domain names, not technical names (`memberId` not `userId`, `dueDate` not `due_at`)
- Do **not** include computed or derived values — those belong in read models
- If an event has no meaningful payload beyond its identity (e.g., a simple state transition), it is fine to have no fields or just the identity key
- Do not pad events with fields just to reach a count — only add what the business needs
- Set `isList: true` only when the field is genuinely a list of values; omit it otherwise. Omit it when unsure.
- At this stage an event has no producing COMMAND yet, so its fields can't be traced upstream. `eventmodeling-identifying-inputs` closes every event field's lineage when the command lands, per `eventmodeling-information-completeness`. Keep the fields to what the command will carry or what the system creates when it records the fact.

## Cell Placement

Brainstorming events has two modes. Choose based on whether the sheet (timeline) already exists.

### Mode A — Brainstorming inside a sheet (sheet exists)

When a sheet is available, place each event directly into it. **Include `cell` in `place_element`** — an element only exists in a cell. A column exists only as part of a slice, so adding a column means inserting a slice.

**Step A — Ensure enough columns exist** (append at end of the sheet). Each event needs its own column, so insert one slice per event. `insert_slice` appends one column by default (`at` inserts at a 1-based column position; `cols` widens it). Name the slice after the event it will hold; it is only a container for the column — `eventmodeling-slicing-event-models` later regroups columns into the real slices (`span_slice`, `split_slice`, `update_slice`).

```
insert_slice { sheet: "Reservation & Lending", name: "Book Reserved" }
```

**Step B — Fetch the sheet to find the swimlane row** (only needed once per sheet):

```
get_sheet_view { sheet: "Reservation & Lending" }   → take the row whose laneKind is "swimlane"
```

**Step C — Compute, for the i-th event:** `cell = <column letter of the i-th column><swimlane row number>`, e.g. `A1`.

**Step D — Place the event with `cell`.**

```
place_element {
  sheet: "Reservation & Lending",
  cell: "A1",
  element: {
    "scope": "event",
    "name": "Book Reserved",
    "fields": [
      {"name": "reservationId", "fieldType": "String",   "example": "res-789"},
      {"name": "copyId",        "fieldType": "String",   "example": "copy-42"},
      {"name": "memberId",      "fieldType": "String",   "example": "mbr-101"},
      {"name": "expiresAt",     "fieldType": "DateTime", "example": "2026-06-01T00:00:00Z"},
      {"name": "reservedAt",    "fieldType": "DateTime", "example": "2026-05-29T10:00:00Z"}
    ]
  }
}
```

For a long, settled list on an empty sheet, the `insert_slice` / `place_element` pairs may go into one `apply_operations` call (atomic: all or nothing; columns append in order, so the nth event lands in the nth column). Otherwise work one event at a time.

> **`place_element` with `cell` is the only placement step needed.** Ids and geometry are generated — never pass them. A cell holds at most one element, so a second event cannot go into an occupied cell (`CELL_OCCUPIED`). Addresses are positional: after inserting or moving a column or lane, re-read the sheet before using one.

> **One EVENT per column (hard rule).** A column is a single moment in the timeline — never place two EVENT elements in the same column, even when they sit in different swimlane lanes (e.g. two systems under Conway's Law). Step A above always appends a fresh column per event, which already satisfies this; if you ever reuse an existing column instead of appending, first confirm no swimlane lane in that column already holds an EVENT (`check_model` reports a column with two as `column-multiple-events`). **Don't resolve this by checkerboarding across swimlanes** — see "Swimlane Rules (Mandatory)" below: this sheet's own swimlane carries the continuous story; the other swimlane only ever gets an isolated single-column handover, then the story resumes back in this swimlane.

### Mode B — Free-form brainstorming (no sheet yet)

When sheets have not been created yet, an element cannot exist — every element lives in a cell. Keep the candidate events as a list in your reply. This is valid during open discovery.

**After free-form brainstorming completes**, all events MUST be assigned to a named sheet before Step 2 (Plotting) can begin:
- Group events into workflows / bounded contexts (see "Sheet Discovery" above)
- Create one sheet per group
- Place each event into its sheet and cell, as in Mode A

An event that is never placed in a cell is not on the model and cannot be sequenced, storyboarded, or used in scenarios.

## Swimlane Rules (Mandatory)

**Use swimlanes sparingly — a swimlane exists for exactly one purpose: marking where integration with another system happens. Nothing else justifies one.** Not a different actor, not a different role, not visual grouping, not "an explicit business rule" in the abstract. Every sheet gets, and in the common case keeps, a single swimlane lane (inserted before the first event is placed) holding all of this bounded context's own domain events. Before adding a lane, check whether an existing lane already covers the element's type. If yes, place the element in that lane.

**The only valid reason to create a second `swimlane`-type lane: another system's own events cross into this sheet as integration triggers for a translation automation** (see `eventmodeling-designing-automation-chains`, Step 4b). Label it for that system and place its trigger events there — never fold them into this sheet's own event swimlane (they are not this bounded context's domain facts) and never treat them as an informal "signal" with no EVENT element at all. Use `external-event` elements for a system you only integrate with (a webhook or API), or `event` elements when it is another team's own timeline. An external EVENT may only ever open the *translation* automation's todo list — never the todo list of the automation that does the actual domain work; that automation is triggered solely by the internal event the translation automation produces, and only exists at all when reacting to that internal event genuinely requires a new decision (Step 4b covers the full translation chain, and when a worker automation belongs after it).

**Never** add a swimlane for any other reason — not a new actor, not a new role, not visual grouping. Human roles get their own **actor** lane during Step 3 (Storyboarding) — a different lane type entirely — never a new swimlane here.

**When two swimlanes exist side by side, tell a continuous story from this sheet's own swimlane as much as possible — don't checkerboard between them.** Having a second swimlane for an external system does not mean every other column should alternate between "our event" and "their event." Keep a run of consecutive columns in this sheet's own swimlane while the domain process is unfolding, and only place an event in the external swimlane where a real cross-system trigger occurs (the external system reacting to, or producing, an event that actually drives the next step of this sheet's process). If two adjacent columns each have an event in a different swimlane but neither one triggers or is triggered by the other, that's a sign the events were placed for visual symmetry rather than because the story actually crossed system boundaries there — revisit the plotting instead.

**An event in the other swimlane is a handover, not a relocation.** It marks a single moment where control passes to the other system — it does not mean the narrative now belongs to that swimlane for a run of columns. Immediately after the handover column, the story resumes in *this* sheet's own swimlane (the next column back in this swimlane), not in the other one. The other swimlane should typically show isolated, single-column events at each genuine handover point, never a multi-column stretch of its own — a long run of consecutive events in the external swimlane is a sign the events belong in a sheet of their own, not this one's.

---

## Workshop Facilitation Guide

**Setting**: This is a collaborative brainstorming workshop — the facilitator guides participants to envision the system and extract events rapidly. See `references/facilitating-event-modeling-workshops.md`'s "Step 1: Brainstorming Events" section for the full facilitation flow (goals framing, free brainstorm, gentle filtering, example dialogue) and general facilitation techniques (handling personalities, pacing, disagreement) that apply throughout.

**The core move, in brief**: ask participants for any event they can think of, capture everything without filtering first, then gently introduce the state-changing test ("did this actually change something?") to separate real events from actions/notifications/internal checks — see the reference for the full example dialogue and phrasing tips. For the questions to put to the people who know the business, and how to word them, see `eventmodeling-workshop-questions`.

**Hotspots**: a "hotspot" (a pain point, open question or disagreement raised during the brainstorm) is a feedback note, not an event. Never place it as an element. If a human has to answer or decide it, leave a `create_pin` on the cell of the event it concerns, worded as the question; if it is context that needs no answer, record it with `set_note` on that cell.

## Workflow

When given domain requirements, perform the following analysis:

### 1. Identify User Roles & Actors (MANDATORY)

Before brainstorming events, define **who** interacts with the system. Every event model needs an explicit role catalog — without it, downstream steps (storyboarding, commands, scenarios) lack clarity on who does what.

Identify all human roles and system actors:
- **Human roles**: Customer, Seller, Admin, Support Agent, Reviewer, etc.
- **System actors**: Payment Gateway, Inventory System, Notification Service, Scheduler, etc.

For each role/actor, document:
- **Name**: Use domain language (e.g., "Seller" not "User Type B")
- **Description**: What this role does in the domain (1-2 sentences)
- **Key actions**: What state changes can this role initiate?
- **Permissions boundary**: What can this role NOT do?

Present as a Role Catalog: for each human role, its name (in domain language), a one-line description, its key actions, and what it explicitly cannot do; for each system actor, its name, whether it's internal or external, what triggers it, and how it communicates (webhooks, event-driven, API). A full worked example (E-commerce domain) is in `references/examples.md`. On the sheet, each human role becomes an `actor` lane in Step 3, with its description, key actions and limits as the lane's note (`set_note`); a system actor is recorded the same way, in the lane its automations sit in. Until those lanes exist, keep the catalog in your reply and in the interview record.

This catalog feeds directly into:
- **Step 3 (Storyboarding)**: One actor lane per human role (not a swimlane — see Swimlane Rules above)
- **Step 4 (Inputs)**: Every command attributed to a specific role/actor
- **Step 7 (Scenarios)**: Scenarios reference roles by name
- **Step 8 (Completeness)**: Verify every role has at least one command path

### 2. Identify Entities (Event Timelines)
Identify the main entities whose story will be told as a timeline of events:
- User/Account
- Order
- Payment
- Shipment
- etc.

For each entity, note:
- Name (use domain language, not technical terms)
- Identity key (what uniquely identifies instances: orderId, paymentId, customerId, etc.)
- What commands will affect it

### 3. Identify Business Processes
Map out critical workflows:
- What steps does a user go through?
- What are the decision points?
- Where do systems integrate?

### 4. Extract State Changes
For each process, identify what state changes occur:
- Customer places order → Order created
- Payment processed → Order confirmed
- Item shipped → Order status changed

These become your domain events.

### 5. Document Business Rules & Constraints
- What rules govern state transitions?
- What validations must pass?
- What are the invariants?

Examples:
- "Order can only be shipped if payment is confirmed"
- "Inventory must be reserved before order confirmation"
- "Customer can only cancel within 24 hours"

### 6. Create Analysis Document

Present findings in this structure in your reply (include facilitation notes for future workshops):

```markdown
## Workshop Notes

**Participants**: [List roles: PO, Dev, QA, Domain Expert]
**Duration**: [Time spent]
**Key facilitation moments**: [What helped clarify understanding?]

---

# Domain Analysis: [Domain Name]

## Role Catalog

### Human Roles
1. **[Role Name]**: [Description]
   - Key actions: [What this role can do]
   - Cannot: [Permission boundaries]

### System Actors
1. **[Actor Name]** ([internal/external]): [Description]
   - Triggers: [What events/commands it initiates]
   - Communication: [Webhooks / Event-driven / API]

## Entities (Event Timelines)
List each entity and its identity:
- **Entity**: Review (Identity: reviewId)
- **Entity**: Seller Response (Identity: responseId)
- **Entity**: Seller (Identity: sellerId)

Note: These are just the logical groupings of events — the story each entity's timeline tells.

## Business Processes
1. **Process Name**: Description
   - Actor: Who initiates?
   - Steps: 1. → 2. → 3.
   - Outcomes: What changes?

## Identified State Changes (Potential Events)
- [Stream] [Verb]: When? Why? (Use past tense: "Review Published", "Seller Response Added")

## Business Rules & Constraints
- Rule 1: Condition and consequence
- Rule 2: Constraint description

## Questions for Domain Expert
- Any gaps in understanding?
- Unclear processes?
```

## Output Format
Present analysis in a clear markdown structure that can be directly used by the eventmodeling-designing-event-models skill. The events themselves go onto their sheets as described above.

## Key Principles
- Use **domain language**, not technical terms
- Focus on **what** happens, not **how** it's implemented
- Identify **state changes** as events, not actions (gently!)
- Document **constraints** and **rules**
- Be **specific** with examples from the requirements
- **Collaborative Process**: This is a group brainstorm, not a solo analysis
- **Rapid Iteration**: Capture quickly, refine later
- **Gentle Filtering**: Introduce "state-changing events" concept conversationally, not as rigid rule
- **Event Sourcing Mindset**: Think in terms of immutable events grouped by entity, not upfront attribute lists
- **Defer Detail**: Don't list all entity attributes upfront — commands, preconditions, and read models get worked out in the design step

## Best Practices for Requirements Analysis

### 1. Be Specific with Requirements
Provide concrete examples and clear scope:
- "Handle orders"
- "Orders have items, pricing, delivery address, and can be cancelled within 24 hours"

### 2. Use Domain Language
Use terms your business understands, not technical jargon:
- "obj1 references obj2"
- "Customer places Order with Products"

### 3. Document Constraints Explicitly
Make implicit rules explicit:
- "Process payments"
- "Authorize payment before marking order confirmed; refund if shipment fails"

### 4. Verify Role Catalog Completeness
Cross-check that the Role Catalog (from Step 1) covers all actors referenced in events and processes:
- "Orders can be created" (by whom?)
- "Customers can create orders; sellers can confirm stock; system can cancel if payment fails"

### 5. Cover Edge Cases
Include error and boundary conditions:
- "What happens if payment is declined?"
- "Can an order be modified after shipping starts?"
- "What triggers order cancellation?"

## Quality Checklist

- [ ] Every event is past tense and names a completed state change (e.g., `Order Placed`, not `Place Order`)
- [ ] **Every event element includes `fields` for all fields that are meaningful from a business perspective** — fields that consumers of this event will use
- [ ] Fields are only included when they add business value (an event with no payload fields is acceptable if the event title alone carries the meaning)
- [ ] Role Catalog lists every actor (human roles and system processors) with distinct responsibilities
- [ ] Each event can be traced back to a specific actor in the Role Catalog
- [ ] No CRUD events (`User Updated`, `Record Deleted`) — events describe business moments, not database operations
- [ ] All known error and boundary conditions have corresponding events
- [ ] No empty columns left in the sheet
- [ ] Events group into at least one recognizable business process flow
- [ ] No overlapping event semantics — two events don't mean the same thing
- [ ] Every event is placed into a **named sheet** — no event left on an untitled or default sheet
- [ ] **Multiple sheets are stacked** (a new sheet goes under the existing ones) — no two sheets overlap on the canvas
- [ ] No unnecessary swimlanes created — a swimlane exists only to mark integration with another system; never added for a new actor, a new role, or visual grouping
- [ ] **When a second (external-system) swimlane exists, events follow one continuous story from this sheet's own swimlane throughout** — the external swimlane only ever holds isolated single-column handovers, never a run of its own, and the story resumes in this sheet's swimlane immediately after each one
- [ ] `check_model` shows no `event-name-not-past-tense` warnings and no `column-multiple-events` errors (`event-disconnected` is expected until commands exist)
