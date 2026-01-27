---
name: design-readmodel
description: Help design read models with fields and set up field mappings from events. Use when a user wants to define what a read model should contain or needs help mapping fields.
---

# Designing Read Models and Field Mappings

Read models are projections of event data - views that serve specific screens or use cases. Every field in a read model should trace back to event fields.

## Your Workflow

When helping design a read model:

### 1. Understand the Context
```bash
# Look at the slice containing the read model
eventmodeler show slice "<slice-name>"

# Check current completeness - what's missing?
eventmodeler show completeness "<read-model-name>"

# See what events feed into this read model
eventmodeler list events
```

### 2. Understand What the Read Model Serves
Ask or infer:
- What screen displays this data?
- What decisions does the user make based on it?
- What queries/filters are needed?

### 3. Design the Fields
Based on the use case, identify what fields are needed. Consider:
- **Identity fields** - IDs to identify the entity
- **Display fields** - What the user sees
- **Status fields** - Current state
- **Computed fields** - Aggregations, counts, derived values
- **Timestamp fields** - When things happened

### 4. Add Fields to the Read Model

Field types: `UUID`, `String`, `Int`, `Long`, `Double`, `Decimal`, `Boolean`, `Date`, `DateTime`, `Custom`

```bash
# Add a simple field
eventmodeler add field --read-model "OrderSummary" --xml '<field name="orderId" type="UUID"/>'

# Add an optional field
eventmodeler add field --read-model "OrderSummary" --xml '<field name="cancelledAt" type="DateTime" isOptional="true"/>'

# Add a list field
eventmodeler add field --read-model "OrderSummary" --xml '<field name="itemIds" type="UUID" isList="true"/>'

# Add a complex/nested field
eventmodeler add field --read-model "OrderSummary" --xml '<field name="shippingAddress" type="Custom">
  <field name="street" type="String"/>
  <field name="city" type="String"/>
  <field name="postalCode" type="String"/>
</field>'
```

### 5. Map Fields from Events

Once the read model has fields, map them from the source events. See the **map-fields** skill for comprehensive documentation on field mappings.

Quick example:
```bash
eventmodeler map fields --flow "OrderPlaced→OrderSummary" --xml '
  <mapping from="orderId" to="orderId"/>
  <mapping from="customerId" to="customerId"/>
  <mapping from="totalAmount" to="total"/>
'
```

### 6. Verify Completeness

After mapping, check that all fields are satisfied:

```bash
eventmodeler show completeness "OrderSummary"
```

This shows which fields have sources and which are still missing.

## When User Has an Idea for a Read Model

If the user says "I want a read model that shows order history with status and total":

1. **Propose the fields** - Get user approval first:
   ```
   I suggest these fields for the OrderHistory read model:
   - orderId (UUID) - identifies the order
   - customerId (UUID) - who placed it
   - status (String) - current status: pending, shipped, delivered, cancelled
   - total (Decimal) - order total
   - placedAt (DateTime) - when it was placed
   - lastUpdatedAt (DateTime) - when status last changed

   Does this look right?
   ```

2. **Add the approved fields**:
   ```bash
   eventmodeler add field --read-model "OrderHistory" --xml '<field name="orderId" type="UUID"/>'
   eventmodeler add field --read-model "OrderHistory" --xml '<field name="customerId" type="UUID"/>'
   eventmodeler add field --read-model "OrderHistory" --xml '<field name="status" type="String"/>'
   eventmodeler add field --read-model "OrderHistory" --xml '<field name="total" type="Decimal"/>'
   eventmodeler add field --read-model "OrderHistory" --xml '<field name="placedAt" type="DateTime"/>'
   eventmodeler add field --read-model "OrderHistory" --xml '<field name="lastUpdatedAt" type="DateTime"/>'
   ```

3. **Set up the mappings** from relevant events:
   ```bash
   eventmodeler map fields --flow "OrderPlaced→OrderHistory" --xml '
     <mapping from="orderId" to="orderId"/>
     <mapping from="customerId" to="customerId"/>
     <mapping from="totalAmount" to="total"/>
     <mapping from="timestamp" to="placedAt"/>
   '
   ```

## Checking Model-Wide Completeness

To find all incomplete mappings across the entire model:

```bash
eventmodeler show model-completeness
```

This shows a summary of all flows and lists any with unsatisfied fields - useful for finding what still needs work.

## Working with Lists and Custom Types

**List of primitives** - use `isList="true"` with any valid type:
```bash
eventmodeler add field --read-model "OrderSummary" --xml '<field name="tags" type="String" isList="true"/>'
```

### Custom Types (Nested Objects)

**CRITICAL: Custom fields MUST contain subfields.** The `type="Custom"` indicates a nested object structure, and the nested `<field>` elements define its properties. A Custom field without subfields is invalid and creates an empty, useless type.

**Custom type with nested fields** - use `type="Custom"` with nested `<field>` elements:
```bash
eventmodeler add field --read-model "OrderSummary" --xml '<field name="shippingAddress" type="Custom">
  <field name="street" type="String"/>
  <field name="city" type="String"/>
  <field name="postalCode" type="String"/>
</field>'
```

The subfields (`street`, `city`, `postalCode`) define the structure of the `shippingAddress` object. Without these subfields, the custom type has no schema and cannot hold meaningful data.

**List of custom objects** - combine `type="Custom"` with `isList="true"`:
```bash
eventmodeler add field --read-model "OrderSummary" --xml '<field name="lineItems" type="Custom" isList="true">
  <field name="productId" type="UUID"/>
  <field name="productName" type="String"/>
  <field name="quantity" type="Int"/>
  <field name="unitPrice" type="Decimal"/>
</field>'
```

This creates a list where each item has the structure defined by the subfields.

**Nested custom types** - subfields can themselves be Custom types:
```bash
eventmodeler add field --read-model "OrderSummary" --xml '<field name="customer" type="Custom">
  <field name="customerId" type="UUID"/>
  <field name="name" type="String"/>
  <field name="billingAddress" type="Custom">
    <field name="street" type="String"/>
    <field name="city" type="String"/>
    <field name="country" type="String"/>
  </field>
</field>'
```

## Common Mistakes

- **Creating empty Custom fields**: Custom fields MUST have subfields that define their structure.
  - Wrong: `<field name="address" type="Custom"/>` (empty - no structure defined)
  - Wrong: `<field name="address" type="Custom"></field>` (also empty)
  - Right:
    ```xml
    <field name="address" type="Custom">
      <field name="street" type="String"/>
      <field name="city" type="String"/>
    </field>
    ```

- **Using "List" as a field type**: "List" is not a valid type. Use `isList="true"` with a valid type instead.
  - Wrong: `<field name="items" type="List"/>`
  - Right: `<field name="items" type="String" isList="true"/>`

- **Using wrong attribute names**: Always use camelCase attributes.
  - Wrong: `optional="true"`, `generated="true"`, `list="true"`
  - Right: `isOptional="true"`, `isGenerated="true"`, `isList="true"`

## Best Practices

1. **Get approval before adding fields** - Propose the design, let the user confirm
2. **Consider all source events** - A read model may need data from multiple events
3. **Use appropriate types** - `Decimal` for money, `DateTime` for timestamps, etc.
4. **Mark optional fields** - Fields that may not always have values (e.g., `cancelledAt`)
5. **Think about updates** - How does the read model change when different events occur?
6. **Verify completeness** - Run `show completeness` after mapping to ensure nothing is missed
