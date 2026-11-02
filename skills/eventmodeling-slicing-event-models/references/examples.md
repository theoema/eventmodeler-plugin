# Event Modeling Slice Examples

## Table of Contents
- [Order Management — Deriving Slices from the Sheet](#order-management--deriving-slices-from-the-sheet)
- [Reading the Slices on the Sheet](#reading-the-slices-on-the-sheet)
- [Correcting a Mixed Slice](#correcting-a-mixed-slice)
- [Dependency Notes](#dependency-notes)
- [Making These Slices Explicit](#making-these-slices-explicit)
- [Checklist for Slice Definition](#checklist-for-slice-definition)

---

## Order Management — Deriving Slices from the Sheet

Given these sheet elements (found via `get_sheet_view`):

```
COMMAND    Place Order                → produces Order Placed
COMMAND    Confirm Order              → produces Order Confirmed
COMMAND    Authorize Payment          → produces Payment Authorized
READMODEL  Order Detail View           → projects Order Placed, Order Confirmed
READMODEL  Payment Status View         → projects Payment Authorized
AUTOMATION Reserve Inventory On Payment → consumes Payment Authorized, issues Reserve Inventory
```

The resulting slices — one per element, never combined:

```
Slice: Place Order                 type: state-change
Slice: Confirm Order                type: state-change
Slice: Authorize Payment            type: state-change
Slice: Order Detail View             type: state-view
Slice: Payment Status View           type: state-view
Slice: Reserve Inventory On Payment   type: automation
```

Note there is no combined "Order Management" or "Payment Processing" slice — each command, read model, and automation gets its own slice.

---

## Reading the Slices on the Sheet

What `get_sheet_view` shows for the elements above, and what to do about each row:

```
Slice (name)                Columns  Status   Subjects                           Verdict
Place Order                 A        planned  command Place Order                OK
Slice                       B        —        command Confirm Order              Placeholder name, no status → rename "Confirm Order", status planned
Authorize Payment           C        planned  command Authorize Payment          OK
Order Detail View           D–E      planned  read model Order Detail View       OK — column E holds only a linked copy of an event
Payment Status View         F        planned  read model Payment Status View     OK
Reserve Inventory On Payment G       planned  automation + Reserve Inventory     OK
```

```
update_slice { slice: "B", name: "Confirm Order", status: "planned" }
```

---

## Correcting a Mixed Slice

Suppose slice `Payment Status View` spans columns F–G and column G holds the command `Authorize Payment` (a second subject):

```
Slice: Payment Status View   columns F–G
  F: Payment Authorized (copy) → Payment Status View (read model) → Payment Screen
  G: Authorize Payment (command) → Payment Authorized (event)
→ slice-mixed-shape: a read model and a command in one slice
```

Split it where the second subject begins, and name the new slice:

```
split_slice  { column: "G" }
update_slice { slice: "G", name: "Authorize Payment", status: "planned" }
```

If both subjects had shared one column, there would be nothing to split. A new slice goes next to it (`insert_slice`) and the second subject is moved there (`move_element`).

---

## Dependency Notes

```
Order Detail View            ← depends on events from Place Order, Confirm Order
Payment Status View          ← depends on events from Authorize Payment
Reserve Inventory On Payment  ← depends on events from Authorize Payment
```

No slice depends on another slice directly — only on the events it produces. Each line goes in the dependent slice's note:

```
set_note { target: "Order Detail View", text: "Depends on events: Order Placed (Place Order), Order Confirmed (Confirm Order)" }
```

---

## Making These Slices Explicit

These elements already exist on the sheet (from `get_sheet_view`), each in a slice already — use `update_slice` to give the slice that holds each element its name, and `split_slice` where one slice holds two of them. Never use `insert_slice` here: it creates a brand-new empty column, which would leave the element already on the sheet in its old slice.

```
update_slice { "slice": "<placeOrderSlice>", "name": "Place Order" }
update_slice { "slice": "<orderDetailViewSlice>", "name": "Order Detail View" }
update_slice { "slice": "<reserveInventoryOnPaymentSlice>", "name": "Reserve Inventory On Payment" }
```

---

## Checklist for Slice Definition

```
[ ] Slice contains exactly one COMMAND, READMODEL, or AUTOMATION — never a
    COMMAND and a READMODEL together
[ ] Slice name matches the element's title exactly (no placeholder name left)
[ ] Slice type matches the element type (state-change / state-view / automation)
[ ] Slice has a status
[ ] Dependencies recorded as "depends on events from X", not "depends on slice X"
[ ] No duplicate slice created for an element that already has one
[ ] No slice exists for a linked copy alone
```