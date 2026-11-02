---
name: eventmodeling-orchestrating-event-modeling
description: "Orchestrates the complete event modeling workflow from requirements to a validated, sliced model. Models architecture as UI/Processor → Command → Event → Read Model. Use when modeling a domain end-to-end from requirements. Do not use for: executing a single step in isolation (invoke the named step skill directly, e.g., eventmodeling-brainstorming-events for Step 1 or eventmodeling-elaborating-scenarios for Step 7), or validating an already-completed model (use eventmodeling-validating-event-models)."
---

# Orchestrating Event Modeling

This step applies the shared element rules in **`eventmodeling-core-rules`**. Read it once per session if you haven't already: it defines what a COMMAND/EVENT/READMODEL/SCREEN/AUTOMATION is, how each is named, and the anti-patterns to reject, so this step doesn't restate them. Its `../eventmodeling-core-rules/references/tools.md` maps each concept to the tool that does it.

Coordinates the 11-step Event Modeling workflow. Each step delegates to a specialized skill. This skill holds the sequence, transition conditions, and what to carry forward between steps.

## Start Here — Read the Real Model, Not Memory

Don't take the conversation's word for what is already built. Before anything else: `list_models` to confirm which model you are in, `get_model_view` for its sheets and slices, `get_sheet_view` on any sheet you are about to touch. What comes back, not a recap from earlier, tells you which steps are done and which is the first incomplete one. If the model has no sheet yet, one is created in Step 1, when the first events are placed.

## Do not cut corners to save tokens or effort

A correct model is the entire point of this workflow. It is not optional scope that can be traded away when a step gets expensive or a placement rule turns out to be inconvenient. If satisfying a rule (a todo-list automation, a missing read model for a command screen, a proper translation-automation chain for an external trigger) requires more columns, more elements, or more tool calls than expected, that is not a signal to remove the requirement or invent an exemption. It is a signal to spend the additional calls. Budget and token cost are never a valid reason to:
- delete an automation, event, or read model that the rules call for, in order to avoid a placement conflict. Solve the placement conflict instead (see the column-layout patterns throughout this file and in `eventmodeling-identifying-outputs`);
- label a command screen's missing read model as "session-context" or "accepted debt" when the rules don't actually exempt it. Only a genuinely blank creation form is exempt; every other command screen needs a real read model, even a small identity/lookup one;
- collapse a two-chained-translation-automation requirement (external EVENT → translation automation → internal EVENT → worker automation) into a single direct connection because it is simpler.

If you catch yourself reasoning "this would need N more columns/elements, let me simplify instead", that is exactly the moment to stop and do the correct, larger version. A model with more elements that is right is a better outcome than a smaller one that skips required elements. Flag genuine scope trade-offs to the user explicitly rather than resolving them unilaterally by cutting the model.

---

## Timeline Alignment Rules

These rules govern how every element is placed on a sheet. Enforce them throughout the workflow. Placement is always by cell address; nobody authors coordinates. If something looks misaligned, the structure (the column, the lane) is wrong, not the pixels.

### Slices exist from the start

A slice is a first-class column or span of columns (see `eventmodeling-core-rules`). Because every element needs a cell, and every cell is in a column, slices come into being as soon as the sheet is built. Early on, give each new column its own slice, named for the unit of work it will become (the event `Tab Opened` sits in the slice `Open Tab`). Step 10 audits and corrects the boundaries; it does not create them.

### State-change slice (SCREEN → COMMAND → EVENT)
- COMMAND and EVENT go in **the same column**: the command produces the event.
- SCREEN (input/command screen) goes in the **actor lane of that same column**.
- **A COMMAND never stands alone.** Every COMMAND must have exactly one issuer in the actor lane of its own column: a SCREEN when a human triggers it, an AUTOMATION when a processor or external-system integration triggers it. There is no third option and no exemption (`check_model` reports `command-without-issuer`, and `command-multiple-issuers` for a second issuer). A command with an empty actor-lane cell is an unresolved gap the moment it is placed, not something to leave for a later step to notice. This applies just as much to a command that only *represents* an externally triggered integration event crossing into this sheet (Step 1/Step 6 territory) as to any other command: place an AUTOMATION for the external actor even when that actor's own decision logic is out of scope for this model. The automation documents *that* something triggers the command, not *how* it decides to.

  **Before placing that automation, check whether the trigger is already a placed EVENT in a second (external) swimlane lane** (from Step 1/brainstorming), as opposed to an unmodeled webhook/API call with no such element. These are not the same case:
  - **No pre-existing EVENT** (a plain webhook/API trigger): place the AUTOMATION+COMMAND in one column as usual; the command produces a new EVENT there. Nothing further needed in Step 4.
  - **A pre-existing EVENT in another system's swimlane lane**: do **not** place the AUTOMATION/COMMAND in that event's column, and do **not** connect `COMMAND → EVENT` to it. That event already happened in another system and cannot be "produced" by a command in this one. Attribute the command to the Role Catalog and leave a note on the column where it will land (`set_note`) saying it is *pending Step 4b*, but leave its placement and every one of its connections to Step 4b. Placing it here only produces a wrong connection that Step 4b then has to delete and redo.

  Every AUTOMATION placed this way still needs its own todo-list READMODEL, and one further rule governs *how* it is triggered: **an automation can only ever be directly triggered by an internal event, never by another system's event.** A signal arriving from a second swimlane must first be translated into an internal event by its own dedicated translation automation (external EVENT → todo-list READMODEL → translation AUTOMATION+COMMAND+*internal* EVENT) before any worker automation reacts to it. Never one automation whose todo list is opened by the external EVENT directly. There is no exemption for a "pure relay" automation either; even one triggered only by its own resulting event still gets a todo list that opens and closes within the same slice. This is designed immediately after Step 4, in **Step 4b, Design Automation Chains** (`eventmodeling-designing-automation-chains`), precisely so it is resolved before Step 5 ever has to catch it as a gap.

### State-view slice (EVENT → READ MODEL → SCREEN)
- READ MODEL goes in the **interaction lane** of a column that is **immediately after the primary source event's column**, never at the end of the sheet.
- SCREEN (view/output screen) goes in the **actor lane of the same column as the READ MODEL**. When that column's interaction cell is unavailable to the read model (e.g. it already holds a COMMAND), the **read model** gets a new column immediately **before** the screen's. The screen's own position is never moved to resolve this (`eventmodeling-identifying-outputs`'s Step 5g has the full mechanics).
- If the primary source event's column already has a COMMAND in the interaction lane, insert a **new slice immediately after** (`insert_slice` with `at` = that column's position + 1) and place the READ MODEL there.

### Never stack read models at the end
Placing all read models in new columns at the very end of the sheet severs the visual connection to the events they are derived from. The sheet must show a coherent left-to-right narrative where each slice is self-contained.

### Place in narrative order, never type by type
The steps below decide *what* exists (events, commands, read models, screens); they do not decide the *order of placement calls*. When you write elements to the sheet, order them **left to right by the story**, one slice at a time: the screen, command and event of a state change, then the read model (and its view screen) that the event feeds, then the next state change. A read model's column goes right after its source event's column, so placing the next command after it is simply the next column.

Never place "all events, then all commands, then all read models" (or any other by-type batch). It forces every read model into a column inserted afterwards, shifts everything already placed, and produces the shape this skill forbids: a row of commands across the sheet with the read models bunched in their own block. If your element list is grouped by type, re-sort it by column before you start placing.

### One read model per component, not one read model per screen
A read model does not necessarily serve a whole screen: **one component in a screen resembles one read model.** Storyboarding (Step 3) only produces a plain screen per screen state; it does not decide components. That decision, and the work of breaking a multi-component screen apart into copies, happens in Step 5 (Identifying Outputs), because a component is defined by its read model.

A screen-wide read model that has to aggregate from events scattered across many columns is the most common source of unnecessary coupling, a "god read model". A screen with a stats row and a list below it is two components: two read models and two screens showing the *same page* (same screen name on both, each a plain screen; they differ only in which component is marked, with the rest blurred), not one screen-wide read model. Each narrower read model then sits naturally close to its own source event, and the `EVENT → READMODEL → SCREEN` chain for each component stays short and forward. See `eventmodeling-identifying-outputs`'s "Step 5a — Enumerate consumers and identify components" and "Step 5c — Break apart multi-component screens into one screen per component" for the mechanics. Apply this during Step 5, before the placement problem exists, rather than reordering columns to patch it afterward.

The narrow exception to "prefer a new read model" is a single homogeneous list/table (e.g. a catalog whose per-row status comes from many different lifecycle events). That is still **one** component: you cannot split "one field's value" into multiple read models just because many event types feed it, and a field like that legitimately keeps its wide fan-in. Reach for this exception only after genuinely failing to find a split, not as the default. **But treat this as a per-field fact about one specific field's irreducible fan-in, never as a label you apply to the whole read model or the whole screen.** There is no "roll-up component" exemption that, once invoked, clears every other field sharing that element from scrutiny. That shortcut is exactly how a god read model smuggles in unrelated fields (e.g. bundling a book's title/author, needing only its own 1-2 events, into the same element as a copy's live availability status, just because both happen to render on the same page). Run the checks below on **every** read model, every time, regardless of whether some field on it was already justified as wide.

### The >3-events heuristic — evaluate every field, every time

**Wide fan-in is the absolute exception, not a comfortable resting point.** Default to assuming a wide read model is wrong and a new, narrower read model is almost always the better answer. The burden of proof is on keeping the connections, not on splitting them. Whenever a read model ends up connected to more than 3 events, your first hypothesis should be "this can and should become two (or more) read models", and you only abandon that hypothesis after the two checks below genuinely fail to find a split. Not because the component looks homogeneous at a glance, not because a prior step called it a roll-up, and not because splitting would mean more elements or columns on the sheet. More elements is a fine outcome; a wide read model that could have been narrower is not.

Two distinct checks, both required, and neither is satisfied by having done it once for a different field on the same element:

1. **Field-level minimality.** Re-derive, field by field, exactly which connected events each field actually needs (`get_slice_view` shows the fields and the flows with their mappings). Any event connected to the read model that no field traces back to is a prunable connection. Remove it (`remove_flow`), regardless of how the read model got wide.
2. **Kind-of-computation split.** Within one visual component, a field needing 1-2 simple, non-overlapping events (e.g. a monotonic counter incremented/decremented by its own dedicated events, or an identity/fact field set once and occasionally updated) is a fundamentally different kind of computation from a field needing wide fan-in across a whole entity's lifecycle (e.g. "is this available right now", which by construction depends on every event that can change that state). When a read model bundles both kinds, split the wide-fan-in field(s) out into their own read model and screen copy, always, with no exception for "but the wide field is a legitimate roll-up so the element as a whole is fine." A legitimate wide field never excuses a narrow field riding along in the same element, and a handful of narrow fields never excuses leaving a wide field un-isolated.

After both checks, a read model may still be wide because one or more of its fields have irreducible, genuinely-single-field fan-in. That is an outcome you document (per "Documenting decisions inline" below, naming the specific field and why), not a category ("roll-up component") you assign to the element up front and stop checking. Re-run this evaluation on every read model at every step that touches it, including ones a prior step already looked at: a field added or a screen re-scoped later can introduce exactly the bundling problem this heuristic exists to catch.

### Recognizing a dedicated business event behind a derived condition

This is a **business-modeling decision, not a technical one**. It is not triggered by a read model's fan-in count and is not a fix for anything the >3-events heuristic flags. A read model landing on wide fan-in is at most a symptom that might prompt you to ask the question below; it is never the reason to answer it one way or the other, and the question is worth asking regardless of how many events are involved.

The question to ask is simply: **is this fact worth a new event?** A derived condition ("is this available right now", "has this moved to its next stage") is worth its own event when it is a fact a domain expert would recognize and name in its own right (not just "some field I compute"), and when that fact is valuable to a later step in the process: another automation would react to it, a different bounded context would want to subscribe to it, some downstream process needs to trigger off of it. If both hold, it deserves to exist as its own dedicated event ("the copy was marked available"), not just as derived read-model logic recomputed from several other events.

When that is the case, model it as its own event, produced via the same todo-list + automation translation pattern `eventmodeling-designing-automation-chains` (Step 4b) uses for external integrations, but triggered internally by whichever raw events can produce that outcome. If the condition is purely for display, with nothing downstream that would ever act on it, it stays a plain read-model projection. No new event is needed, no matter how many raw events feed it or how wide the resulting fan-in looks.

**Only fold together causes that are genuinely redundant for the same outcome, never causes that carry distinct business meaning.** A derived condition's causes typically split into two groups: several distinct events that all mean the *same* thing from the business's point of view (e.g. `Copy Reservation Released`, `Copy Returned`, and `Copy Returned From Repair` all mean "the copy is available again"), and several distinct events that each mean something the business still wants told apart (e.g. `Copy Reserved` → Reserved, `Copy Checked Out` → CheckedOut, `Copy Sent For Repair` → UnderRepair, `Copy Reported Lost` → Lost, `Copy Withdrawn` → Withdrawn). Only the first group is safe to consolidate. A dedicated event like `Copy Marked Available` collapses "N different reasons, same outcome" into one reusable signal without losing information. Collapsing the second group into something like a generic `Copy Marked Unavailable` would erase the *why*, which some downstream consumer may actually need. Leave those as separate events and direct connections, even though the read model's fan-in then stays wider than the fully consolidated ideal. That remaining fan-in is not a failure of anything: it means those events are each individually meaningful, not synonymous with each other, and collapsing them would have been the actual modeling mistake.

### No stranded elements

An element can't be left at a default position, because it only exists in a cell. What can be left stranded is structure that connects to nothing. After each step that creates elements (Steps 1–5), run `check_model` for each sheet in scope and read these findings: `empty-slice` (a column with nothing in it), `event-disconnected`, `screen-disconnected`, `orphan-element`, `read-model-unconsumed`, `read-model-without-source`, `command-without-event`, `dangling-flow`.

**For each one:**
- **If it belongs in the current model**: connect it (`create_flow`) or move it to the cell where it belongs (`move_element`).
- **If it is an orphan (duplicate or no longer needed)**: remove it (`remove_target`).

Never leave a stranded element on the sheet when proceeding to the next step.

### No backward arrows
The sheet must always progress left to right. This is the goal to design toward, not just a validation check to run afterward. Every connection arrow (SCREEN→COMMAND, COMMAND→EVENT, READMODEL→SCREEN, READMODEL→AUTOMATION, AUTOMATION→COMMAND) must point to the right or downward (within the same column). A right-to-left arrow among these is always a layout error, full stop. The tools accept a backward flow without complaint, but `check_model` reports every one as `flow-backward` (an error), so run it rather than scanning by eye.

**`EVENT → READMODEL` has exactly one exception, and it is narrow.** A read model that already carries a `READMODEL → AUTOMATION` edge (a todo-list read model feeding an automation, per `eventmodeling-designing-automation-chains`, Step 4b) may also be fed by a later-column event closing an item it opened earlier. That accumulator shape is what the todo-list pattern exists for. **Outside that one case the connection is wrong, for good reason:** without the rule, a read model would become a moving target for whatever screen or scenario later reaches back into it.

For every other read model, in particular one feeding a SCREEN rather than an AUTOMATION, never connect a later event back into it, no matter how convenient. If a later event needs to update what a screen already shows, resolve it the same way Step 5c resolves multi-component screens: place a **new READMODEL** in a new column immediately after the later event's column, make it a linked copy of the earlier instance (`copy_element`; it is the same read model, not an unrelated new one; see `eventmodeling-core-rules`'s Linked Copies section), connect the later event forward into the linked copy, update the field(s) that event actually changes, and place a matching screen there: a plain screen with the earlier one's name (not a linked copy, which would show the earlier design exactly), showing the updated data, with what changed marked (`marks`).

A wide fan-in read model (many connected events, one column) is a different problem with a different fix: see "one read model per component" and the >3-events heuristic above. It is never a justification for a backward arrow. The one real signal to treat as a prunable connection regardless of column position is a **connected event that isn't actually used by any field** on the read model.

Before wiring any of the five forward-only pairs, verify that `column(source) ≤ column(target)`. If this is violated:
- Move the earlier-placed element to the correct column (`move_element`), OR
- Insert a new slice at the right position to restore the correct order.

Screens placed during Step 3 (Storyboarding) are provisional positions. Steps 4 and 5 may need to move them to align with the commands or read models placed later.

### Column insertion
Use `insert_slice` with `at` (the 1-based column position; existing columns shift right) to create a slice exactly where it belongs, and `cols` when it needs to be wider than one column. Do not append when placing read models or view screens; always target the correct position. To make room for a second element on the same lane, widen the slice (`span_slice`) instead of stacking.

Placing an element never connects it to anything. The one exception is `source` on a field, which creates the flow it names. Every other connection is yours to create with `create_flow`, so wire an automation chain's edges deliberately instead of expecting any to appear.

### Read and write economically

Read in tiers: `get_model_view` for orientation, `get_sheet_view` for the grid, `get_slice_view` only for a slice you are about to edit. Reserve `get_cell_view` for pinpoint reads. For "what is on the sheet and how is it wired right now" between steps, `get_sheet_view` is the orientation read and the input to choosing an insertion position.

Prefer the granular tools, one placement or flow at a time in story order. Use `apply_operations` only for a unit that must land all-or-nothing, and `build_sheet_from_spec` only to scaffold a whole sheet whose shape is already settled.

**Structural validation is one call, not a scan.** `check_model` runs every mechanical check (wiring, who issues each command, flow direction, one event per column, slice shape, naming, field traceability, scenario coverage and shape, dangling references) and returns only findings. Use it for the mandatory post-step check and as the first move in Step 9. It does not judge; the rules in this skill and its step skills still apply on top.

### Documenting decisions inline, at any step

Separate from the Step 11 sheet-level reasoning note: at **any** step (1–10), if that step makes a decision or assumption important enough that a later reader could otherwise misread the model, leave a small note where the decision applies (`set_note` on the cell, slice or lane; read the existing note first, since a write replaces it). Use sparingly. This is for a genuine "why is it like this" moment (an assumption that fills a gap the brief left open, a rejected alternative, a non-obvious constraint), not routine narration of what a step did.

---

## Interview Phase

**Skip if**: the user has provided a clear domain description, requirements or scope, and a stated output goal (design, learning, validation, documentation).

**When interviewing**, ask:

1. **Domain**: "What are you modeling? Describe the business process in 2-3 sentences."
2. **Requirements state**: "(A) Written requirements/user stories, (B) Rough ideas, (C) Existing system to reverse-engineer?"
3. **Goal**: "(A) Learning event modeling, (B) A model ready to build from, (C) Design validation, (D) Team documentation?"
4. **Constraints**: "Any constraints? (timeline, external integrations, team size)"
5. **Starting point**: "Are you starting from scratch, or do you already have outputs from earlier steps (event list, commands, scenarios)?" Confirm the answer against the real model state read above; don't take it at face value.

Confirm understanding before proceeding: "So we're modeling [domain], goal is [goal], constraints are [constraints]. Starting from [step]. Does that match?"

**Capture findings**: follow **`eventmodeling-interview-protocol`** to record this step's own findings. This step starts the record: the note on the first slice of the first sheet opens with this header, and every later step adds its section and trail row to it. Until a sheet with a first slice exists (Step 1 creates it), keep the record in the conversation and write it into the note the moment the slice does:

```markdown
# Event Modeling: [Project Name]

**Started**: [ISO date]
**Goal**: [learning / ready to build / design validation / documentation]
**Constraints**: [timeline, integrations, team size]

## Interview Trail

| Step | Skill | Status | Key Output |
|------|-------|--------|------------|
```

Add this step's own row ("Orchestration" / `eventmodeling-orchestrating-event-modeling` / domain scoped, starting point confirmed).

---

## Phase Transition Protocol (Mandatory After Every Step)

After each step completes, before invoking the next skill, record a phase summary. In the conversation, and appended to the first-slice note per `eventmodeling-interview-protocol`:

```markdown
### Step N complete — [Skill Name]
- **What was done**: [2-4 bullet points — key artifacts created, decisions made, gates passed]
- **Carry-forward**: [what the next step needs from this step]
- **Open questions**: [anything unresolved or deferred]
```

Update the Interview Trail row for this step (Status → Done, Key Output → one-line summary). Anything in "Open questions" that needs an answer from a person is also a pin (`create_pin`) on the area it concerns. Then re-read the sheet you just touched (`get_sheet_view`) to confirm what actually landed before loading the next skill.

---

## Mid-Workflow Entry

To get oriented in a model someone else built, start with `eventmodeling-analyzing-existing-models` (read-only: what is there, how complete, what is open). If the user already has outputs from earlier steps, start from where they are. Don't ask them to recite which steps are complete; read it from the model: events with no commands means Steps 1-2 are done and Step 4 isn't; screens with no wired commands means Step 3 is done and Step 4 isn't; and so on. Ask only what the model can't tell you. Do not re-run completed steps. Pick up from the first incomplete one.

---

## Workflow

### Step 1: Brainstorm Events

Invoke `eventmodeling-brainstorming-events`.

**Input**: Domain requirements and any existing knowledge about the domain.
**Output to carry forward**: Event list + Role Catalog + dedicated sheets (one per workflow / bounded context) with events already placed in their correct sheet. The Role Catalog (all human roles and system actors) feeds into every subsequent step. Sheet discovery happens here. It does not happen in Step 2.
**Gate**: Do not proceed until the Role Catalog exists, events cover all known business processes, and every event is placed on a named sheet.

---

### Step 2: Plot Events

Invoke `eventmodeling-plotting-events`.

**Input**: Events already placed on their sheets from Step 1. This step focuses on **one sheet at a time**. Ask the user which sheet to sequence first if multiple exist.
**Output to carry forward**: Chronological event ordering within the chosen sheet, showing causal dependencies between events.
**Gate**: The chosen sheet reads as a coherent narrative before proceeding. Repeat for each additional sheet before moving to Step 3.

---

### Step 3: Storyboard

Invoke `eventmodeling-storyboarding-events`.

**Input**: Event sheet + Role Catalog.
**Output to carry forward**: Screens with one `actor` lane per human role, showing what data each screen displays and collects.
**Gate**: Every human role from the Role Catalog has at least one screen.

Give every screen a real HTML design (`place_element` with `design`, or `set_screen_design`); a bare labelled box is an unfinished thought.

You can reuse columns if screens can be matched to existing events: place the screen in the same column as the event, in the actor lane.

---

### Step 4: Identify Inputs

Invoke `eventmodeling-identifying-inputs`.

**Input**: Storyboards + Role Catalog.
**Output to carry forward**: Command definitions, each attributed to a specific role or system processor.
**Gate**: Every UI action in the storyboards maps to a named command.

---

### Step 4b: Design Automation Chains

Invoke `eventmodeling-designing-automation-chains`.

**Input**: Every AUTOMATION placed in Step 4, each already paired with its own COMMAND in one column.
**Output to carry forward**: Every automation's todo-list READMODEL, placed and wired; every externally triggered automation resolved into a two-stage translation chain (external EVENT → todo-list READMODEL → translation AUTOMATION+COMMAND+internal EVENT → worker automation's own todo list).
**Gate**: No AUTOMATION on the sheet lacks an incoming `READMODEL → AUTOMATION` connection, and no automation's todo list is opened directly by another system's (second-swimlane) event. Skip this step only if Step 4 placed zero automations.

---

### Step 5: Identify Outputs

Invoke `eventmodeling-identifying-outputs`.

**Input**: Event list + Commands from Step 4 + automation chains already resolved in Step 4b + the plain screens placed in Step 3.
**Output to carry forward**: Read model definitions (projections of events optimized for UI queries), one per screen component, with any multi-component screen already broken apart into same-named screens, each with its component marked (this step's Step 5a/5c, not Step 3's job). Automation todo-list read models are already complete from Step 4b and are not re-derived here. This step only ever designs screen-facing read models.
**Gate**: Every screen data need from the storyboards is satisfied by a read model, and no read model spans more than one component.

---

### Step 6: Apply Conway's Law

Invoke `eventmodeling-applying-conways-law`.

**Input**: Full event model so far (events, commands, read models).
**Output to carry forward**: System swimlanes mapping events and commands to team boundaries.
**Gate**: Each boundary can be independently owned by a team. Skip this step if Conway's Law boundaries are not relevant to the project.

---

### Step 7: Elaborate Scenarios

Invoke `eventmodeling-elaborating-scenarios` **before writing the first scenario**. Its scenario types, business-rules-only rule and payload shape are not repeated here, so scenarios written from this file alone come out wrong.

**Input**: Commands and read models.
**Output to carry forward**: Given-When-Then specifications (or storylines, for walkthrough-style coverage) for every command **and every read model**, placed in the specification-lane cell of the slice they specify.
**Gate**: Every command has scenarios covering **all applicable types** from the elaborating-scenarios workflow, not just happy path + one error case, **and every READMODEL on the sheet has at least one view scenario**. See the gate checklist below. A command-only pass is an incomplete Step 7, even if every command's coverage looks exhaustive.

> **Do not reduce scenarios to a simple good-case / bad-case pair**, but do not pad them with technical cases either. Which scenario types apply is defined in `eventmodeling-elaborating-scenarios` and decided by the domain.

> **Read models need scenarios too. It's easy to forget, since the seven types are command-shaped.** Every READMODEL needs at least one view scenario (GWT or storyline); a read model with zero scenarios is as incomplete as a command with zero. `eventmodeling-elaborating-scenarios`'s own checklist covers the details (connectivity rules, GWT-vs-storyline judgment per read model, and avoiding redundancy between a storyline and its GWTs). Don't re-derive those here, just enforce the gate.

---

### Step 8: Check Completeness

Invoke `eventmodeling-checking-completeness`.

**Input**: Full model: events, commands, read models, scenarios, Role Catalog.
**Output to carry forward**: Field traceability matrix confirming every field has an origin and a destination. List of any gaps found.
**Gate**: All gaps resolved or explicitly accepted before proceeding.

---

### Step 9: Validate

Invoke `eventmodeling-validating-event-models`.

**Input**: Complete event model.
**Output**: Validation report with PASS / PASS WITH WARNINGS / FAIL verdict.
**Gate**: PASS verdict before declaring the model ready to build from.

If FAIL: address findings and re-invoke `eventmodeling-validating-event-models`.

**Optional — Structural Checklist**: Invoke `eventmodeling-validating-event-models-checklist` for a second, mechanical pass. It runs 12 structural checks across 6 phases and returns a PASS / PASS WITH WARNINGS / FAIL verdict independently of Step 9. A PASS on Step 9 does not substitute for this checklist.

---

### Step 10: Slice the Model

Invoke `eventmodeling-slicing-event-models`.

On a sheet the slices already exist as first-class columns, created as the model was built. This step is the audit: confirm that every independently deployable unit of work has exactly one slice sized to it (`span_slice`, `split_slice`, `move_column`, `update_slice`), and that no slice bundles unrelated work or splits one unit of work in two.

**Input**: Complete, validated event model (PASS from Step 9).
**Output to carry forward**: One slice per COMMAND (state-change), per READMODEL (state-view), and per AUTOMATION on the sheet: the model's independently deployable feature boundaries.
**Gate**: Every COMMAND, READMODEL, and AUTOMATION on the sheet sits in a matching slice; no duplicates exist for elements that already had one; linked copies imply no slice of their own.

---

### Step 11: Document Reasoning

Not delegated to a separate skill. It is performed directly by this orchestrating skill, since the reasoning being documented is the *orchestrator's own* accumulated context across all prior steps, not something a single-step skill has visibility into.

**Input**: The complete, sliced, validated model (Steps 1–10) plus this session's own record of decisions made along the way: assumptions added beyond the literal brief, sequencing corrections, business rules deliberately encoded as scenarios rather than events, read-model sharing choices, and any cross-context/integration gaps found (e.g. during Step 6 or Step 9).

**Output to carry forward**: One reasoning note per sheet, on that sheet's first slice (`set_note` on column `A`), containing the full modeling reasoning for that bounded context in as much detail as the session actually has to give, not a boilerplate template filled in thinly.

**Gate**: Every sheet has a reasoning note on its first slice, non-empty, written after the model for that sheet was already complete (so it can describe the *finished* shape, not a plan).

**Mechanics**: the first slice's note already holds the interview record (`eventmodeling-interview-protocol`). Read it first (`get_slice_view` on the first slice) and add a `## Modeling Reasoning` section to it. Do not overwrite what is there. Write it as markdown source: headings, lists, bold, tables.

**What the note should actually contain.** Write for the next person (or next session) who opens this sheet cold, not for whoever just built it. Wherever it names an element on the sheet, use the element's name, not its address: addresses shift when columns are inserted.
- **Scope**: what business process this sheet covers, and its entities (identity keys).
- **Assumptions added beyond the literal brief**: anything invented to fill a gap the requirements left open, and why (e.g. adding a resolution event so a state isn't a one-way trap door).
- **Business rules deliberately encoded as scenarios, not new events**, so a reader doesn't mistake a missing event for an oversight.
- **Sequencing or design corrections made mid-workflow**, e.g. a column reorder because an event's original placement implied the wrong causality.
- **Read model design rationale**, especially where one read model deliberately serves several screens/automations, so it doesn't read as a missing 1:1 mapping.
- **Any cross-context or integration gaps found** (Step 6 Conway's Law, or discovered incidentally, e.g. a same-sheet connection constraint blocking a needed cross-sheet data dependency). State the finding and the viable resolutions, matching whatever note or pin was also left on the affected area.
- **Closing summary**: element counts and the validation verdict for this sheet's slice of the model.

If a sheet's story is genuinely simple, say so briefly rather than padding. But for any sheet with real design decisions behind it, this note is the place those decisions survive past the session that made them.

---

## Final Output

A complete, sliced event model consisting of:
- Role Catalog (human roles and system actors with permissions)
- Chronological event sheets
- Screens with role-based actor lanes
- Command definitions with actor attribution
- Read model designs
- System boundaries (if Conway's Law applied)
- Given-When-Then scenarios
- Completeness verification
- Validation report with readiness verdict
- Slices marking every independently deployable feature boundary
- A reasoning note on each sheet's first slice, documenting the design decisions, assumptions, and any integration gaps behind that sheet's model

### Optional Follow-on Skills

These skills are not part of the 11-step main path but extend the model for specific needs:

- **`eventmodeling-designing-event-models`**: use when the Command/Event/Read-Model structure or event causality need detailed design work. Can be applied at any step where those decisions arise, most commonly during or after Step 1.
- **`eventmodeling-optimizing-stream-design`**: use after the model is complete to validate that every entity's event timeline is anchored on a single business identity, not a disguised collection or event log.
- **`eventmodeling-translating-external-events`**: use when external systems (webhooks, IoT, third-party APIs) need to feed into the domain model.
- **`eventmodeling-modeling-from-documents`**: use when the requirements arrive as a document, a URL or an image: it reads the source into a fact register and drives this workflow from it.
- **`eventmodeling-assessing-scenario-coverage`**: use after Step 7 to rate how much of each slice's behaviour its scenarios specify and list the meaningful cases still missing (stored on the slice, shown on its header).
- **`eventmodeling-adding-next-slice`**: use when the model is complete as far as it goes and a genuinely new capability is next.
- **`eventmodeling-reviewing-business-fit`**: use after Step 9 for a business analyst's read of the model: what the business would question, as pins.
- **`eventmodeling-analyzing-stressors`**: use after Step 9 to test the design against what could hit it (residuality stressor analysis) and redesign where it breaks.
- **`eventmodeling-tracing-data`**, **`eventmodeling-adding-attributes`**, **`eventmodeling-adding-examples`**: use to follow one piece of information through the model, to add or rename a field along its whole chain, and to give fields and scenarios realistic example values.
- **`eventmodeling-workshop-questions`**: use when questions should go to the domain people rather than to the requester.

### Further Reading

- **[Project Planning with Event Modeling](references/project-planning-with-event-modeling.md)**: why explicit step contracts produce a flat cost curve and let teams build in parallel, plus velocity-based estimation and capacity planning built on workflow steps instead of story points.

---

## Quality Checklist

- [ ] No stranded structure: `check_model` shows no empty slices, disconnected events or screens, orphan elements, or dangling flows
- [ ] No `EVENT → READMODEL` flow points backward unless the read model already has a `READMODEL → AUTOMATION` edge (todo-list pattern); `check_model` shows no `flow-backward`. Every other later-event update uses a new read model + screen copy, never a link back to the earlier instance
- [ ] All 11 modeling steps completed (plus Step 4b whenever Step 4 placed any automations), no step skipped without explicit reason
- [ ] Every AUTOMATION has a todo-list READMODEL, and no automation's todo list is opened directly by another system's event (Step 4b)
- [ ] Every COMMAND, READMODEL, and AUTOMATION sits in a matching slice (`check_model` shows no `slice-mixed-shape`)
- [ ] Every sheet has a reasoning note on its first slice, written after that sheet's model was complete
- [ ] Role Catalog exists with named human roles and system processors
- [ ] Every command is attributed to a specific role from the Role Catalog
- [ ] Every read model satisfies at least one UI or processor query need
- [ ] At least one Given-When-Then scenario exists per command
- [ ] At least one view scenario (GWT or storyline) exists per READMODEL, not just per command
- [ ] Completeness check shows no unresolved field traceability gaps
- [ ] Validation returns PASS or PASS WITH WARNINGS with all critical issues resolved
- [ ] The Interview Trail on the first slice's note shows the status of each completed step
- [ ] A phase summary was recorded after every completed step before loading the next skill
