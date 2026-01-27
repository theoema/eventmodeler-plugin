---
name: manage-fields
description: Update or remove fields on commands, events, read models, screens, and processors. Use when modifying field properties (optional, generated, type) or removing fields from elements.
---

# Managing Fields

This skill covers updating field properties and removing fields from elements. For adding new fields, see the slice creation skills. For mapping fields between elements, see the `map-fields` skill.

## Updating Field Properties

Use `update field` to modify field attributes without changing the field's name or structure.

### Command Syntax

```bash
eventmodeler update field --<element-type> "<name>" --field "<field-name>" [options]
```

Element types: `--command`, `--event`, `--read-model`, `--screen`, `--processor`

### Available Options

| Option | Values | Description |
|--------|--------|-------------|
| `--optional` | `true`/`false` | Mark field as not required for completeness |
| `--generated` | `true`/`false` | Mark field as system-generated (not from user input) |
| `--user-input` | `true`/`false` | Mark field as user-provided (screens only) |
| `--type` | Field type | Change the field's data type |

### Examples

**Mark a field as optional:**
```bash
eventmodeler update field --event "OrderPlaced" --field "promoCode" --optional true
```

**Mark a field as system-generated:**
```bash
eventmodeler update field --event "OrderPlaced" --field "orderId" --generated true
eventmodeler update field --event "OrderPlaced" --field "timestamp" --generated true
```

**Mark screen fields as user input:**
```bash
eventmodeler update field --screen "Checkout" --field "email" --user-input true
eventmodeler update field --screen "Checkout" --field "shippingAddress" --user-input true
```

**Change a field's type:**
```bash
eventmodeler update field --read-model "OrderSummary" --field "total" --type Decimal
```

**Combine multiple updates:**
```bash
eventmodeler update field --event "UserRegistered" --field "createdAt" --generated true --optional false
```

### Updating Nested Fields

Use dot notation for fields inside Custom types:

```bash
eventmodeler update field --event "OrderPlaced" --field "shippingAddress.postalCode" --optional true
```

## Field Property Meanings

### isOptional
- **Purpose**: Indicates the field may not always have a value
- **Effect**: Completeness checks don't require this field to have a mapping source
- **Use for**: Nullable fields, conditional data, fields set by later events

```bash
# cancelledAt is only set when an order is cancelled
eventmodeler update field --read-model "OrderSummary" --field "cancelledAt" --optional true
```

### isGenerated
- **Purpose**: Indicates the field is system-generated, not derived from other fields
- **Effect**: Completeness checks don't require this field to have a mapping source
- **Use for**: IDs, timestamps, sequence numbers, computed values

```bash
# System generates the order ID
eventmodeler update field --event "OrderPlaced" --field "orderId" --generated true
# System captures the timestamp
eventmodeler update field --event "OrderPlaced" --field "timestamp" --generated true
```

### isUserInput (Screens only)
- **Purpose**: Indicates the field comes from user input on the screen
- **Effect**: Identifies which screen fields are inputs vs. display-only
- **Use for**: Form fields, search inputs, user selections

```bash
# These are input fields on the checkout screen
eventmodeler update field --screen "Checkout" --field "email" --user-input true
eventmodeler update field --screen "Checkout" --field "cardNumber" --user-input true
# This is display-only (from read model)
eventmodeler update field --screen "Checkout" --field "orderTotal" --user-input false
```

## Removing Fields

Use `remove field` to delete a field from an element.

### Command Syntax

```bash
eventmodeler remove field --<element-type> "<name>" --field "<field-name>"
```

### Examples

```bash
# Remove a field from an event
eventmodeler remove field --event "OrderPlaced" --field "legacyId"

# Remove a field from a read model
eventmodeler remove field --read-model "OrderSummary" --field "deprecatedStatus"

# Remove a field from a command
eventmodeler remove field --command "PlaceOrder" --field "unusedFlag"

# Remove a nested field using dot notation
eventmodeler remove field --event "OrderPlaced" --field "metadata.debugInfo"
```

### Caution

Removing a field:
- Deletes the field from the element
- May break existing field mappings that reference it
- Cannot be undone (except via git)

Before removing, check if the field is used in mappings:

```bash
eventmodeler show completeness "<element-name>"
```

## Workflow: Cleaning Up Field Properties

### 1. Review Current State

```bash
# See all fields and their properties
eventmodeler show event "OrderPlaced"
```

Output shows each field with its attributes:
```xml
<event name="OrderPlaced">
  <field name="orderId" type="UUID"/>
  <field name="customerId" type="UUID"/>
  <field name="timestamp" type="DateTime"/>
  <field name="promoCode" type="String"/>
</event>
```

### 2. Identify System-Generated Fields

Mark fields that the system generates (not derived from other data):

```bash
eventmodeler update field --event "OrderPlaced" --field "orderId" --generated true
eventmodeler update field --event "OrderPlaced" --field "timestamp" --generated true
```

### 3. Identify Optional Fields

Mark fields that may not always be present:

```bash
eventmodeler update field --event "OrderPlaced" --field "promoCode" --optional true
```

### 4. Verify Completeness

```bash
eventmodeler show completeness "OrderPlaced"
```

Generated and optional fields should no longer appear as "unsatisfied".

## Workflow: Fixing Screen Field Types

### 1. List Screen Fields

```bash
eventmodeler show slice "Place Order"
```

### 2. Mark Input Fields

```bash
# User provides these
eventmodeler update field --screen "Checkout" --field "email" --user-input true
eventmodeler update field --screen "Checkout" --field "cardNumber" --user-input true
eventmodeler update field --screen "Checkout" --field "billingAddress" --user-input true
```

### 3. Ensure Display Fields Are Not Marked

```bash
# These come from the read model, not user input
eventmodeler update field --screen "Checkout" --field "orderSummary" --user-input false
eventmodeler update field --screen "Checkout" --field "totalAmount" --user-input false
```

## Common Patterns

### Event with Generated ID and Timestamp

```bash
eventmodeler update field --event "UserRegistered" --field "userId" --generated true
eventmodeler update field --event "UserRegistered" --field "registeredAt" --generated true
```

### Read Model with Optional Status Fields

```bash
eventmodeler update field --read-model "OrderStatus" --field "shippedAt" --optional true
eventmodeler update field --read-model "OrderStatus" --field "deliveredAt" --optional true
eventmodeler update field --read-model "OrderStatus" --field "cancelledAt" --optional true
```

### Screen with Mixed Input/Display

```bash
# Inputs
eventmodeler update field --screen "Profile" --field "displayName" --user-input true
eventmodeler update field --screen "Profile" --field "bio" --user-input true
# Display-only
eventmodeler update field --screen "Profile" --field "memberSince" --user-input false
eventmodeler update field --screen "Profile" --field "totalOrders" --user-input false
```

## Error Handling

**Field not found:**
```
Error: Field "wrongName" not found on event "OrderPlaced"
Available fields:
  - orderId
  - customerId
  - timestamp
```

**Element not found:**
```
Error: Event "WrongEvent" not found
```

**Invalid type:**
```
Error: Invalid field type. Valid types: UUID, String, Int, Long, Double, Decimal, Boolean, Date, DateTime, Custom
```

## Best Practices

1. **Mark generated fields early** - When creating slices, immediately mark IDs and timestamps as generated
2. **Use optional sparingly** - Only mark truly optional fields; prefer explicit modeling over nullable everything
3. **Document user inputs** - Mark all screen input fields for clarity in code generation
4. **Check mappings before removing** - Removing a field can break existing mappings
5. **Batch updates** - Update multiple fields in one session to maintain consistency
