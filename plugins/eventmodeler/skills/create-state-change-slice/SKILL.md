---
name: create-state-change-slice
description: Create a complete state-change slice with Screen, Command, Event, flows, and field mappings. Use when a user action triggers a state change in the system.
---

# Creating State-Change Slices

A state-change slice represents a **user-initiated action** that changes system state:

```
Screen → Command → Event
```

- **Screen**: The UI where the user initiates the action
- **Command**: The intent/request to change state
- **Event**: The fact that state changed

## When to Use

Use state-change slices when:
- A user performs an action (clicks a button, submits a form)
- That action changes the state of an entity
- Examples: "Place Order", "Cancel Subscription", "Update Profile", "Add Item to Cart"

## Command Syntax

```bash
eventmodeler create state-change-slice --xml '<state-change-slice name="Slice Name" [after="Other Slice"]>
  <screen name="Screen Name">
    <field name="fieldName" type="Type"/>
  </screen>
  <command name="CommandName">
    <field name="fieldName" type="Type"/>
  </command>
  <event name="EventName">
    <field name="fieldName" type="Type"/>
  </event>
</state-change-slice>'
```

## Field Types

`UUID`, `String`, `Int`, `Long`, `Double`, `Decimal`, `Boolean`, `Date`, `DateTime`, `Custom`

## Field Attributes

- `isOptional="true"` - Field may not have a value
- `isGenerated="true"` - System-generated (e.g., IDs, timestamps), not from user input
- `isUserInput="true"` - Entered by the user on the screen
- `isList="true"` - Field is a list/array of the specified type

## Automatic Behavior

1. **Positioning**: Places slice at end, or use `after="Slice Name"` or `before="Slice Name"`
2. **Field mappings**: Automatically inferred by matching field names between:
   - Screen → Command
   - Command → Event
3. **Flows**: Automatically creates Screen→Command and Command→Event flows

## Example: Place Order Slice

```bash
eventmodeler create state-change-slice --xml '<state-change-slice name="Place Order">
  <screen name="Checkout Screen">
    <field name="customerId" type="UUID"/>
    <field name="items" type="String" isUserInput="true"/>
    <field name="shippingAddress" type="String" isUserInput="true"/>
  </screen>
  <command name="PlaceOrder">
    <field name="customerId" type="UUID"/>
    <field name="items" type="String"/>
    <field name="shippingAddress" type="String"/>
  </command>
  <event name="OrderPlaced">
    <field name="orderId" type="UUID" isGenerated="true"/>
    <field name="customerId" type="UUID"/>
    <field name="items" type="String"/>
    <field name="shippingAddress" type="String"/>
    <field name="placedAt" type="DateTime" isGenerated="true"/>
  </event>
</state-change-slice>'
```

## Workflow

### 1. Understand the User Action
Ask or determine:
- What is the user trying to do?
- What information do they provide?
- What changes as a result?

### 2. Design the Fields
- **Screen fields**: What the user sees/enters
- **Command fields**: What's needed to execute the action
- **Event fields**: What facts are recorded (include generated IDs and timestamps)

### 3. Propose Before Creating
```
I'll create a "Place Order" slice with:

Screen (Checkout Screen):
- customerId (UUID) - identifies the customer
- items (String) - what they're ordering
- shippingAddress (String) - where to ship

Command (PlaceOrder):
- customerId, items, shippingAddress (from screen)

Event (OrderPlaced):
- orderId (UUID, generated) - unique order identifier
- customerId, items, shippingAddress (from command)
- placedAt (DateTime, generated) - when the order was placed

Does this look right?
```

### 4. Create the Slice
After approval, run the command.

### 5. Verify
```bash
eventmodeler show slice "Place Order"
```

## Working with Lists and Custom Types

**List of primitives** - use `isList="true"` with any valid type:
```xml
<field name="tags" type="String" isList="true"/>
<field name="quantities" type="Int" isList="true"/>
```

### Custom Types (Nested Objects)

**CRITICAL: Custom fields MUST contain subfields.** The `type="Custom"` indicates a nested object structure, and the nested `<field>` elements define its properties. A Custom field without subfields is invalid and creates an empty, useless type.

**Custom type with nested fields** - use `type="Custom"` with nested `<field>` elements:
```xml
<field name="address" type="Custom">
  <field name="street" type="String"/>
  <field name="city" type="String"/>
  <field name="postalCode" type="String"/>
</field>
```

The subfields (`street`, `city`, `postalCode`) define the structure of the `address` object. Without these subfields, the custom type has no schema and cannot hold meaningful data.

**List of custom objects** - combine `type="Custom"` with `isList="true"`:
```xml
<field name="lineItems" type="Custom" isList="true">
  <field name="productId" type="UUID"/>
  <field name="quantity" type="Int"/>
  <field name="unitPrice" type="Decimal"/>
</field>
```

This creates a list where each item has the structure defined by the subfields.

**Nested custom types** - subfields can themselves be Custom types:
```xml
<field name="customer" type="Custom">
  <field name="customerId" type="UUID"/>
  <field name="name" type="String"/>
  <field name="billingAddress" type="Custom">
    <field name="street" type="String"/>
    <field name="city" type="String"/>
    <field name="country" type="String"/>
  </field>
</field>
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
  - Wrong: `optional="true"`, `generated="true"`, `list="true"`, `user-input="true"`
  - Right: `isOptional="true"`, `isGenerated="true"`, `isList="true"`, `isUserInput="true"`

## Best Practices

1. **Get approval first** - Propose the design before creating
2. **Use meaningful names** - Slice name should describe the action ("Place Order" not "Order Slice")
3. **Include generated fields** - Events should have IDs and timestamps
4. **Match field names** - Same names enable automatic mapping
5. **Consider what's user input vs system data** - Mark appropriately
