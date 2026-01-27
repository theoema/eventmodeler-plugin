---
name: map-fields
description: Create field mappings on flows to define how data moves between connected elements. Use when setting up or modifying field mappings on Event→ReadModel, ReadModel→Screen, or other flows.
---

# Mapping Fields on Flows

Field mappings define how data flows from source elements to target elements. When you connect an Event to a ReadModel with a flow, the field mappings specify which event fields populate which read model fields.

## Why Field Mappings Matter

Field mappings are essential for:
- **Information completeness** - The model tracks whether all target fields have sources
- **Code generation** - Generators use mappings to create projection logic
- **Documentation** - Clear traceability of where each field's data comes from

## Command Syntax

```bash
eventmodeler map fields --flow "<Source>→<Target>" --xml '<mapping from="sourceField" to="targetField"/>'
```

Or with multiple mappings:

```bash
eventmodeler map fields --flow "<Source>→<Target>" --xml '
  <mapping from="orderId" to="orderId"/>
  <mapping from="customerId" to="customerId"/>
  <mapping from="totalAmount" to="total"/>
'
```

### Flow Identifier Formats

The `--flow` argument accepts:

1. **Name arrow format** (most common): `"SourceName→TargetName"` or `"SourceName->TargetName"`
   ```bash
   eventmodeler map fields --flow "OrderPlaced→OrderSummary" --xml '...'
   ```

2. **Element ID prefix** (for duplicates): `"id:abc12345→TargetName"`
   ```bash
   eventmodeler map fields --flow "id:abc12345→OrderSummary" --xml '...'
   ```

3. **Flow ID directly**: The UUID of the flow itself
   ```bash
   eventmodeler map fields --flow "550e8400-e29b-41d4-a716-446655440000" --xml '...'
   ```

## JSON Alternative

You can use JSON instead of XML:

```bash
eventmodeler map fields --flow "OrderPlaced→OrderSummary" --json '[
  {"from": "orderId", "to": "orderId"},
  {"from": "customerId", "to": "customerId"}
]'
```

## Workflow: Setting Up Field Mappings

### 1. Find the Flow

First, identify what flows exist and need mapping:

```bash
# Check model-wide completeness to find flows missing mappings
eventmodeler show model-completeness

# Or check a specific element's completeness
eventmodeler show completeness "OrderSummary"
```

The completeness output shows:
- Which flows exist
- Which target fields are satisfied (have mappings)
- Which target fields are unsatisfied (need mappings)

### 2. Understand Source and Target Fields

```bash
# See the source element's fields
eventmodeler show event "OrderPlaced"

# See the target element's fields
eventmodeler show completeness "OrderSummary"
```

### 3. Create the Mappings

Map source fields to target fields:

```bash
eventmodeler map fields --flow "OrderPlaced→OrderSummary" --xml '
  <mapping from="orderId" to="orderId"/>
  <mapping from="customerId" to="customerId"/>
  <mapping from="items" to="lineItems"/>
  <mapping from="totalAmount" to="total"/>
  <mapping from="placedAt" to="orderDate"/>
'
```

### 4. Verify Completeness

```bash
eventmodeler show completeness "OrderSummary"
```

## Mapping Nested Fields

For Custom (nested) field types, use dot notation:

```bash
eventmodeler map fields --flow "OrderPlaced→OrderSummary" --xml '
  <mapping from="shippingAddress.street" to="deliveryAddress.street"/>
  <mapping from="shippingAddress.city" to="deliveryAddress.city"/>
  <mapping from="shippingAddress.postalCode" to="deliveryAddress.zip"/>
'
```

## Handling Multiple Flows to Same Target

A read model often receives data from multiple events. Map each flow separately:

```bash
# Initial data from OrderPlaced
eventmodeler map fields --flow "OrderPlaced→OrderSummary" --xml '
  <mapping from="orderId" to="orderId"/>
  <mapping from="customerId" to="customerId"/>
  <mapping from="totalAmount" to="total"/>
  <mapping from="placedAt" to="orderDate"/>
'

# Status updates from OrderShipped
eventmodeler map fields --flow "OrderShipped→OrderSummary" --xml '
  <mapping from="shippedAt" to="shippedDate"/>
  <mapping from="trackingNumber" to="trackingNumber"/>
'

# Delivery confirmation from OrderDelivered
eventmodeler map fields --flow "OrderDelivered→OrderSummary" --xml '
  <mapping from="deliveredAt" to="deliveredDate"/>
'
```

## Handling Duplicate Element Names (Linked Copies)

When the model has linked copies of an element (same name, different instances), use element IDs:

```bash
# List events to find IDs
eventmodeler list events
# Output shows:
#   <event id="abc12345-..." name="OrderPlaced" fields="4"/>
#   <event id="def67890-..." name="OrderPlaced" fields="4"/>  <!-- linked copy -->

# Use ID prefix in the flow identifier
eventmodeler map fields --flow "id:abc12345→OrderSummary" --xml '
  <mapping from="orderId" to="orderId"/>
'
```

The CLI prefers non-linked-copy elements when resolving names, but if you need a specific instance, use the ID prefix.

## Updating Existing Mappings

Running `map fields` again **merges** with existing mappings:
- New mappings are added
- If a target field already has a mapping, it's replaced with the new source

```bash
# Initial mapping
eventmodeler map fields --flow "OrderPlaced→OrderSummary" --xml '
  <mapping from="orderId" to="orderId"/>
'

# Add more mappings (orderId mapping preserved, new ones added)
eventmodeler map fields --flow "OrderPlaced→OrderSummary" --xml '
  <mapping from="customerId" to="customerId"/>
  <mapping from="totalAmount" to="total"/>
'

# Change a mapping (total now comes from different field)
eventmodeler map fields --flow "OrderPlaced→OrderSummary" --xml '
  <mapping from="grandTotal" to="total"/>
'
```

## Common Mapping Patterns

### Identity Mapping
Same field name on both sides:
```xml
<mapping from="orderId" to="orderId"/>
```

### Rename Mapping
Different names, same data:
```xml
<mapping from="totalAmount" to="total"/>
<mapping from="placedAt" to="orderDate"/>
```

### Nested to Flat
Extract nested fields to top level:
```xml
<mapping from="customer.id" to="customerId"/>
<mapping from="customer.name" to="customerName"/>
```

### Flat to Nested
Populate nested structure from flat fields:
```xml
<mapping from="street" to="address.street"/>
<mapping from="city" to="address.city"/>
```

## Error Handling

The CLI provides helpful errors:

**Flow not found:**
```
Error: Flow not found: OrderPlaced→OrderSummary
Available flows:
  - OrderPlaced→OrderDetails (abc12345-...)
  - OrderShipped→OrderDetails (def67890-...)
```

**Source field not found:**
```
Error: Source field not found: totalPrice
Available source fields:
  - orderId (uuid-1234...)
  - customerId (uuid-5678...)
  - totalAmount (uuid-9abc...)
```

**Target field not found:**
```
Error: Target field not found: grandTotal
Available target fields:
  - orderId (uuid-1234...)
  - total (uuid-5678...)
  - orderDate (uuid-9abc...)
```

## Best Practices

1. **Check completeness first** - Use `show model-completeness` to find what needs mapping
2. **Map all fields at once** - More efficient than one mapping at a time
3. **Use meaningful names** - If renaming, ensure the target name reflects the domain
4. **Verify after mapping** - Run `show completeness` to confirm all fields are satisfied
5. **Handle linked copies carefully** - Use ID prefixes when names are ambiguous
6. **Document transformations** - If a mapping involves a transformation concept (e.g., timestamp to date), note it in scenarios
