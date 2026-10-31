# Event Modeling Slice Examples

## Table of Contents
- [Order Management — Checking Slices Against the Structure](#order-management--checking-slices-against-the-structure)
- [Correcting a Mixed Slice](#correcting-a-mixed-slice)
- [Dependency Notes](#dependency-notes)
- [Checklist for Slice Verification](#checklist-for-slice-verification)

---

## Order Management — Checking Slices Against the Structure

Given these elements on the sheet (read from `get_sheet_view`):

```
COMMAND    Place Order                  → produces Order Placed
COMMAND    Confirm Order                → produces Order Confirmed
COMMAND    Authorize Payment            → produces Payment Authorized
READMODEL  Order Detail                 → folds Order Placed, Order Confirmed
READMODEL  Payment Status               → folds Payment Authorized
AUTOMATION Reserve Inventory On Payment → consumes Payment Authorized (via its todo list), issues Reserve Inventory
```

The slices the structure should have — one subject each, never combined:

```
Slice: Place Order                    type: state-change
Slice: Confirm Order                  type: state-change
Slice: Authorize Payment              type: state-change
Slice: Order Detail                   type: state-view
Slice: Payment Status                 type: state-view
Slice: Reserve Inventory On Payment   type: automation
```

Note there is no combined "Order Management" or "Payment Processing" slice — each command, read model, and automation gets its own slice. The automation's todo-list read model (one column before it) is its own state-view slice as well.

What `get_sheet_view` actually shows, and what to do about each row:

```
Slice (name)               Columns  Status   Subjects                       Verdict
Place Order                A        planned  command Place Order            OK
Slice                      B        —        command Confirm Order           Placeholder name, no status → rename "Confirm Order", status planned
Authorize Payment          C        planned  command Authorize Payment       OK
Order Detail               D–E      planned  read model Order Detail         OK — column E holds only a linked copy of an event
Payment Status             F        planned  read model Payment Status      OK
Reserve Inventory ...      G        planned  automation + Reserve Inventory OK
```

```
update_slice { slice: "B", name: "Confirm Order", status: "planned" }
```

---

## Correcting a Mixed Slice

Suppose slice `Payment Status` spans columns F–G and column G holds the command `Authorize Payment` (a second subject):

```
Slice: Payment Status   columns F–G
  F: Payment Authorized (copy) → Payment Status (read model) → Payment Screen
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
Order Detail                 ← depends on events from Place Order, Confirm Order
Payment Status               ← depends on events from Authorize Payment
Reserve Inventory On Payment ← depends on events from Authorize Payment
```

No slice depends on another slice directly — only on the events it produces. Each line goes in the dependent slice's note:

```
set_note { target: "Order Detail", text: "Depends on events: Order Placed (Place Order), Order Confirmed (Confirm Order)" }
```

---

## Checklist for Slice Verification

```
[ ] Slice contains exactly one COMMAND, READMODEL, or AUTOMATION — never a
    COMMAND and a READMODEL together
[ ] Slice name matches the element's name exactly (no placeholder name left)
[ ] Slice type matches the element type (state-change / state-view / automation)
[ ] Slice has a status
[ ] Dependencies recorded as "depends on events from X", not "depends on slice X"
[ ] No slice exists for a linked copy alone
```
