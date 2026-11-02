# Identifying Inputs — Worked Examples

These are conceptual worked examples (Order/Payment domain) illustrating the command-identification reasoning this step applies — useful for understanding the reasoning, but the actual mechanics live in the numbered Workflow sub-steps of the main SKILL.md (placing COMMAND elements on the sheet), not in writing a document like this.

## 1. Extract Commands from UI Actions

```text
Storyboard: Order Creation Screen
User action: Click "Create Order" button
  ↓
Command: Create Order
Input data from form:
    - customerId
    - items[] (product selections + quantities)
    - shippingAddress
Validation:
    - customerId must exist
    - items must not be empty
    - quantities must be > 0
Produces event: Order Created
```

## 2. Identify Processor Triggers

```text
Processor trigger: Payment gateway webhook received
  ↓
Command: Authorize Payment (from Processor, not UI)
Input data from webhook:
    - orderId
    - paymentId
    - authorizationCode
Validation:
    - orderId must exist and be in Confirmed state
    - authorizationCode must be valid
Produces event: Payment Authorized
```

## 2b. Understand the Processor "Todo List" Pattern

```text
Event Stream (Domain events):
Payment Authorized → triggers Inventory system

Processor: Inventory Reserver

Todo List:
When Payment Authorized event arrives:
    1. Add item to todo: "Reserve inventory for order-123"

Processor Logic (continuously):
FOR EACH todo item IN todo_list:
    - Check if inventory available
    - If yes: Reserve inventory, produce Inventory Reserved event, mark done
    - If no: Produce Inventory Failed event, mark failed
    - If error: Keep in todo for retry

Example:
Event: Payment Authorized(orderId=order-123, items=[{prodId: P1, qty: 2}])
    ↓
Todo added: Reserve P1 qty 2
    ↓
Processor checks: P1 has 5 available, need 2 
    ↓
Action: Reserve 2 units
    ↓
Event produced: Inventory Reserved(orderId=order-123, reserved=[...])
    ↓
Todo marked done
```

## 2c. Document Processor Automation (Gears Symbol)

```text
Command Catalog with Role Attribution (from Role Catalog):

UI-Issued Commands (attributed to specific human roles):
  1. Create Order (Order Entry screen) [ Customer]
  2. Confirm Order (Confirmation screen) [ Customer]
  3. Cancel Order (Status screen) [ Customer]
  4. Request Return (Order page) [ Customer]
  5. Override Order Status (Admin panel) [ Support Agent]

Processor-Issued Commands (attributed to system actors):
  6. Authorize Payment (Payment gateway webhook) [ Payment Gateway]
  7. Reserve Inventory (Triggered by Payment Authorized) [ Inventory System]
  8. Create Shipment (Triggered by Inventory Reserved) [ Fulfillment System]
  9. Notify Customer (Triggered by multiple events) [ Notification Service]
```

## 3. Document Command Specifics

```text
Command: Confirm Order
Source: UI (user clicks button)
Input:
    orderId: string (from URL/context)
    paymentMethod: enum ('card' | 'transfer')
    [paymentDetails]: depends on method

Validation rules:
    - Order must exist
    - Order must be in Draft state
    - Payment method must be supported
    - Funds must be available (pre-check)

Preconditions (from stream state):
    - Order Created event exists
    - No Confirm Order previously processed

Success result: Order Confirmed event

Failure results:
    - "Order not found" → Command rejected, no event
    - "Order already confirmed" → Command rejected, no event
    - "Payment method not supported" → Command rejected, no event
```

## 4. Create Command Catalog

```text
Command Catalog: Order System

### UI-Issued Commands

1. Create Order
   Source: User (Order Entry screen)
   Input: customerId, items[], shippingAddress
   Produces: Order Created event

2. Confirm Order
   Source: User (Confirmation screen)
   Input: orderId, paymentMethod
   Produces: Order Confirmed event

3. Cancel Order
   Source: User (Status screen)
   Input: orderId, reason
   Produces: Order Cancelled event

### Processor-Issued Commands

4. Authorize Payment
   Source: Payment Processor (webhook)
   Input: orderId, paymentId, authCode
   Produces: Payment Authorized event

5. Fail Payment
   Source: Payment Processor (webhook)
   Input: orderId, paymentId, reason
   Produces: Payment Failed event

6. Reserve Inventory
   Source: Inventory Processor (triggered by Payment Authorized)
   Input: orderId, items[]
   Produces: Inventory Reserved event

7. Create Shipment
   Source: Fulfillment Processor (triggered by Inventory Reserved)
   Input: orderId, items[]
   Produces: Order Shipped event
```

## 5. Map Data Sources

```text
Command: Confirm Order

Data origin matrix:
  orderId
    ↑ Source: UI context (from Order Created, displayed to user)
    ↑ Captured: Hidden in URL or session
    ↑ Validation: Must match Order from stream

  paymentMethod
    ↑ Source: UI form selection
    ↑ Captured: User selects checkbox/radio
    ↑ Validation: Must be in allowed list

[paymentDetails] (conditional)
    ↑ Source: Depends on paymentMethod
    ↑ For 'card': Card number, CVV, expiry (from payment form)
    ↑ For 'transfer': Bank account, routing number (from form)
    ↑ Validation: Format and validity checks
```

## 6. Identify Implicit Context

```text
Command: Ship Order
Explicit input (from UI/Processor):
    orderId
    shipmentId (from fulfillment system)

Implicit context (from stream state):
    Order must exist
    Order must be in Inventory Reserved state
    Payment must be authorized (from Payment Authorized event)
    Inventory must be reserved (from Inventory Reserved event)

These implicit checks use stream state:
    currentState.orderId === orderId 
    currentState.status === 'InventoryReserved' 
    currentState.paymentId exists 
    currentState.shipmentId can be set 
```

## Legacy markdown-document format (superseded — kept for reference only)

Older versions of this skill wrote the command catalog as a markdown document instead of placing elements on the sheet. The actual mechanism today is the sheet's tools (the "Output Format" and "Creating a COMMAND with fields" sections in the main SKILL.md) — this template is kept only so the shape of the information (what a complete command catalog covers) stays documented somewhere.

```markdown
# Inputs: [Domain Name]

## Commands Summary

| Command | Role/Actor | Source | Trigger | Input | Event |
|---------|------------|--------|---------|-------|-------|
| Create Order | Customer | UI | User action | customerId, items, address | Order Created |
| Confirm Order | Customer | UI | User action | orderId, paymentMethod | Order Confirmed |
| Cancel Order | Customer | UI | User action | orderId, reason | Order Cancelled |
| Authorize Payment | Payment Gateway | Processor | Webhook | orderId, paymentId | Payment Authorized |
| Reserve Inventory | Inventory System | Processor | Payment Authorized event | orderId, items | Inventory Reserved |
| Ship Order | Fulfillment System | Processor | Inventory Reserved event | orderId, shipmentId | Order Shipped |

---

## Detailed Commands

### Command: Create Order

**Source**: User (Order Entry screen)

**Input Data**:
- customerId: string
- items: Array<{productId: string, quantity: number}>
- shippingAddress: {street, city, state, zip}

**Validation**:
- customerId must exist in system
- items array must not be empty
- quantities must be > 0
- address fields must be non-empty

**Preconditions** (from stream state):
- Stream Order:X does not exist yet

**Success**: Produces Order Created event

**Failure**: Command rejected, no event
- "Customer not found"
- "Items invalid"
- "Address incomplete"

--- [Repeat for each command]

---

## Data Completeness Check

### Data Input → Event

Verify every command input becomes event data:

| Command Input | Event Data | Status |
|---------------|-----------|--------|
| customerId | orderId |  Stored in Order Created |
| items | items |  Stored in Order Created |
| shippingAddress | shippingAddress |  Stored in Order Created |

### Missing Data

Document any input that doesn't make it to events:
- None identified 

---

## Processor Commands

Document all processor-triggered commands:
[List each with source system and trigger condition]
```
