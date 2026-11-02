# Timeline Boundary Design Patterns

## Contents
- Entity Boundary Design Patterns (5 patterns with examples)
- Timeline Boundary Decision Tree
- Red Flags: Boundary Is Wrong
- Tips for Finding the Right Boundary

---

## Entity Boundary Design Patterns

### Pattern 1: Single Entity (Most Common)

CORRECT: One timeline boundary per business entity
```
Entity: Order
Root Identity: orderId (e.g., 'order-123')
Lifetime: 1-2 years

Events in timeline:
  1. Order Created (2024-01-15)
  2. Order Line Added (2024-01-15)
  3. Order Line Added (2024-01-15)
  4. Order Confirmed (2024-01-16)
  5. Payment Processed (2024-01-16)
  6. Order Shipped (2024-01-20)
  7. Order Delivered (2024-01-25)

Identity Principle: orderId is the natural business key
Boundary: Everything about THIS specific order, nothing else
```

---

### Pattern 2: Composite Entity (Proper Composition)

CORRECT: Entity contains related child entities
```
Entity: Order
Root Identity: orderId (e.g., 'order-456')

Contains related children (same lifetime):
  - OrderLines: 3 items
    * Line 1: productId=prod-A, qty=2, price=$50
    * Line 2: productId=prod-B, qty=1, price=$100
    * Line 3: productId=prod-C, qty=5, price=$10

  - ShippingAddress:
    street: 123 Main St, City: Portland, State: OR

  - PaymentInfo:
    method: credit_card, amount: $400

Events in timeline:
  1. Order Created (customer-789, 3 items)
  2. Order Line Added (item 1)
  3. Order Line Added (item 2)
  4. Order Line Added (item 3)
  5. Order Confirmed (payment method selected)
  6. Payment Processed (authorization complete)
  7. Order Shipped (tracking 123456)

Pattern: Small, bounded number of children per parent
Lifetime: Parent and all children created/destroyed together
Together: they change as a unit (can't ship without payment, etc.)
```

---

### Pattern 3: Collection (ANTI-PATTERN - DO NOT USE)

WRONG: Treating a collection as an entity
```
Bad Entity: All Orders
Root Identity: "all-orders-collection" (artificial, meaningless)

Contains: Every order ever created
Events:
    1. Order Created (customer-001, order-001)
    2. Order Created (customer-002, order-002)
    3. Order Created (customer-001, order-003)
    4. Order Created (customer-003, order-004)
    ... (continues forever, unbounded)

Problems with this approach:
  - No single business identity (it's a collection, not an entity)
  - Timeline grows unbounded — no natural end to its lifetime

Solution: Use a projection/read model query instead, not an entity
  - Query: "All Orders By Customer" (customer-id)
  - Query: "Orders By Status" (status)
  - Rebuild from individual Order timelines on-demand
```

---

### Pattern 4: Event Log (ANTI-PATTERN - DO NOT USE)

WRONG: Using an entity as an event log
```
Bad Entity: System Log
Root Identity: "system-log" (meaningless placeholder)

Contains: Every system event imaginable
Events:
    1. User Logged In (user-123)
    2. Order Created (order-456)
    3. Payment Processed (payment-789)
    4. Inventory Updated (sku-101)
    5. User Logged Out (user-123)
    6. User Logged In (user-223)
    ... (grows indefinitely, no pattern)

Problems with this approach:
  - No business identity (log of everything)
  - Events unrelated to each other (mixing user, order, payment, inventory)
  - Can't answer "what's the state of X?" (too mixed)

Solution: Use separate event timelines per business entity
  - Keep dedicated event timelines: Order, Payment, Inventory, User
```

---

### Pattern 5: Historical Entity (GOOD - When Needed)

CORRECT: Keep historical data for audit/compliance as its own boundary
```
Entity: Archived Order
Root Identity: archivedOrderId (e.g., 'archived-order-001')
Purpose: Regulatory compliance (7-year retention)

Contains: A record + audit trail of an order
Events:
    1. Order Archived (original order-123 on 2023-12-31)
       - reason: compliance_retention
       - originalData: { id, customerId, items, total, dates }

    2. Audit Log Added (accessed by accounting, 2024-01-15)
       - accessor: accounting@company.com
       - action: viewed for tax audit

    3. Audit Log Added (accessed by auditor, 2024-02-01)
       - accessor: auditor@firm.com
       - action: reviewed for compliance

  ... (additional audit entries over time)

Lifetime: 7 years (regulatory requirement)

Key principles:
  - Completely separate from the active Order entity
  - Active Order is for current business operations
  - Archived Order is an immutable historical record
  - Different access patterns, different lifecycles
```

---

## Timeline Boundary Decision Tree

Use this to decide whether a timeline is bounded around the right business identity:

```
Does your timeline have a natural business identity?
 NO → This is not an entity, it's a log/report
    SOLUTION: Use a read model/projection, not an entity

 YES → Does every event in the timeline belong to that one entity's lifecycle?
   
    YES → GOOD: Boundary is correctly scoped to one identity
   
    NO → The boundary is too wide — it's absorbing events that
         belong to a different entity or a different concern
        REDESIGN: Split by the entity each event actually concerns
          Examples: User → User Profile + User Sessions
                    Order → Order + Order Line Items (if line items
                    have their own independent lifecycle)
```

---

## Red Flags: Boundary Is Wrong

If your timeline exhibits ANY of these, the fix is a narrower or different identity — not a technical workaround:

```
Red Flag 1: Timeline growing continuously with no natural end
   Cause: The timeline identity spans an unbounded population, not one entity
   Solution: Re-scope to a single business entity's lifecycle
   Example: "All Orders" timeline → "Order" per customer order

Red Flag 2: Events with no shared business meaning
   Cause: Treating a log as an entity
   Solution: Use a read model/query instead of an entity
   Example: "System Metric Recorded" → use a metrics/observability system

Red Flag 3: Timeline contains unrelated entities
   Cause: Entity boundary is wrong
   Solution: Split into separate entities
   Example: "All Orders" → "Order" per customer

Red Flag 4: Can't explain what single business question the timeline answers
   Cause: Not a real entity — probably a collection or log
   Solution: Convert to a read model/projection
   Example: "System Events" → query specific per-entity timelines instead
```

---

## Tips for Finding the Right Boundary

### 1. Anchor on a Single Business Identity
```
Every timeline should answer: "the history of exactly which entity?"
If the answer is "a category of things" or "everything," the
boundary is wrong — it's describing a collection, not an entity.
```

### 2. Understand Event Granularity
```
RIGHT: One event per meaningful state change
WRONG: Multiple events per semantic operation
Example: "User Updated Profile" (1 event)
NOT: "First Name Changed", "Last Name Changed", ... (N events)
```

### 3. Split When the Identity Is Actually Two Identities
```
All Orders (growing unbounded, no single identity)
→ Order (per order)
→ Order Line (per line item, if it has its own independent lifecycle)

User Account (everything about a user, several unrelated concerns)
→ User Profile (personal info)
→ User Preferences (settings)
→ User Sessions (login history)
```

### 4. Separate Active From Historical Concerns
```
Keeping everything in one entity forever conflates two different
lifecycles: the entity while it's active, and its record afterward.
Example:
  - Active Subscription (current state)
  - Archived Subscription (after cancelled)
```
