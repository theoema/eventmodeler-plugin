---
name: create-automation-slice
description: Create a complete automation slice with Processor, Command, Event, and flows. Use when system logic automatically triggers a state change in response to events.
---

# Creating Automation Slices

An automation slice represents a **system-initiated action** (saga, policy, reactor):

```
Processor → Command → Event
```

- **Processor**: The system component that decides to act (saga, policy, background job)
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
2. **Field mappings**: Automatically inferred by matching field names (Command → Event)
3. **Flows**: Automatically creates Processor→Command and Command→Event flows

Note: Processors don't have fields, so there's no Processor→Command field mapping.

## Example: Auto-Fulfill Order

```bash
eventmodeler create automation-slice --xml '<automation-slice name="Auto Fulfill Order" after="Place Order">
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

### 1. Identify the Trigger
What event(s) cause this automation to run?
- The processor "listens" to events and decides when to act
- This connection (Event→Processor) is set up separately via cross-slice flows

### 2. Design the Fields
- **Command fields**: What data the processor needs to act
- **Event fields**: What facts are recorded (include generated values)

### 3. Propose Before Creating
```
I'll create an "Auto Fulfill Order" slice:

Processor: Order Fulfillment Processor
- Triggers when an order is placed and payment confirmed

Command (FulfillOrder):
- orderId (UUID) - which order to fulfill
- warehouseId (UUID, generated) - selected warehouse

Event (OrderFulfilled):
- orderId, warehouseId (from command)
- fulfilledAt (DateTime, generated) - when fulfillment started

Does this look right?
```

### 4. Create the Slice
After approval, run the command.

### 5. Verify
```bash
eventmodeler show slice "Auto Fulfill Order"
```

## Connecting to Trigger Events

After creating the automation slice, you'll typically want to show what events trigger it. This is done by creating flows from events/read models to the processor (handled separately, not part of this command).

## Best Practices

1. **Name the processor descriptively** - "Order Fulfillment Processor" not just "Processor"
2. **Include context fields** - The command needs enough data to act
3. **Add generated fields** - Computed values, timestamps
4. **Consider failure cases** - What if the automation fails? (May need separate error events)
5. **Think about idempotency** - Can this safely run multiple times?
