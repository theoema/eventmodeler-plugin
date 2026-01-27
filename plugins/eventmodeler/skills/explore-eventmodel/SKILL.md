---
name: explore-eventmodel
description: How to explore and navigate an event model using the eventmodeler CLI. Use when you need to understand the structure of an event model, find specific elements, or get an overview.
---

# Exploring Event Models with the CLI

The `eventmodeler` CLI lets you query and explore `.eventmodel` files. It auto-detects the file in the current directory, or use `-f <path>` to specify one.

## Getting an Overview

**Start here** to understand the model:

```bash
eventmodeler summary
```
Shows statistics: total slices, events, commands, read models, etc.

```bash
eventmodeler list slices
```
Lists all slices with their status (created, in-progress, blocked, done). Slices represent vertical features.

```bash
eventmodeler list slices --chapter "<name>"
```
Lists only slices within a specific chapter.

```bash
eventmodeler list chapters
```
Lists chapters - timeline sections that group slices.

## Listing Elements

```bash
eventmodeler list events      # All events in the model
eventmodeler list commands    # All commands in the model
eventmodeler list aggregates  # All aggregates (event groupings)
eventmodeler list actors      # All actors (user/system roles)
```

## Searching

```bash
eventmodeler search "<term>"
```
Searches across all entity names: slices, events, commands, read models, aggregates, and actors.

## Drilling Into Details

### Show a Slice
```bash
eventmodeler show slice "<name>"
```
Returns XML with:
- All components in the slice (screens, commands, events, read-models, processors)
- **Inline flow annotations** on each component showing `<flows-from>` and `<flows-to>` connections
- Fields on each component with types and attributes
- Information flow (which components connect to which)
- **Inbound flows**: External elements flowing INTO this slice (with field mappings)
- **Outbound flows**: Elements in this slice flowing OUT to other slices (with field mappings)
- Scenarios (Given-When-Then test cases)

The inbound/outbound flows show cross-slice dependencies - crucial for understanding how a slice integrates with the rest of the system.

**Scenarios are key** - they define the slice's behavior and give it context. A slice's scenarios show:
- **Given**: What events have already occurred (the starting state)
- **When**: What command is being executed with what field values
- **Then**: The expected outcome - either:
  - Events that should be produced
  - An error that should occur
  - A read model assertion (expected state)

Example scenario in slice output:
```xml
<scenarios>
  <scenario name="Successfully place order">
    <given>
      <event type="CustomerRegistered">customerId: "123"</event>
    </given>
    <when>
      <command type="PlaceOrder">customerId: "123", items: [...]</command>
    </when>
    <then>
      <event type="OrderPlaced">orderId: "456", customerId: "123"</event>
    </then>
  </scenario>
  <scenario name="Cannot place order without items">
    <when>
      <command type="PlaceOrder">customerId: "123", items: []</command>
    </when>
    <then>
      <error type="ValidationError">Order must contain at least one item</error>
    </then>
  </scenario>
</scenarios>
```

Reading scenarios helps understand:
- What preconditions the slice expects
- What inputs the command accepts
- What the happy path looks like
- What error cases are handled

### Show an Event or Command
```bash
eventmodeler show event "<name>"
eventmodeler show command "<name>"
```
Returns XML with the element's fields and connections.

### Show a Chapter
```bash
eventmodeler show chapter "<name>"
```
Shows the chapter with:
- All slices within the chapter
- **Flow graph**: How slices connect to each other (which events flow to which read models, etc.) with field mappings
- **External dependencies**:
  - Inbound: Flows from outside the chapter into slices within it
  - Outbound: Flows from slices in this chapter to elements outside it

Use `show chapter` to understand the relationships between slices in a feature area. Use `list slices --chapter` for a simple list without flow details.

### Show an Actor
```bash
eventmodeler show actor "<name>"
```
Shows the actor and lists all screens associated with it.

## Managing Slice Status

Slices have a status that tracks their progress: `created`, `in-progress`, `blocked`, `done`.

```bash
# Mark a slice as in progress
eventmodeler mark "Place Order" in-progress

# Mark a slice as done
eventmodeler mark "Place Order" done

# Mark a slice as blocked
eventmodeler mark "Place Order" blocked

# Reset to created
eventmodeler mark "Place Order" created
```

Use `list slices` to see current statuses:
```bash
eventmodeler list slices
```

## Checking Information Completeness

Information completeness tracks whether fields flow correctly between connected elements.

### Check a Specific Element
```bash
eventmodeler show completeness "<name>"
```
Auto-detects the element type (command, event, read model, screen, or processor) and shows:
- Incoming flows to that element
- Which fields are satisfied (have a source)
- Which fields are unsatisfied (missing a source)

### Check the Entire Model
```bash
eventmodeler show model-completeness
```
Project-wide view of all information flows:
- Summary: complete count, incomplete count, total
- Lists all incomplete flows with their unsatisfied fields

Example output:
```xml
<model-completeness>
  <summary complete="45" incomplete="3" total="48"/>
  <incomplete-flows>
    <flow from="CreateOrder" to="OrderCreated" target-type="event" flow-type="CommandToEvent">
      <unsatisfied-fields>
        <field name="customerId"/>
      </unsatisfied-fields>
    </flow>
  </incomplete-flows>
</model-completeness>
```

### Check Aggregate ID Fields
```bash
eventmodeler show aggregate-completeness "<aggregate-name>"
```
Checks if all events in an aggregate have the aggregate's ID field.

## Exploration Strategy

When exploring an unfamiliar event model:

1. **Get the big picture**: `eventmodeler summary` → understand scale
2. **See the features**: `eventmodeler list slices` → what does this system do?
3. **Understand organization**: `eventmodeler list chapters` → how is it structured over time?
4. **Dive into a slice**: `eventmodeler show slice "<name>"` → how does one feature work?
5. **Read the scenarios**: They tell you what the slice actually does, its preconditions, and edge cases
6. **Search for specifics**: `eventmodeler search "<term>"` → find related elements

When checking model health:
1. **Overall completeness**: `eventmodeler show model-completeness` → any broken flows?
2. **Specific element**: `eventmodeler show completeness "<name>"` → what's missing here?
3. **Aggregate consistency**: `eventmodeler show aggregate-completeness "<name>"` → ID fields present?

When looking for something specific:
- Know the feature name? → `show slice`
- Know an event name? → `show event` or `search`
- Want all events? → `list events`
- Checking data flow? → `show completeness` or `show model-completeness`
