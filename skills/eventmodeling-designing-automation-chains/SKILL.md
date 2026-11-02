---
name: eventmodeling-designing-automation-chains
description: "Step 4b of Event Modeling - design every automation's todo-list read model and, for externally-triggered automations, its two-stage translation chain (external EVENT to internal EVENT). Runs immediately after eventmodeling-identifying-inputs places an AUTOMATION+COMMAND pair, before eventmodeling-identifying-outputs. Use right after Step 4 places any AUTOMATION. Do not use for: screen-facing read models (use eventmodeling-identifying-outputs), placing the automation's own COMMAND (use eventmodeling-identifying-inputs), or system/team boundary swimlanes (use eventmodeling-applying-conways-law)."
---

# Designing Automation Chains

This step applies the shared element rules in **`eventmodeling-core-rules`** — read it once per session if you haven't already; it defines what a COMMAND/EVENT/READMODEL/SCREEN/AUTOMATION is, how each is named, and the anti-patterns to reject, so this step doesn't restate them.

Every tool call carries `modelId` (from `list_models`); `sheet` may be omitted only when the model has exactly one sheet. Positions are cell addresses (`C2`: column letter, lane number), never coordinates. The tool for each concept below is in [tools](../eventmodeling-core-rules/references/tools.md).

This is **Step 4b**. It runs immediately after `eventmodeling-identifying-inputs` (Step 4) places an AUTOMATION and its COMMAND, and before `eventmodeling-identifying-outputs` (Step 5) designs any screen-facing read model. Designing an automation's todo list — and, when it applies, its translation chain — while the automation is still fresh from Step 4 is what prevents the alternative: discovering the gap only once Step 5's own verification pass runs, after commands and flows already built on top of the incomplete shape.

**Input**: Every AUTOMATION placed in Step 4, each already paired with its own COMMAND in the actor/interaction lanes of one column (per `eventmodeling-orchestrating-event-modeling`'s Timeline Alignment Rules).
**Output to carry forward**: Every automation's todo-list READMODEL, placed and wired; every externally-triggered automation resolved into a two-stage translation chain (external EVENT → todo-list READMODEL → translation AUTOMATION+COMMAND+internal EVENT → worker automation's own todo list).
**Gate**: No AUTOMATION on the sheet lacks an incoming `READMODEL → AUTOMATION` flow from a todo-list read model, and no automation's todo list is opened directly by another system's (second-swimlane) event.

---

## Every automation needs a todo-list read model — no exemption

**Every AUTOMATION needs a todo-list read model — this is not optional, and there is no "pure relay" exemption.** An automation is a processor: it reacts to events, decides what work is outstanding, and issues commands to get that work done. The read model that tells it what's outstanding is its **todo list** — a queue of pending work items, not a snapshot of current entity state.

**The pattern**: one or more events *open* an item on the todo list (something now needs doing); one or more events *close* it (the work is done — remove the item). A todo list can be opened and closed by more than one event type on either side, and the set of opening events doesn't need to match the set of closing events in count or shape — whatever the domain calls for.

**Worked example**: an automation that reacts to `Customer Registered` by sending a welcome notification.
- Todo list: **Notifications To Send** — one row per pending notification.
- `Customer Registered` **opens** a row (a notification now needs sending).
- `Notification Sent` (the event this automation's own resulting command produces) **closes** that row (removes it — the list only ever shows outstanding work).
- The automation (`Send Welcome Notification`) reads `Notifications To Send`, and for every open row issues `Send Notification`.
- In Step 7 that open → close lifecycle is specified as one storyline on the todo list (empty → `Customer Registered` → its row → `Notification Sent` → empty), see `eventmodeling-elaborating-scenarios`; `check_model` reports a todo list without one as `todo-list-without-storyline`.

Even an automation that looks like a "pure signal relay" still has a todo list — model it anyway: it documents that the automation is idempotent/complete once its own event fires, and keeps every automation consistent with the same `READMODEL → AUTOMATION → COMMAND → EVENT` pattern instead of silently exempting some as "too simple to need one."

## The translation-chain rule for external triggers

**There is no such thing as an invisible or informal "signal" — a trigger is always a real EVENT**, placed in a second swimlane when it belongs to another system (see `eventmodeling-brainstorming-events`'s swimlane rule).

**An automation can only ever be directly triggered by an internal event — never by another system's event.** A "trigger" arriving from a second swimlane is not itself the thing that drives your domain's work; it first has to be *translated* into an internal event. Do not model this as one automation whose todo list is opened by the external EVENT and that also does the real work (e.g. an automation reading a todo list opened by `Reservation Requested` from another system's swimlane and directly issuing `Reserve Copy`) — that lets an external system trigger domain work with no translation step, which this model doesn't allow. **Never connect an external event straight to a processor** — the `ExternalEventToProcessor` flow type exists, and this step forbids using it. **Apply this now, in Step 4b** — Conway's Law (Step 6) only confirms the boundary, it doesn't introduce the chain. Every integration trigger from another team's system needs a **translation automation** — that part is never optional:

1. **Translation automation** — converts the external fact into an internal one; the other system's own decision logic is out of scope. Its todo list is opened by the external EVENT — the one and only place an external EVENT may open a todo list. **It is not closed by the internal EVENT its own command produces, and gets no backward arrow.** A translation automation's job is an instantaneous, always-succeeding relay: there is no real work-in-progress window between "signal arrived" and "internal fact recorded" worth modeling as an open/close lifecycle, unlike a worker automation that can genuinely have pending items. This read model exists only to satisfy "every automation reads from a todo list," not to accumulate and drain a queue — so it stays open-ended: fed by the external EVENT, read by the automation, never closed.
   - **Three separate columns**, left to right: `[external EVENT] → [todo-list READMODEL] → [AUTOMATION + COMMAND + internal EVENT]`. Never crammed into one or two — the "one EVENT per column" rule applies here too.
   - **The external EVENT is an `external-event` element in the other system's swimlane** (or an `event` in a second swimlane lane named for that system). It reaches the todo-list read model by an `ExternalEventToReadModel` flow (`EventToReadModel` for an `event`); `create_flow` infers the type.
   - **Name the internal EVENT for its business meaning, not the transport** — usually the same name as the external EVENT (e.g. external `Copy Reserved` → internal `Copy Reserved`; the swimlane already shows which is which), never a mechanical `<X> Signal Received`/`<X> Request Received`/`<X> Synced` suffix. Same for the automation/command: `Record Reservation`, not `Record Reservation Signal`. The internal EVENT is a genuine new domain fact recorded in this side's own language — place it as its own `event` element in this sheet's own swimlane lane, never as a linked copy of the external one. Since the two events usually share a name, always refer to them by address (`B3.1`), never by name — a name matches both.
   - Its command and event carry no business decision — they only exist to produce the internal fact a worker automation would need, *if one is even needed* (see below).

### Does a worker automation come after it, or does the chain stop there?

The translation automation never decides anything — it only records a fact. Whether something needs to happen *after* that fact depends on whether there is still a decision left to make:

- **Build a worker automation only when reacting to the internal EVENT requires a genuinely new decision on this side** — an invariant to check, a choice to make, data to compute that the internal EVENT doesn't already carry. That decision is what earns its own COMMAND, which is what earns the automation that issues it (see item 2 below for its shape).
- **Do not build one when the internal EVENT already *is* the business-relevant fact.** If the external system already made every decision involved, and the "next" EVENT you were about to model would just restate data the internal EVENT already carries under a different name (same identity, no new field, no invariant checked, no choice made) — stop after the translation automation. That is not a second stage of the same chain; it is no chain at all, just a fact recorded once. Anything else that needs this fact reads the internal EVENT directly through a state-view READMODEL projection — it does not get an automated command reaction bolted on solely to keep a fixed "two automations" shape.

  A `<X> Synced` event feeding a second translation on top of an already-decided fact is the most common shape of this violation — the "Synced" name is itself the tell that no new decision was made.

  *Example*: an external `Reservation Placed` opens a translation chain and already carries `reservationId`, `copyId`, and `memberId` — the copy to reserve was chosen upstream, by the other system. Translating it into an internal `Reservation Placed` is still required (per the rule above — an automation may never react to the external event directly). But adding a second `Reservations To Apply` todo list + `Reserve Copy` automation + `Copy Reserved` event on top of it is redundant: `Copy Reserved` doesn't decide anything `Reservation Placed` didn't already decide, it's the same fact under a different name. A read model showing "reserved copies" should project directly off the internal `Reservation Placed` event, not off a manufactured second automation stage.

2. **Worker automation** (only when the check above says one is needed) — the one that does the actual work (a real domain decision the process is about, e.g. checking copy availability before issuing `Reserve Copy`). Its todo list is opened **only** by the internal EVENT the translation automation produced (this sheet's own swimlane) — never by the external EVENT directly — and closed by whatever event marks that work done. Unlike the translation stage, the worker's todo list keeps its normal open/close accumulator shape, because real pending work can sit there (it's the point where actual domain decisions happen).

**Two chained automations only when there genuinely are two decisions** — the translation (none, by definition) and the worker (a real one). Whether a worker stage exists at all is a judgment call driven by whether that second decision actually exists, not a fixed shape to stamp onto every cross-swimlane trigger.

When a worker automation *is* warranted, wire the todo lists differently for each automation in the chain: for the translation automation, the external EVENT (second swimlane) **opens** the row and nothing closes it; for the worker automation, the internal EVENT **opens** its row and its own resulting EVENT **closes** it (backward arrow, per the exemption below). `EVENT → READMODEL` flows from both swimlanes are unaffected by which swimlane the event sits in.

The shape of a complete chain, one column per step:

```
B  [other system's swimlane lane]   Copy Reserved                 (external-event)
        │ ExternalEventToReadModel
C  [interaction lane]               Reservations To Record        (readmodel, list-shaped)
        │ ReadModelToProcessor
D  [actor lane]       Reservation Recorder   (processor)
        │ ProcessorToCommand
   [interaction lane] Record Reservation     (command)
        │ CommandToEvent
   [own swimlane lane] Copy Reserved         (event — the internal fact)
```

## Fields on a todo-list read model

A todo-list read model's fields describe the pending item — the identity it's about (e.g. `customerId`) plus enough context to act on it (e.g. `email`, `notificationType`). Do not add a `status` field to mark items done — a todo list's "open" state is *membership in the list itself* (the row exists at all), not a status flag on a row that never leaves. If the same underlying data is also useful with an explicit status column for a different consumer, that is a different read model, not this one.

Every field needs a known source per `eventmodeling-information-completeness`: declare it as you place the read model with `source: { from, field? }` on the field (`from` = the opening event's address or name on the sheet), or with `isGenerated: true` where the information starts. A field that genuinely holds a list of values is `isList: true`. The field shape is exactly `{name, fieldType, isList?, isGenerated?, isOptional?, isUserInput?, isPii?, example?, subfields?}`.

The fields continue down the chain: the automation carries the fields it passes on (each with `source` from the todo-list read model), its command's fields take their `source` from the automation, and the resulting event's from the command. A processor with no fields leaves every field of its command untraceable. Because each `source` creates its flow and mapping, placing the elements left to right (read model, automation, command, event) with their field sources wires `ReadModelToProcessor`, `ProcessorToCommand` and `CommandToEvent` as a side effect.

A todo-list read model is always list-shaped — it's a queue of pending items, never a single record — so it always gets **`isList: true`** on the read model itself (`place_element` / `update_element`; see `eventmodeling-core-rules`'s READMODEL section). This is a flag on the read model itself, separate from any field's own `isList`.

## Placement — todo-list READMODEL, one column before its automation

The todo-list read model goes in the interaction lane, **one column before** its automation (actor lane) — the automation's own column already holds the COMMAND it issues, so the read model can never share that column.

A cell holds one element, so look first: `get_sheet_view` shows what the column before the automation holds in its interaction lane. If it's occupied, or no column exists there, insert one immediately before the automation's (`insert_slice`, `at` = the automation's 1-based column position, `cols: 1`, named for the todo list; the automation and everything after it shift one column right). Then `place_element` puts the read model in the interaction-lane cell, with its fields and `isList: true` in the same call:

```
place_element {
  "cell": "C2",
  "element": {
    "scope": "readmodel",
    "name": "Notifications To Send",
    "isList": true,
    "fields": [...]
  }
}
```

Placement never connects anything (apart from the flows a field `source` declares), so wire the chain yourself in the flows below and confirm it with `check_model`. Addresses shift after every column insertion: re-read the sheet (`get_sheet_view`) before using another address.

**A column cannot exist outside a slice.** Every column you insert comes with its own slice (`insert_slice`, one column): name it for what the column holds — the todo-list read model, the automation. A slice is one command, one read model or one automation, never a command and a read model together (see `eventmodeling-core-rules`, "Slices") — so the todo-list column and the automation's own column are separate slices. `eventmodeling-slicing-event-models` reviews the final boundaries.

For a **translation-chain automation**, place its three columns left to right in the same pass: `[external EVENT]`, `[todo-list READMODEL]`, `[AUTOMATION + COMMAND + internal EVENT]` — the external EVENT normally already exists (placed in Step 1/brainstorming, second swimlane); if it doesn't, place it there first. **Insert these columns immediately after the external EVENT's own column** (`insert_slice` with `at` = that column's position + 1) — never at the sheet's start. A translation chain lives exactly where its trigger already sits in the narrative; being an "input" is not a reason to relocate it ahead of everything else. A whole chain is one atomic unit; `apply_operations` can run its insertions, placements and flows as one all-or-nothing batch if you prefer that to single calls.

## Wiring flows

All of these are `create_flow` calls (`from` and `to` are element addresses; the type is inferred from the pair: `EventToReadModel`, or `ExternalEventToReadModel` from an `external-event`).

1. **READMODEL → AUTOMATION first** — the automation reads its own todo list. The closing flow below is only legitimate once this one exists (`check_model` otherwise reports it as `flow-backward`).
   ```
   create_flow { "from": "<readmodel address>", "to": "<automation address>" }
   ```
2. **Every opening EVENT → READMODEL.**
   ```
   create_flow { "from": "<opening event address>", "to": "<readmodel address>" }
   ```
3. **Every closing EVENT → READMODEL — worker-stage todo lists only.** Including the automation's own resulting event, even though that event is produced by the command this same automation issues. This is not a backward arrow: `EVENT → READMODEL` flows are exempt from column ordering (`check_model` exempts them from `flow-backward`) when the read model already has a `READMODEL → AUTOMATION` flow — this todo-list read model qualifies because of the flow from step 1 above. A read model in this shape is a live projection, not a frozen snapshot — a later event closing an earlier-opened item is the normal case, not an exception to reach for only when convenient. **Do not add this flow for a translation automation's todo list** — it has no closing event at all (see the translation-chain rule above); wiring one back is a modeling error, not a convenience.
4. **Down the chain**: `ProcessorToCommand` from the automation to its command, `CommandToEvent` from the command to its event (Step 4 usually made these).

Where a source and target field have different names, state the correspondence with `set_flow_mapping` (by field name). Field lineage across the whole chain follows `eventmodeling-information-completeness`; `check_model`'s `untraceable-field` warnings show where a hop is missing a mapping.

Skip a flow silently if the target cell is empty (the element may not exist yet). Log each created arrow, e.g. `→ connected READMODEL→AUTOMATION "Notifications To Send"→"Send Welcome Notification"`.

If this step is designing more than one automation's chain in the same pass, keep each automation's own three-flow sequence intact (READMODEL→AUTOMATION before its closing EVENT→READMODEL); different automations' triples can be interleaved or concatenated freely since they don't depend on each other.

## Verification (run before moving to Step 5)

Run `check_model` first — it reports `orphan-element` and `read-model-without-source` for chain elements left unwired, `command-without-event` for a chain whose command produces nothing, `untraceable-field` for fields no flow supplies, `automation-without-todo-list` for an automation no todo-list read model feeds, `flow-backward` for a flow that runs right to left (a worker todo list's closing `EventToReadModel` is exempt), `command-without-issuer` and `command-multiple-issuers`, and `external-event-direct` for a flow straight from an external event into an automation. It cannot tell a translation automation from a worker, see who opens or closes a todo list, or judge whether a stage is redundant, so read the sheet (`get_sheet_view` lists every element with its address and every flow with its endpoints) and, for every AUTOMATION on it:

1. Does it have an incoming `READMODEL → AUTOMATION` flow from a todo-list read model? An AUTOMATION is **never** exempt — if not, design and wire it now.
2. Is its todo list opened by an internal event only, **unless it is itself a translation automation** (which is the one case a second-swimlane event may open a todo list)? If a worker automation's todo list is opened directly by another system's event, that automation is missing its translation chain — split it into translation + worker automations per the rule above. Also confirm no external event has a flow straight to a processor (`external-event-direct`).
3. If it's a **translation** automation, does its todo list have no closing edge — i.e. no flow from its own resulting internal EVENT back to its own todo-list read model? If one exists, remove it (`remove_flow`): a translation automation's todo list is never closed.
4. If it's a **worker** automation sitting right after a translation automation, does its own COMMAND make a genuinely new decision the internal EVENT didn't already carry? If its fields and logic just restate the internal EVENT's own data under a new name, it's a redundant stage — remove the worker automation, its todo list, and its resulting event (`remove_target`), and have downstream consumers project a state-view READMODEL directly off the internal EVENT instead.
5. If it's a real worker automation, does its todo list have a proper closing edge from its own resulting event? If not, add it.
6. Does the todo-list read model's field set avoid a `status` flag (membership in the list is the state)?
7. Does every flow point forward? `check_model` reports a flow whose source column is later than its target's as `flow-backward`; the only allowed one is a worker todo list's closing `EventToReadModel`, which it exempts. It cannot tell a translation todo list's wrong closing flow from a worker's right one, which is item 3.
8. Is each command issued by only its own automation (exactly one issuer)? `check_model` reports `command-multiple-issuers` otherwise.

List the result (connected / chain-resolved) for every automation checked — this is the evidence that `eventmodeling-identifying-outputs`'s later per-node verification pass (which re-checks automations defensively) finds nothing left to do here.

## Quality Checklist

- [ ] Every AUTOMATION has an incoming `READMODEL → AUTOMATION` flow from a todo-list read model — no exemption, even for a simple relay
- [ ] No automation's todo list is opened directly by another system's (second-swimlane) EVENT unless that automation is itself the translation automation
- [ ] No external event has a flow straight to a processor
- [ ] Every externally-triggered automation has a translation automation ahead of it — never triggered by the external EVENT directly
- [ ] A worker automation exists after the translation stage only when it makes a genuinely new decision — not merely to keep a fixed "two automations" shape when the internal EVENT already is the business-relevant fact
- [ ] Every **worker**-stage todo-list read model's opening and closing events are identified, including the automation's own resulting event as a closing event
- [ ] Every **translation**-stage todo-list read model has an opening event (the external EVENT) and **no closing event at all** — no backward arrow from its own internal EVENT back to its own todo list
- [ ] No todo-list read model uses a `status` field instead of list membership
- [ ] Every todo-list read model carries `isList: true` on the read model
- [ ] Every todo-list read model sits one column before its automation, never sharing its column
