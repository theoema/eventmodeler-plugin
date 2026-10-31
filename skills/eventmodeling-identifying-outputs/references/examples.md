# Identifying Outputs — Worked Examples

These are conceptual worked examples of the read-model design thinking this step applies — useful for understanding the reasoning, but the actual mechanics live in the numbered Step 5a–5i sections of the main SKILL.md (placing elements on the sheet), not in writing a document like this.

## Conceptual walkthrough — mapping events to a read model (Order domain)

### 1. Map Event Data to UI Screens

```
Screen: Order Status View
Displays data from events:
  orderId ← Order Created event
  customerId ← Order Created event
  items ← Order Created event
  total ← Order Created event
  status ← Order Confirmed event (or Order Cancelled)
  confirmedAt ← Order Confirmed event
  paymentId ← Payment Authorized event
  shipmentId ← Order Shipped event
  shippedAt ← Order Shipped event

This screen is a projection of these events:
  - Order Created
  - Order Confirmed
  - Payment Authorized
  - Order Shipped
```

### 2. Define Read Models

```
ReadModel: Order Status View
Purpose: UI displays current order status
Events subscribed: Order Created, Order Confirmed, Payment Authorized, Order Shipped, Order Cancelled
Data:
{
  orderId: String (from Order Created)
  customerId: String (from Order Created)
  status: String (from events: Draft → Confirmed → Authorized → Shipped → Delivered)
  createdAt: DateTime (from Order Created)
  confirmedAt: DateTime (from Order Confirmed)
  paymentId: String (from Payment Authorized)
  shipmentId: String (from Order Shipped)
  shippedAt: DateTime (from Order Shipped)
}
```

(Every one of those lines is a field on the read model with a source: the flow from the named event, same-named or mapped with `set_flow_mapping`. A line that names no event would be `isGenerated`, or a gap.)

### 3. Document Event → Data Mapping

```
Event: Order Created
Provides to UI/Processors:
  orderId
  customerId
  items[]
  total
  shippingAddress
  createdAt

Event: Order Confirmed
Provides to UI/Processors:
  orderId (link to stream)
  paymentMethod (user selected method)
  confirmedAt (timestamp)
  paymentId (payment system reference)

Event: Payment Authorized
Provides to UI/Processors:
  orderId (link to stream)
  paymentId
  authCode
  authorizedAt (timestamp)
  amount (verified amount)

Event: Order Shipped
Provides to UI/Processors:
  orderId (link to stream)
  shipmentId
  shippedAt (timestamp)
  carrier (shipping company)
  trackingNumber (for delivery tracking)
```

### 4. Create Output Catalog

```
ReadModel Catalog: Order System

1. Order Status View
   Purpose: UI shows current order status
   Events: Order Created, Order Confirmed, Payment Authorized, Order Shipped, Order Cancelled
   Data: orderId, status, createdAt, confirmedAt, paymentId, shipmentId
   Consumed by:
     - Order Status screen (UI)
     - Customer Dashboard (UI)

2. Order List View   (list-shaped)
   Purpose: UI lists all orders for a customer
   Events: Order Created, Order Confirmed, Order Cancelled
   Data: orderId, customerId, total, status, createdAt
   Consumed by:
     - Customer Order History (UI)
     - Order Search/Filter (UI)

3. Payment Status View
   Purpose: UI shows payment status
   Events: Order Confirmed, Payment Authorized, Payment Failed
   Data: orderId, paymentId, status, authCode, failureReason, timestamp
   Consumed by:
     - Payment Status screen (UI)

4. Shipment Tracking View
   Purpose: UI shows tracking information
   Events: Order Shipped, Delivery Confirmed
   Data: orderId, shipmentId, trackingNumber, carrier, shippedAt, estimatedDelivery
   Consumed by:
     - Order Tracking screen (UI)

(The automations below are not consumers of these screen-facing read models: each has its own todo list — see section 6.)
```

### 5. Identify Missing Data

```
Question: What if UI needs "estimated delivery date"?
Event: Order Shipped has carrier + trackingNumber
Action needed: Add estimatedDelivery to the Order Shipped event
  (or mark it isGenerated on the read model, if it is computed)

Question: What if UI needs to show "payment method" on status?
Event: Order Confirmed has paymentMethod
Action needed: Include paymentMethod in the relevant read models

Question: What if UI needs "item descriptions"?
Event: Order Created has items[]
But: items[] only has productId
Action needed: The product descriptions must come from some event on the sheet
  (a product catalog event feeding the read model) — otherwise the field has no source and is a gap
```

### 6. Processor Outputs

Each automation reads its own todo list (`eventmodeling-designing-automation-chains`): outstanding work, opened and closed by events — not a status column on an entity view.

```
Automation: Inventory Reserver
Reads todo list: Orders Awaiting Reservation (list-shaped)
  opened by:  Payment Authorized
  closed by:  Inventory Reserved
Issues command:
  - Reserve Inventory

Automation: Fulfillment Creator
Reads todo list: Orders Awaiting Shipment (list-shaped)
  opened by:  Inventory Reserved
  closed by:  Shipment Created
Issues command:
  - Create Shipment

Automation: Notification Sender
Reads todo list: Notifications To Send (list-shaped)
  opened by:  Order Created, Order Confirmed, Order Shipped, Delivery Confirmed
  closed by:  Notification Sent
Issues command:
  - Send Notification
```

## Shape of a complete read model catalog

The same information a catalog states — kept on the sheet as elements, fields, flows and notes, and presented as text at the end of the step.

```markdown
# Outputs: [Domain Name]

## Read Models Summary

| ReadModel | Purpose | Events | Consumed By |
|-----------|---------|--------|-------------|
| Order Status | Show order state | Order Created, Order Confirmed | UI |
| Order List | List orders | Order Created, Order Cancelled | UI |
| Payment Status | Payment info | Order Confirmed, Payment Authorized | UI |
| Shipment Tracking | Track delivery | Order Shipped, Delivery Confirmed | UI |

---

## Detailed Read Models

### ReadModel: Order Status View

**Purpose**: Order Status screen displays current order state

**Events subscribed**:
- Order Created
- Order Confirmed
- Payment Authorized
- Order Shipped
- Order Cancelled
- Delivery Confirmed

**Data**:
```
{
  orderId: String
  customerId: String
  status: String ('Draft' | 'Confirmed' | 'Authorized' | 'Shipped' | 'Delivered' | 'Cancelled')
  items: list of {productId, quantity, unitPrice}
  total: Decimal
  shippingAddress: Custom

  createdAt: DateTime
  confirmedAt: DateTime
  paymentId: String
  paymentMethod: String ('card' | 'transfer')
  authorizedAt: DateTime

  shipmentId: String
  carrier: String
  trackingNumber: String
  shippedAt: DateTime
  estimatedDelivery: DateTime
}
```

**Update Logic**:
- Order Created: Insert with status='Draft'
- Order Confirmed: Update status='Confirmed'
- Payment Authorized: Update status='Authorized', set paymentId
- Order Shipped: Update status='Shipped', set shipmentId, carrier, trackingNumber
- Delivery Confirmed: Update status='Delivered'
- Order Cancelled: Update status='Cancelled'

**Consumed By**:
- Order Status Screen (displays)

--- [Repeat for each read model]

---

## Data Completeness Check

### Events → UI Needs

Verify all UI needs have event sources:

| UI Need | Event Source | Status |
|---------|-------------|--------|
| Order status | Order Confirmed, Order Shipped | covered |
| Tracking number | Order Shipped | covered |
| Order items | Order Created | covered |
| Estimated delivery | Order Shipped | covered |
| Cancellation reason | Order Cancelled | covered |

### Missing Data

Identify UI needs without event sources:
- None identified

---

## Processor Consumption

### Automations and their todo lists:

| Automation | Todo list (opened by → closed by) | Issues command |
|------------|-----------------------------------|----------------|
| Inventory Reserver | Orders Awaiting Reservation (Payment Authorized → Inventory Reserved) | Reserve Inventory |
| Fulfillment Creator | Orders Awaiting Shipment (Inventory Reserved → Shipment Created) | Create Shipment |
| Notification Sender | Notifications To Send (several → Notification Sent) | Send Notification |
```
