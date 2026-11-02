---
name: eventmodeling-assessing-scenario-coverage
description: "Assess how much of each slice's business logic its scenarios (Given/When/Then and storylines) specify, and store the result on the slice: a percentage plus the meaningful cases still missing. Use when asked how complete the scenarios are, which slices need more scenarios, or before declaring the scenario work done. Do not use for: writing the scenarios themselves (use `eventmodeling-elaborating-scenarios`), checking that every field has a source (use `eventmodeling-checking-completeness`) or validating structure and method rules (use `eventmodeling-validating-event-models`)."
---

# Assessing Scenario Coverage

This step applies the shared element rules in **`eventmodeling-core-rules`** — read it once per session if you haven't already. Tool names and conventions are in `eventmodeling-core-rules` → `../eventmodeling-core-rules/references/tools.md`.

You are in **Critic Mode**: judge what the scenarios specify, honestly, and record the result. The only thing you write is a slice's coverage. You never add or change scenarios, elements or slice status; the cases you find missing are the input for `eventmodeling-elaborating-scenarios`, which is where they get written.

## What coverage is

A slice's coverage is a percentage, the share of its meaningful use cases that its scenarios specify, plus up to 10 of the meaningful cases still missing. It sits on the slice (`update_slice` with `coverage`) and the canvas shows the percentage on the slice header: red below 50, orange below 70, green from 70. The server stamps when it was assessed and which scenarios, in which state, it was based on. `get_sheet_view` and `get_slice_view` return it per slice as `percentage`, `missing`, `assessedAt` and `stale`; `get_model_view` gives only the percentage. `stale: true` means a scenario of the slice was added, removed or edited after the assessment: the number no longer describes the slice, so assess it again.

The number guides what to model next. A flattering number is worse than none.

## Step 1 — Scope

From the person's request: one sheet, a few slices, or every sheet of the model. If a name matches several sheets or slices, ask. `list_models` and `get_model_view` give the slices and the coverage percentage they already carry; `get_sheet_view` adds whether each is stale.

When the request covers the whole model, leave out slices whose coverage exists and is not stale, and say how many you left out; assess them again only if the person asks for a full reassessment.

Run `check_model` for the sheets in scope before you read them. Its scenario findings tell you which existing scenarios cannot be trusted yet: `scenario-missing-ref` (a step points at an element that is gone), `scenario-without-examples` (the steps only name elements, or a storyline read-model beat says nothing about its state), `scenario-when-shape`, `scenario-then-shape` and `scenario-on-linked-copy`. A scenario flagged like that does not yet specify its outcome. Count it as not covering its case and say so in your reply, so that fixing it is the next step, not a rewrite of the number.

## Step 2 — Read each slice

`get_slice_view` returns the slice's elements with their fields, its flows and its scenarios with full bodies: the steps, the rows of a list read model, a storyline's beats. Read it before you rate the slice.

- **Slices with no rating of their own.** A column whose only command or read model is a linked copy continues an original elsewhere; the scenarios live on the original, and the original's slice is the one to rate. A slice with no element (`empty-slice`) has nothing to rate. Name every slice you skipped, with the reason.
- **Which slice a scenario counts for.** A scenario counts for the slice of the command or read model it specifies, wherever its card happens to sit. If it sits in another slice's cell, mention it in your reply; nothing is moved here.
- Fields, notes and flows tell you what the logic is. Where the model states a rule (a field description, a cell note, a scenario's context note), it is part of the logic.

## Step 3 — Derive the real use cases, per slice

Work out what the slice's business logic actually has to do, from its command, read model or automation, its fields, its flows and the events involved:

- the happy path and its meaningful variants;
- each business rule, and **every way the business would reject or change the outcome**: a field with a rule behind it, a state precondition, a limit, a duplicate business action, an authorisation the model shows. The scenario-type questions in `eventmodeling-elaborating-scenarios` (happy path, validation failure, state violation, duplicate action, alternative path, external failure, compensation) are the checklist;
- for a command, every distinct event outcome it can produce;
- for a read model, every distinct projection behaviour: what appears, what changes, what disappears. A list read model also has its empty list;
- for an automation, every distinct decision its command takes for an item on its todo list.

**Only meaningful cases count, both for what exists and for what is missing.** A case that is technically possible but means nothing to the business is not a use case: don't invent it, don't list it as missing. An existing scenario that only restates another one, tests machinery (a missing or malformed field with no rule behind it, colliding ids, a retried request, an infrastructure failure) or can never occur in the process adds nothing to the coverage. Ask of each candidate: *would a domain expert care that this case behaves as specified?* If not, drop it.

When the model doesn't tell you whether a case exists (a limit nobody has decided, a rule only hinted at), that is a question for the person, not a missing scenario: raise it with `create_pin` and leave the case out of both counts. A decided failure is a use case; an undecided one is a question (`eventmodeling-core-rules` → "Open Questions vs. Decided Failures").

## Step 4 — Match scenarios to use cases

- A Given/When/Then scenario covers the use case whose given state, trigger and outcome it specifies, with the values that decide it. A scenario that only names the elements, or an outcome with no values, does not.
- A **storyline** covers every transition its beats walk. A beat specifies what appears, what changes or what disappears, so a todo list's storyline covers the item arriving and the item leaving in one scenario. A Given/When/Then that only restates one step of a storyline covers nothing new: count the transition once.
- A rejection scenario (a `then` that is an expected failure) covers the rejection case it names.
- A read model's use cases are covered by view scenarios: Given/When/Then or storyline. A read model with none rates 0, however thoroughly the command beside it is specified.

## Step 5 — Calculate

`percentage = round(100 × covered meaningful use cases ÷ all meaningful use cases)`. A slice with no business logic to speak of (a pure pass-through) and one scenario is 100. A slice with no use cases you can derive gets no value: say so instead of writing one. Don't round up to look good.

For every uncovered meaningful use case add an entry to `missing`: a short `title` and, where the model lets you say it, `given`, `when` and `then`, one short plain sentence each, in the model's own vocabulary, with concrete values only where the model has them. Leave a part out rather than guess it. At most 10 entries, the most important first; if more are missing, the percentage still counts all of them. For a read model the title may say the missing scenario is a storyline.

## Step 6 — Store it

One call per slice:

```
update_slice {
  sheet: "Bike Rental", slice: "Rent a Bike",
  coverage: {
    percentage: 78,
    missing: [
      { title: "Reject a rental of an already rented bike", given: "bike B-1 is rented", when: "a customer rents B-1", then: "the rental is rejected" }
    ]
  }
}
```

Writing coverage again replaces the old value. `coverage: null` clears it: use that for a slice that now has no derivable use cases but still carries an old value. Read the slice back (`get_slice_view`) once for a sample to check the value landed and is not stale.

## Step 7 — Report

Put it in your reply to the person, a table with one row per slice: Slice, Coverage, Missing (the titles). Beneath it:

- slices rated, the average, and the lowest three;
- slices skipped, with the reason (copy-only, empty, still fresh);
- scenarios you did not count because `check_model` flagged them, and what is wrong with each;
- questions you raised as pins.

Offer nothing beyond that: the missing cases are listed, and writing them is `eventmodeling-elaborating-scenarios`.

## Quality Checklist

- [ ] `check_model` ran first and the flagged scenarios were not counted as covering
- [ ] Every rated slice was read in full (`get_slice_view`), not judged from names
- [ ] Every counted and every missing case passes "would a domain expert care?"
- [ ] A read model with only a command-side specification beside it was not rated on the command's scenarios
- [ ] A storyline's transitions were counted once, not again for a Given/When/Then that restates them
- [ ] Each `missing` entry says what to specify, in the model's words, and nothing is invented
- [ ] Undecided rules went to a pin, not into `missing`
- [ ] Skipped slices are named with the reason
- [ ] Nothing but coverage was written
