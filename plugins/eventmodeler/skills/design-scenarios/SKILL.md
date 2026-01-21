---
name: design-scenarios
description: Help design and add scenarios (Given-When-Then specifications) to slices. Use when a user wants to specify behavior for a slice, add test cases, or define invariants.
---

# Designing Scenarios for Slices

Scenarios define the behavior of a slice using Given-When-Then format. They specify what happens when a command is executed under certain conditions.

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

### 2. Understand the Slice Structure
From the slice output, identify:
- **The Command** - What action is being performed?
- **The Event(s)** - What facts are recorded when it succeeds?
- **Existing scenarios** - What's already specified?
- **Fields** - What data flows through this slice?

### 3. Design Scenarios
Think about these scenario types:

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

**Edge Cases** - Boundary conditions, empty lists, etc.

### 4. Add Scenarios with Example Field Values

Always include realistic example field values and a description explaining the scenario's purpose.

```bash
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

For error scenarios:
```bash
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

For read model assertions:
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

## If the User Mentions a Specific Invariant

When the user says something like "add a scenario where you can't cancel a shipped order":

1. Identify the slice (probably "Cancel Order" or similar)
2. Look at what events exist (e.g., OrderShipped)
3. Create the scenario directly:

```bash
eventmodeler add scenario --slice "Cancel Order" --xml '<scenario name="Cannot cancel shipped order" description="Once an order has been shipped, it can no longer be cancelled - customers must use the returns process instead">
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

## Best Practices

1. **Write clear descriptions** - Explain why this scenario matters and what business rule it captures
2. **Use realistic example values** - "cust-123" not "string", "2024-01-15" not "date"
3. **Name scenarios descriptively** - "Cannot cancel shipped order" not "Error case 1"
4. **Include all relevant fields** - Don't skip fields that matter for the scenario
5. **Think about state** - What events need to exist for this scenario to make sense?
6. **Cover the negative cases** - These often reveal missing business rules
