# Elaborating Scenarios — Worked Examples

Full worked Given-When-Then examples for each scenario category, all drawn from one Order/Payment domain so the shapes are easy to compare. The main SKILL.md keeps one compact example per category — this file has the complete set.

## 1. Command Scenarios (Given-When-Then)

```
Feature: Order Creation

Scenario: Create order successfully
Given a customer with ID "cust-123"
And products exist with IDs ["prod-1", "prod-2"]
And customer has valid shipping address
When the customer creates an order with items:
    | productId | quantity | unitPrice |
    | prod-1    | 2        | 50.00    |
    | prod-2    | 1        | 30.00    |
Then the order should be created with status "Draft"
And the total should be calculated as 130.00
And an "Order Created" event is produced with:
    | field | value |
    | orderId | {uuid} |
    | customerId | cust-123 |
    | items | [...] |
    | total | 130.00 |
    | status | Draft |

Scenario: Reject order with invalid customer
Given a customer ID "invalid-cust"
And no customer exists with that ID
When the customer tries to create an order
Then the command should be rejected
And the rejection reason is "Customer not found"
And no event is produced

Scenario: Reject order with empty items
Given a customer with ID "cust-123"
And an empty items list []
When the customer tries to create an order
Then the command should be rejected
And the rejection reason is "Order must contain items"
And no event is produced

Scenario: Reject order with invalid address
Given a customer with ID "cust-123"
And an incomplete shipping address (missing city)
When the customer tries to create an order
Then the command should be rejected
And the rejection reason is "Invalid shipping address"
And no event is produced
```

## 2. Command Scenarios — State Validation

```
Feature: Order Confirmation

Scenario: Confirm order in Draft state
Given an order "order-456" in Draft state
And Order Created event exists
And no Order Confirmed event exists
When the customer confirms the order with payment method "card"
Then the order should be confirmed
And an "Order Confirmed" event is produced with:
    | field | value |
    | orderId | order-456 |
    | paymentMethod | card |
    | confirmedAt | {timestamp} |

Scenario: Reject confirming already-confirmed order
Given an order "order-456" in Confirmed state
And Order Confirmed event already exists
When the customer tries to confirm the order again
Then the command should be rejected
And the rejection reason is "Order already confirmed"
And no Order Confirmed event is produced

Scenario: Reject confirming cancelled order
Given an order "order-456" in Cancelled state
And Order Cancelled event exists
When the customer tries to confirm the order
Then the command should be rejected
And the rejection reason is "Cannot confirm cancelled order"
And no event is produced
```

## 3. View Scenarios (Given-When-Then)

```
Feature: Order Status View

Scenario: Display order after creation
Given an Order Created event with:
    | field | value |
    | orderId | order-789 |
    | customerId | cust-123 |
    | items | [{...}] |
    | total | 150.00 |
When the Order Status view processes this event
Then the view should display:
    | field | value |
    | Order ID | order-789 |
    | Status | Draft |
    | Total | $150.00 |
    | Items | 3 products |
    | Created | {date} |

Scenario: Update status after confirmation
Given an Order Created event already processed
And Order Status view showing status "Draft"
When an Order Confirmed event is received with:
    | field | value |
    | orderId | order-789 |
    | confirmedAt | 2024-12-31T10:00:00Z |
Then the view should update to display:
    | field | value |
    | Status | Confirmed |
    | Confirmed Date | 12/31/2024 10:00 AM |

Scenario: Accumulate payment information
Given Order Confirmed event processed (status=Confirmed)
When a Payment Authorized event arrives with:
    | field | value |
    | orderId | order-789 |
    | paymentId | pay-123 |
    | authCode | AUTH-456 |
Then the view should accumulate:
    | field | value |
    | Payment ID | pay-123 |
    | Auth Code | AUTH-456 |
    | Payment Status | Authorized |
```

## 3b. List-type Read Model Scenarios

Expected rows and empty-list intent when the THEN read model is a list (the whole read model is a set of rows):

```
Feature: Product Catalog

Scenario: Products list shows all created products
Given a Product Created event with name "Shoes", index "0", family_id "22222..."
And a Product Created event with name "Clothing", index "1", family_id "33333..."
When the Product List view processes these events
Then the list should contain:
    | name     | index | family_id |
    | Shoes    | 0     | 22222...  |
    | Clothing | 1     | 33333...  |

Scenario: Products list is empty after last item is deleted
Given a Product Deleted event for the last remaining item
When the Product List view processes this event
Then the list should be empty
```

## 4. Error Path Scenarios

```
Feature: Payment Authorization Failure

Scenario: Handle declined payment
Given an order "order-001" in Confirmed state
And customer initiates payment
When the payment gateway declines the card
Then a Payment Failed event is produced with:
    | field | value |
    | orderId | order-001 |
    | reason | Card declined |
    | timestamp | {now} |

Scenario: Update order view on payment failure
Given Order Status view shows status "Confirmed"
When Payment Failed event arrives for order-001
Then the view should update:
    | field | value |
    | Payment Status | Failed |
    | Failure Reason | Card declined |
    | Retry Available | Yes |

Scenario: Allow retry after payment failure
Given a Payment Failed event exists
And order status is still "Confirmed"
When customer retries payment
Then the new Authorize Payment command is accepted
And can produce new Payment Authorized event
```

## 5. Compensation Scenarios

```
Feature: Order Cancellation

Scenario: Cancel order in Draft state
Given an order "order-555" in Draft state
And only Order Created event exists
When customer cancels the order with reason "Changed mind"
Then an Order Cancelled event is produced with:
    | field | value |
    | orderId | order-555 |
    | reason | Changed mind |
    | cancelledAt | {timestamp} |

Scenario: Cannot cancel completed order
Given an order "order-555" in Delivered state
And Delivery Confirmed event exists
When customer tries to cancel
Then the command should be rejected
And the rejection reason is "Cannot cancel delivered order"

Scenario: Trigger compensation on cancellation
Given an order in Confirmed state
And Payment Authorized event exists
When Order Cancelled event is produced
Then a Refund Payment command should be automatically triggered
And Refund Initiated event should follow
```

## 6. The same scenario on the sheet

How one of the scenarios above becomes a scenario element. The slice "Confirm Order" has the command `Confirm Order` in `C2`, the event `Order Confirmed` in `C3`, and the specification lane is row 5. Field names come from `get_slice_view`; the example values are only those the rule depends on (the order's id and its state).

```
set_scenario {
  scenario: "Reject confirming an already-confirmed order",
  cell: "C5",
  given: [{ element: "Order Confirmed", fieldValues: { orderId: "order-456" } }],
  when:  [{ element: "Confirm Order",    fieldValues: { orderId: "order-456" } }],
  then:  [{ errorType: "OrderAlreadyConfirmed", errorMessage: "Order already confirmed" }]
}
```

The happy path of the same slice, with the asserted outcome as the filled values:

```
set_scenario {
  scenario: "Confirm order in Draft state",
  cell: "C5",
  given: [{ element: "Order Created",   fieldValues: { orderId: "order-456" } }],
  when:  [{ element: "Confirm Order",   fieldValues: { orderId: "order-456", paymentMethod: "card" } }],
  then:  [{ element: "Order Confirmed", fieldValues: { orderId: "order-456", paymentMethod: "card" } }]
}
```

Both scenarios sit in `C5` — the specification lane is the one place where cells stack. `cell` creates each one there and fills it in the same call.

## 7. A storyline on the sheet

The list read model `Open Orders` (column `D`, specification lane row 5) shows the orders still waiting to be confirmed. One storyline says what three GWTs would only say in pieces: which event moves the list where, with the same order throughout.

```
set_scenario {
  scenario: "An order is open until it is confirmed",
  cell: "D5",
  storyline: [
    { element: "Open Orders",     expectEmptyList: true },
    { element: "Order Created",   fieldValues: { orderId: "order-456", total: "120.00" } },
    { element: "Open Orders",     rows: [{ orderId: "order-456", total: "120.00" }] },
    { note: "The customer pays the next morning." },
    { element: "Order Confirmed", fieldValues: { orderId: "order-456", paymentMethod: "card" } },
    { element: "Open Orders",     expectEmptyList: true }
  ]
}
```

It starts with the read model it walks through, in its first state; each later state follows the event that causes it, and the last beat is the state the last event leaves it in. A single-record read model's state is its `fieldValues` instead (`{ element: "Order Status", fieldValues: { orderId: "order-456", status: "Confirmed" } }`).
