---
name: eventmodeling-adding-attributes
description: "Add a new attribute, or rename an existing one, across the chain of elements from a source element to a target element, keeping every field's source intact. Use when a field has to appear on, or change name along, several connected elements (a screen field that must reach a read model, a rename that must follow its lineage). Do not use for: the rule for where a field's value comes from (use `eventmodeling-information-completeness`), only finding where a field goes (use `eventmodeling-tracing-data`) or auditing every field of the model (use `eventmodeling-checking-completeness`)."
---

# Adding Attributes

This step applies the shared element rules in **`eventmodeling-core-rules`** — read it once per session if you haven't already. Tool names and conventions are in `eventmodeling-core-rules` → `../eventmodeling-core-rules/references/tools.md`. Every field you write follows **`eventmodeling-information-completeness`**: it has a source the moment it exists.

You are in **Modeling Mode**: you propagate one attribute change (add or rename) along a chain of connected elements, from where the value starts to where it is needed. Apply the change at every element of the chain, and keep the lineage whole.

> **Fast path.** When the request names the elements and asks for a local edit (add or rename one field on those elements, nothing flowing further), read only those slices (`get_slice_view`), edit in one `update_element` per element, and verify once. Don't walk the chain. Walk it only when lineage is at stake: a rename across a chain, or a new field that must flow downstream or come from upstream.

## Step 1 — Gather inputs

Ask for what is missing **in one message**, never one question at a time:

1. **Target element**: the end of the chain, by name or address (and its sheet).
2. **Source element**: the start of the chain, by name or address. If the person doesn't know, take the real origin from the trace (Step 2).
3. **Operation**: add a new attribute, or rename an existing one.
4. Rename: the name to rename from and the name to rename to. Add: the new name, and its type and meaning if the name doesn't make them obvious. Pick the `fieldType` the name implies (a `*At` field is a `DateTime`, a `*Id` field a `UUID`, otherwise `String`) and say what you picked.

Anything already in the person's request needs no asking.

## Step 2 — Resolve the chain

Resolve target and source to elements: `get_model_view` for the sheets, `get_sheet_view` for names and addresses, `get_slice_view` for fields and the flows with their mappings. A linked copy carries no fields of its own: its original does, so the chain runs through the original, and the copies show the change by themselves. Read each element's fields from the original.

- **Rename.** The field exists, so follow it: `trace_field { sheet, element: <target>, field: <oldName> }` returns the steps upstream back to where it starts and every element it flows into downstream. The chain is the part of the upstream steps between the target and the source (or all the way to the origin when no source was given). The downstream steps beyond the target are not in the chain but they are affected: see Step 3.
- **Add.** The field doesn't exist yet, so there is nothing to trace. Read the chain from the flows: from the target walk back through the inbound flows (the upstream rules in `eventmodeling-information-completeness`: a read model from its events, an event from its command, a command from the screen or automation that issues it, a screen from the read model it shows) until you reach the source. If the target already has a sibling field that travels the same way from the source, `trace_field` on it lists the elements for you; confirm with the flows. A flow can cross sheets: the chain may too.

The source element is where the new field starts, unless one of its own inbound neighbours already carries the information. In that case the chain stops too early: extend it to the real origin and tell the person.

If the chain is longer than the person likely meant, or a hop doesn't exist yet (a missing flow or element), say so and ask before you add structure.

## Step 3 — Plan the change per element

Work out every element's new field list before you write anything, from the origin forward.

**Add.** Skip an element that already has a field of that name and log it. Otherwise append the field:

- Along the chain the same name carries the value from hop to hop, so a hop needs nothing beyond the field: no mapping, the same name is the mapping.
- Where the information starts: `isUserInput` on a screen where a person types it, `isGenerated` on the element where anything else creates it. Don't mark the origin by guess: a field marked generated to make a gap go away is the mistake the rule exists to stop.
- A hop that needs the value under a different name, or from one specific upstream element, gets `source: { from: "<upstream element>", field: "<its field>" }` on the field, which creates the flow and the mapping if they are missing. `set_flow_mapping` does the same on an existing flow, and replaces that flow's whole mapping list.
- `isPii`: the value is about a person (name, email, address, phone). Mark it on every element the value passes through.

**Rename.** Find the field by its old name. Skip an element that doesn't have it and log it. Otherwise change only the name: send the field with its existing `id` and the new `name`, so the mappings that point at it survive. A downstream field that matched the old name, or took it as its `source`, loses its lineage with the rename: in the same pass either rename it too or point its source at the new name. That includes fields outside the chain (`eventmodeling-information-completeness` → "Renames move the lineage too"). Scenario values (step `fieldValues`, list `rows`) follow the rename on their own when the field is sent with its `id`; only on an element renamed without its `id` do they stay under the old name, and `set_scenario` is how they are corrected.

## Step 4 — Write it

A `fields` patch in `update_element` **replaces the whole field list**: send every field of the element, with the existing ids, changing only the one you mean to. Field shape is exactly `{name, fieldType, isList?, isGenerated?, isOptional?, isUserInput?, isPii?, example?}`, plus `source` at write time.

A chain is one logical edit, so send it as one `apply_operations` batch of `update_element` calls, ordered from the origin forward so every `source` refers to a field that already exists. If a step fails the batch applies nothing: fix the cause and resend the whole batch. For one or two elements, plain `update_element` calls are enough.

## Step 5 — Verify

- `trace_field` for the new name from the target: the steps lead back to the origin you intended, via the mappings you meant.
- `check_model` for the sheets touched. Fix `untraceable-field` and `unsourced-fields` on the changed fields, and `pii-mismatch` where a flow now carries a personal value into an unmarked field. Nothing here is a detail to leave for later.

## Step 6 — Report

Say it to the person, short:

- **Operation**: add "customerSegment" / rename "customerId" to "clientId".
- **Chain**, in flow order from the origin: each element with its type, slice and sheet, marked changed or skipped, with the reason for a skip (field already exists, field not found).
- **Lineage**: where the attribute starts (which element is `isUserInput` or `isGenerated`) and each mapping you set for a renamed hop.
- **Downstream fixed**: fields outside the chain that you renamed or re-pointed.
- **Open**: anything you couldn't decide, as a pin you raised (`create_pin`).

```
Operation: rename "customerId" to "clientId"

Chain (origin to target):
  Register Customer (screen)    changed
  Register Customer (command)   changed
  Customer Registered (event)   skipped, field not found
  Customer Overview (read model) changed

Lineage: starts at Register Customer, where the value is generated. Every later element carries it under the same new name, so no mapping was needed.
```

## Quality Checklist

- [ ] The chain was found from the model (`trace_field`, flows), not assumed from names
- [ ] Every new or renamed field has a source: same name upstream, a `source` or mapping, or a flag where the information starts
- [ ] Nothing is marked `isGenerated` or `isUserInput` only to silence a gap
- [ ] Every `fields` patch carried the element's whole field list, with existing ids
- [ ] Linked copies were left alone; the originals were edited
- [ ] A rename also fixed the fields downstream that lost their source, and scenario values keyed by the old name
- [ ] `isPii` was set on every element a personal value passes through
- [ ] `trace_field` and `check_model` were run on the result
