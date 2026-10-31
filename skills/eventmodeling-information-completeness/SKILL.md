---
name: eventmodeling-information-completeness
description: The rule for where every field's value comes from. Use it whenever a field is added, renamed or reviewed on any element (SCREEN, COMMAND, EVENT, READMODEL, AUTOMATION), whether in a full modeling run, a one-off edit, or a fix. Each field either matches a same-named field on a connected upstream element, maps to a differently named field or a connected event, or is marked generated or user-input where the information starts. Anything else is a gap to close. Not a step of its own. Other skills reference it instead of restating it.
---

# Information Completeness

Information doesn't appear out of nowhere. Every field on every element either comes from an element **connected upstream** of it, or starts at this element. Check this **each time you add or rename a field**, before you write it. Don't leave it for a later completeness pass.

## The four valid states of a field

| Where the value comes from | What to write | Example |
|---|---|---|
| A connected upstream element has a field **with the same name** | nothing: no mapping, the same name *is* the mapping | `Bike Reserved.bikeId` ← `Reserve Bike.bikeId` |
| A connected upstream element has it **under a different name** | `source: { from: "<upstream element>", field: "<sourceFieldName>" }` on the field (a subfield as `"<field>.<subfield>"`), or `set_flow_mapping` on the flow | `Reserve Bike.endTime` sourced from `Reserve a Bike` screen field `returnTime` |
| It comes from one specific connected **event** (or other connected element), e.g. because several upstream elements carry the field | `source: { from: "<Event name>" }` (add `field` if the name differs) | `Active Reservation.expiresAt` sourced from `Reservation Confirmed` |
| **Nothing upstream** supplies it | `isUserInput: true` when a person types it into a screen; `isGenerated: true` for everything else that starts here: a session value, a generated id or timestamp, a value computed at this element | `Reserve a Bike.startTime` (user input), `Bike Reserved.reservationId` (generated) |

These are the forms the model resolves. A flow carries field mappings by field name, and the traceability check follows them. Write the field name or the element name **bare**. Combined forms such as `"Bike Reserved.bikeId"`, `"user-input"`, `"session:x"` or `"derived:..."` resolve to nothing.

Both flags mean "this information starts here". Neither is a way to say "I don't know where this comes from".

## Upstream means inbound, per element type

Lineage always points to where the value comes **from**. It never points downstream.

| Element | Its upstream (where its fields can come from) |
|---|---|
| SCREEN | the READMODEL it displays. The COMMAND it submits is **downstream**: never map a screen field to it. |
| COMMAND | the SCREEN or AUTOMATION issuing it |
| EVENT | the COMMAND producing it |
| READMODEL | the EVENTs connected to it (`EVENT → READMODEL`) |
| AUTOMATION | the READMODEL (todo list) it reads |

So information typed into a form is `isUserInput` **on the screen**. The command, event and read model then pick it up by the same name, or by a mapping where it is renamed. Contextual values a command's validation reads (current time or date, acting user, tenant) are `isGenerated` on the **command**. Commands carry everything they need to validate (`eventmodeling-identifying-inputs` → "Command input is self-contained"), and the event takes a recorded time from there. Only an id the system creates when recording the fact is `isGenerated` on the event. A count, sum or duration the projection calculates is `isGenerated` on the read model.

## How to check a field

1. Look at the element's inbound neighbours: `get_slice_view` shows its fields and the flows into it, with their mappings. `check_model` lists what it can already see as `untraceable-field` (some inbound flow exists but covers the field) and `unsourced-fields` (no inbound flow at all and no stated origin).
2. Pick the row above that matches. One neighbour with the same name, and nothing to write, is the common case.
3. **No row fits → it's a gap.** The field is missing upstream, the connection is missing, or the element is missing. Close it: add the field upstream (and carry it through the chain the same way), add the connection (`create_flow`), or add the element. Only when the source genuinely can't be decided from the model, ask (`create_pin`).

Don't mark a field `isGenerated` to make a gap go away, and don't invent a generated origin in the middle of a flow.

In a write, declare the source while you still know it: `source` on the field in `place_element` / `update_element` creates the flow if it is missing and records the mapping in one step.

```json
{ "scope": "command", "name": "Reserve Bike", "fields": [
  { "name": "bikeId", "fieldType": "UUID" },
  { "name": "endTime", "fieldType": "DateTime", "source": { "from": "Reserve a Bike", "field": "returnTime" } },
  { "name": "requestedAt", "fieldType": "DateTime", "isGenerated": true }
] }
```

## Planned sources that aren't on the sheet yet

While storyboarding, a displayed screen field is often meant to come from a read model that hasn't been designed yet. Don't mark it generated, and flag it as a gap: leave the field unflagged and record it in the note on the screen's cell (`set_note`; read the note first, since it replaces the text), one line per planned read model: `Planned read model "<Read Model Name>": <field>, <field>`. `eventmodeling-identifying-outputs` reads those lines to design the read models. Once the read model is connected, a same-named field needs no mapping, and a differently named one gets `source: { from: "<Read Model Name>", field: "<readModelFieldName>" }`. A displayed field is never marked generated just because its read model is missing.

## Renames move the lineage too

Renaming a field breaks every downstream field that matched it by name, or that was sourced from `<oldName>`. In the same pass, either rename those too or point their source at the new name. When you rename with `update_element`, send the field with its existing `id` and the new `name` so mappings that point at it survive, and send the whole field list: a `fields` patch replaces it. For a chain, walk the flows downstream with `get_slice_view` and fix each field.

## Worked example — reserving a bike

| Element | Field | How it's written |
|---|---|---|
| SCREEN `Reserve a Bike` | `bikeId`, `startTime`, `returnTime` | `isUserInput` |
| | `customerId` | `isGenerated` (from the session) |
| | `bikeCategory` | `source: { from: "Available Bike View" }` |
| COMMAND `Reserve Bike` | `bikeId`, `startTime`, `customerId` | same names as the screen, nothing to write |
| | `endTime` | `source: { from: "Reserve a Bike", field: "returnTime" }` |
| | `requestedAt` | `isGenerated` (current time, read by "startTime must be in the future") |
| EVENT `Bike Reserved` | `bikeId`, `startTime`, `endTime`, `customerId` | same names |
| | `reservationId` | `isGenerated` |
| | `reservedAt` | `source: { from: "Reserve Bike", field: "requestedAt" }` |
| READMODEL `Active Reservation` | `reservationId`, `bikeId`, `startTime` | same names |
| | `expiresAt` | `source: { from: "Reservation Confirmed" }` |
| | `minutesLeft` | `isGenerated` (calculated by the projection) |
| SCREEN `Reservation` | `reservationId`, `startTime`, `expiresAt` | same names, read model connected |

Every field on the way traces back to a generated or user-input origin. Nothing appears without a source.
