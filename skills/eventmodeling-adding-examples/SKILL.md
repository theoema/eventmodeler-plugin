---
name: eventmodeling-adding-examples
description: "Add or complete example values on the fields of one or more elements (element field examples), consistent with the neighbouring elements, or put concrete example values into a scenario's steps. Use when fields have no example, an element's examples are weak or contradict the elements around them, or a scenario's steps carry no values. Do not use for: designing which scenarios exist (use `eventmodeling-elaborating-scenarios`), judging how much a slice's scenarios cover (use `eventmodeling-assessing-scenario-coverage`) or adding, renaming or sourcing fields (use `eventmodeling-adding-attributes`)."
---

# Adding Examples

This step applies the shared element rules in **`eventmodeling-core-rules`** — read it once per session if you haven't already. Tool names and conventions are in `eventmodeling-core-rules` → `../eventmodeling-core-rules/references/tools.md`.

You are in **Modeling Mode**: give the model realistic, consistent values for its fields, and change nothing else.

## Two kinds of example values

They are written differently and say different things. Know which one the person asks for.

| | Element field example | Scenario value |
|---|---|---|
| What it is | the `example` on a field of an element: an illustrative value for that field, wherever the element is read | a value in one scenario step, `fieldValues` on a given, when or then entry, or a row of a list read model |
| Written with | `update_element` (a `fields` patch) | `set_scenario` |
| How much | every field gets one | only the values the case depends on: what the rule decides on, the ids that link the steps, the asserted outcome. The other fields stay empty on purpose |

The two do not copy into each other: an element's `example` does not appear in a scenario until you write it there, and a scenario's values don't change the element. Keep them consistent (the same customer in both) but don't copy every example into every scenario.

Which one: the person names an element or its fields, so element examples; the person names a scenario, or says a scenario's steps lack values, so scenario values. If it is unclear, ask in one line.

## Element field examples

### Step 1 — Find the elements

Resolve each target from the person's request: by name or address, on its sheet (`get_model_view`, `get_sheet_view`). Read it with `get_slice_view`: its fields with their current `example` values, and the flows with their mappings. A **linked copy** has no fields of its own: its original does, so edit the original and the copy follows.

**Many targets at once** ("fill in the examples across this sheet"): read every slice of the scope once, before writing anything, and build one pool of values per field. Don't run the steps once per element, and don't re-read a neighbour per element.

### Step 2 — Choose consistent values

The same information carries the same value wherever it appears. Pick the canonical value for each piece of information from the pool and reuse it:

- A field with the same name on a connected neighbour (`customerId` on the screen, the command, the event) takes that neighbour's existing example. Where a field is renamed along a flow, `trace_field` shows the chain, and every element on it gets the same value under its own name.
- A value some element already has is the canonical one. Never overwrite an existing example unless the person asks you to. If existing examples contradict each other along a chain, don't pick a winner silently: tell the person which ones disagree.
- A calculated field (`isGenerated` on a read model: a total, a count, a duration) must agree with its inputs: a total of 149.99 comes from the unit prices and quantities in the example.

For a field with no neighbour to copy, infer the value from its name and `fieldType`, in the language of the element's domain:

| Field | Example |
|---|---|
| `email` | `jane.smith@example.com` |
| a person's name | a real-sounding name |
| `id`, `*Id` | a short readable id such as `ORD-2024-0042` (a `UUID` field gets a UUID) |
| `amount`, `price`, `total` | `149.99` |
| `date`, `*At` | ISO 8601, `2024-03-15T10:30:00Z` |
| `status` | a plausible value derived from the element's name |
| a `Boolean` | `true` or `false` |
| `count`, `quantity` | a small integer such as `3` |
| a list field (`isList: true`) | one representative item, as a string |
| any other string | a short realistic value matching the name |

Domain-specific, not generic: for "Order Placed" use order values; for "Customer Registered" registration values. Short: one value per field, no paragraphs. A `Custom` field takes its examples on its subfields. For a field marked `isPii`, make up the person: never use a real one, and use `example.com` for addresses.

### Step 3 — Write them

`example` is a string. In `update_element` use `updateFields: [{ name, example }]` to fill examples without resending the other fields (a `fields` patch would replace the whole list). A field with a good example comes back untouched. One `update_element` per element; use one `apply_operations` batch when several elements should land together.

### Step 4 — Report

- **Updated**: each element, with the examples you set (field and value), and which you reused from a neighbour (name the neighbour).
- **Kept**: fields whose examples were already good.
- **Disagreements**: examples along a chain that contradict each other, left as they were.

```
Updated: Order Placed (event)
  orderId     ORD-2024-0042   reused from Place Order (command)
  total       149.99          2 x 50.00 + 49.99, from the lines of Place Order
  placedAt    2024-03-15T10:30:00Z

Kept: email jane@example.com
```

## Scenario values

Fill a scenario's steps with the values it depends on, following **`eventmodeling-elaborating-scenarios`** ("Fill the values essential to the scenario"): the values the rule decides on, the ids that tie the steps together, and the outcome the scenario asserts. Leave every other field empty: the empty fields show which data matters.

1. Read the scenario and its elements' field names (`get_slice_view`). `set_scenario` accepts only field names the element really has.
2. Take ids and shared facts from the element examples above, so the scenario and the elements agree (the same `customerId` in every step). Pick values on the interesting side of the rule: one day past the limit, not "long ago".
3. Write each section in full. Each section you pass replaces that section: send all of the section's entries, not only the one you changed.

```
set_scenario {
  scenario: "Reject a reservation for a date in the past",
  when: [{ element: "Reserve Bike", fieldValues: { customerId: "CUST-8819", date: "2026-10-12" } }],
  then: [{ errorType: "ReservationDateInPast", errorMessage: "reservation date is in the past" }]
}
```

A list read model's outcome is its `rows` or `expectEmptyList`, on the scenario; a storyline carries its values per beat. Run `check_model`: `scenario-without-examples` is the finding for a scenario whose steps only name elements. If a value the rule depends on has no field on any element, the field is missing: add it (`eventmodeling-adding-attributes`) instead of writing the value in a note.

## Quality Checklist

- [ ] The request was read as element examples or scenario values, and the report says which
- [ ] Every field without an example got one (element examples); only the essential values were filled (scenarios)
- [ ] The same information has the same value on every element it flows through, under whatever name it has there
- [ ] Existing examples were kept; contradictions were reported, not overwritten
- [ ] Calculated values agree with their inputs
- [ ] No real person's data in any `isPii` field
- [ ] Examples went in with `updateFields` (or a `fields` patch carrying the element's whole list); linked copies were left alone
