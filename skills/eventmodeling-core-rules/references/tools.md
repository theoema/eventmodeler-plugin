# Concept → Tool

The one place that says which EventModeler tool does what. Every `eventmodeling-*` skill speaks in modeling concepts; this file turns them into calls. You are already connected: there is nothing to log in to, no token, no other interface. Tools are named bare, e.g. `place_element`.

Every tool takes `modelId` (from `list_models`). Tools that act on a sheet take `sheet` (name or id); omit it only when the model has exactly one sheet.

## Vocabulary

| Event-modeling concept | In EventModeler |
|---|---|
| Timeline / board / chapter | A **sheet** (`create_sheet`). A model has many sheets. A sheet is a journey someone lives through, start to outcome. |
| A journey branching into another sheet's journey | `connect_sheets`, from the slice where it branches. A visual connector only, with no fields and no semantics, and rare; see the core rules' Connecting Sheets. Elements shared between sheets are linked copies. |
| Column | A grid column, lettered `A`, `B`, … left to right. Left to right is time. |
| Slice | A **first-class** column or contiguous span of columns, with a name and a status. `insert_slice`, `span_slice`, `split_slice`, `update_slice`, `move_slice`. |
| Lane / row | A numbered, typed row (`insert_lane`, `update_lane`, `move_lane`). Four kinds, below. |
| Cell | Column × lane, addressed `B2`. Holds **one** element. Only scenarios may stack. |
| Element in a cell | `B2.1` (1-based position in the cell) |
| COMMAND | `command` — `interaction` lane |
| READMODEL | `readmodel` — `interaction` lane |
| SCREEN | `screen` — `actor` lane |
| AUTOMATION | `processor` — `actor` lane |
| EVENT | `event` — `swimlane` lane |
| External system event | `external-event` — `swimlane` lane. It opens a todo-list read model (`ExternalEventToReadModel`). An `event` placed in a second swimlane lane (named for the other system) is the other way to show another team's own timeline. |
| SCENARIO | `scenario` — `specification` lane |
| Connection | A **flow** (`create_flow`) |
| Field mapping | `set_flow_mapping`, or `source` on a field |
| Linked copy | `copy_element` |
| Comment / finding / decision note | `set_note` |
| Open question for the human | `create_pin` |
| Validate | `check_model` |
| Look at the board | `get_model_view`, `get_sheet_view`, `get_slice_view`, `get_cell_view`, `render_sheet` |
| Screen mockup | `place_element` with `design`, or `set_screen_design` |
| Call out part of a screen | `marks` on `place_element` / `set_screen_design` |

Lane kinds and what each accepts (strict; a wrong element is refused with `LANE_MISMATCH`):

| Lane kind | Accepts |
|---|---|
| `actor` | `screen`, `processor` |
| `interaction` | `command`, `readmodel` |
| `swimlane` | `event`, `external-event` |
| `specification` | `scenario` |
| `notes` | nothing: each cell's content is its cell note, shown as markdown. Write it with `set_note` on the cell |

A sheet may have several lanes of one kind: one `actor` lane per human role, one `swimlane` lane per system whose events appear. Conventional order, top to bottom: actor, interaction, swimlane, specification. A notes lane goes where it reads best, usually at the top or the bottom.

## Addresses

- Column: letter (`C`) or slice name. Row: number (`2`) or lane name. Cell: `C2`. Element: `C2.1`. A few tools (`remove_target`, the screen tools) accept a bare `C2` for the cell's only element.
- Addresses shift when columns or rows are inserted or moved. Re-read before you reuse one from earlier. In notes and pins, name elements; don't quote addresses.
- A slice's columns are lettered individually; a slice spanning three columns owns three letters. Cells belong to a column, not to a slice.

## Reading

Read in tiers, narrowest that answers the question.

| Tool | Gives you |
|---|---|
| `list_models` | Models you can reach, with your role |
| `get_model_view` | Orientation: sheets, slices (name, status, columns), counts |
| `get_sheet_view` | One sheet's grid: element names and types per cell, flows, scenario titles. No fields, no scenario bodies. |
| `get_slice_view` | One slice in working detail: elements with fields, cell notes, scenario bodies, flows with mappings, the slice's scenario coverage. Read this before editing a slice. |
| `trace_field` | One field followed through the model: `upstream` (where its value comes from, back to where it starts), `downstream` (everywhere it flows), `origins` (user input, generated, or `unsourced` — a gap). Across flows and sheets, by mapping or same-named field. |
| `get_cell_view` | One cell (or one element) at full detail, plus its slice and lane notes |
| `render_sheet` | A picture of the sheet (`cols: "A:D"` to crop wide sheets) |
| `render_screen` | One screen's HTML design as an image |
| `get_history` / `revert_model` | What changed, and rewinding. Every removal is recoverable. |

A linked copy shows the original's name and carries **no fields of its own**: read fields from the original (`originalStickyId`).

## Structure

| To | Use |
|---|---|
| Create a model | `create_model` |
| Create a sheet | `create_sheet` (goes under the existing sheets; `below` / `above` / `rightOf` / `leftOf` another sheet to put it elsewhere) |
| Arrange sheets on the canvas | `move_sheet` (`below` / `above` / `rightOf` / `leftOf` another sheet; sheets in the way move along) |
| Add a slice (one or more columns) | `insert_slice` (`name`, `at` = 1-based column position, `cols`) |
| Widen / narrow a slice | `span_slice` (`to` or `cols`) |
| Break a spanned slice in two | `split_slice` |
| Rename / set status / annotate a slice | `update_slice` (`status`: draft, created, planned, in-progress, blocked, done) |
| Record a slice's scenario coverage | `update_slice` with `coverage: { percentage, missing: [{ title, given?, when?, then? }] }` (`null` clears); see `eventmodeling-assessing-scenario-coverage` |
| Reorder | `move_slice`, `move_column`, `move_lane` |
| Add a lane | `insert_lane` (`kind`, `name`) |
| Rename a lane / change its kind | `update_lane` (refused while the lane holds elements the new kind rejects) |
| Remove an element, a row or a column | `remove_target` (cascades to flows, mappings, scenario entries) |
| Delete a sheet | `delete_sheet` (reports the linked copies on other sheets that go with it) |
| Group sheets in a named box (a bounded context, a team's area) | `set_context` with `name` + `sheets`; pass `context` to rename it or re-fit it around other sheets. `delete_context` removes the box, never the sheets. `get_model_view` lists contexts and their sheets. Moving a sheet doesn't move its box: re-fit it. |

One cell holds one element. To fit a second element on the same lane in the same slice, widen the slice first (`span_slice`, or `insert_slice` with `cols`). Never fight the grid by stacking.

## Elements

- `place_element`: `cell`, and `element` = `{ scope, name, fields? }` where `scope` is `command`, `event`, `readmodel`, `screen`, `processor`, `external-event` or `scenario`. For a screen also pass `design` (HTML) in the same call. Ids and geometry are generated: never pass them.
- `update_element`: `element` address + `patch`. Merges by default (only the keys you pass change). A `fields` patch **replaces the whole field list**: send every field. Unchanged field names keep their ids, so mappings survive. To rename a field without losing mappings that point at it, send it with its existing `id` and the new `name`.
- `move_element`: to another cell; keeps identity, flows and scenarios.
- `copy_element`: a linked copy of a `command`, `event`, `readmodel`, `screen` or `external-event` in another cell. From another sheet: `fromSheet` names the original's sheet, `sheet` the copy's. A copy of a copy resolves to the original. Automations cannot be copied. Copy only what you then wire on this sheet; `check_model` reports a copy wired to nothing as `linked-copy-unwired`.
- `remove_target`: removal. Removing an original removes its linked copies and the flows touching them; the reply's `removedCopies` names each one, with its sheet.
- A list-shaped read model (a todo list, any query that returns rows) is marked by the node-level list flag, `isList: true` on the read model itself in `place_element` / `update_element`. It is not a field's `isList`. The views return it on the read model; a linked copy reports its original's, and takes it from there (set it on the original).

### Field shape

Exactly `{ name, fieldType, isList?, isGenerated?, isOptional?, isUserInput?, isPii?, example?, subfields? }`. `fieldType` is one of `UUID`, `Boolean`, `Double`, `Decimal`, `Date`, `DateTime`, `Long`, `Int`, `String`, `Custom` (a `Custom` field has `subfields`). Any other key is refused: the call fails, changes nothing, and names the keys that are accepted.

- `isList`: this field holds several values.
- `isGenerated`: the information starts here (minted or computed at this element).
- `isUserInput`: the information starts here because a person types it into a screen.
- `isOptional`: may be absent.
- `isPii`: the value identifies or describes a person (name, email, address, phone). It belongs to the value, not the element: mark it on every element the value passes through.
- `example`: an illustrative value, a string. On a `Custom` field put examples on its subfields.

### Field lineage

A field's source is recorded structurally, in flows:

1. Same name on a connected upstream element: nothing to write.
2. Different name, or a specific upstream element: `source: { from, field? }` on the field in `place_element` / `update_element` (`from` = address or element name on this sheet; `field` defaults to this field's own name), or `set_flow_mapping` on the flow (`mappings: [{ source, target }]` by field name, dotted for `Custom` subfields). `source` creates the flow if it doesn't exist yet and the pairing is valid. `set_flow_mapping` replaces the flow's whole mapping list.
3. The information starts here: `isUserInput` (typed into a screen) or `isGenerated` (anything else that starts here).

`source` is consumed at write time and never stored on the field.

## Flows

`create_flow` takes `from` and `to` element addresses; the type is inferred from the pair. Valid flow types:

| From → To | Flow type |
|---|---|
| command → event | `CommandToEvent` |
| command → external-event | `CommandToExternalEvent` |
| event → readmodel | `EventToReadModel` |
| readmodel → screen | `ReadModelToScreen` |
| readmodel → processor | `ReadModelToProcessor` |
| screen → command | `ScreenToCommand` |
| processor → command | `ProcessorToCommand` |
| external-event → processor | `ExternalEventToProcessor` |
| external-event → screen | `ExternalEventToScreen` |
| external-event → readmodel | `ExternalEventToReadModel` |

Any other pair is refused (`INVALID_FLOW`). Placing an element never connects anything; every flow is yours to create. `create_flow` does not police direction or count: it accepts a backward flow, a second issuer for a command and a screen wired to several commands. `check_model` reports each of them (`flow-backward`, `command-multiple-issuers`, `screen-multiple-commands`), so run it after wiring.

`remove_flow` (by `from` + `to`, or `flowId`). `ExternalEventToProcessor` and `ExternalEventToScreen` are on the list, but the method never uses them: an external event opens a todo-list read model (`ExternalEventToReadModel`), an automation is triggered only through its own todo-list read model, and a screen reads a read model. `check_model` warns about both as `external-event-direct`.

## Scenarios

`set_scenario` writes a scenario in one call: `{ scenario: "<name>", cell: "<specification cell>", … }` creates it in that `specification`-lane cell of its subject's column and fills it; on an existing scenario (by name or address) `cell` is ignored. A scenario is one of two kinds:

- **Given/When/Then** — `given`, `when`, `then` (below): one isolated transition.
- **Storyline** — `storyline: [beats]`: one read model walked through its states. A beat is an element reference, `{ element, fieldValues }` (an event's data or a single-record read model's state), `{ element, rows }` / `{ element, expectEmptyList: true }` (a list read model's state), or `{ note }`. It starts with the read model, events move it on, and each state follows the events that cause it; `set_scenario` refuses a storyline that breaks that order. `storyline` replaces all beats and never mixes with `given`/`when`/`then`.

For a Given/When/Then scenario:

- `given`: events, external events and read models (prior state). `when`: the command (or event) that triggers. `then`: events, commands, read models, or an expected failure `{ errorType, errorMessage }`.
- Entries are element addresses or names, or `{ element, fieldValues: { fieldName: value } }` for concrete example data. Each section you pass **replaces** that section; omit it to leave it unchanged. `[]` clears.
- An entry may name an element on **another sheet**: `Reset Password!B3.1`, or its name. A name is looked up on the scenario's sheet first, then across the model. Never place a linked copy for a step. In the views each entry has a `ref` (`B3.1`, or `Sheet!B3.1`) saying where its element sits. In `build_sheet_from_spec`, scenario steps are spec addresses or `Sheet!B3.1`.
- When a copy of an element exists on the sheet, a name matches both and is ambiguous: use the original's address. Reference the original, never the copy: a copy has no fields to take values, so an entry addressed at a copy is stored against its original.
- A scenario has `name`, optional `description`, optional `contextNote` (one line of background shown under the title, for what no element can express: "The payment provider declines the card."; `""` clears it), and `position` (order among the slice's scenarios).
- A decided rejection is a `then` entry `{ errorType, errorMessage }`. There is no separate flag.
- When `then` is exactly one list read model (`isList: true`), `rows: [{ <field name>: value }, …]` gives its expected rows and `expectEmptyList: true` says it comes out empty. They sit on the scenario, not on the `then` entry, exclude each other, and are refused for any other `then`. `rows: []` / `expectEmptyList: false` clear them.

## Notes, questions, findings

- `set_note`: free text on a **cell** (`B2`), a **column/slice** (`A` or slice name) or a **lane** (`2`). Never on an element. It **replaces** the existing note: read the note first (`get_cell_view`, `get_slice_view`) and write old plus new. Empty text clears. In a `notes` lane the cell note is the cell's content, written in markdown and shown in full on the canvas.
- `create_pin`: a question or open item for the human, anchored to what it is about: `regions`, each a `sheet` with `cells` (`B2`, `B2:D4`, `C`, `3`), or with neither for the whole sheet. Give several when the question spans them (the same step on two sheets). It waits on them until they reply. `list_pins` with `status: "waiting"` lists the questions still unanswered.
- A pin a human leaves can cover several regions across sheets; `get_pin` returns each with its cells and one picture per sheet. Read all of them: together they are the context the note is about.
- Pins a human leaves for you: `list_pins` → `get_pin` → `claim_pin` → do the work → `complete_pin`, or `reply_pin` to ask first.

Use a note for something decided or observed. Use a pin for something that needs an answer.

## Screens

`place_element` with `design`, or `set_screen_design` afterwards: sanitized HTML and CSS (no scripts or handlers). `set_screen_theme` sets the model-wide look once, so each screen stays a few lines of semantic markup. Check with `render_screen`.

Marks call out one part of a design: put `data-em-mark-id="<id>"` on the element and pass `marks: [{ id, color?, blurOutside?, whiteOutside? }]` with the design. The element is outlined (default red `#e74c3c`); `blurOutside` blurs the rest of the page, `whiteOutside` whites it out. They show on the canvas, in `render_screen`/`render_sheet`/`get_pin` pictures, and in board files. `set_screen_design` without `marks` keeps the marks whose element is still in the new HTML; `marks: []` clears them. A linked screen copy shows its original's design and marks; for the same page with another part marked, place a plain screen of the same name.

## Validation

`check_model` (optionally one `sheet`) runs the mechanical checks and returns findings, errors first, each with a severity, a stable `check` code, the sheet, a target (an element address, a slice name or `column C`) and a message naming the cells to look at. It does not judge anything: a clean result means the structure is sound, not that the model is right. Run it before any hand-rolled checking, then apply the rules by reading the slices.

| Area | Codes (severity) |
|---|---|
| Wiring | `command-without-event` (error), `command-without-issuer` (error), `command-multiple-issuers` (error), `screen-multiple-commands` (error), `event-without-command` (warning), `automation-without-todo-list` (error), `read-model-without-source` (error), `read-model-unconsumed`, `screen-disconnected`, `screen-mark-unanchored`, `event-disconnected`, `orphan-element`, `linked-copy-unwired` (warnings), `dangling-flow` (error) |
| Direction and layout | `flow-backward` (error; the closing event into a todo-list read model is exempt), `column-multiple-events` (error), `sheet-starts-with-command` (warning), `state-change-without-trigger` (warning), `external-event-direct` (warning) |
| Slices | `slice-mixed-shape` (error), `empty-slice` (info) |
| Naming | `event-name-not-past-tense`, `command-name-past-tense` (warnings) |
| Field sources | `untraceable-field` (warning), `unsourced-fields` (info), `pii-mismatch` (warning: a flow carries a value from a PII-marked field into an unmarked one, or the reverse) |
| Scenarios | `scenario-when-shape`, `scenario-then-shape`, `scenario-on-linked-copy`, `scenario-without-examples`, `storyline-shape` (warnings), `slice-without-scenario`, `no-error-scenario`, `todo-list-without-storyline` (info), `scenario-missing-ref` (error) |
| Candidates, worth a look | `command-fan-out`, `read-model-fan-in`, `slice-scenario-outlier` (info) |

Linked copies resolve to their originals throughout: a flow landing on a copy credits the original, a screen and its copy are one issuer, and a copy implies no slice shape of its own.

## Replies

A write answers with what it did plus the sheet's grid **outline**: column letter → slice, row number → lane. That is enough to re-address after columns or rows shift. For contents, read `get_sheet_view` / `get_slice_view` again.

A parameter or key a tool doesn't take fails the call (nothing changes) and the error lists what it does take. If the instructions or a skill name a parameter the tool refuses, the session's tool list predates the server: ask the user to reconnect the EventModeler MCP server, then retry.

## Working alongside others

People and other agents edit the same model live.

- `claim_pin` answers with `alsoInProgress`: the other pins agents are working on, and their sheets. Stay off those sheets where you can. Never delete an original that another sheet copies from without asking (`create_pin` or `reply_pin`): its copies go with it.
- Before re-adding something that isn't there, read `get_history` for that sheet: a person may have removed it on purpose. Ask rather than restore it.
- A pin region with `sheetDeleted` points at a sheet that is gone. Read `get_history`, and ask the human if the note no longer makes sense.

## Batching

Prefer the granular tools: one `place_element`, one `create_flow` at a time, in story order, left to right. Use `apply_operations` only for a unit that must land all-or-nothing (a failure applies nothing; resend the whole batch). A step for a tool that takes no arguments may omit `args`. `build_sheet_from_spec` creates a whole sheet from one spec, all-or-nothing; reach for it only to scaffold a sheet whose shape is already settled. A screen in a spec takes `design` as HTML, as `place_element` does.
