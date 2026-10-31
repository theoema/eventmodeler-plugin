# Applying Conway's Law — Worked Examples

These are conceptual worked examples of the team/system-boundary reasoning this skill applies — useful for understanding the reasoning, but the actual analysis steps are the numbered Workflow steps in the main SKILL.md.

## 1. Identify System Boundaries

```
System Boundaries:

Order Management System
  - Owns: Order entity and its lifecycle
  - Events: Order Created, Order Confirmed, Order Cancelled
  - Owns: Order state machine

Payment Processing System
  - Owns: Payment authorization and processing
  - Events: Payment Authorized, Payment Failed
  - Owns: Payment state machine

Inventory System
  - Owns: Stock levels and reservations
  - Events: Inventory Reserved, Inventory Allocated
  - Owns: Inventory state machine

Fulfillment System
  - Owns: Shipments and delivery
  - Events: Order Shipped, Delivery Confirmed
  - Owns: Shipment state machine
```

## 2. Create Swimlane Diagram

```
                Event Stream Timeline
                →

Order Team      Order Created   Order Confirmed   Order Cancelled

Payment Team                    Payment Authorized    Payment Failed

Inventory Team                  Inventory Reserved

Fulfillment Team                Order Shipped    Delivery Confirmed

Each team owns their swimlane events
Coordination via events crossing swimlanes
```

(An analysis diagram. On the sheet only the other systems' handover events get a lane of their own, and a lane named for a system carries a note naming its owning team.)

## 3. Map Team Responsibilities

```
Order Management Team
  Commands they handle:
    - Create Order
    - Confirm Order
    - Cancel Order
  Events they produce:
    - Order Created
    - Order Confirmed
    - Order Cancelled
  Read Models they maintain:
    - Order Status View
    - Order List View
  Systems they call:
    - Payment System (to confirm payment)
    - Inventory System (to check stock)

Payment Processing Team
  Commands they handle:
    - Authorize Payment
    - Process Payment
    - Refund Payment
  Events they produce:
    - Payment Authorized
    - Payment Failed
    - Payment Refunded
  Read Models they maintain:
    - Payment Status View
    - Transaction History
  Systems they depend on:
    - Payment Gateway (external)
    - Order System (for context)

Inventory Team
  Commands they handle:
    - Reserve Inventory
    - Release Reservation
    - Allocate Stock
  Events they produce:
    - Inventory Reserved
    - Reservation Released
    - Stock Allocated
  Read Models they maintain:
    - Inventory Level View
    - Reservation View
  Systems they depend on:
    - Order System (triggers)
    - Warehouse System (stock source)

Fulfillment Team
  Commands they handle:
    - Create Shipment
    - Mark Shipped
    - Confirm Delivery
  Events they produce:
    - Shipment Created
    - Order Shipped
    - Delivery Confirmed
  Read Models they maintain:
    - Shipment Tracking View
    - Delivery Schedule View
  Systems they depend on:
    - Inventory System (items to ship)
    - Carrier APIs (tracking)
```

## 4. Identify Inter-System Communication

Every crossing enters the consuming system through a translation chain (`eventmodeling-designing-automation-chains`): the other system's event opens a translation automation's todo list; its command records the internal event; a worker automation follows only when a genuinely new decision remains.

```
Communication Patterns:

Order System  →  Payment System
  Order System produces: Order Confirmed event (in the Order System's lane)
  Payment System receives it through a translation chain:
    Order Confirmed (Order lane) → Confirmed Orders (todo list) →
    Record Confirmed Order (translation automation + command) → Order Confirmed (Payment System's own lane)
  Payment System reacts: Authorize Payment (worker automation — the gateway call is a real decision)
  Payment System produces: Payment Authorized event

Order System  →  Inventory System
  Payment System produces: Payment Authorized event
  Inventory System receives it through its own translation chain:
    Payment Authorized (Payment lane) → Authorized Payments (todo list) →
    Record Authorized Payment (translation automation + command) → Payment Authorized (Inventory System's own lane)
  Inventory System reacts: Reserve Inventory (worker automation — checks stock)
  Inventory System produces: Inventory Reserved event

Inventory System  →  Fulfillment System
  Inventory System produces: Inventory Reserved event
  Fulfillment System receives it through its own translation chain, then
    reacts: Create Shipment (worker automation — decides how to ship)
  Fulfillment System produces: Order Shipped event
```

## 5. Define System Interfaces

```
Order System Interface
  Commands it accepts:
    - Create Order (from UI)
    - Confirm Order (from UI)
    - Cancel Order (from UI or Processors)
  Events it produces:
    - Order Created
    - Order Confirmed
    - Order Cancelled
  Read Models it provides:
    - Order Status View
    - Order List View

Payment System Interface
  Commands it accepts:
    - Authorize Payment (from its Payment Processor)
    - Process Payment (from its Payment Processor)
    - Record Confirmed Order (from its translation automation)
  Events it produces:
    - Payment Authorized
    - Payment Failed
    - Payment Processed

Inventory System Interface
  Commands it accepts:
    - Reserve Inventory (from its Inventory Processor, reading its todo list)
    - Record Authorized Payment (from its translation automation)
  Events it produces:
    - Inventory Reserved
    - Inventory Failed
```

## 6. Identify Processors vs Systems

```
Processors (autonomous automation):

1. Payment Processor
   Todo list opened by: Order Confirmed (internal, recorded by the translation automation)
   Todo list closed by: Payment Authorized, Payment Failed
   Logic: Calls external payment gateway
   Produces: Authorize Payment command
   Lives in: Payment System

2. Inventory Processor
   Todo list opened by: Payment Authorized (internal, recorded by the translation automation)
   Todo list closed by: Inventory Reserved, Inventory Failed
   Logic: Checks stock, reserves inventory
   Produces: Reserve Inventory command
   Lives in: Inventory System

3. Fulfillment Processor
   Todo list opened by: Inventory Reserved (internal, recorded by the translation automation)
   Todo list closed by: Shipment Created
   Logic: Creates shipment records
   Produces: Create Shipment command
   Lives in: Fulfillment System

4. Notification Processor
   Todo list opened by: Order Created, Order Confirmed, Order Shipped (internal)
   Todo list closed by: Notification Sent
   Logic: Sends emails/SMS
   Produces: Send Notification command
   Lives in: Notification System (cross-cutting)
```

## Output Format — full worked example (Order/Payment/Inventory/Fulfillment domain)

````markdown
# System Organization: [Domain Name]

## System Boundaries

### System: Order Management
- **Team**: Order Team
- **Responsibilities**: Create, confirm, cancel orders
- **Commands**: Create Order, Confirm Order, Cancel Order
- **Events Produced**: Order Created, Order Confirmed, Order Cancelled
- **Events Consumed**: Payment Authorized, Inventory Reserved (for state updates)
- **Read Models**: Order Status View, Order List View
- **Scope**: One stream type (Order)

### System: Payment Processing
- **Team**: Payment Team
- **Responsibilities**: Authorize and process payments
- **Commands**: Authorize Payment, Process Payment, Refund Payment
- **Events Produced**: Payment Authorized, Payment Failed, Payment Refunded
- **Events Consumed**: Order Confirmed (from Order System)
- **Read Models**: Payment Status View, Transaction History
- **Dependencies**: External payment gateway
- **Scope**: One stream type (Payment)

### System: Inventory Management
- **Team**: Inventory Team
- **Responsibilities**: Stock management and reservations
- **Commands**: Reserve Inventory, Release Reservation, Allocate Stock
- **Events Produced**: Inventory Reserved, Reservation Released, Stock Allocated
- **Events Consumed**: Payment Authorized (from Payment System)
- **Read Models**: Inventory Level View, Reservation View
- **Dependencies**: Warehouse system
- **Scope**: One stream type (Inventory Reservation)

[Continue for each system]

---

## Event Flow Across System Boundaries

### Flow: Order → Payment → Inventory → Fulfillment

```
Time →

Order System
Order Created →
Order Confirmed →
                              (triggers)
Payment System
Payment Authorized →
                       (triggers)
Inventory System
Inventory Reserved →
                       (triggers)
Fulfillment System
Order Shipped
Delivery Confirmed
```

---

## Team Responsibilities Matrix

| Team | Creates Commands | Produces Events | Owns Read Models |
|------|-----------------|-----------------|------------------|
| Order | Create Order, Confirm Order | Order Created, Order Confirmed | Order Status View |
| Payment | Authorize Payment | Payment Authorized | Payment Status View |
| Inventory | Reserve Inventory | Inventory Reserved | Inventory Level View |
| Fulfillment | Create Shipment | Order Shipped | Shipment Tracking View |

---

## Inter-System Communication

### Order → Payment
- Trigger: Order Confirmed event (Order System's lane)
- Received by: Payment System's translation chain (Confirmed Orders → Record Confirmed Order)
- Result: Authorize Payment command issued by the Payment Processor

### Payment → Inventory
- Trigger: Payment Authorized event (Payment System's lane)
- Received by: Inventory System's translation chain (Authorized Payments → Record Authorized Payment)
- Result: Reserve Inventory command issued by the Inventory Processor

[Document all communication patterns]

---

## Dependencies

### External Systems

| System | Owns | Called By | Purpose |
|--------|------|-----------|---------|
| Payment Gateway | Payment provider | Payment System | Authorization |
| Warehouse | Inventory source | Inventory System | Stock info |
| Carrier API | Shipping | Fulfillment System | Tracking |

---

## Independent Development

Each system can:
- Develop independently
- Use different tech stacks
- Scale independently
- Deploy independently
- Own their events
- Maintain their read models

Coordination via:
- Events (async messaging)
- Processors (listen and react, through translation chains)
- Read models (shared views)
````
