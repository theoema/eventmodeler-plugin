---
name: eventmodeling-analyzing-stressors
description: "Residuality-theory stressor analysis of one sheet, run as a conversation. Finds wide stressors (most from outside the model), describes the residue each leaves, builds a stressor-by-element incidence matrix to expose hidden coupling, and proposes redesigns as new slices. Read-only until the person confirms; what it finds is then recorded as notes and pins. Use when a sheet looks right but you want to know how it behaves when the world goes wrong: an outage, a new law, a key person leaving, a partner stopping delivery. Do not use for: structural validation (use `eventmodeling-validating-event-models`), business questions about the flows as modeled (use `eventmodeling-reviewing-business-fit`), or designing the translation of an external system's events (use `eventmodeling-translating-external-events`)."
---

# Analyzing Stressors — What Is Left of the Model When Things Go Wrong

This step applies the shared element rules in **`eventmodeling-core-rules`** — read it once per session if you haven't already; it defines what a COMMAND/EVENT/READMODEL/SCREEN/AUTOMATION is and the translation chain this analysis often ends in, so this step doesn't restate them. Tool names and conventions are in `eventmodeling-core-rules` → `../eventmodeling-core-rules/references/tools.md`.

The analysis runs over several turns of the conversation: each step ends with one message, and the person's answer is the next turn. Find where you are from the conversation: the person's last answer says which step comes next.

**This skill never changes the model** until the person says yes in Step 3. Then it builds the redesign and records what it found (Step 4). Until then it only reads.

---

## Concepts (get these right)

- **Stressor**: anything that can happen to the system or its environment: technical, business, legal, market, people, nature. Deliberately WIDE, far-fetched ones included. Not a risk list, not a prediction: the theory holds that *which* stressors you pick barely matters; what matters is that they come from many directions. So most must come from **outside** the model: a power outage, a pandemic, a new law, the company being acquired, the key administrator quitting, a competitor going free. Only loosely related to the sheet is fine.
- **Residue**: what is LEFT of the system after the stressor hits: which slices still work, which state or data survives or is lost. You observe a residue; you don't build it. The fix is NOT the residue.
- **Goal**: change the model until every residue is acceptable. The architecture becomes the sum of those residues, so it also survives stressors nobody listed.
- **Incidence matrix**: stressors by elements. Elements hit by the SAME stressors are hidden-coupled.

---

## Step 0 — Read once

The sheet comes from the person's request. If the model has several sheets and the request names none, ask which (one short question listing them from `list_models` and `get_model_view`) and end the turn.

Read once and reuse for the whole round:

- `check_model` for the sheet. A residue is read off the wiring, so a missing or backward flow gives a wrong one. Don't fix findings here; if one distorts the analysis, say so in a line.
- `get_model_view` (slice statuses) and `get_sheet_view` for the grid, then `get_slice_view` for the slices, with their fields.
- `list_pins` with `status: "all"`: questions already asked about this sheet, so you don't ask them again.

Residues name the real slices and elements they hit, by name, never by cell address and never with field names. Every element you name is one you read here.

---

## Step 1 — Stressors and their residues

About **15** wide stressors, mixed:

- **at least 5 external**: things that happen to the business or the world, not to the model: power outage, office flooded, pandemic, new privacy law, company acquired, key person leaves, a competitor goes free, the main customer goes bankrupt, the cloud region is gone for a week;
- the rest from the sheet's own business (a customer changes their mind, prices change, a partner stops delivering), still at business level.

Write them the way a business person would say them, short: **no** field names, attribute values, concurrency races, token or API mechanics. One stressor per line, no "(e.g. X, Y)" sub-lists. Order the list so external and internal stressors alternate, not grouped.

Per stressor: an id (`S1`, `S2`, and so on), the stressor in at most 8 words, the element most affected, and `Residue: <what still works / what is lost>` in **one short sentence** of plain language that names slices or elements. A table:

```markdown
| # | Stressor | Most affected | Residue |
|---|---|---|---|
| S1 | Payment provider is down for a day | Order Paid | Orders still come in, but nothing ships until payments return |
| S2 | Power outage in the warehouse | Shipping List | Orders are paid, nobody can pick them, and customers aren't told |
```

End the message with two asks: **which residues are unacceptable** (they answer with the ids, e.g. "S2, S5, S9"), or **more stressors first**.

### "More"

Offer more only **before** the person has named the unacceptable ones; naming them is the go-ahead.

- **More** (before any ids were named): one new table holding **every stressor offered so far plus** about 10 **fresh** ones (15 + 10 = 25, then 35, and so on), never just the 10 new ones. Earlier stressors keep their id, wording, element and residue unchanged and come first; the fresh ones follow, none already offered, with the same external and internal mix and the same plain business wording as above (no races, outages of a lookup, or other mechanics), aimed at the elements and categories (nature, legal, people, market, data, and so on) the earlier lists hit least. Ids continue the numbering (`S16`, `S17`, and so on). Say how many are new ("10 new ones added below S15, 25 in total") without naming elements or mechanics. The newest table is the one to answer from.
- **Ids named**: go **straight to Step 2** in this same turn with every stressor named across all tables (the union by id). Never ask again whether to build the matrix or offer more; however few were named, build the matrix, and with fewer than 5 say that the coupling it shows is still thin.
- **More** or **No more** said after the matrix is built: ignore it with one line; the analysis has moved on.

---

## Step 2 — Incidence matrix

For each named stressor, list every element it hits (not just the most affected one), taken from the read in Step 0, never guessed. Then one table:

- one row per element hit at least once, sorted by hits (most first), so the hotspots are on top;
- columns: the element's name, "Hits", then one column per named stressor labelled `S1`, `S2`, and so on, with "x" where it hits and blank where it doesn't;
- more than 10 named stressors: replace the stressor columns with one "Hit by" column ("S1, S4, S9");
- a legend under it: "S1 = (stressor); S4 = (stressor)".

```markdown
| Element | Hits | S1 | S4 |
|---|---|---|---|
| Order Paid | 2 | x | x |
| Shipping List | 1 | x |   |

S1 = Payment provider down; S4 = Price changes mid-checkout
```

Below it, name the **clusters** (elements hit by the same stressor pair), the shared cause behind each cluster, and the hotspots (most hits). Ask which cluster to redesign first; when the person answers in words, go on with that cluster. Keep the text short; the detail is in the table. Name elements, never addresses.

---

## Step 3 — Redesign one cluster

Propose new slices (translations, reconciliation todo lists, grace periods, second entry paths, and so on) and state the **new residue** for each stressor that hit the cluster. Ask one yes-or-no question: "Add these N slices for <cluster>?", with the slices and their new residues in the message, kept short.

A yes covers building the slices **and** recording the analysis (Step 4). A no: acknowledge it and offer the next cluster. If the person declines every redesign but named unacceptable residues, ask once whether to leave those as pins: an unacceptable residue with no decided fix is an open question.

---

## Step 4 — On yes: build it, then record it

**Build.** Add each new slice to the sheet with the modeling skills, one slice at a time, in story order: `insert_slice` where it belongs, then the elements through the step skill that owns them (`eventmodeling-translating-external-events` and `eventmodeling-designing-automation-chains` for translations and todo lists, `eventmodeling-identifying-inputs` for a second entry path, `eventmodeling-elaborating-scenarios` for the cases the new slice must pass). Run `check_model` afterwards and fix its errors. A redesign that changes slices that already exist, rather than adding new ones, is a change the person didn't confirm: ask first (`create_pin`, or in the conversation).

**Record.** Findings that should outlive the conversation go in the model:

- **A note on the sheet's first slice** (`set_note` on column A; read it first and keep everything already there): a "Stressor Analysis" section with the round, the stressors named unacceptable, the clusters and their shared causes, the redesign agreed and the new residue for each stressor, and the residual index once Step 5 has one (see `eventmodeling-interview-protocol` for how a sheet-wide record is kept).
- **A line on each hotspot or redesigned element's cell** (`set_note` on the cell; read it first): the stressor, the residue, and what resolved it. Only for the hotspots and what you changed, not for every element.
- **A pin** (`create_pin`) for every residue still unacceptable that has no agreed fix, and for anything only the business can answer ("how long can the warehouse be offline before customers must be told?"). Anchor it on the cells the residue hits. A pin waits for an answer; a note records something settled.

Reply in a few lines: what was added and what was recorded.

---

## Step 5 — Validate

Generate a **fresh** set of stressors not used in the earlier rounds, and compare how many have an acceptable residue **before** and **after** the redesign (the residual index). The "before" is the model as it was: `get_history` shows what changed if you need it. Present them as in Step 1; the person names the unacceptable ones, and your message states the index ("before: 6 of 12 acceptable, after: 10 of 12"). Add the index to the sheet's "Stressor Analysis" note. Repeat rounds until new stressors stop hitting new elements.

---

## Conversation style

- Explain a residue with a before and after example if asked (same stressor, different leftover).
- Keep each message short; the detail goes in the table.
- No ids or cell addresses in the text: element names are enough.
