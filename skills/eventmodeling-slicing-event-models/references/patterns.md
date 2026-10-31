# Event Modeling Slice Patterns

## Table of Contents
- [Slice Definition Template](#slice-definition-template)
- [Slice Shapes Across the Lanes](#slice-shapes-across-the-lanes)
- [Cross-Slice Communication Patterns](#cross-slice-communication-patterns)
- [Tips for Effective Slicing](#tips-for-effective-slicing)

---

## Slice Definition Template

Use this template for each slice:

```
Slice: [Command name | Read model name | Automation name]
Type: state-change | state-view | automation

COMMAND (state-change/automation only): [name] — what it does
READMODEL (state-view only): [name] — what it shows

EVENTS PRODUCED:
  - [Event]: when it occurs

EVENTS CONSUMED:
  - [Event from another slice]: why needed

UPSTREAM DEPENDENCIES:
  - [Slice X]: needed because it produces [Event]

DOWNSTREAM DEPENDENTS:
  - [Slice Y]: consumes [Event] we produce
```

The dependency lines are what goes into a slice's note (`set_note` on the slice).

---

## Slice Shapes Across the Lanes

A slice's columns hold one of these shapes. Lanes are the sheet's typed rows: actor (screens, processors), interaction (commands, read models), swimlane (events, external events), specification (scenarios).

| Slice | Actor lane | Interaction lane | Swimlane lane |
|---|---|---|---|
| state-change | Screen or Automation | Command | Event(s) it produces |
| state-view | Screen (or Automation) it feeds | Read Model | Event(s) it folds, as linked copies of the originals |
| automation | Automation | Command it issues | Event(s) it produces |

Specification lane: the scenarios of the slice's one command or read model.

**The automation chain** spans three slices, left to right, never one:

```
[External Event]  →  [Todo-list Read Model]  →  [Automation + Command + Event]
 (feeds the next      state-view slice            automation slice
  slice's column)
```

The external event's column has no subject of its own, so it sits inside the todo-list read model's slice (it is that read model's source). The todo-list read model is a state-view slice; the automation with its command and internal event is an automation slice.

**A column that only repeats something** (a linked copy of a read model that feeds a screen, say) has no subject and implies no slice: it sits inside the slice it serves.

---

## Cross-Slice Communication Patterns

### Pattern 1: Command Slice → Event → Command Slice

```
Slice: Place Order (state-change)
  produces Order Placed
       ↓
Slice: Confirm Order (state-change)
  its command validates against Order Placed (a linked copy of it in this slice)
```

Advantages: loose coupling, asynchronous, each slice owns its own state — no shared state between them. The second command still has its own trigger (a screen or an automation in its own slice): two commands in a row with nothing issuing the second is a gap, not a shortcut.

### Pattern 2: Command Slice → Event → Read Model Slice

```
Slice: Place Order (state-change)
  produces Order Placed
       ↓
Slice: Order Detail (state-view)
  folds Order Placed into its read model
```

This is the most common dependency: nearly every state-view slice depends on the events produced by one or more state-change slices. The state-view slice can be developed in parallel, but its projections only produce data once the upstream events exist.

---

## Tips for Effective Slicing

### 1. One Element Per Slice

See the main SKILL.md's "Core Concept" section — never combine a COMMAND and a READMODEL into one slice, even under an inviting broader "feature" name.

### 2. Name the Slice After Its Element

Use the exact command, read model, or automation name as the slice name — don't invent a broader "feature" name that spans multiple elements.

### 3. Express Dependencies as Events, Never as Shared State

Slices never call each other directly and never share state. If slice B needs something from slice A, it's because slice B's command or read model builds on an event that slice A produces — not because they're coupled.

### 4. Split Where the Second Subject Begins

A multi-column slice with two subjects is split at the column where the second subject starts (`split_slice`), and the new slice is renamed straight away. A single column holding two subjects can't be split — give the second subject a new slice (`insert_slice`) and move it there (`move_element`).
