---
name: eventmodeling-elaborating-scenarios
description: "Step 7 of Event Modeling - Elaborate scenarios using Given-When-Then format. Specify behavior of commands and views. Each spec tied to exactly one command or view. Use after defining systems and boundaries. Do not use for: architectural validation (use eventmodeling-validating-event-models) or verifying field completeness across the model (use eventmodeling-checking-completeness)."
---

# Elaborating Scenarios

This step applies the shared element rules in **`eventmodeling-core-rules`** — read it once per session if you haven't already; it defines what a COMMAND/EVENT/READMODEL/SCREEN/AUTOMATION is, how each is named, and the anti-patterns to reject, so this step doesn't restate them. Tool names and conventions are in `eventmodeling-core-rules` → `../eventmodeling-core-rules/references/tools.md`.

You are in **Modeling Mode**: build the scenarios, flag what is unclear and move on.

## Examples Are the Scenario — Non-Negotiable

**A scenario without concrete example values is not a scenario.** The element names in
`given`/`when`/`then` only say *which* facts are involved; the example values say *what* happened
and *why* the outcome follows — they are the actual specification, and the only thing an
implementation can later be checked against. Every value **essential to the scenario** — the ones the rule decides on
and the ones that tie the steps together — is filled in as `fieldValues: { fieldName: value }` on the
step it belongs to (`given` events, the `when` command, `then` events, `given`/`then` read
models), and list read models carry their expected rows (or an explicit empty list). Never record a scenario, never report one done, and never mark this step
complete while an essential value is missing. Details: "Fill the values essential to the scenario"
below.

## GWT vs. Storyline — Decision Rule

A GWT scenario asserts one isolated transition (one precondition → one action → one outcome). A
storyline instead narrates one use case as an ordered sequence of **beats**, walking the *same*
element (usually a read model) through multiple states in one flow — something no single GWT can
express.

**Commands: always GWT.** A command has no state progression to narrate — it validates one
input against one state and either succeeds or is rejected. Never write a storyline for a command.

**Read models: storyline when the read model goes through a series of clear state transitions
driven by events; GWT otherwise.** Decide this **per read model**, not once for the whole pass:
does replaying this read model's actually-connected event(s) more than once produce an interesting
accumulated/changed state worth narrating? A single event type recurring with different data counts
just as much as a multi-event lifecycle — e.g. `Account Funded` ($40) then `Account Funded` ($70)
walking a balance from $40 to $110 is a genuine storyline driver. **Todo lists are a prime
candidate**: an item appears when opened and disappears when completed, which is exactly the kind
of state progression a storyline is for — as is any read model whose rows accumulate, update, or
get removed across a sequence of events (balances, counters, statuses). Yes → storyline. No →
GWT-only. Make this call before drafting any payload — a batch pass that reuses one schema for
every command *and* every read model is a sign the per-read-model judgment got skipped.

Even where a read model gets a storyline, still write GWTs for the specific scenarios the
storyline doesn't cover — validation failures, cross-context sourcing, or any transition outside
the narrated flow. Storyline and GWT are complementary, not exclusive: the storyline covers the
narrated lifecycle, GWT covers everything else about that read model.

**Example** — a customer-activation walkthrough of a Todos read model:
1. **Beat 1** — Todos read model, empty.
2. **Beat 2** — after `Customer Registered`, Todos shows one entry ("activate your account").
3. **Beat 3** — after `Customer Activated`, Todos is empty again (todo completed and removed).

A storyline threads all three beats into one walkthrough; three separate GWTs could each assert one
transition but not the lifecycle.

> **Tool availability:** this server has no storyline tool yet. Still make the per-read-model decision and write the line for it (step 0 below). Where the answer is "storyline", write the GWTs that assert each transition, and leave a `set_note` on the read model's cell naming the lifecycle that still needs its storyline.

### Storyline shape

A storyline has a title, an optional description, a layout (horizontal or vertical) and an ordered list of beats. Each beat is one state of the walkthrough and refers to an element on the same sheet; the *same* element (like the Todos read model above) may appear many times, once per state.

- **Beat order rules**: the first beat is a READMODEL; after a READMODEL comes an EVENT (or several); after an EVENT comes another EVENT or a READMODEL. Beats are only READMODEL or EVENT — no COMMAND, SCREEN or other types.
- A beat may be marked as an alternate/error branch off the previous beat.
- Example values, expected rows and an explicit empty list are set **per beat** — a list READMODEL beat carries its own expected rows (or is marked empty), so every intermediate state is asserted, not just "not empty". This is storyline-only: in a GWT scenario the same data goes on the scenario, never on the `then` step (see Step 4b).
- A note beat may sit anywhere and is skipped by the order rules — see "Notes — Last Resort Only" before adding one.

## Notes — Last Resort Only

A scenario can carry a one-line `contextNote` (shown under its title; set it with `set_scenario`) and a free-text `description`, and a storyline can hold note beats. **Notes are only for what cannot be expressed
with events and the other elements.** Before writing one, ask: could this be a `given` event, a
read model in `given`, a field example on the command, or a `then` outcome? If yes, model it that
way — a scenario that tells its story in a note instead of in events is incomplete, and anything built from it
can't check what's only in prose.

Legit uses: a condition outside the system ("the payment provider is in maintenance"), a business
assumption or regulation behind the case, why the case matters, a time gap between storyline beats
("14 days later"). Not legit: "the customer has already ordered twice" (→ `given` events), "the
cart total is 120 €" (→ field examples), "an email is sent" (→ a `then` event), restating the
title. Most scenarios need no note; keep any note to a few lines.

### Fill the values essential to the scenario — mandatory

The goal is not to fill every field, but every value **the scenario depends on**: change one of
them and the outcome changes, or the steps no longer belong together. Essential are:

- **the values the rule decides on** — the dates of an expiry rule, the wrong token, the outcome
  `Unclear` that routes a check into review, the second registration's duplicate email;
- **the ids linking the steps** — the same `customerId` in the given events, the command and the
  outcome, so it is clear they are about the same thing;
- **the asserted outcome** — the `then` event's or read model's values that the scenario expects
  (the `status: Rejected` and its reason, the row that appears in the list).

**Leave every other field empty — that is best practice, not laziness.** The empty fields are
what make it clear which data is relevant: a reader sees at a glance that this case turns on the
dates and the token, not on the name, address or phone number. Padding every step with every field
buries the values that carry the rule. The test for the result: someone reading only the filled
values can tell why the outcome follows. A
scenario whose essential values are missing is incomplete, even when a note explains the
situation: the fields are the specification, the note is not.

The values are what make the rule visible and checkable. For an expiry rule, the dates go into the
fields — `Customer Registered` with `registeredAt: 2026-10-12`, `Confirm Email` with `confirmedAt: 2026-10-13` —
not into a note saying "registered on 2026-10-12, confirms on 2026-10-13". Keep ids consistent
across the steps of one scenario (the same `customerId` in every given event, the command and the
outcome), and pick values that sit on the interesting side of the rule (one day past the limit, not
a vague "long ago").

Read the elements' field names once (`get_slice_view` shows every element of a slice with its fields) and pick the essential ones per scenario from that. `set_scenario` accepts only field names the element really has — a typo'd key is rejected, never silently dropped. If a value the rule depends on has no field on any element (e.g. the expiry rule needs a
timestamp no event carries), add that field to the element (`update_element`) — don't move the value into the note.

## Interview Phase (Optional)

**When to Interview**: Skip if the user has already specified: scenario coverage depth (happy path + validation + state violations), known edge cases to include, and stakeholders available for review. Interview when coverage goals are unclear or edge cases haven't been identified.

### Critical Questions

1. **Scenario Depth & Coverage Goals** (Impact: Determines scope—happy path only vs. comprehensive coverage)
   - Question: "How comprehensive should scenario coverage be? (A) Happy path + basic validation, (B) All command variations, (C) Comprehensive including edge cases and error paths"
   - Why it matters: Affects time investment and implementation complexity; production code needs (C), design validation might use (A) or (B)
   - Follow-up triggers: If (C) → ask "How many scenarios per command is reasonable?"; if (A) → clarify MVP vs. production distinction

2. **Known Edge Cases & Business Rules** (Impact: Ensures critical scenarios aren't missed)
   - Question: "What specific edge cases or business rules are critical to test? (e.g., 'order cancellation within 24 hours', 'payment decline recovery')"
   - Why it matters: Business rules often generate overlooked scenarios; edge cases reveal missing events
   - Follow-up triggers: For each rule → ask "What scenarios demonstrate this rule? What happens when it's violated?"

3. **Testing & Automation Strategy** (Impact: Shapes scenario detail and executable format)
   - Question: "Will these scenarios be: (A) Automated tests, (B) Manual QA reference, (C) Documentation only?"
   - Why it matters: Automated tests need precise Given/When/Then; documentation can be more narrative
   - Follow-up triggers: If (A) → ask about test framework; if (B) → ask about QA process

4. **Stakeholder Review & Validation** (Impact: Determines who validates business logic correctness)
   - Question: "Who will review and validate scenarios? (A) Product Owner, (B) QA/Tester, (C) Multiple roles, (D) Engineering only?"
   - Why it matters: Multi-role review catches business logic errors; single role may miss perspective
   - Follow-up triggers: If (A) only → ask "Will PO have time for detailed review?"; if (C) → plan review workshop

Follow **`eventmodeling-interview-protocol`** to run this interview and record its findings — label this step "**7. Scenarios** (`eventmodeling-elaborating-scenarios`)". Findings should cover: coverage goals, critical edge cases, business rules requiring scenarios, testing strategy, review/validation plan, and the key scenario specifications themselves (GWT format for critical commands and views).

Scenarios are done, but the model is not yet ready for implementation — Steps 8–11
(Completeness, Validation, Slicing, Documentation) still follow. Proceed to Step 8
(`eventmodeling-checking-completeness`).

---

## Workshop Facilitation Guide

**Context**: Scenarios are created in a collaborative workshop with multiple stakeholders. Use this approach for rapid, real-time scenario creation:

### Before the Workshop

**Participants** (required roles):
- **Product Owner/Domain Expert**: Knows business rules, priorities, edge cases
- **Developer**: Technical feasibility, implementation concerns
- **QA/Tester**: Test coverage, edge cases, error scenarios
- **Facilitator**: Keeps pace, ensures shared understanding, captures scenarios

**Setup**:
- The sheet with the previously completed steps in view: the events timeline and the system boundaries
- Timer for time-boxing per command/view

### During Workshop (Per Command/View)

**Rapid Cycle** (15-20 min per command):
1. **Happy Path First**: "What's the normal success case?" (Product Owner leads)
   - Facilitator writes scenario live
   - All roles review in real-time
   - Adjust based on feedback

2. **Validation Failures**: "What could go wrong with inputs?" (Developer/QA)
   - Common: invalid format, missing fields, invalid references
   - Quick scenarios, add to the sheet

3. **State Violations**: "What if system is in wrong state?" (Domain Expert)
   - Example: "Can't confirm if already confirmed"
   - These are business rules, often overlooked

4. **Alternative Paths**: "Are there other ways this could work?" (Product Owner)
   - Different paths through the same command
   - Different outcomes based on business logic

5. **Error Handling**: "What if external systems fail?" (Developer)
   - Payment decline, inventory unavailable
   - Recovery/retry scenarios

6. **Compensation**: "Can this be undone?" (Domain Expert)
   - Cancellation, reversal, refund flows
   - Often reveal missing events

**Review Style**:
- Each scenario read aloud by facilitator
- Quick check: "Is this right?" (Everyone nods or speaks up)
- Move forward—don't perfect, iterate later
- Target: 3-5 scenarios per command, 10-20 minutes per command

### Multi-Role Review

As scenarios are written, ensure each role checks:
- **Product Owner**: "Is this the right business behavior?"
- **Developer**: "Can we implement this? Need system state? Edge cases?"
- **QA**: "Can we test this? Is it clear enough?"
- **Facilitator**: "Do we have enough detail for coding?"

### Common Workshop Mistakes to Avoid

- **Perfectionism**: Don't spend 30 minutes on one scenario. Capture and move.
- **Missing roles**: One person can't represent all perspectives.
- **Too technical**: Use domain language, not code. Adjust in implementation.
- **Incomplete givens**: "Given an order" is too vague. Specify state.
- **Unclear events**: Every scenario must show what event is produced (or why not).

### Tips for Rapid Creation

- **Use templates**: Have Given/When/Then templates ready
- **Parallel work**: Different people write different scenarios simultaneously
- **Capture edge cases**: When someone says "What if...?" → immediately capture as scenario
- **Reference past decisions**: Point to the event timeline and boundary decisions
- **Record decisions**: Why did we choose this behavior? (Helps implementers later)

## Scenario Types

**Do not reduce scenarios to a simple good-case / bad-case pair.** For each command, ask the questions below and write a scenario for every answer that the business case supports. Which types apply and how many scenarios each type generates is always determined by the domain — not by a fixed count or a static rule.

| # | Type | Question to ask |
|---|------|----------------|
| 1 | **Happy Path** | "What is the normal success case?" |
| 2 | **Validation Failure** | "What input does the *business* refuse?" (e.g. a booking date in the past) — not schema checks like missing/malformed fields |
| 3 | **State Violation** | "What if the system is in a state that makes this command invalid?" |
| 4 | **Duplicate Action** | "What if someone performs this business action again on something already in that state?" (e.g. resolving an already-resolved todo) — never a re-sent or colliding id |
| 5 | **Alternative Path** | "Are there different valid outcomes depending on context?" |
| 6 | **External Failure** | "What if an external system or scheduler fails during this command?" |
| 7 | **Compensation** | "Can this be undone or reversed? What triggers the cleanup?" |

Work through each question with the domain in mind. If the answer is "that situation cannot occur in this business" then no scenario is needed for that type — but that judgment must come from the domain, not from a desire to write fewer scenarios.

**Scenarios specify business rules, not technical guarantees.** Every rejection must name a rule a domain expert would recognise and could argue about ("a closed day takes no new todos"). Do not write scenarios for: duplicate or colliding ids, retries/idempotency of the same request, missing or malformed required fields, type/format checks, or infrastructure failures — the implementation handles those and they say nothing about the domain. Rules the brief states explicitly (e.g. "adding on a closed day is rejected") are the core of the coverage and must all be present.

**The error message is domain language** — the reason as a user would read it ("Day is already closed"), never an id, a placeholder or a truncated example value.

## Workflow

### 0. Decide GWT vs. storyline for every READMODEL — before drafting any scenario payload

Apply the decision test from "GWT vs. Storyline — Decision Rule" above to every READMODEL on the
sheet (or in scope). Produce a visible artifact — one line per READMODEL, "yes → storyline" or
"no → GWT-only" with a one-clause reason — before writing the first scenario, not as a
checklist review after. Deciding this list up front is what stops a batch pass from silently
collapsing into one reused schema for every command and read model alike.

For each command and view, write scenarios in Given-When-Then format:

The full worked example set for every category below (command scenarios, state validation, view scenarios, list-type views, error paths, compensation) lives in `references/examples.md` — one compact example per category is shown here.

### 1. Command Scenarios (Given-When-Then)
Specify command behavior:

```
Scenario: Create order successfully
Given a customer with ID "cust-123"
And products exist with IDs ["prod-1", "prod-2"]
When the customer creates an order with items:
    | productId | quantity | unitPrice |
    | prod-1    | 2        | 50.00    |
Then the order should be created with status "Draft"
And an "Order Created" event is produced with orderId, customerId, items, total, status

Scenario: Reject order with invalid customer
Given a customer ID "invalid-cust" that doesn't exist
When the customer tries to create an order
Then the command should be rejected, reason "Customer not found"
And no event is produced
```

### 2. Command Scenarios - State Validation
Specify how prior events affect the command:

```
Scenario: Confirm order in Draft state
Given an order "order-456" in Draft state, Order Created event exists
When the customer confirms the order with payment method "card"
Then an "Order Confirmed" event is produced with orderId, paymentMethod, confirmedAt

Scenario: Reject confirming already-confirmed order
Given an order "order-456" in Confirmed state, Order Confirmed event already exists
When the customer tries to confirm the order again
Then the command should be rejected, reason "Order already confirmed"
```

### 3. View Scenarios (Given-When-Then)
Specify how read models display data:

```
Scenario: Display order after creation
Given an Order Created event with orderId, customerId, items, total
When the Order Status read model processes this event
Then the view should display Order ID, Status "Draft", Total, Items, Created

Scenario: Accumulate payment information
Given Order Confirmed event processed (status=Confirmed)
When a Payment Authorized event arrives with paymentId, authCode
Then the view should accumulate Payment ID, Auth Code, Payment Status "Authorized"
```

### 3b. List-type Read Model Scenarios
Specify expected rows and empty-list intent when the THEN read model is a list (the whole read model is a set of rows):

```
Scenario: Products list shows all created products
Given a Product Created event with name "Shoes", index "0"
When the Product List view processes this event
Then the list should contain: | name | index | ... | Shoes | 0 | ... |

Scenario: Products list is empty after last item is deleted
Given a Product Deleted event for the last remaining item
When the Product List view processes this event
Then the list should be empty
```

### 4. Error Path Scenarios
Specify how the system handles failures:

```
Scenario: Handle declined payment
Given an order "order-001" in Confirmed state
When the payment gateway declines the card
Then a Payment Failed event is produced with orderId, reason, timestamp

Scenario: Allow retry after payment failure
Given a Payment Failed event exists, order status is still "Confirmed"
When customer retries payment
Then the new Authorize Payment command is accepted
```

### 5. Compensation Scenarios
Specify rollback/cancellation flows:

```
Scenario: Cancel order in Draft state
Given an order "order-555" in Draft state, only Order Created event exists
When customer cancels the order with reason "Changed mind"
Then an Order Cancelled event is produced with orderId, reason, cancelledAt

Scenario: Trigger compensation on cancellation
Given an order in Confirmed state, Payment Authorized event exists
When Order Cancelled event is produced
Then a Refund Payment command should be automatically triggered
And Refund Initiated event should follow
```

## Output Format

There is no document output for this step — the deliverable is the scenarios placed directly in the specification-lane cells of the sheet (see "Putting Scenarios on the Sheet" below), using the same command/view/list/error/compensation scenario shapes already illustrated in the Workflow section above (§1–5). Group scenarios under their command or view exactly as shown there; there's no separate presentation format to produce first.

## Putting Scenarios on the Sheet

After designing all scenarios, put them on the sheet. Do this for every command and view that has scenarios.

A scenario is an element in the **specification lane**, in a cell of the slice it specifies. It is the one element kind allowed to stack: a specification cell holds as many scenarios as the slice needs. Each scenario is tied to exactly one command or one view. Scenario names must be unique on the sheet — `set_scenario` finds a scenario by its address or its name.

### Step 1 — Find the slice, the column and the specification cell

`get_sheet_view` shows the grid: slices with their column letters, lanes with their kinds, and every element by cell address. Find the COMMAND or READMODEL you are specifying; its column letter plus the specification lane's row number is the scenario's cell (e.g. the command sits in `C2`, the specification lane is row 5, so the cell is `C5`). If the sheet has no specification lane yet, add one (`insert_lane` with kind `specification`). `render_sheet` gives a visual read when the layout isn't obvious.

**Scenarios go on the original element, never on a linked copy.** If the COMMAND/READMODEL you are specifying is a linked copy (`linkedCopy: true`, with `originalStickyId`), resolve it to the original (the element that is not a copy) and put the scenario in the original's column. A copy's column gets no scenario of its own; the original's column holds the one spec for that fact. If the original sits on another sheet, put the scenario there. Never delete the original.

To see what a slice already has, `get_slice_view` returns its elements with fields, its flows, and the scenarios already there with full bodies — read it before adding more.

### Step 2 — Know the valid step elements

A given/when/then step refers to the element where it lives, **on any sheet**. A command's `given` often needs an event another journey records (a password reset, an account registered elsewhere): name that event on its own sheet. Collect the events, the command and the read models you need from `get_sheet_view` (names and addresses; `get_model_view` lists the other sheets), and their field names from `get_slice_view`. A step is referred to by element address (`B3.1` on the scenario's sheet, `Reset Password!B3.1` on another sheet) or by element name. A name is looked up on the scenario's sheet first, then across the model. A name resolves to the original of a linked copy, an address of a copy points the step at its original, and an ambiguous name (original and copy both on the sheet, or two elements sharing a name) is rejected with the matching addresses — use the address then.

**Never place a linked copy just so a step can name it.** A copy is there to be wired to its neighbors on this sheet (`eventmodeling-core-rules`, Linked Copies); a step needs no copy. In the scenario views each step carries a `ref` saying where its element sits (`B3.1`, or `Reset Password!B3.1`), and the scenario card on the canvas shows the sheet name on a step from another sheet.

### Step 3 — Resolve step references

For each step (given/when/then entry) decide the element it refers to. An unresolved or ambiguous reference makes `set_scenario` fail as a whole, with nothing changed — resolve it (by address) and send the call again. A step whose true source event lives on a different sheet names that event there (`Sheet!B3.1`); don't fabricate a same-sheet event that isn't the real source, and don't leave the `given` empty for it. In a `build_sheet_from_spec` spec, a scenario's steps are addresses within the spec, or `Sheet!B3.1` for an element already on another sheet.

### Step 4 — Place the scenario and fill it

```
place_element { cell: "<specification cell>", element: { scope: "scenarios", name: "Reject a reservation for a date in the past" } }
set_scenario  { scenario: "<that name or the returned address>", given: [...], when: [...], then: [...], description?: "..." }
```

A step is an element reference (address or name) or `{ element, fieldValues: { fieldName: value } }` to carry the example values. Provided sections replace that section wholesale; omitted sections are untouched. Place a scenario and fill it in one go by sending both calls in one `apply_operations` batch (by the scenario's unique name), and group all the scenarios of one command or view the same way. `position` orders the card among its slice's scenarios.

**Rules for the steps (the method — keep to them even where a call would accept more):**
- `given`: prior EVENTs. Where the case depends on what a read model was showing, that read model's state goes in `given` too (the context).
- `when`: at most one COMMAND; empty when `then` is a READMODEL.
- `then`: EVENTs only OR exactly one READMODEL — never mixed.
- A step names its element where it lives, on whichever sheet: never a linked copy placed for the step.

`set_scenario` itself only checks that each entry is an element of an allowed kind for its section; it does not enforce these tighter rules, so keep to them yourself. `check_model` reports `scenario-when-shape` where a `when` holds more than one entry, an event, or anything beside a read-model `then`, and `scenario-then-shape` where a `then` mixes a read model with other entries, names several read models, puts an error beside other entries, or holds a command.

### Step 4b — List-type read model scenarios

When `then` is a READMODEL whose whole result is a set of rows (a list-shaped read model — every todo-list read model, `Product List`, `Invoice List`), the scenario needs two things **at the scenario level, never on the `then` step**:

- **the expected rows** — one object per expected row, keyed by the read model's field names. Without this the scenario has no expected output and cannot be verified.
- **an explicit empty-list flag** when the expected result is an empty list (e.g. after a delete). An empty set of rows alone is ambiguous ("no data provided yet" vs. "intentionally empty") and must not be left standing for the empty case.

In `set_scenario`, pass the expected rows as `rows: [{ <field name>: value, … }, …]`, one object per row, keyed by the read model's field names, or `expectEmptyList: true` for the empty case. Never both. Both are accepted only when `then` is exactly that one read model and it is marked as a list (`isList: true` on the read model; if it isn't yet, `update_element` it first). Rows with a key that is not one of its fields are refused. `rows: []` or `expectEmptyList: false` clears a stored outcome. `get_slice_view` and `get_cell_view` return them with the scenario.

```
set_scenario {
  scenario: "Products list shows all created products",
  given: [{ element: "Product Created", fieldValues: { name: "Shoes", index: "0" } }],
  then: ["Product List"],
  rows: [{ name: "Shoes", index: "0" }]
}
set_scenario {
  scenario: "Products list is empty after the last item is deleted",
  given: ["Product Deleted"],
  then: ["Product List"],
  expectEmptyList: true
}
```

### Step 4c — Error/rejection scenarios

When a scenario represents a command being **rejected** (validation failure, state violation, or business rule rejection), the outcome is an **error entry** in `then`:

```
set_scenario {
  scenario: "Reject a reservation for a date in the past",
  given: [],
  when: [{ element: "Reservation Command", fieldValues: { reservationId: "r-123", date: "2026-10-12" } }],
  then: [{ errorType: "ReservationDateInPast", errorMessage: "reservation date is in the past" }]
}
```

- `errorMessage` is the short human-readable reason for the rejection (e.g. `"reservation date is in the past"`, `"reservation already confirmed"`). Never leave it empty. `errorType` is a short name for the same rule.
- When the scenario is a rejection, `then` is that error entry alone — no events are produced on rejection. Don't add outcomes next to it.
- `when` contains the COMMAND being rejected.
- `given` contains any prerequisite EVENTs needed to establish the wrong state (may be empty for an input the business refuses).
- An empty-list expectation is only for list READMODELs; never combine it with a rejection.

State-violation rejection (given events establish the wrong state):

```
set_scenario {
  scenario: "Reject confirming an already-confirmed reservation",
  given: [{ element: "Reservation Confirmed", fieldValues: { reservationId: "r-123", confirmedAt: "2026-10-12" } }],
  when: [{ element: "Confirm Reservation", fieldValues: { reservationId: "r-123" } }],
  then: [{ errorType: "ReservationAlreadyConfirmed", errorMessage: "reservation already confirmed" }]
}
```

### Step 5 — Verify and report back

Run `check_model`. For scenarios it reports `slice-without-scenario` (a slice that changes state but has none), `no-error-scenario` (only happy paths — consider a rejection), `scenario-missing-ref` (a step pointing at an element that no longer exists), `scenario-on-linked-copy` (a scenario that names a linked copy as its command or read model, or sits in a column whose only command or read model is a copy), `linked-copy-unwired` (a copy wired to nothing, typically placed only so a step could name it: remove it, and its slice if that leaves it empty; the step already names the original), and the step-shape codes above. It does not check that every read model has a view scenario, that the steps carry example values, or that each scenario sits in its subject's own slice: check those by hand from the Quality Checklist. Fix what it flags, then tell the user:
- How many scenarios were put on the sheet (per command/view)
- Anything you could not express or reference, and why
- The cell address of each specification cell you filled

---

## Quality Checklist

**Per command — all 7 scenario types considered (skip only with explicit reason):**
- [ ] **Happy Path** — normal success case written
- [ ] **Validation Failure** — invalid/missing input rejected (or marked N/A with reason)
- [ ] **State Violation** — command issued in wrong system state (or marked N/A with reason)
- [ ] **Duplicate Action** — second issuance after success handled (or marked N/A with reason)
- [ ] **Alternative Path** — different valid outcomes captured (or marked N/A with reason)
- [ ] **External Failure** — external system failure handled (or marked N/A with reason)
- [ ] **Compensation** — reversal/undo flow written (or marked N/A with reason)

**No command has only 2 scenarios unless all other types were reviewed and found inapplicable.**

**Per read model — this is a separate, equally mandatory pass, not an afterthought of the command pass above:**
- [ ] **Essential example values filled (blocking — no exceptions)** — every value the rule decides on, the ids linking the steps and the asserted outcome carry a concrete example on their step; every other field left empty, so it is clear which data is relevant; a scenario whose values only appear in its description is not done
- [ ] **Notes carry nothing an element could** — every scenario `contextNote`, description and note beat holds only background no event, command field or read model can express; anything else is moved into `given`/`when`/`then` and the note removed
- [ ] **Every READMODEL on the sheet has at least one view scenario** — GWT (`given`: source EVENTs, `when`: empty, `then`: the READMODEL) or a storyline. A model with dozens of command scenarios and 0 read-model scenarios is not a complete Step 7 — it's easy to walk away thinking coverage is thorough because the command side looks exhaustive, so check the read-model side explicitly before reporting this step done. (`check_model` does not check this: compare the read models on the sheet with the view scenarios.)
- [ ] **Population scenario** — the view shows correct data after its source event(s)
- [ ] **Removal/update scenario, where applicable** — a row disappears or changes (an empty list, for list-type views) after an event that supersedes it (expiry, return, archival, withdrawal, status change, etc.). A later event connecting back to an earlier-placed todo-list read model is normal for that accumulator shape (the read model already feeds an AUTOMATION), so add the connection if it's missing rather than assuming the scenario is impossible. If the read model has no automation to feed, a backward connection is not allowed — by Step 5 this should already have been modeled the forward way (see `eventmodeling-identifying-outputs`: a new read model + screen copy in the later event's column, never a link back). If it wasn't, write the scenario against that forward-placed copy's original. When the superseding event lives on a different sheet, name it there in the `given` (`Sheet!B3.1`).
- [ ] **GWT vs. storyline decided per read model, not applied uniformly** — apply the decision test from "GWT vs. Storyline — Decision Rule" above to each read model individually; some may qualify for a storyline while the rest of the same model are correctly GWT-only. Don't default to one format for every read model just because it worked for the first one.
- [ ] **No redundancy or contradiction between a read model's GWT scenarios and its storyline** — if both exist for the same read model, read the storyline's beats before finalizing the GWTs. A GWT that asserts the same state a beat already shows is redundant (delete it); a GWT written without tracing the same causal sequence the storyline encodes can end up asserting something the storyline's beats actually contradict (e.g. claiming two entities coexist in a view when the storyline correctly shows one superseding the other) — delete or fix it, never leave a contradiction on the sheet.
- [ ] **Cross-sheet sources named where they live** — if a step's true source event (a command's prior state, a read model's source) lives on a different sheet, the step names it on that sheet (`Sheet!B3.1`), rather than omitting the scenario, leaving `given` empty, fabricating a same-sheet event that isn't the real source, or placing a linked copy only so the step can name it (`check_model` reports such a copy as `linked-copy-unwired`)

**Format and placement:**
- [ ] State preconditions are explicit in Given (not just "Given an order")
- [ ] Actions are clear in When
- [ ] Outcomes are verifiable in Then (event produced or rejection with reason)
- [ ] Every scenario sits in the specification-lane cell of its subject's original column (never a linked copy's)
- [ ] **Workshop facilitation approach documented** (when the scenarios were made in a session with the human)
- [ ] **All stakeholder roles (PO, Dev, QA, Domain Expert) perspectives captured**
- [ ] **Edge cases suggested by participants captured**
- [ ] **Why behind business rules documented (not just the what)**

## Gherkin Best Practices

```
Good:
Given an order in "Draft" state
When the customer confirms the order
Then the status changes to "Confirmed"

Bad:
Given order
When stuff happens
Then it works

Good:
And an Order Confirmed event is produced with:
    | field | value |
    | orderId | order-123 |

Bad:
And an event is produced

Good:
Then the command is rejected
And the error is "Customer not found"

Bad:
Then there's an error
```

## Key Principles

1. **One Scenario = One Test**: Each scenario is testable
2. **Explicit Preconditions**: State is clear in "Given"
3. **Clear Actions**: "When" describes user/processor action
4. **Verifiable Outcomes**: "Then" checks results
5. **Event-Centric**: Every scenario produces or updates events
6. **Business Language**: Use domain terms, not technical jargon
