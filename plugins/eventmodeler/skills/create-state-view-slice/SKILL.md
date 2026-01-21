---
name: create-state-view-slice
description: Create a state-view slice with a Read Model. Use when you need to display projected data to users or provide data to processors.
---

# Creating State-View Slices

A state-view slice represents a **read model** - a projection of event data for querying:

```
Read Model (fed by events)
```

- **Read Model**: A denormalized view optimized for a specific use case

## When to Use

Use state-view slices when:
- You need to display data to users
- A processor needs to query current state
- You want to aggregate data from multiple events
- Examples: "Order Summary", "User Dashboard", "Inventory Levels", "Account Balance"

## Command Syntax

```bash
eventmodeler create state-view-slice --xml '<state-view-slice name="Slice Name" [after="Other Slice"]>
  <read-model name="ReadModelName">
    <field name="fieldName" type="Type"/>
  </read-model>
</state-view-slice>'
```

## Field Types

`UUID`, `String`, `Int`, `Long`, `Double`, `Decimal`, `Boolean`, `Date`, `DateTime`, `Custom`

## Field Attributes

- `optional="true"` - Field may not always have a value

## Automatic Behavior

1. **Positioning**: Places slice at end, or use `after="Slice Name"` or `before="Slice Name"`

Note: Flows from events to the read model are created separately using the `map fields` command.

## Example: Order Summary View

```bash
eventmodeler create state-view-slice --xml '<state-view-slice name="View Order Summary" after="Place Order">
  <read-model name="OrderSummary">
    <field name="orderId" type="UUID"/>
    <field name="customerId" type="UUID"/>
    <field name="customerName" type="String"/>
    <field name="status" type="String"/>
    <field name="totalAmount" type="Decimal"/>
    <field name="itemCount" type="Int"/>
    <field name="placedAt" type="DateTime"/>
    <field name="lastUpdatedAt" type="DateTime"/>
  </read-model>
</state-view-slice>'
```

## Example: User Dashboard

```bash
eventmodeler create state-view-slice --xml '<state-view-slice name="View User Dashboard">
  <read-model name="UserDashboard">
    <field name="userId" type="UUID"/>
    <field name="displayName" type="String"/>
    <field name="email" type="String"/>
    <field name="orderCount" type="Int"/>
    <field name="totalSpent" type="Decimal"/>
    <field name="memberSince" type="DateTime"/>
    <field name="lastActivityAt" type="DateTime"/>
  </read-model>
</state-view-slice>'
```

## Workflow

### 1. Understand the Use Case
Ask or determine:
- Who views this data? (User on a screen, or a processor?)
- What decisions are made based on it?
- What filters/queries are needed?

### 2. Design the Fields
Consider:
- **Identity fields** - IDs to identify the entity
- **Display fields** - What the user sees
- **Status fields** - Current state
- **Computed fields** - Aggregations, counts, sums
- **Timestamp fields** - When things happened

### 3. Propose Before Creating
```
I'll create a "View Order Summary" slice with:

Read Model (OrderSummary):
- orderId (UUID) - identifies the order
- customerId (UUID) - who placed it
- customerName (String) - for display
- status (String) - pending, shipped, delivered, cancelled
- totalAmount (Decimal) - order total
- itemCount (Int) - number of items
- placedAt (DateTime) - when placed
- lastUpdatedAt (DateTime) - last status change

Does this look right?
```

### 4. Create the Slice
After approval, run the command.

### 5. Connect to Events
After creating the read model, set up field mappings from source events:

```bash
# Check what events exist
eventmodeler list events

# Map fields from events to the read model
eventmodeler map fields --flow "OrderPlaced→OrderSummary" --xml '
  <mapping from="orderId" to="orderId"/>
  <mapping from="customerId" to="customerId"/>
  <mapping from="totalAmount" to="totalAmount"/>
'
```

### 6. Verify Completeness
```bash
eventmodeler show completeness "OrderSummary"
```

## Best Practices

1. **Name describes the view** - "Order Summary" not "Order Read Model"
2. **Design for the consumer** - What does the screen/processor need?
3. **Include computed fields** - Counts, sums, derived status
4. **Consider multiple source events** - A read model often combines data from several events
5. **Use appropriate types** - `Decimal` for money, `DateTime` for timestamps
6. **Mark optional fields** - Fields that may not always have values (e.g., `cancelledAt`)
7. **Think about updates** - Which events update which fields?

## Connecting Events After Creation

The `create state-view-slice` command only creates the read model. To complete the picture:

1. Use `eventmodeler map fields --flow "EventName→ReadModelName"` to set up mappings
2. Use `eventmodeler show completeness "ReadModelName"` to verify all fields have sources

See the `design-readmodel` skill for more details on field mapping.
