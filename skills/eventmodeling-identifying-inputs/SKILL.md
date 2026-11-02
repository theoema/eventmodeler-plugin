---
name: eventmodeling-identifying-inputs
description: "Step 4 of Event Modeling - Identify Commands/Inputs from UI and Processor actions. Map user actions to commands and data. Use after storyboarding UI. Do not use for: identifying read models or outputs (use eventmodeling-identifying-outputs) or elaborating behavior specifications (use eventmodeling-elaborating-scenarios)."
---

# Identifying Inputs

This step applies the shared element rules in **`eventmodeling-core-rules`** — read it once per session if you haven't already; it defines what a COMMAND/EVENT/READMODEL/SCREEN/AUTOMATION is, how each is named, and the anti-patterns to reject, so this step doesn't restate them.

Tool names and conventions are in [tools.md](../eventmodeling-core-rules/references/tools.md).

Every tool call carries `modelId` (from `list_models`). `sheet` may be omitted only when the model has exactly one sheet. Element kinds are written as `command`, `event`, `screen` and `processor` (an AUTOMATION). Positions are cell addresses (`B2`: column letter, row number), an element inside a cell is `B2.1`, never coordinates. Addresses are positional: after inserting or moving a column or lane, re-read the sheet before using one.

## Interview Phase (Optional)

**When to Interview**: Skip if the user has already identified UI actions/commands and processor triggers. Interview when it's unclear which actions are user-initiated vs. processor-automated.

### Critical Questions

1. **Automation Level** (Impact: Determines which commands are UI-triggered vs. processor-triggered)
   - Question: "Are there actions that should be: (A) User-initiated only, (B) Processor-automated, (C) Mix of both?"
   - Why it matters: Knowing automation vs. manual separates command types
   - Follow-up triggers: If (C) → ask "Which specific user actions trigger automation? What does the processor decide on its own?"

2. **External System Triggers** (Impact: Determines if there are processor commands from webhooks/integrations)
   - Question: "Will commands be triggered by: (A) UI only, (B) External webhooks (payments, notifications, etc.), (C) Scheduled processors, (D) All of above?"
   - Why it matters: External triggers are processor commands, not UI commands
   - Follow-up triggers: If (B) or (D) → ask which external systems send webhooks and what data they include

Follow **`eventmodeling-interview-protocol`** to run this interview and record its findings — label this step "**4. Identifying Inputs** (`eventmodeling-identifying-inputs`)". Findings should cover: automation classification (user-initiated vs. processor-automated), external system triggers, and a command attribution summary (UI-issued vs. processor-issued, each with its role/source).

---

## Workflow

Given UI storyboards and event timeline, identify all inputs.

**PREREQUISITE**: The **Role Catalog** from Step 1 (eventmodeling-brainstorming-events) must exist. Every command identified below MUST be attributed to a specific role or system actor from that catalog.

> **Scope note**: This step only places the AUTOMATION actor and its COMMAND (per the Timeline Alignment Rules in `eventmodeling-orchestrating-event-modeling`). Do not attempt to design an automation's todo-list READMODEL here, and do not resolve an externally-triggered automation into a translation chain here — that is Step 4b (`eventmodeling-designing-automation-chains`), which runs immediately after this step completes, before Step 5. **Do not place anything at all** for a command whose only trigger is an EVENT already sitting in another swimlane (see "Identify Processor Triggers" below) — leave that entire case, placement included, to Step 4b.

### 1. Extract Commands from UI Actions
For each user action in storyboard, create a command attributed to a specific role: name the command, list the input fields the form captures, state the validation rules that apply, and record the event it produces. A full worked example (Order Creation → Create Order) is in `references/examples.md`.

### 2. Identify Processor Triggers
Identify automation-triggered commands: for each processor-triggered action (a webhook, a scheduled job), create a command attributed to the source system, documenting its input data, validation rules, and the event it produces. A full worked example (a payment webhook → Authorize Payment) is in `references/examples.md`.

**Stop and check before placing any of these**: is the "processor trigger" already a placed EVENT in a second (external) swimlane (an `external-event` element, or an `event` in another system's swimlane) — e.g. a `Copy Reserved (ext)` event from Step 1's brainstorming — rather than an unmodeled webhook with no element on the sheet? The two cases are handled completely differently:

- **No pre-existing EVENT** (a genuine webhook/API call, like `Authorize Payment` above): proceed as written below — place the AUTOMATION+COMMAND in one column, wire `COMMAND → EVENT` normally.
- **A pre-existing EVENT in another swimlane**: do **not** place the AUTOMATION/COMMAND in that event's column, and do **not** wire `COMMAND → EVENT` to it — that event already happened in another system; this chapter's command cannot be the thing that produces it. List the command in the Command Catalog as *deferred to Step 4b* and stop there. `eventmodeling-designing-automation-chains` places the correct multi-column translation-chain-plus-worker shape from scratch; anything placed here for this case would only have to be deleted and redone there.

### 2b. Understand the Processor "Todo List" Pattern
Processors don't directly process events—they maintain a todo list driven by events: a triggering event adds an item to the processor's todo list, and the processor continuously walks that list, checking a condition for each item and either succeeding (producing a success event, marking the item done) or failing (producing a failure event or leaving it for retry). A full worked example (Payment Authorized → Inventory Reserver's todo list) is in `references/examples.md`.

**Key insight**: Processors are reactive. They listen for events and create todo items, then execute those todos by issuing commands that produce new events.

### 2c. Document Processor Automation (Gears Symbol)
Show which commands come from automation vs. user actions: catalog every command with its role/actor attribution, separating UI-issued commands (attributed to specific human roles) from processor-issued commands (attributed to system actors/services). A full worked example is in `references/examples.md`.

**Validation**: Every command MUST have a role/actor attribution. If a command says `[ User]` instead of a specific role name, it's incomplete — go back to the Role Catalog and assign the correct role.

### 3. Document Command Specifics
For each command, define structure: its source, its typed input fields, the validation rules it enforces, the preconditions it checks against stream state, the success event it produces, and each distinct failure outcome. A full worked example (Confirm Order) is in `references/examples.md`.

### 3b. Command input is self-contained

A command must carry **everything needed to validate it**. For each validation rule and precondition, list every value the rule reads. Each value is either:
- **a field on the command**, or
- **stream state**: data from prior events of this entity (step 6).

Nothing else. Watch for **contextual information** that is easy to overlook because "the system just knows it":
- the current time or date (`now`, `today`), for deadlines, expiry, "must be in the future", opening hours, cut-off dates
- the acting user, their role, and the tenant or organization, for "only the owner may…", limits per customer
- the timezone or locale, when a rule depends on the local date ("before midnight", "on a business day")

Each one becomes a command field, `isGenerated: true` (it enters the model at the command, supplied when the command is issued, per `eventmodeling-information-completeness`). The decision then never reads a clock or session on its own, and the same command always gets the same outcome. That is also what lets a GWT scenario state it: *when `Reserve Bike` with `requestedAt = 09:00`, then rejected, because `startTime` is in the past*.

A timestamp the event records (`reservedAt`) comes from that command field, not from a clock read when the event is stored.

### 4. Create Command Catalog
List all commands the system accepts, grouped into UI-issued and processor-issued sections, each with its source, input fields, and the event it produces. A full worked example is in `references/examples.md`.

### 5. Map Data Sources
Document where each command input comes from — UI context, a form selection, a conditional field that depends on another field's value — and how it's validated. A full worked example is in `references/examples.md`.

### 6. Identify Implicit Context
Document what comes from stream state beyond the command's explicit input: the preconditions the command implicitly relies on (e.g. prior events that must already exist, a required prior status) that its validation logic checks against. A full worked example is in `references/examples.md`.

## Output Format

Instead of writing a markdown document, **place each COMMAND on the sheet** using `place_element`.

> **CRITICAL: Every COMMAND MUST include `fields`, and every field must have a known source.** A command without fields — or with fields whose origin is unknown — is an incomplete model that cannot be traced, validated, or turned into code.

### Field data lineage — mappings, `isGenerated` and `isUserInput`

Apply `eventmodeling-information-completeness` to every COMMAND field and every EVENT field as you add it. A command's upstream is the SCREEN or AUTOMATION issuing it. An event's upstream is the COMMAND producing it. User input is `isUserInput` on the screen that captures it, not on the command. Contextual values the validation reads (current time, acting user, tenant) are `isGenerated` on the command (see 3b). Only an id the system creates when recording the fact is `isGenerated` on the event. **A field whose source you can't name is a gap. Resolve it before you place the field.**

Lineage lives in the flows between elements, not on the field. A field named the same as a field on the element upstream of it needs no mapping: once the flow exists, the same name is the mapping. A renamed field is declared with `set_flow_mapping` on the flow between the two elements. `check_model` reports every required field that no inbound flow supplies as `untraceable-field`.

The stored field shape is exactly `{name, fieldType, isList?, isGenerated?, isOptional?, isUserInput?, isPii?, example?}`; any other key is refused. Set `isList: true` only when the field genuinely holds a list of values (e.g. multi-select); leave it off when unsure. Only add fields the command actually needs to do its job — do not add speculative fields; enrich later via `eventmodeling-adding-attributes`.

### Field traceability rule for events

Every EVENT field comes from its COMMAND (same name, or a `set_flow_mapping` entry for a rename) or is created by the system when the event is recorded (`isGenerated: true`). Anything else means the command, or the screen behind it, is missing that field.

```json
{
  "scope": "command",
  "name": "Reserve Bike",
  "fields": [
    {"name": "customerId",  "fieldType": "String",   "example": "cust-42",               "isGenerated": true},
    {"name": "bikeId",      "fieldType": "String",   "example": "bike-17"},
    {"name": "stationId",   "fieldType": "String",   "example": "stn-03"},
    {"name": "startTime",   "fieldType": "DateTime", "example": "2026-06-01T09:00:00Z"},
    {"name": "endTime",     "fieldType": "DateTime", "example": "2026-06-01T17:00:00Z"},
    {"name": "requestedAt", "fieldType": "DateTime", "example": "2026-06-01T08:45:00Z",  "isGenerated": true}
  ]
}
```

`customerId` (from the session) and `requestedAt` (the current time) are contextual: the rules "a customer may hold one active reservation" and "`startTime` must be after `requestedAt`" read them, so they are on the command, `isGenerated`. `bikeId`, `stationId` and `startTime` come from same-named fields on the `Reserve a Bike` screen. `endTime` is called `returnTime` on that screen, so once the screen is wired to the command:

```
set_flow_mapping { "from": "<screen address>", "to": "<command address>", "mappings": [{ "source": "returnTime", "target": "endTime" }] }
```

Event fields produced by this command:
- `reservationId` → created by the system → `isGenerated: true`
- `customerId`, `bikeId`, `stationId`, `startTime`, `endTime` → same name on `Reserve Bike` → no mapping
- `reservedAt` → mapped from `requestedAt`, the command's current time, never a clock read when the event is stored:

```
set_flow_mapping { "from": "<command address>", "to": "<event address>", "mappings": [{ "source": "requestedAt", "target": "reservedAt" }] }
```

`set_flow_mapping` replaces the flow's mappings wholesale, so send every renamed pair on that flow in one call. (A field may instead carry `source: { from, field? }` when you write it, which creates the flow and the mapping together. If you use it, don't create that flow again by hand.)

Commands go in the `interaction` lane — same column as their resulting event.

### Creating a COMMAND with fields

Commands go in the **interaction lane**, same column as the event they produce. Use the sheet's interaction lane (it also holds read models). `place_element` puts the COMMAND in that lane's cell of the event's column; never pass ids or coordinates:
```
place_element {
  "cell": "<column letter + interaction lane row>",
  "element": { "scope": "command", "name": "Reserve Bike", "fields": [ ... ] }
}
```
A cell holds one element, so an occupied cell is refused (`CELL_OCCUPIED`): widen the slice first (`span_slice`, or `insert_slice` with `cols`). If the sheet has no interaction lane yet, add one between the actor lanes and the event lane (`insert_lane` with kind `interaction`). Placing a command creates no flows; wiring comes next. Pass the `fields` in the same call when the attributes are already known; otherwise follow up with `update_element` and a `fields` patch, which replaces the whole list (send every field; unchanged names keep their ids, so mappings survive).

### Preventing backward arrows (mandatory pre-placement check)

A `SCREEN → COMMAND` connection must always go downward within the same column (actor row → interaction row). A `COMMAND → EVENT` connection must also be within the same column. Both are inherently forward because they share a column.

However, if a screen placed in Step 3 (Storyboarding) is in a column that is **different** from the event column, the connection will span columns and may go backwards. Before placing a command:

1. Confirm the screen for this command is in the **same column** as the event it produces.
2. If the screen is in an earlier column, move it to the event's column before wiring (`move_element`).
3. If the screen is in a later column, move the event's column (by inserting a new column at the correct position with `insert_slice`) or move the screen to match.

The timeline must always read left-to-right: SCREEN and COMMAND belong in the same column as the EVENT they produce. `check_model` reports any flow that runs right to left as `flow-backward`.

### Wire connections after placing each COMMAND

After `place_element` returns the COMMAND's address, create the arrows that complete the slice (`create_flow` with `from` and `to`; the type is inferred from the pair, and a pairing that isn't valid is refused with `INVALID_FLOW`):

1. **SCREEN → COMMAND** — find the SCREEN in the actor row of the same column (the issuer of a processor-triggered command is the AUTOMATION in the actor lane, and the flow is `AUTOMATION → COMMAND`).

   Read the cell: `get_cell_view` for that cell, or `get_sheet_view` for the whole sheet (an empty cell means no SCREEN placed yet). Then connect:
   ```
   create_flow { "from": "<screen address>", "to": "<command address>" }
   ```
   `create_flow` does not skip a flow that already exists between the two elements: read the cell first when in doubt. If wiring more than one COMMAND in the same pass, put every SCREEN→COMMAND and COMMAND→EVENT pair across all of them into one `apply_operations` call (see step 2 below) instead of one call per pair.

2. **COMMAND → EVENT** — find the EVENT in the swimlane row of the same column.

   **Guard**: if the occupying EVENT belongs to a different (external) swimlane than this sheet's own default swimlane (or is an `external-event`), stop — do not create this connection (the tool would accept it; the rule forbids it). An external system's own event cannot be the thing this sheet's command produces; that case belongs entirely to Step 4b's translation chain. (This should not arise if the "Identify Processor Triggers" check above was followed, but re-check here before connecting, since it's the point of no return for a wrong edge.)

   Read the cell the same way, then:
   ```
   create_flow { "from": "<command address>", "to": "<event address>" }
   ```

Skip a connection silently if the target cell is empty (the element may be placed in a later step). Log each created arrow: `→ connected SCREEN→COMMAND "Place Order"` or `→ connected COMMAND→EVENT "Place Order"→"Order Placed"`.

If wiring more than one COMMAND in this pass, send all the SCREEN→COMMAND and COMMAND→EVENT flows in one `apply_operations` call.

**After wiring, run `check_model` (`sheet`).** Its `command-multiple-issuers` finding flags any COMMAND that ended up with two issuers — the classic symptom of cross-wiring a previous-column SCREEN into a command that an AUTOMATION already drives (or vice versa). It also reports a command that produces no event (`command-without-event`), a command with no issuer (`command-without-issuer`), a screen wired to more than one command (`screen-multiple-commands`), a flow that runs right to left (`flow-backward`), fields no inbound flow supplies (`untraceable-field`) and fields with no stated origin (`unsourced-fields`). Fix each by removing the wrong flow (`remove_flow`).

After all commands are placed and wired, present the Command Catalog summary as text to the user.

---

Older versions of this skill wrote the command catalog as a markdown document rather than placing elements on the sheet — that legacy template is kept in `references/examples.md` for reference only; it is not the actual output mechanism.

## Quality Checklist

- [ ] Every UI action maps to a command
- [ ] Every processor action maps to a command
- [ ] **Every command is attributed to a specific role/actor from the Role Catalog**
- [ ] **No command uses generic "User" — must name the specific role (Customer, Seller, Admin, etc.)**
- [ ] Every command input is documented
- [ ] Every input validates against rules
- [ ] Preconditions from stream state are explicit
- [ ] Success and failure outcomes documented
- [ ] Implicit context from stream state is identified
- [ ] **Every value a validation rule reads is a command field or stream state, including contextual information (current time/date, acting user, tenant)**
- [ ] No undocumented commands exist
- [ ] Command naming is consistent and clear
- [ ] Processor triggers are explicit
- [ ] **Processor todo list pattern explained for each automation**
- [ ] **Event-to-todo triggering mechanism documented**
- [ ] **Automation marked with [AUTO] to distinguish from user actions [USER]** (on the sheet the issuing flow tells them apart: `ScreenToCommand` vs. `ProcessorToCommand`)
- [ ] **Processor failure/retry handling specified**

## Key Principles

1. **Source Clarity**: Every command comes from UI or Processor
2. **Input Completeness**: All data needed to validate is on the command, including contextual information like the current time
3. **Validation Explicit**: All rules documented
4. **State Awareness**: Preconditions from stream state clear
5. **Event Mapping**: Every input becomes event data

## Common Patterns

### User Command Pattern
```
User action on UI screen
  ↓
Captures form/selection data
  ↓
Validation checks
  ↓
Command issued
  ↓
Event created or rejection
```

### Processor Command Pattern
```
External event/webhook received
  ↓
Triggers processor logic
  ↓
Processor validates and decides
  ↓
Command issued (if valid)
  ↓
Event created or decision recorded
```

### Conditional Input Pattern
```
Command: Payment Confirm
Input:
  - paymentMethod (user selected)
  - paymentDetails (conditional on method)
    If method='card': cardNumber, CVV, expiry
    If method='transfer': bankAccount, routingNumber
```
