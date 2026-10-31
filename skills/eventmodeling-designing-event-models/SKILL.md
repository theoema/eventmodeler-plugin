---
name: eventmodeling-designing-event-models
description: "Designs the event model for a business process — maps Commands, Events, and Read Models with clear causality. Events are business facts; Read Models are projections built from them. Use when designing the Command/Event/Read-Model structure from domain analysis. Do not use for: brainstorming events from scratch (use eventmodeling-brainstorming-events), or checking whether an event belongs to the right entity/timeline (use eventmodeling-optimizing-stream-design)."
---

# Designing Event Models

This step applies the shared element rules in **`eventmodeling-core-rules`**. Read it once per session if you haven't already: it defines what a COMMAND/EVENT/READMODEL/SCREEN/AUTOMATION is, how each is named, and the anti-patterns to reject, so this step doesn't restate them. Its `../eventmodeling-core-rules/references/tools.md` maps each concept to the tool that does it.

## Purpose

Converts domain analysis into the event modeling structure:

**UI/Processor** → **Command** → **Event** → **Read Model** → **UI/Processor**

- **UI/Processor**: entry points that trigger intent
- **Command**: business intent, expressed in business language, can be rejected
- **Event**: a business fact, the result of a successful command
- **Read Model**: a projection of events, shaped for a specific query need

The sheet is the artifact. Each part of the design below has a place on it: events, commands and read models are elements with fields; causality is the flows and the column order; preconditions are a note on the command's cell; rejections and state transitions are scenarios. Present the design to the user in the conversation in the shapes below, and build it on the sheet.

## Workflow

Given a domain analysis, design the event model.

### 1. Design Events

Events are business facts, not technical occurrences:
- **Event Type**: what changed, in past tense business language (Created, Confirmed, Shipped)
- **Event Data**: the business fields, noting which came directly from the triggering command versus which are implicit context (e.g. the entity's identity)
- **Causality**: which command triggered it

Document each entity's events as a chronological list, giving each event's triggering command and its data. A full worked example (an Order) is in `references/examples.md`.

**Key Rules**:
- Events are immutable business facts
- Events use past tense, business language
- Event order matters. It tells the story
- Never modify or delete an event once it exists
- Events only exist if the triggering command succeeded
- Event data is captured, never calculated. A total, count or average that is derived from other data is `isGenerated` on the read model that shows it (`eventmodeling-information-completeness`)

On the sheet: each event is an `event` in a `swimlane` lane, left to right in the order it happens, one event per column (`check_model` reports `column-multiple-events`).

### 2. Design Commands

Commands are business intent from a UI or Processor:
- Represent what a user or system wants to do
- Can be rejected (validation failure)
- Only a UI or Processor can issue a command
- Produce events if valid, or reject if invalid

Document each command's source, its input fields, the preconditions it is checked against, the event(s) it produces on success (noting which data comes from the command versus implicit context), and its rejection outcomes. A full worked example (Confirm Order) is in `references/examples.md`.

**Key Rules**:
- Only a UI or Processor can issue commands (entry points)
- One command per UI/Processor action
- Successful command → Event(s) created
- Failed validation → Command rejected, no event

On the sheet: the command is a `command` in the `interaction` lane of the column of the event it produces, with its input as fields. The preconditions are a note on the command's cell, listing the events its decision depends on (`set_note`). The rejection outcomes are scenarios whose Then is an expected failure (`eventmodeling-elaborating-scenarios`).

### 3. Design Read Models

Read models are projections of events, built for UI/Processor queries:
- Built from events (their only source)
- Optimized for a specific query pattern
- Consumed by UI (for display) or Processor (for a decision)
- Can be regenerated from events at any time

Document each read model's purpose, which events it subscribes to, its query-shaped data, how each subscribed event updates that data, and who consumes it. A full worked example (Order Summary View) is in `references/examples.md`.

On the sheet: a `readmodel` in the `interaction` lane of the column immediately after its source event, fed by `EventToReadModel` flows and feeding a screen or an automation. See `eventmodeling-identifying-outputs` and `eventmodeling-designing-automation-chains` for placement.

### 4. Document Event Causality

Trace the causal chain from the first command through the events and downstream commands it can trigger, showing how a later command's decision depends on events produced earlier. A full worked example (an Order create→confirm→ship chain) is in `references/examples.md`.

On the sheet, this is the left-to-right column order plus the flows: `ScreenToCommand`, `CommandToEvent`, `EventToReadModel`, `ReadModelToScreen`/`ReadModelToProcessor`, `ProcessorToCommand`.

### 5. Document State Transitions

Map the valid states a process can be in, which command causes which transition, and which transitions are invalid from a given state (e.g. cancelling an already-shipped order), ending at any terminal states. A full worked example (Order transitions) is in `references/examples.md`.

On the sheet: a valid transition is a scenario (Given the prior events, When the command, Then the new event). An invalid transition is a scenario whose Then is an expected failure.

### Output Format

Present the complete design in the conversation, with sections for Events (per entity, chronological, each with its triggering command), Commands (input, preconditions, events produced, rejection outcomes), and Read Models (purpose, subscribed events, consumers). A full worked example of this structure is in `references/examples.md`. It is a summary of what is on the sheet, not a replacement for it, and not a file.

## Key Principles

See `eventmodeling-core-rules` for the element definitions and causality rules this design must follow (events are immutable facts, commands are checked against documented preconditions, read models never drive command validation).

## Quality Checklist

- [ ] All events are immutable business facts (past tense)
- [ ] Events contain only captured business data, no computed fields
- [ ] Each command's preconditions are documented explicitly
- [ ] Commands either produce events or are rejected. No silent failures
- [ ] Event causality is clear and traceable end to end
- [ ] State transitions are documented, including which ones are invalid and why
- [ ] No read model is needed for a command's own validation; screens and automations get the read models their own rules require
- [ ] The sheet reflects the design: events in `swimlane`, commands and read models in `interaction`, flows wired, `check_model` clean
