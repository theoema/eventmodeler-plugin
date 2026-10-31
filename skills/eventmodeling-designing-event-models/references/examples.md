# Designing Event Models — Worked Examples

These are conceptual worked examples (Order domain) illustrating the event-model design reasoning this skill applies. They are useful for understanding the reasoning; the actual instructions to follow live in the numbered Workflow sections of the main SKILL.md. Each is the summary you would give in the conversation. On the sheet, the same facts are elements, flows, fields, notes and scenarios.

## 1. Design Events — worked example

```text
Entity: Order (order-123)

Events (chronological):
1. Order Created
   Triggered by: Create Order command
   Data: customerId, items[], shippingAddress, createdAt
   (from command: customerId, items[], shippingAddress)
   (implicit: orderId, createdAt)

2. Order Confirmed
   Triggered by: Confirm Order command
   Data: paymentId, confirmedAt
   (from command: paymentId)
   (implicit: orderId, previous status verified)

3. Order Shipped
   Triggered by: Ship Order command
   Data: shipmentId, shippedAt
   (from command: shipmentId)
   (implicit: orderId, confirmed status verified)
```

The order total is not on `Order Created`. It is calculated from the items, so it is a field of the read model that shows it (`Order Summary View`, below), marked `isGenerated`.

## 2. Design Commands — worked example

```text
Command: Confirm Order
Source: UI or Processor (only these can issue)
Input: orderId, paymentId

Preconditions:
  - order status is 'Draft' (reject: already confirmed)
  - paymentId is valid (reject: invalid payment)

Outcomes:
   Success: Order Confirmed event
     Data: paymentId, confirmedAt
     Implicit: orderId, previous status
   Rejection: error returned, no event created
```

## 3. Design Read Models — worked example

```text
ReadModel: Order Summary View
Purpose: UI displays customer order list, Processor checks order status

Subscribed to events:
  - Order Created
  - Order Confirmed
  - Order Shipped
  - Order Cancelled

Data (optimized for queries):
  {
    orderId: string
    customerId: string
    total: number            (isGenerated: calculated from the items)
    status: string
    createdAt: Date
    confirmedAt?: Date
    shippedAt?: Date
  }

Update from events:
  - Order Created → Insert row (id, customer, total, status='Draft')
  - Order Confirmed → Update status='Confirmed', set confirmedAt
  - Order Shipped → Update status='Shipped', set shippedAt
  - Order Cancelled → Update status='Cancelled'

Consumed by:
  - UI: displays list of orders
  - Processor: checks if order can be shipped
```

## 4. Document Event Causality — worked example

```text
Command Flow:
Create Order command
    → Order Created event
       ↓ (may trigger downstream action)
Confirm Order command (depends on Order Created having happened)
    → Order Confirmed event
       ↓ (may trigger downstream action)
Ship Order command (depends on Order Created + Order Confirmed having happened)
    → Order Shipped event
```

## 5. Document State Transitions — worked example

```text
Order State Transitions:

Initial state: (no order yet)
  ↓
Create Order → Order Created
  ↓
State: Draft

Draft state:
  → Confirm Order → Order Confirmed → State: Confirmed
  → Cancel Order → Order Cancelled → State: Cancelled

Confirmed state:
  → Ship Order → Order Shipped → State: Shipped
  → Cancel Order (rejected - already confirmed)

Shipped state:
  → No more transitions allowed
```

Each arrow is a scenario on the sheet. `Cancel Order (rejected - already confirmed)` is a scenario whose Given is `Order Created` and `Order Confirmed`, whose When is `Cancel Order`, and whose Then is the expected failure.

## Output Format — full worked example

```markdown
# Event Model: [Domain]

## Events

### Entity: Order

**Identity**: orderId

**Events**:
- Order Created: Initial event creating the order
  Data: customerId, items[], shippingAddress

- Order Confirmed: Payment confirmed
  Data: paymentId, confirmedAt

- Order Shipped: Order shipped
  Data: shipmentId, shippedAt

- Order Cancelled: Order cancelled
  Data: cancelledAt, reason

---

## Commands

### Command: Create Order
- Input: customerId, items[], shippingAddress
- Preconditions: Items valid, customerId exists
- Events produced: Order Created
- Possible outcomes: Success (Order Created) or validation error

### Command: Confirm Order
- Input: orderId, paymentId
- Preconditions: Order in Draft status, payment validated
- Events produced: Order Confirmed
- Possible outcomes: Success or "Already confirmed" error

---

## Read Models

### ReadModel: Order Summary View
- Purpose: Quick lookup of order status
- Events: Order Created, Order Confirmed, Order Shipped, Order Cancelled
- Queries served: Get Order (orderId), List Orders By Customer (customerId)
```
