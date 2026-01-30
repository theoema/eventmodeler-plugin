---
name: design-scenarios
description: Help design and add scenarios (Given-When-Then specifications) to slices. Use when a user wants to specify behavior for a slice, add test cases, or define invariants.
---

# Designing Scenarios for Slices

Scenarios define the behavior of a slice. There are two main scenario patterns:

1. **State-Change Scenarios** (for state-change slices): Given events → When command → Then events/error
2. **Automation Scenarios** (for automation slices): Given events → Then command (simpler Given-Then format)

## Your Workflow

When helping design scenarios for a slice:

### 1. Gather Context
```bash
# Look at the slice to understand its components
eventmodeler show slice "<slice-name>"

# See what chapter it belongs to and nearby slices for context
eventmodeler list chapters
eventmodeler show chapter "<chapter-name>"

# List all events to understand what's available for Given clauses
eventmodeler list events
```

### 2. Identify the Slice Type

**State-Change Slice** (Screen → Command → Event):
- User triggers a command
- Command produces event(s)
- Use: Given → When command → Then events/error

**Automation Slice** (ReadModel → Processor → Command → Event):
- Events trigger the processor to issue a command
- Use simpler Given → Then format (no When)
- Given: All events that lead to the trigger condition
- Then: The command that should be dispatched (or noCommand)

### 3. Design Scenarios by Type

---

## State-Change Scenarios

For slices where a user action triggers a command:

**Happy Path** - The command succeeds under normal conditions
- Given: Any required preconditions (existing events)
- When: The command with valid input
- Then: The expected event(s) with field values

**Validation Errors** - Bad input is rejected
- Given: (may be empty)
- When: Command with invalid input
- Then: Error with message

**Business Rule Violations** - Valid input but rule prevents action
- Given: Events that create a state where the action isn't allowed
- When: The command
- Then: Error explaining why

### State-Change Examples

```bash
# Happy path - command succeeds
eventmodeler add scenario --slice "<slice-name>" --xml '<scenario name="Successfully place order" description="A registered customer can place an order with valid items">
  <given>
    <event name="CustomerRegistered">
      <field name="customerId">cust-123</field>
    </event>
  </given>
  <when>
    <command name="PlaceOrder">
      <field name="customerId">cust-123</field>
      <field name="items">[{"productId": "prod-1", "quantity": 2}]</field>
    </command>
  </when>
  <then type="events">
    <event name="OrderPlaced">
      <field name="orderId">order-456</field>
      <field name="customerId">cust-123</field>
    </event>
  </then>
</scenario>'
```

```bash
# Validation error
eventmodeler add scenario --slice "<slice-name>" --xml '<scenario name="Cannot place empty order" description="Orders must contain at least one item - empty item lists are rejected">
  <when>
    <command name="PlaceOrder">
      <field name="customerId">cust-123</field>
      <field name="items">[]</field>
    </command>
  </when>
  <then type="error" errorType="ValidationError">Order must contain at least one item</then>
</scenario>'
```

```bash
# Business rule violation
eventmodeler add scenario --slice "Cancel Order" --xml '<scenario name="Cannot cancel shipped order" description="Once an order has been shipped, it can no longer be cancelled">
  <given>
    <event name="OrderPlaced">
      <field name="orderId">order-123</field>
    </event>
    <event name="OrderShipped">
      <field name="orderId">order-123</field>
      <field name="shippedAt">2024-01-15T10:00:00Z</field>
    </event>
  </given>
  <when>
    <command name="CancelOrder">
      <field name="orderId">order-123</field>
    </command>
  </when>
  <then type="error" errorType="BusinessRuleViolation">Cannot cancel an order that has already shipped</then>
</scenario>'
```

---

## Automation Scenarios

For slices where events trigger a processor to issue a command. Use the simpler **Given-Then** format:

**Trigger Conditions Met** - Events cause processor to act
- Given: All the events that have occurred (including the trigger events)
- Then: The command the processor issues

**Trigger Conditions Not Met** - Events don't cause action
- Given: Events that don't meet the trigger condition
- Then: noCommand (processor should not dispatch)

### Automation Examples

```bash
# Payment received triggers shipment initiation
# Given: OrderPlaced + PaymentReceived → Then: InitiateShipment command
eventmodeler add scenario --slice "Auto Ship Order" --xml '<scenario name="Payment triggers shipment" description="Given an order has been placed and payment received, the system automatically initiates shipment">
  <given>
    <event name="OrderPlaced">
      <field name="orderId">order-123</field>
      <field name="customerId">cust-456</field>
    </event>
    <event name="PaymentReceived">
      <field name="orderId">order-123</field>
      <field name="amount">99.99</field>
    </event>
  </given>
  <then type="command">
    <command name="InitiateShipment">
      <field name="orderId">order-123</field>
      <field name="warehouseId">wh-001</field>
    </command>
  </then>
</scenario>'
```

```bash
# Multiple events from different contexts trigger automation
eventmodeler add scenario --slice "Send Order Confirmation" --xml '<scenario name="Order and payment trigger confirmation email" description="Given a customer is registered, order placed, and payment received, send confirmation email">
  <given>
    <event name="CustomerRegistered">
      <field name="customerId">cust-123</field>
      <field name="email">customer@example.com</field>
    </event>
    <event name="OrderPlaced">
      <field name="orderId">order-456</field>
      <field name="customerId">cust-123</field>
    </event>
    <event name="PaymentReceived">
      <field name="orderId">order-456</field>
      <field name="amount">150.00</field>
    </event>
  </given>
  <then type="command">
    <command name="SendOrderConfirmation">
      <field name="orderId">order-456</field>
      <field name="email">customer@example.com</field>
    </command>
  </then>
</scenario>'
```

```bash
# Negative case - conditions not met, no command dispatched
eventmodeler add scenario --slice "Auto Ship Order" --xml '<scenario name="No shipment without payment" description="Given only an order placed (no payment), the system should not initiate shipment">
  <given>
    <event name="OrderPlaced">
      <field name="orderId">order-123</field>
      <field name="customerId">cust-456</field>
    </event>
  </given>
  <then type="noCommand"/>
</scenario>'
```

---

## Read Model Assertions

For state-view slices, test that events project correctly to read models:

```bash
eventmodeler add scenario --slice "<slice-name>" --xml '<scenario name="Order appears in summary" description="After placing an order, it should be visible in the order summary with pending status">
  <then type="readModelAssertion">
    <read-model name="OrderSummary">
      <field name="orderId">order-456</field>
      <field name="status">pending</field>
    </read-model>
  </then>
</scenario>'
```

---

## Scenario Types Summary

| Slice Type | Format | Then Contains |
|------------|--------|---------------|
| State-Change | Given → When (Command) → Then | Events or Error |
| Automation | Given (Events) → Then | Command or noCommand |
| State-View | Given → Then | Read Model Assertion |

## JSON Format Alternative

You can also use JSON format:

**State-Change Scenario:**
```bash
eventmodeler add scenario --slice "<slice-name>" --json '{
  "name": "Successfully place order",
  "description": "A registered customer can place an order",
  "given": [
    { "event": "CustomerRegistered", "fieldValues": { "customerId": "cust-123" } }
  ],
  "when": {
    "command": "PlaceOrder",
    "commandFieldValues": { "customerId": "cust-123", "items": [] }
  },
  "then": {
    "type": "events",
    "events": [
      { "event": "OrderPlaced", "fieldValues": { "orderId": "order-456" } }
    ]
  }
}'
```

**Automation Scenario (Given-Then format):**
```bash
eventmodeler add scenario --slice "<slice-name>" --json '{
  "name": "Payment triggers shipment",
  "description": "Given order placed and payment received, shipment is initiated",
  "given": [
    { "event": "OrderPlaced", "fieldValues": { "orderId": "order-123" } },
    { "event": "PaymentReceived", "fieldValues": { "orderId": "order-123", "amount": 99.99 } }
  ],
  "then": {
    "type": "command",
    "command": "InitiateShipment",
    "commandFieldValues": { "orderId": "order-123" }
  }
}'
```

**Automation Scenario (negative case):**
```bash
eventmodeler add scenario --slice "<slice-name>" --json '{
  "name": "No shipment without payment",
  "description": "Given only order placed (no payment), no command dispatched",
  "given": [
    { "event": "OrderPlaced", "fieldValues": { "orderId": "order-123" } }
  ],
  "then": {
    "type": "noCommand"
  }
}'
```

## Best Practices

1. **Write clear descriptions** - Explain why this scenario matters and what business rule it captures
2. **Use realistic example values** - "cust-123" not "string", "2024-01-15" not "date"
3. **Name scenarios descriptively** - "Cannot cancel shipped order" not "Error case 1"
4. **Include all relevant fields** - Don't skip fields that matter for the scenario
5. **Think about state** - What events need to exist for this scenario to make sense?
6. **Cover the negative cases** - These often reveal missing business rules (use `noCommand` for automations)
7. **Match format to slice type** - Use Given-When-Then for state-change, Given-Then for automation
