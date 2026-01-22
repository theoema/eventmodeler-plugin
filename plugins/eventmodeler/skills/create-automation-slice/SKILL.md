---
name: create-automation-slice
description: Create a complete automation slice with ReadModel, Processor, Command, Event, and flows. Use when system logic automatically triggers a state change in response to events.
---

# Creating Automation Slices

An automation slice represents a **system-initiated action**:

```
ReadModel → Processor → Command → Event
```

- **ReadModel**: The state/data the processor needs to make decisions
- **Processor**: The system component that decides when and how to act
- **Command**: The intent/request to change state
- **Event**: The fact that state changed

## When to Use

Use automation slices when:
- The system automatically responds to events
- No direct user interaction triggers the action
- Examples: "Auto-assign Reviewer", "Send Reminder Email", "Calculate Shipping", "Process Payment"

## Command Syntax

```bash
eventmodeler create automation-slice --xml '<automation-slice name="Slice Name" [after="Other Slice"]>
  <read-model name="ReadModelName">
    <field name="fieldName" type="Type"/>
  </read-model>
  <processor name="Processor Name"/>
  <command name="CommandName">
    <field name="fieldName" type="Type"/>
  </command>
  <event name="EventName">
    <field name="fieldName" type="Type"/>
  </event>
</automation-slice>'
```

## Field Types

`UUID`, `String`, `Int`, `Long`, `Double`, `Decimal`, `Boolean`, `Date`, `DateTime`, `Custom`

## Field Attributes

- `optional="true"` - Field may not have a value
- `generated="true"` - System-generated (e.g., computed values, timestamps)

## Automatic Behavior

1. **Positioning**: Places slice at end, or use `after="Slice Name"` or `before="Slice Name"`
2. **Field mappings**: Automatically inferred by matching field names:
   - ReadModel → Command (same-named fields)
   - Command → Event (same-named fields)
3. **Flows**: Automatically creates:
   - ReadModel → Processor
   - Processor → Command
   - Command → Event

## Example: Auto-Fulfill Order

```bash
eventmodeler create automation-slice --xml '<automation-slice name="Auto Fulfill Order" after="Place Order">
  <read-model name="OrderReadyForFulfillment">
    <field name="orderId" type="UUID"/>
    <field name="isPaid" type="Boolean"/>
    <field name="shippingAddress" type="String"/>
  </read-model>
  <processor name="Order Fulfillment Processor"/>
  <command name="FulfillOrder">
    <field name="orderId" type="UUID"/>
    <field name="warehouseId" type="UUID" generated="true"/>
  </command>
  <event name="OrderFulfilled">
    <field name="orderId" type="UUID"/>
    <field name="warehouseId" type="UUID"/>
    <field name="fulfilledAt" type="DateTime" generated="true"/>
  </event>
</automation-slice>'
```

## Example: Send Welcome Email

```bash
eventmodeler create automation-slice --xml '<automation-slice name="Send Welcome Email" after="Register User">
  <read-model name="NewUserForWelcome">
    <field name="userId" type="UUID"/>
    <field name="email" type="String"/>
    <field name="firstName" type="String"/>
  </read-model>
  <processor name="Welcome Email Sender"/>
  <command name="SendWelcomeEmail">
    <field name="userId" type="UUID"/>
    <field name="email" type="String"/>
    <field name="firstName" type="String"/>
  </command>
  <event name="WelcomeEmailSent">
    <field name="userId" type="UUID"/>
    <field name="email" type="String"/>
    <field name="sentAt" type="DateTime" generated="true"/>
  </event>
</automation-slice>'
```

## Workflow

### 1. Identify the Trigger Events
What event(s) cause this automation to run?
- The read model collects data from trigger events
- External events flow INTO the automation's read model

### 2. Design the Read Model
What state does the processor need to make its decision?
- Include identity fields (IDs)
- Include any conditions the processor checks
- This is an "automation read model" - it exists to feed the processor

### 3. Design the Command & Event Fields
- **Command fields**: What data is needed to perform the action
- **Event fields**: What facts are recorded (include generated values)

### 4. Propose Before Creating
```
I'll create an "Auto Fulfill Order" slice:

ReadModel (OrderReadyForFulfillment):
- orderId (UUID) - which order
- isPaid (Boolean) - processor checks this
- shippingAddress (String) - needed for fulfillment

Processor: Order Fulfillment Processor
- Triggers when order is paid and has shipping address

Command (FulfillOrder):
- orderId (UUID) - which order to fulfill
- warehouseId (UUID, generated) - selected warehouse

Event (OrderFulfilled):
- orderId, warehouseId (from command)
- fulfilledAt (DateTime, generated) - when fulfillment started

Does this look right?
```

### 5. Create the Slice
After approval, run the command.

### 6. Verify
```bash
eventmodeler show slice "Auto Fulfill Order"
```

### 7. Connect Trigger Events
After creating the automation slice, connect external events to its read model:

```bash
# Events from other slices feed the automation's read model
eventmodeler create flow --from "OrderPlaced" --to "OrderReadyForFulfillment"
eventmodeler create flow --from "PaymentReceived" --to "OrderReadyForFulfillment"
```

## Automation vs State-View Read Models

| Automation ReadModel | State-View ReadModel |
|---------------------|---------------------|
| Inside automation slice | Standalone slice |
| Feeds a Processor | Feeds a Screen |
| "What the processor needs" | "What the user sees" |
| Created with automation-slice | Created with state-view-slice |

## Code Generation

Use `eventmodeler codegen slice "<name>"` to get JSON output for code generation:

```bash
eventmodeler codegen slice "Auto Fulfill Order"
```

This outputs:
- `sliceType: "AUTOMATION"` for routing to the right generator
- Full schemas for all elements
- Inbound dependencies (events feeding the read model)
- Internal flows with field mappings
- Scenarios with resolved event schemas

## Best Practices

1. **Name the read model for its purpose** - "OrderReadyForFulfillment" not "OrderData"
2. **Include decision fields** - What does the processor check?
3. **Name the processor descriptively** - "Order Fulfillment Processor" not just "Processor"
4. **Include context fields** - The command needs enough data to act
5. **Add generated fields** - Computed values, timestamps
6. **Consider failure cases** - What if the automation fails? (May need separate error events)
7. **Think about idempotency** - Can this safely run multiple times?
