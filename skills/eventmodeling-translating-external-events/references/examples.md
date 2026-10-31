# Translating External Events — Worked Examples

These are conceptual worked examples of the translation reasoning this skill applies — useful for understanding the reasoning, but the actual mechanics are the numbered Workflow steps in the main SKILL.md. Each translation ends up as a translation chain on the sheet (`eventmodeling-designing-automation-chains`).

## 1. Identify External Event Sources

```
External System: Payment Gateway (Stripe)

Events received:
  - charge.succeeded
  - charge.failed
  - charge.refunded
  - charge.dispute.created

Example payload: charge.succeeded
{
  "id": "ch_1234567890",
  "amount": 15000,
  "currency": "usd",
  "customer": "cus_9876543210",
  "status": "succeeded",
  "created": 1640995200
}

External System: GPS Location Service (Google Maps)

Events received:
  - location_update
  - geofence_enter
  - geofence_exit

Example payload: geofence_exit
{
  "userId": "user-123",
  "geoFenceId": "hotel-front-entrance",
  "timestamp": 1640995200,
  "latitude": 40.7128,
  "longitude": -74.0060
}
```

On the sheet: a swimlane lane "Payment Gateway" holding an event `Charge Succeeded` with fields `id`, `amount`, `currency`, `customer`, `status`, `created`.

## 2. Analyze Technical Representation

```
External Event: charge.succeeded (Stripe)

Technical fields:
  - id: UUID of charge in Stripe (not meaningful to us)
  - amount: Integer cents (15000 = $150.00)
  - currency: ISO code ("usd")
  - customer: Stripe customer ID (not our customer ID)
  - status: String indicating success
  - created: Unix timestamp

Problems with using directly:
  - We don't use Stripe customer IDs (we have our own customer IDs)
  - Currency and amount require interpretation
  - Status is one field in their model, we care about the fact it succeeded
  - Stripe charge ID isn't the same as our order ID
  - We need to correlate back to our Order stream
```

## 3. Define Domain Translation Rules

```
Translation: External charge.succeeded → Domain Payment Authorized

Mapping rules:
  charge.id (Stripe) → paymentGatewayRef (keep for reconciliation, don't use as primary)
  charge.customer (Stripe) → Look up: Which of OUR customers has this Stripe ID?
  charge.amount → paymentAmount (convert from cents)
  charge.currency → paymentCurrency
  created → timestamp
[NEED TO FIND] → orderId (Stripe doesn't tell us! This is critical—how do we know which order?)

Problem identified:
Stripe webhook comes with charge details but NOT our order ID.

Solutions:
A. Record the Stripe charge ID against our Order when we initiate payment
     When webhook arrives: charge.id → find the order it was recorded against
     Find orderId → record Payment Authorized

B. Store custom metadata in Stripe charge
     When creating charge: Include our orderId in metadata
     When webhook arrives: Extract orderId from metadata

Choose A or B based on Stripe integration approach.
```

In the model, the mapping is declared on the flow from the external event into the todo-list read model:

```
place_element { "cell": "C2", "element": { "scope": "readmodel", "name": "Charges To Record",
  "fields": [
    {"name": "paymentGatewayRef", "fieldType": "String", "example": "ch_1234567890", "source": {"from": "Charge Succeeded", "field": "id"}},
    {"name": "paymentAmount",     "fieldType": "Decimal", "example": "150.00",        "source": {"from": "Charge Succeeded", "field": "amount"}},
    {"name": "paymentCurrency",   "fieldType": "String", "example": "usd",           "source": {"from": "Charge Succeeded", "field": "currency"}},
    {"name": "orderId",           "fieldType": "String", "example": "order-123",     "source": {"from": "Payment Initiated"}}
  ] } }
```

`Payment Initiated` is the internal event of section 4 — the fact that carries the correlation. `amount` (cents) becoming `paymentAmount` (a decimal amount) is a conversion decision; state it in the translation note on the automation's cell.

## 4. Handle Correlation

```
Pattern: Correlation via Reference Tracking

Our system flow:
  1. Order created in our system: order-123
  2. We initiate payment with Stripe:
     - Send amount, customer info
     - Receive charge ID: ch_1234567890
     - Record the reference: Payment Initiated { orderId: order-123, paymentGatewayRef: ch_1234567890 }

When webhook arrives:
  1. Webhook: charge.succeeded { id: ch_1234567890, amount: 15000, ... }
  2. Match paymentGatewayRef = ch_1234567890 to the recorded reference
  3. Get orderId from the reference
  4. Record Payment Authorized event: { orderId: order-123, amount: 150.00, ... }

Key insight: You must create the correlation bridge when initiating external action.
```

The reference is a fact on our side (`Payment Initiated`), so it is an event on the sheet whose fields reach the todo-list read model; the correlation is visible in the model and every field has a source.

## 5. Define Translation Scenarios

```
External Event: charge.succeeded
Trigger: Stripe webhook arrives with charge details
Precondition: Payment Initiated was recorded for this charge ID
Translation logic:
  1. Extract charge.id from webhook
  2. Find the orderId recorded for it
  3. Validate order exists and is in Confirmed state
  4. Record domain event: Payment Authorized { orderId, amount, timestamp, ... }
Success: Domain event produced
Failure scenarios:
  - Charge ID not recorded against any order → rejected, no event (manual review)
  - Order not in Confirmed state → rejected, no event
  - Duplicate webhook → rejected as a duplicate (already recorded)

--- External Event: geofence_exit
Trigger: Guest leaves hotel area (GPS geofence)
Precondition: Guest has opted in to location tracking
Translation logic:
  1. Extract userId and geoFenceId from webhook
  2. Validate guest is currently in hotel
  3. Check geofence_exit is "hotel-front-entrance" (not just any geofence)
  4. Record domain event: Guest Left Hotel { guestId: userId, timestamp, ... }
Success: Domain event produced
Failure scenarios:
  - Guest hasn't opted in → no event (respect privacy)
  - Guest not checked in → no event (shouldn't be in geofence)
  - Unknown geofence → rejected, no event
```

As scenarios on the translation command `Record Payment Authorization` — each scenario card is placed first (`place_element`, `scope: "scenario"`, in the specification lane's cell of the command's column), then authored:

```
set_scenario { "scenario": "Charge for a confirmed order",
  "given": [ "Charge Succeeded", "Payment Initiated", "Order Confirmed" ],
  "when":  [ "Record Payment Authorization" ],
  "then":  [ "Payment Authorized" ] }

set_scenario { "scenario": "Charge not recorded against any order",
  "given": [ "Charge Succeeded" ],
  "when":  [ "Record Payment Authorization" ],
  "then":  [ { "errorType": "UnknownCharge", "errorMessage": "No order was recorded for this charge" } ] }
```

(Use addresses rather than names where two elements share a name.)

## 6. Handle Duplicates and Ordering

```
Problem: Stripe retries charge.succeeded webhook
Webhook 1: charge.succeeded { id: ch_123 } → Arrives at 10:00 AM
Webhook 2: charge.succeeded { id: ch_123 } → Arrives at 10:05 AM (retry)

Solution: Idempotent translation

Before recording the event:
  1. Take the external ID: ch_123
  2. Has a Payment Authorized event with paymentGatewayRef = ch_123 been recorded?
  3. If yes: reject as a duplicate (already processed)
  4. If no: record the event

This requires the external ID on the event:
Payment Authorized event {
    orderId: order-123,
    amount: 150.00,
    paymentGatewayRef: ch_123,  ← keep the external ID for deduplication
    ...
  }
```

As a scenario: Given `Payment Authorized` with `paymentGatewayRef: ch_123` and `Charge Succeeded` with `id: ch_123`, When `Record Payment Authorization`, Then `{ errorType: "DuplicateCharge", errorMessage: "Charge ch_123 was already recorded" }`.

## 7. Handle Partial or Missing Information

```
External Event: geofence_exit

Available data:
  - userId
  - geoFenceId
  - timestamp
  - latitude, longitude (raw GPS)

Missing data:
  - Guest name (not in webhook payload)
  - Reason for leaving (not tracked)
  - Expected return time (not available)

Handling strategy:
A. Translation enriches from our own events:
     Domain event: Guest Left Hotel {
       guestId: userId,  ← From webhook
       timestamp: ...,   ← From webhook
       guestName: "John Smith",  ← From the guest's own events (Guest Checked In)
       roomNumber: "502",  ← From the guest's own events (Guest Checked In)
       geoFenceId: "front-entrance"  ← From webhook
     }

B. Some data we don't need:
     We ignore: latitude, longitude (we just care that guest left)

C. Some data we can infer:
     We can assume: Guest is now outside hotel
                    Cleaning crew can visit room
```

Enriched fields (`guestName`, `roomNumber`) need a source on the sheet: the `Guest Checked In` event feeds the read model the translation automation reads. Ignored fields are left unmapped. Inferred values are `isGenerated`.

## Shape of a complete translation write-up

The same information a translation note covers — kept on the sheet (events, fields, flows, scenarios, the note on the translation automation's cell) and presented as text at the end of the step.

````markdown
# External Event Translation: [Domain Name]

## External Systems & Events

### System: [External System Name]

**Connection Type**: [Webhook/API polling/WebSocket/Streaming]

**Events Received**:
- event1_name
- event2_name
- event3_name

---

## Translation Rules

### External Event: [Event Name]

**Source System**: [System name]

**Technical Representation**:
```json
{
  "field1": "value",
  "field2": "value"
}
```

**Domain Translation**:
| External Field | Our Field | Mapping | Notes |
|---|---|---|---|
| externalId | externalRef | Kept for deduplication | Reference only |
| customer | [enrich] | Our customer ID from our own events | Must correlate |

**Correlation Method**:
[How do we link back to our domain entities?]

**Domain Event Produced**:
- Event Name: [Event Name]
- Fields: [List with sources]

**Translation Logic**:
```
1. Extract from the external event
2. Validate preconditions
3. Enrich from our own events
4. Record the domain event
```

**Success Scenario**:
[What success looks like]

**Failure Scenarios**:
- Scenario 1: Consequence
- Scenario 2: Consequence

**Duplicate Handling**: [Idempotent strategy]

--- [Repeat for each external event]

---

## Correlation Reference

Track how external IDs map to our domain:

| Our Entity | External System | External ID Field | Recorded By | Matched On |
|---|---|---|---|---|
| Order | Stripe | charge.id | Payment Initiated | paymentGatewayRef |
| Guest | GPS Service | userId | Guest Checked In | guestId |

---

## Failure & Recovery

### Webhook Arrives for Non-existent Order
**Symptom**: Stripe sends charge.succeeded for unknown order
**Cause**: Race condition or data inconsistency
**Detection**: No recorded reference matches the charge ID
**Recovery**: Rejected as an error scenario; queued for manual review

### Duplicate Webhooks
**Symptom**: Same webhook received multiple times
**Cause**: Stripe retry mechanism or network duplication
**Detection**: Domain event already recorded with same external reference
**Recovery**: Duplicate is rejected; no second event
````
