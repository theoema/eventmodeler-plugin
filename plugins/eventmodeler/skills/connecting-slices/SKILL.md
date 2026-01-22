---
name: connecting-slices
description: Connect slices together with flows to show how data moves through the system. Use after creating slices to complete the event model.
---

# Connecting Slices

After creating slices, you need to connect them with flows to show how data moves through the system. This creates the complete picture of your event model.

## The Data Flow Pattern

In event modeling, data flows in a specific pattern:

```
[State-Change Slice]              [State-View Slice]              [Next Slice]

     Screen                            ReadModel                     Screen
       │                                  ↑  │                         ↑
       ↓                                  │  ↓                         │
    Command                               │  ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┘
       │                                  │                    OR
       ↓                                  │                   Processor
     Event  ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┘                      ↑
                                                                  │
                                          └ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┘
```

**Key flows:**
1. **Event → ReadModel**: Events project their data into read models
2. **ReadModel → Screen**: Read models provide data to screens (for user viewing)
3. **ReadModel → Processor**: Read models provide data to processors (for automation)

Within-slice flows (Screen→Command, Command→Event, Processor→Command) are created automatically when you create a slice.

## Command Syntax

```bash
eventmodeler create flow --from "<source>" --to "<target>"
```

## Valid Flow Combinations

| Source | Target | Use Case |
|--------|--------|----------|
| Event | ReadModel | Project event data into a view |
| ReadModel | Screen | Provide data for user display |
| ReadModel | Processor | Provide data for automation logic |

## Flow Directions (Visual)

Flows are rendered with specific directions:
- **Event → ReadModel**: From event's top to read model's bottom (events are below)
- **ReadModel → Screen/Processor**: From read model's top to screen/processor's bottom

## Workflow: Connecting a Complete Model

### 1. Create Your Slices First

```bash
# User places an order
eventmodeler create state-change-slice --xml '<state-change-slice name="Place Order">
  <screen name="Checkout">...</screen>
  <command name="PlaceOrder">...</command>
  <event name="OrderPlaced">...</event>
</state-change-slice>'

# View for order status
eventmodeler create state-view-slice --xml '<state-view-slice name="View Order Status" after="Place Order">
  <read-model name="OrderStatus">...</read-model>
</state-view-slice>'

# System automatically fulfills order (includes its own read model)
eventmodeler create automation-slice --xml '<automation-slice name="Auto Fulfill" after="View Order Status">
  <read-model name="OrderReadyToFulfill">
    <field name="orderId" type="UUID"/>
    <field name="isPaid" type="Boolean"/>
  </read-model>
  <processor name="Fulfillment Processor"/>
  <command name="FulfillOrder">...</command>
  <event name="OrderFulfilled">...</event>
</automation-slice>'
```

### 2. Connect Events to Read Models

Events project their data into read models:

```bash
# OrderPlaced feeds into OrderStatus view
eventmodeler create flow --from "OrderPlaced" --to "OrderStatus"

# OrderFulfilled also updates OrderStatus view
eventmodeler create flow --from "OrderFulfilled" --to "OrderStatus"
```

### 3. Connect Events to Automation Read Models

Automation slices have their own internal read model. Connect external events to feed it:

```bash
# OrderPlaced feeds the automation's read model
eventmodeler create flow --from "OrderPlaced" --to "OrderReadyToFulfill"

# PaymentReceived also feeds the automation's read model
eventmodeler create flow --from "PaymentReceived" --to "OrderReadyToFulfill"
```

Note: The flow from ReadModel → Processor is internal to the automation slice and created automatically.

### 4. Connect Read Models to Screens (for State-View slices)

State-view read models provide data to screens:

```bash
# OrderStatus provides data to a screen in another slice
eventmodeler create flow --from "OrderStatus" --to "Order Details Screen"
```

### 5. Set Up Field Mappings

After creating flows, map the fields. The `--flow` argument uses the source and target names separated by `→`:

```bash
# Map fields from OrderPlaced to OrderStatus
eventmodeler map fields --flow "OrderPlaced→OrderStatus" --xml '
  <mapping from="orderId" to="orderId"/>
  <mapping from="customerId" to="customerId"/>
  <mapping from="placedAt" to="placedAt"/>
'
```

### 6. Verify Completeness

Check that all flows have proper field mappings:

```bash
eventmodeler show model-completeness
eventmodeler show completeness "OrderStatus"
```

## Handling Duplicate Names with IDs

When multiple elements share the same name (e.g., linked copies of events or read models), you'll need to use element IDs instead of names.

### Finding Element IDs

```bash
# List events with their IDs
eventmodeler list events
# Output:
# <events>
#   <event id="abc12345-..." name="OrderPlaced" fields="4"/>
#   <event id="def67890-..." name="OrderPlaced" fields="4"/>  <!-- linked copy -->
# </events>
```

### Using IDs in Commands

Prefix the ID (or ID prefix) with `id:`:

```bash
# Create flow using full ID
eventmodeler create flow --from "id:abc12345-1234-5678-9abc-def012345678" --to "OrderStatus"

# Create flow using ID prefix (first 8 characters is usually enough)
eventmodeler create flow --from "id:abc12345" --to "OrderStatus"

# Map fields using ID prefix
eventmodeler map fields --flow "id:abc12345→OrderStatus" --xml '
  <mapping from="orderId" to="orderId"/>
'
```

### When the CLI Finds Duplicates

If you use a name that matches multiple elements, the CLI will show you the options:

```bash
eventmodeler create flow --from "OrderPlaced" --to "OrderStatus"
# Error: Multiple events found with name "OrderPlaced"
# Please specify using the element ID:
#   - "OrderPlaced" (id: abc12345-1234-5678-9abc-def012345678)
#   - "OrderPlaced" (id: def67890-1234-5678-9abc-def012345678)
#
# Usage: eventmodeler create flow --from "id:abc12345" --to "OrderStatus"
```

## Common Patterns

### Pattern 1: Event Sourced View
Multiple events feed into one read model:

```bash
eventmodeler create flow --from "OrderPlaced" --to "OrderSummary"
eventmodeler create flow --from "OrderShipped" --to "OrderSummary"
eventmodeler create flow --from "OrderDelivered" --to "OrderSummary"
```

### Pattern 2: Automation Triggered by Events
Events feed an automation slice's internal read model:

```bash
# Events feed the automation's read model (created with create automation-slice)
eventmodeler create flow --from "OrderPlaced" --to "OrderReadyForShipment"
eventmodeler create flow --from "PaymentReceived" --to "OrderReadyForShipment"

# The ReadModel → Processor flow is internal to the automation slice
# (created automatically when you create the automation slice)
```

### Pattern 3: User Dashboard
A read model feeds a screen for user viewing:

```bash
eventmodeler create flow --from "UserDashboard" --to "Dashboard Screen"
```

## Error Handling

The CLI prevents invalid flows:

```bash
# This will error - can't go directly from Event to Screen
eventmodeler create flow --from "OrderPlaced" --to "Checkout Screen"
# Error: Cannot create flow directly from Event to Screen.
# Events flow to ReadModels, which then flow to Screens.

# This will error - can't go from ReadModel to ReadModel
eventmodeler create flow --from "OrderStatus" --to "CustomerProfile"
# Error: Cannot create flow from ReadModel to ReadModel.
```

## Best Practices

1. **Create all slices first** - Easier to see the full picture before connecting
2. **Connect events to read models** - Every event should feed at least one read model
3. **Think about what triggers what** - Automation slices need data from read models
4. **Map fields after creating flows** - Use `map fields` to complete the data flow
5. **Verify completeness** - Run `show model-completeness` to find missing mappings
6. **Use IDs for duplicates** - When names aren't unique, use `id:` prefix with element IDs
