---
name: eventmodeling-core-rules
description: Shared, foundational rules for every event-modeling step — what a COMMAND/EVENT/READMODEL/SCREEN/AUTOMATION is, how each is named, where each lives on a sheet, and the anti-patterns to reject. Not a step of its own — load it once per session before any modeling, then the step skill. Every other eventmodeling-* skill refers to it instead of restating it.
---

# Event Modeling — Core Rules

These rules apply to every step of event modeling, regardless of which step's SKILL.md you're currently following. Read this once per session — a step's own file only states what's specific to that step, not these fundamentals. [tools](references/tools.md) maps each concept here to the tool that does it.

## Modes

Which step you're in sets the posture — don't mix them.

- **Modeling Mode** — `eventmodeling-brainstorming-events`, `eventmodeling-plotting-events`, `eventmodeling-storyboarding-events`, `eventmodeling-identifying-inputs`, `eventmodeling-identifying-outputs`, `eventmodeling-designing-automation-chains`, `eventmodeling-elaborating-scenarios`, `eventmodeling-translating-external-events`: explore and build. Capture the process as completely as you can. A naming slip or an incomplete precondition is something to flag and move past, not a reason to stall the whole step — don't over-correct early.
- **Critic Mode** — `eventmodeling-validating-event-models`, `eventmodeling-validating-event-models-checklist`, `eventmodeling-checking-completeness`, `eventmodeling-reviewing-business-fit`, `eventmodeling-optimizing-stream-design`: review what's already there. Apply every rule in this file strictly. Surface every violation, gap, and inconsistency — don't soften a finding because the model "mostly works." Record findings as notes (`set_note`) and questions as pins (`create_pin`) rather than silently fixing them, unless the step explicitly says to fix.

Don't mix them in one pass: a modeling step is not the place to run a full rules audit, and a critic step is not the place to quietly add missing structure on the model's behalf instead of flagging it.

## Lanes — What Lives Where

Every column on a sheet has typed lanes (rows), and each element type belongs in exactly one lane kind:

| Lane type | Elements allowed | What it represents |
|---|---|---|
| `actor` | SCREEN, AUTOMATION | Who or what initiates — a human via a screen, or a system via an automation |
| `interaction` | COMMAND, READMODEL | The business intent going in (COMMAND), or the query result coming out (READMODEL) |
| `swimlane` | EVENT, EXTERNAL EVENT | The business fact recorded — which system/entity's own story this event belongs to |
| `specification` | SCENARIO | The Given/When/Then behavior spec for this column's COMMAND or READMODEL |
| `notes` | none | Free markdown per column, e.g. the modeling reasoning behind it. Its content is the cells' notes |

A cell holds one element (scenarios alone may stack). An EVENT never shares a column with another EVENT even across different swimlane lanes — one column is one moment in time (`check_model` reports `column-multiple-events`). See [tools](references/tools.md) for the placement mechanics; this table is only "what belongs where," not "how to place it."

## Slices

A **slice** is the thinnest possible vertical cut through the model — exactly one COMMAND, one READMODEL, or one AUTOMATION's command, never combined:

```
state-change slice = SCREEN/AUTOMATION → COMMAND → EVENT(s)
state-view slice   = EVENT(s) → READMODEL → SCREEN/AUTOMATION
automation slice   = EVENT(s) → AUTOMATION → COMMAND → EVENT(s)
```

A slice never mixes a COMMAND and a READMODEL — those are two different slice types (`state-change` and `state-view`), even when they belong to the same feature. Slices are independently deployable and only communicate with each other via events. `check_model` reports `slice-mixed-shape` for a slice that holds more than one of: a command, a read model, an automation with its command. Linked copies are not counted.

On a sheet a slice is a first-class column, or a contiguous span of columns, with a name and a status (`insert_slice`, `span_slice`, `split_slice`, `update_slice`) — it is not derived afterwards. Every COMMAND on the sheet implies a `state-change` slice, every READMODEL implies a `state-view` slice, every AUTOMATION implies an `automation` slice — **except** a linked copy (see Linked Copies below), which implies no slice of its own: it's the same underlying COMMAND/EVENT/READMODEL reused elsewhere, not a second independently-deployable thing, so only the original's column counts as a slice. Every column still belongs to some slice: a column holding only copies sits in a slice named for what it shows, whose note says it continues the original. It is not counted as a separate unit of work. Such a column exists because its copies are wired to something on this sheet; a column of copies wired to nothing is not a slice, it is clutter (see Linked Copies). `eventmodeling-slicing-event-models` checks the slice boundaries once modeling is done; it never invents a slice that isn't already implied by an element that's there, and skips columns whose only COMMAND/READMODEL is a linked copy.

## Elements

### EVENT
- Represents a business fact that already happened.
- Naming: past tense, business language.
  - Valid: `Order Placed`, `Payment Authorized`, `User Registered`
  - Invalid: `Sidebar Opened`, `Request Completed`, `Api Called` — these describe machinery or UI state, not business facts
- Immutable — never modify or delete an event once created; a correction is a new event, not an edit.
- Contains only captured facts. No computed or derived values — those belong in a READMODEL.

### COMMAND
- Represents business intent — what an actor wants to do.
- Can be rejected; only succeeds if its documented preconditions hold.
- Carries every piece of information its validation needs that isn't in the stream's prior events. That includes context such as the current time or date, the acting user and the tenant. The decision never reads a clock or a session on its own. See `eventmodeling-identifying-inputs` → "Command input is self-contained".
- Naming: imperative, business language.
  - Valid: `Place Order`, `Confirm Payment`, `Cancel Subscription`
  - Invalid: `Load Orders`, `Fetch Data`, `Open Dialog` — these are queries or UI-only actions, not business intent
- Issued by exactly one SCREEN or AUTOMATION — never two. `check_model` counts the flows into the command (`ScreenToCommand` or `ProcessorToCommand`; a screen and its linked copy are one issuer). One is right. None is `command-without-issuer`, and means the issuer is missing: place the screen or automation in the actor lane of the command's column. Two or more is `command-multiple-issuers`: decide who really issues it and `remove_flow` the other, which usually means a second command was hiding in one. The mechanics are in [tools](references/tools.md).
- Produces one or more EVENTs on success, or is rejected with no event.

### READMODEL
- A projection built from events, shaped for a specific query.
- Naming: noun phrase, business language, named for the data — not the machinery that produces it.
  - Valid: `Order Summary`, `Invoice List`
  - Invalid: `Order Summary Projector`, `Invoice List Repository` — implementation detail leaking into the model
- Optional — nothing requires a READMODEL to exist for a COMMAND or a downstream consumer.
- Never drives COMMAND validation directly — a COMMAND is checked against its own documented preconditions, not a read model.
- **List-shaped read models**: when the whole READMODEL represents multiple rows (each row shaped by its defined fields), not a single projected record, set `isList: true` on the read model itself (`place_element` / `update_element`) — this is a flag on the read model, distinct from a field's own `isList` (a single field holding multiple values). Every todo-list read model (see `eventmodeling-designing-automation-chains`) is list-shaped and always gets `isList: true`; the same applies to any other READMODEL whose query naturally returns a set of rows (e.g. `Product List`, `Invoice List`). `eventmodeling-elaborating-scenarios` reads this flag to decide whether a scenario needs row-level `rows`/`expectEmptyList` instead of a single accumulated result.

### SCREEN
- Represents what a user sees and can act on.
- Naming: what a user would call it, not its internals.
  - Valid: `Dashboard`, `Order Overview Page`
  - Invalid: `Order Overview Component`, `Projector View`

### AUTOMATION
- Represents an automated actor — a processor, scheduler, or system reacting to events.
- Naming: what it does, in business language.
  - Valid: `Billing Scheduler`, `Inventory Reserver`
  - Invalid: `Order Service Handler` — implementation-flavored, not a business actor's name

Names are readable business words with spaces. Tense and business language are the rules; stay consistent with the casing already on the sheet.

## Causality

- Every EVENT traces to the COMMAND that produced it (`check_model` reports `event-without-command` where nothing produces an event; an event that is another system's own timeline, in a second swimlane lane, is the expected exception).
- Every COMMAND traces to the SCREEN or AUTOMATION that issued it — exactly one.
- A process starts with a read (SCREEN + READMODEL) or an AUTOMATION reacting to an existing EVENT — never with an unmotivated COMMAND.

## Role Catalog

Every event model needs one, established before brainstorming events (it's the first mandatory step) — without it, downstream steps have no way to say *who* does what:

- **Human roles** (Customer, Seller, Admin...) and **system actors** (Payment Gateway, Scheduler...), each with: a name in domain language, a one-line description, its key actions, and what it explicitly cannot do.
- On a sheet a role is an `actor` lane named for it, its description, key actions and limits are the lane's note (`set_note`), and a command's role is the lane of the screen or automation that issues it.
- No COMMAND may be attributed to a generic "User" — it must name a specific role or actor from the catalog.
- Every COMMAND is issued by exactly one role/actor (this is the same rule as the COMMAND single-issuer rule above, stated from the Role Catalog's side).
- Every human role needs at least one COMMAND and one READMODEL — a role in the catalog that never acts and never sees anything is decorative, not modeled.

See `eventmodeling-brainstorming-events` for how the catalog is built, `eventmodeling-identifying-inputs` for attributing commands to it, and `eventmodeling-validating-event-models` for re-checking it holds once the model is complete.

## Field Lineage (Information Completeness)

Every field on every element has a known source: a same-named field on a connected upstream element, a mapping to a differently named field or a connected event (`source`, or `set_flow_mapping`), or `isGenerated`/`isUserInput` where the information starts. Check it each time you add or rename a field. **A field with no source is a gap, not a detail to fill in later.** The full rule, per element type, is in `eventmodeling-information-completeness`.

## Connections Read Forward

A connection either goes **downward within the same column** (actor → interaction → swimlane → spec) or **forward to a later column** — never backward (right-to-left). This is why elements that belong together — a SCREEN and the READMODEL it queries, a SCREEN/AUTOMATION and the COMMAND it issues, a COMMAND and the EVENT it produces — should share a column whenever possible: the moment they don't, a backward arrow becomes a real risk. The tools do not refuse a backward flow, but `check_model` reports every one as `flow-backward` (an error), except the todo-list closing event described below.

When an element's natural column is already occupied by something else, insert a new column immediately before or after (whichever keeps every connection forward; `insert_slice` with `at`) rather than wiring across the gap. See each step's own placement section for where "before" vs. "after" applies.

## Updating a Read Model That Already Feeds a Screen

A later EVENT sometimes needs to change data an earlier READMODEL already shows on a SCREEN — e.g. a cancellation affecting an "active items" view placed several columns earlier. Wiring that later EVENT straight back into the existing READMODEL is never the fix: it's a backward connection, and the only backward `EVENT → READMODEL` edge the method allows is into a READMODEL that already has a `READMODEL → AUTOMATION` edge (the todo-list pattern — see Translation Chain below; `check_model` reports every other backward flow as `flow-backward`). A READMODEL that only feeds a SCREEN never qualifies for that exemption.

Instead, place a **new READMODEL** in a new column **immediately after** the later event's column — it *is* the same read model, carried forward to a later state, not an unrelated new one. (The method marks it as a linked copy whose changed field is then updated; here a linked copy carries no fields of its own and cannot change one, so place a plain read model with the **same name** and the fields as they stand after the event, `status` updated, rather than using `copy_element`.) Connect the later event forward into it; the fields the new event doesn't touch stay as they were. Give it a matching SCREEN in that same new column — same screen name as the earlier one, showing the updated data, optionally re-marked/highlighted with `marks` (`set_screen_design`) to call out what changed. The new SCREEN is placed as a plain element matching the earlier one's name, not as a linked copy (a copy would show the earlier design exactly). Leave the earlier READMODEL/SCREEN instance exactly as it is — it's still what that earlier point in the timeline correctly showed. Never delete it.

See `eventmodeling-identifying-outputs` Step 5g for the full placement/wiring mechanics.

## Linked Copies

A **linked copy** is a COMMAND/EVENT/READMODEL element that mirrors another element elsewhere in the model — most often needed because a flow (`create_flow`) only ever pairs elements on the same sheet, so an element on a different sheet can't be wired to directly. The same marking obligation applies whenever an element is reused rather than newly wired anywhere in the model, same sheet included: never place an element that repeats another swimlane's already-recorded fact under its own unlinked identity — either connect forward to the original, or mark it a linked copy. The one exception is a translation chain's own internal EVENT (`eventmodeling-designing-automation-chains`): it is a genuine new domain fact recorded in this side's own language, not a copy, even when it shares the external EVENT's name.

To make one, use `copy_element` with the original's address and the destination cell (for an original on another sheet, `fromSheet` names the original's sheet and `sheet` the copy's). Commands, events, read models, screens and external events can be copied; automations cannot. The copy takes the original's name and fields; wire it to its own neighbors normally afterward. A copy of a copy resolves to the original. A screen copy also shows its original's design and marks, exactly, and setting a design through it sets the original's. To show the same page with a different part called out (a split screen, a screen showing a later state), place a plain screen of the same name with its own marked design instead.

A linked copy is allowed to diverge from its origin afterward in the specific field(s) a newly-connected event actually drives — e.g. the "Updating a Read Model That Already Feeds a Screen" copy above, where `status` is updated post-link to the event's new value. The link establishes "this is the same underlying fact, not a duplicate" at creation time; it doesn't freeze the copy's fields identical to the origin's forever. (Not possible here yet: a copy carries no fields of its own, so a divergent version is a plain read model of the same name, as noted above.)

**Scenarios (and storylines) are always added to the original, never to a copy.** A copy is the same fact reused, so its GWT spec lives once, in the origin's column. Before calling `set_scenario`, check whether the target is a copy (`get_cell_view` shows `originalStickyId`) and follow it to the original. `eventmodeling-elaborating-scenarios` carries the step-by-step.

**A copy is for wiring, never for a scenario step.** A scenario's given/when/then steps may name elements on any sheet (`Reset Password!B3.1`), so a Given event from another journey needs no copy on this sheet. Never place a copy, or a slice to hold one, only so a step can name it: it can't be wired to anything (an event flows into no command), so it reads as a broken slice. `check_model` reports a copy wired to nothing as `linked-copy-unwired`.

Never delete the original (the element that is not a copy) once copies of it exist — copies reference it, and removing it removes every copy too, with the flows touching them. `eventmodeling-checking-completeness` already treats any linked copy as an intentional copy, never a duplicate or missing-slice gap to flag.

## Arranging Sheets

All of a model's sheets sit on one canvas, and people read it like a map, so lay them out in the order the story is told. You never set coordinates: a new sheet goes under the existing ones, and `create_sheet` and `build_sheet_from_spec` take `below` / `above` / `rightOf` / `leftOf` (another sheet's name) to put it somewhere else. `move_sheet` rearranges existing sheets the same way. Sheets in the way move along, so placing one never covers another.

- Top to bottom is story order: each sheet goes `below` the one whose story comes before it. Creating sheets in story order gets this for free.
- Side by side (`rightOf`) is for journeys that run as alternatives: the divergent journeys one choice leads to sit next to each other.
- `get_model_view` shows each sheet's neighbours in `layout`. `check_model` reports `sheets-overlap` (a warning) when two sheets lie on top of each other; fix it with `move_sheet` before calling the work done.
- Sheets that belong to one bounded context (or one team's area) can be grouped in a named box with `set_context`. A box follows the sheets you name, never coordinates, and doesn't move when they do: after `move_sheet`, re-fit it with `set_context` and the same `context`.

## Connecting Sheets

`connect_sheets` draws an arrow from one sheet to another. It carries no fields and nothing `check_model` can verify, so it is the exception: most models need none, and a model with no sheet connectors is normal.

Draw one only where the actor's journey **branches** into another sheet's journey: a reader who reaches one particular slice here has to turn to the other sheet to follow what that actor does next (the divergent journey of `eventmodeling-brainstorming-events`). Anchor it on that slice (`fromColumn`) and, where you can name it, on the slice it lands on (`toColumn`). If you can't name the slice where the branch happens, don't draw it. Check `get_model_view`'s `sheetFlows` first so you don't draw it twice.

Never draw one:

- because one sheet comes after another in the list, or happens later in time;
- because a sheet's story simply continues another's: they are one sheet, so merge them;
- because two sheets share an event, read model, command or screen: that is a linked copy (`copy_element`);
- for a handover between systems or teams: the receiving sheet shows it with a linked copy of the event it starts from, or an external event and its translation chain;
- to quiet `sheet-starts-with-command`: a connector doesn't put a read on this sheet, a linked copy does;
- to make the model look connected.

## Translation Chain

An AUTOMATION reacting to an event from another system — external (a webhook/API) or second-swimlane (another team's own timeline) — needs a two-stage shape, never a direct reaction:

```
external EVENT → todo-list READMODEL → translation AUTOMATION + COMMAND → internal EVENT → worker automation's own todo list
```

- The external/second-swimlane EVENT opens only a *translation* automation's todo list — never a *worker* automation's todo list directly. (An `external-event` reaches that todo list by an `ExternalEventToReadModel` flow, an `event` in a second swimlane lane by `EventToReadModel`; `create_flow` infers the type.)
- A translation automation's own todo list is opened by that external event and is never closed — it doesn't track "done," it only relays.
- Its resulting COMMAND produces the internal EVENT the domain actually works with — that's the moment external data becomes a business fact in this domain's own language, ready for a worker automation (or anything else) to react to.

Every AUTOMATION, translation or worker, needs its own todo-list READMODEL (opened by triggering events, closed by completion events) — no "pure relay" exemption: `check_model` reports `automation-without-todo-list` for an automation with no `ReadModelToProcessor` flow into it, and `external-event-direct` (a warning, since legacy models use them) for an `ExternalEventToProcessor` or `ExternalEventToScreen` flow, which the method never uses. See `eventmodeling-designing-automation-chains` for the todo-list pattern and placement mechanics, and `eventmodeling-translating-external-events` for mapping the external payload's raw fields to the internal EVENT's domain fields (correlation, idempotency, duplicate handling).

## Open Questions vs. Decided Failures

Two different things are easy to conflate, and conflating them corrupts completeness checking:

- **Open question** — something genuinely undecided. Raise it as a pin (`create_pin`) on the relevant area, worded as a question. It stays open (unanswered) until the human actually answers it — resolving means answering, not deleting the pin.
- **Decided failure** — a rejection or error case whose behavior is already decided (e.g. "payment fails → reject, no event"). Model it as a SCENARIO whose Then is an expected failure (`{ errorType, errorMessage }`, see `eventmodeling-elaborating-scenarios`), never as a pin. It's permanent, specified behavior, not something waiting on an answer.

A decided failure left as a lingering, unanswered pin looks unresolved when it isn't. When counting completeness (`eventmodeling-checking-completeness`), only a genuinely unanswered question (`list_pins` with `status: "waiting"`) counts as a gap — a decided failure path needs its error scenario, not a pin standing in for one.

## Offline-First Thinking

Model the business process as it would work without any software first, then translate to elements:

- How would this work manually, with people, on paper?
- Who acts, and what triggers each action?
- What information do they need before they can act?

Question any step that exists only because of the system — a loading spinner, a cache refresh, a session check is not a business step and does not belong on the sheet. This is the discipline behind the Anti-Patterns below; the anti-patterns are what it looks like when this principle is skipped.

## Concrete Over Generic

Looking at the model, a reader must understand how the business works. Model one real flow with real examples and real data, not a generic model that covers every case. This applies to generic platforms too: model one real case the platform handles, not the platform's own structure.

Example (a template-driven health record system):
- ❌ `Commit Composition` with fields `content`, or `name`/`category`/`entries` — still says nothing about the business.
- ✅ `Record Blood Pressure` → `Blood Pressure Recorded` with `systolic: 132`, `diastolic: 92`, `pulse: 113`, `composer: Dr. Marcus Johnson`.

## Anti-Patterns

- **Data-loading commands**: `Load Orders`/`Fetch Data` are queries, not business intent — model as a READMODEL, not a COMMAND.
- **UI-interaction events**: `Sidebar Opened`/`Button Clicked` are UI state, not business facts — don't model them unless the business genuinely cares.
- **Technical events**: `Api Called`/`Response Received` describe machinery, not an outcome — find the business fact underneath.
- **Calculated events**: an event whose value is recomputed as source data changes (a running total, an average) is a READMODEL, not an EVENT.

## Structural Shapes

Beyond the local anti-patterns above, four recurring **connection shapes** are worth checking whenever the model's wiring is visible — a fan-out or fan-in count crossing a rough threshold. Only one is a real anti-pattern regardless of context; the other three are **candidates** — worth checking against the business context, not automatically wrong. Reason about the actual events/fields/scenarios involved before flagging a candidate; if the domain justifies the count, it's not a violation. These are internal shorthand names for spotting the shape — never surface "bed", "left chair", "right chair", or "shelf" in anything shown to a business stakeholder (a note, a pin, a report); describe the concern in plain terms instead.

- **The bed (real anti-pattern — always flag)** — one SCREEN wired to more than one COMMAND (`check_model` reports `screen-multiple-commands`; a screen and its linked copies count as one screen). A screen is where the user has already committed to one decision, so it should trigger exactly one command; wiring several to it means the choice is being made somewhere invisible to the model.
- **The left chair (candidate)** — one COMMAND resulting in more than two EVENTs (`check_model` reports `command-fan-out`, info). May mean the command is doing more than one job, or it may be a single business outcome that legitimately fans out. Check whether the outcomes always happen together or could happen independently before treating it as a concern.
- **The right chair (candidate)** — one READMODEL built from more than three EVENTs (`check_model` reports `read-model-fan-in`, info). May mean the view is answering more than one question at once, or it may be one coherent picture that genuinely needs that many sources. Check whether the fields shown belong to a single thing the user is checking before treating it as a concern.
- **The shelf (candidate)** — one slice with noticeably more SCENARIOs than the others on the same sheet (a rough outlier, not a fixed threshold — compare against the typical count for that sheet; `check_model` reports `slice-scenario-outlier`, info, from five scenarios and above twice the sheet's median). May mean the step is quietly covering ground that belongs to a separate step, or it may just be genuinely more complex. Check what the extra scenarios actually cover before treating it as a concern.

## Flow & Causality

Model causality, not strict sequence:

- A sheet starts with a **state-view** (READMODEL feeding a SCREEN) or an **automation** reacting to an EVENT already on the sheet — never with an unmotivated COMMAND. `check_model` reports `sheet-starts-with-command` (a warning) when the first slice is a state change whose screen shows no read model. If the read the user acts on lives on another sheet, place a linked copy of it here; a sheet connector doesn't count. A genuinely blank creation form is the one case that may start a sheet; it still gets the warning, so say so in a note.
- Every COMMAND must be traceable to a trigger: a SCREEN (a user decision) or an AUTOMATION (a system reaction): `command-without-issuer`.
- A `state-change` slice must not follow another `state-change` slice without a trigger in between (a new SCREEN or AUTOMATION) — two COMMANDs in a row with nothing issuing the second one is a gap, not a shortcut. `check_model` reports `state-change-without-trigger` where the second command has an issuer, but every issuer sits at or before the previous slice; a command with no issuer at all is `command-without-issuer`, and one screen wired to both commands is `screen-multiple-commands`.

Valid transitions between slices:
- state-view → state-change
- state-change → state-view
- state-change → automation
- automation → state-change
- automation → state-view

## Quick Self-Check

Before treating a model as done, run `check_model` for the mechanical pass (the items below that name a code are answered by it), then verify the rest yourself:

- [ ] Every EVENT is past tense and describes a business fact, not machinery or UI state
- [ ] Every COMMAND is imperative business intent, not a data-loading query
- [ ] Every READMODEL is named for its data, not its implementation
- [ ] No EVENT holds a computed/aggregated value — that belongs in a READMODEL
- [ ] Every COMMAND has exactly one issuer (one inbound SCREEN or AUTOMATION edge): `command-without-issuer`, `command-multiple-issuers`
- [ ] Every COMMAND traces back to a SCREEN (user decision) or AUTOMATION (system reaction)
- [ ] The sheet starts with a state-view or an automation reacting to an event — not a bare COMMAND: `sheet-starts-with-command`
- [ ] No two `state-change`/`state-change` slices are chained without a new trigger between them: `state-change-without-trigger`
- [ ] No SCREEN is wired to more than one COMMAND (the bed — see Structural Shapes): `screen-multiple-commands`
- [ ] Every connection stays in its column or runs forward: `flow-backward`; one EVENT per column: `column-multiple-events`; every slice is one subject: `slice-mixed-shape`; every AUTOMATION reads a todo list: `automation-without-todo-list`
- [ ] The model follows one real flow with real data, not a generic one (see Concrete Over Generic)

This is the fast pass — the full, deeper check is `eventmodeling-validating-event-models-checklist`.
