---
name: eventmodeling-information-completeness
description: The rule for where every field's value comes from. Use it whenever a field is added, renamed or reviewed on any element (SCREEN, COMMAND, EVENT, READMODEL, AUTOMATION), whether in a full modeling run, a one-off `eventmodeling-adding-attributes` or `place_element` call, or a fix. Each field either matches a same-named field on a connected upstream element, maps to a differently named field or a connected event, or is marked generated or user-input where the information starts. Anything else is a gap to close. Not a step of its own. Other skills reference it instead of restating it.
---

# Information Completeness

Information doesn't appear out of nowhere. Every field on every element either comes from an element **connected upstream** of it, or starts at this element. Check this **each time you add or rename a field**, before you write it. Don't leave it for a later completeness pass.

## The four valid states of a field

| Where the value comes from | What to write | Example |
|---|---|---|
| A connected upstream element has a field **with the same name** | nothing: no mapping, the same name *is* the mapping | `Bike Reserved.bikeId` ← `Reserve Bike.bikeId` |
| A connected upstream element has it **under a different name** | `source: { from: "<upstream element>", field: "<sourceFieldName>" }` on the field (a sub-field as `"<field>.<subfield>"`), or `set_flow_mapping` on the flow | `Reserve Bike.endTime` with `source: { from: "Reserve a Bike", field: "returnTime" }` |
| It comes from one specific connected **event** (or other connected element), e.g. because several upstream elements carry the field | `source: { from: "<Event Name>" }`, the element's name on its own | `Reservation View.expiresAt` with `source: { from: "Reservation Confirmed" }` |
| **Nothing upstream** supplies it: manual input typed into a screen, a session value, a generated id or timestamp, a value computed at this element | `isUserInput: true` when a person types it into a screen; `isGenerated: true` for everything else that starts here | `SCREEN "Reserve a Bike".startTime` (user input), `Bike Reserved.reservationId` (generated) |

These are the forms the model resolves. A flow carries field mappings by field name, and `trace_field` follows them. Write the field name or the element name **bare**. Combined forms like `"Bike Reserved.bikeId"`, `"user-input"`, `"session:x"` or `"derived:..."` resolve to nothing.

## Upstream means inbound, per element type

Lineage always points to where the value comes **from**. It never points downstream.

| Element | Its upstream (where its fields can come from) |
|---|---|
| SCREEN | the READMODEL it displays. The COMMAND it submits is **downstream**: never map a screen field to it. |
| COMMAND | the SCREEN or AUTOMATION issuing it |
| EVENT | the COMMAND producing it |
| READMODEL | the EVENTs connected to it (`EVENT → READMODEL`) |
| AUTOMATION | the READMODEL (todo list) it reads |

So information typed into a form is `isUserInput: true` **on the screen**. The command, event and read model then pick it up by the same name, or by a `source` where it's renamed. Contextual values a command's validation reads (current time or date, acting user, tenant) are `isGenerated: true` on the **command**. Commands carry everything they need to validate (`eventmodeling-identifying-inputs` → "Command input is self-contained"), and the event takes a recorded time from there. Only an id the system creates when recording the fact is `isGenerated: true` on the event. A count, sum or duration the projection calculates is `isGenerated: true` on the read model.

## How to check a field

1. Look at the element's inbound neighbours, from the flows (`get_slice_view` shows its fields and the flows into it with their mappings; `trace_field` follows one field the whole way: upstream to where its value starts (`origins`: user input, generated, or `unsourced`, a gap) and downstream to everywhere it goes, see `eventmodeling-tracing-data`), or from the grid (`get_sheet_view`) when the sheet is unwired. `check_model` lists what it can already see as `untraceable-field` (some inbound flow exists but doesn't cover the field) and `unsourced-fields` (a screen or command with no inbound flow at all and no stated origin).
2. Pick the row above that matches. One neighbour with the same name, and nothing to write, is the common case.
3. **No row fits → it's a gap.** The field is missing upstream, the connection is missing, or the element is missing. Close it: add the field upstream (and carry it through the chain the same way), add the connection (`create_flow`), or add the element. Only when the source genuinely can't be decided from the model, apply the questioning rule: ask, and leave a pin (`create_pin`).

`isGenerated` and `isUserInput` both mean "this information starts here". Neither is ever a way to say "I don't know where this comes from". Don't mark a field `isGenerated` to make a gap go away, and don't invent a generated origin in the middle of a flow.

In a write, declare the source while you still know it: `source` on the field in `place_element` / `update_element` creates the flow if it is missing and records the mapping in one step. A field is exactly `{name, fieldType, isList?, isGenerated?, isOptional?, isUserInput?, isPii?, example?, subfields?}`; any other key is refused (an update may also send a field's existing `id`, which is how a rename keeps its mappings).

```json
{ "scope": "command", "name": "Reserve Bike", "fields": [
  { "name": "bikeId", "fieldType": "UUID" },
  { "name": "endTime", "fieldType": "DateTime", "source": { "from": "Reserve a Bike", "field": "returnTime" } },
  { "name": "requestedAt", "fieldType": "DateTime", "isGenerated": true }
] }
```

## Planned sources that aren't on the sheet yet

While storyboarding, a displayed screen field is often meant to come from a read model that hasn't been designed yet. A `source` can only name an element that is already on the sheet, so record it in the note on the screen's cell instead (`set_note`; read the note first, a write replaces it), one line per planned read model, `Planned read model "<Read Model Name>": <field>, <field>`, and flag it as a gap. `eventmodeling-identifying-outputs` groups screen fields by those lines to design the read model. Once the read model is connected, a same-named field needs no mapping, and a differently named one gets `source: { from: "<Read Model Name>", field: "<readModelFieldName>" }`. A displayed field is never marked `isGenerated` just because its read model is missing.

## Renames move the lineage too

Renaming a field breaks every downstream field that matched it by name, or that has a `source` pointing at `<oldName>`. In the same write, either rename those too or set their `source` to the new name. `eventmodeling-adding-attributes` does this for a chain. When you rename with `update_element`, send the field with its existing `id` and the new `name` (a `fields` patch replaces the whole list), so mappings that point at it survive. Scenario example values keyed by the old name follow a rename sent with the field's `id` automatically.

## Worked example — reserving a bike

```
SCREEN      "Reserve a Bike"   bikeId (user input), startTime (user input), returnTime (user input),
                               customerId (generated, from session), bikeCategory (source: Available Bike View)
COMMAND     Reserve Bike        bikeId, startTime, customerId (same names, nothing to write), endTime (source: returnTime),
                               requestedAt (generated: current time, read by "startTime must be in the future")
EVENT       Bike Reserved       bikeId, startTime, endTime, customerId (same names), reservationId (generated),
                               reservedAt (source: requestedAt)
READMODEL   Active Reservation  reservationId, bikeId, startTime (same names), expiresAt (source: Reservation Confirmed),
                               minutesLeft (generated)
SCREEN      "Reservation"      reservationId, startTime, expiresAt (same names, read model connected)
```

Every field on the way traces back to a generated or user-input origin. Nothing appears without a source.
