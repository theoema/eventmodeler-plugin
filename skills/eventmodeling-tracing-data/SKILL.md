---
name: eventmodeling-tracing-data
description: "Follow one piece of information through the model, such as \"where does the customer email come from, and where is it used?\", and show it as a table: one stop per element it passes through, in flow order, each with its field name and slice, plus the gaps the trace exposes. Use when asked where a value originates, where it is stored, renamed, shown or used, or what a field change would touch. Read-only. Do not use for: adding or renaming a field along a chain (use `eventmodeling-adding-attributes`), the rule for where a field's value must come from (use `eventmodeling-information-completeness`) or auditing every field of the model (use `eventmodeling-checking-completeness`)."
---

# Tracing Data

This step applies the shared element rules in **`eventmodeling-core-rules`** — read it once per session if you haven't already. Tool names and conventions are in `eventmodeling-core-rules` → `../eventmodeling-core-rules/references/tools.md`. Where a field's value must come from is the rule in `eventmodeling-information-completeness`; this skill shows what the model actually does.

You are in **Critic Mode** and **read-only**: follow the information, report what the model says and what it does not. Nothing on the model changes. If the person then asks for a change (add the field to a read model, mark it as personal data), that is ordinary modeling work, with `eventmodeling-adding-attributes` for fields along a chain.

Typical questions: "Where is the customer email used?", "Where does the delivery address come from?", "Which events store the IBAN?", "What happens to the discount code?"

## Step 1 — The subject

The **subject** is one piece of information, in the person's words ("customer email", "order total"). A field the person points at or names is the subject.

If there is no subject, or several equally likely ones ("the customer data"), ask first and stop: list the candidate field names you found (at most about eight) and let the person choose. One piece of information per trace: "name and email" is two traces. Do the first and offer the second.

## Step 2 — Find where the model holds it

Read in tiers (`eventmodeling-core-rules` → `../eventmodeling-core-rules/references/tools.md`, "Reading"). `get_model_view` lists the sheets and slices; `get_slice_view` shows each element with its fields, and the flows with their mappings. There is no search by field name, so read the slices that plausibly touch the subject (by slice and element names), including the obvious variants (`email`, `mail`, the model's own language).

Collect the elements and fields that look like the subject: the same name, a prefixed or renamed one (`customerEmail`, `newEmail`, `recipient`). A field that only sounds related (`emailVerified`, `emailCount`) is a different piece of information: leave it out, or mention it in the note of the stop it sits on.

Then let the model do the following, not guessing from names. Take the element where the subject is most plainly held and call:

```
trace_field { sheet: "Registration", element: "B3.1", field: "email" }
```

It returns `upstream` (where the value comes from, step by step back to where it starts) and `downstream` (every element it flows into), across flows and sheets. Each step carries `sheet`, `address`, `element`, `type`, `field` (a dotted path for a subfield, such as `address.street`), `via` ("mapping" or "same-name") and `depth`. `origins` lists where the value starts: a field marked `isUserInput` or `isGenerated`, or an element with nothing flowing into it, which is a gap when the field is not marked.

The first trace may start mid-chain and so miss the branches off the upstream steps. Trace again from each origin to get everything downstream. Then check the fields you collected that the trace did not reach: each is either a different piece of information, or the same information the model never connected (see Step 4).

Note `isPii` on the fields: if the subject is personal data, say so in the summary.

## Step 3 — Order and classify the stops

Order the stops the way the information flows, from where it enters to where it ends (screen → command → event → read model → screen; event → read model → automation → command → event). Branches follow their source: after the event that stores it, the read models fed by it, then the automations. `depth` gives the order within a chain.

| Role | The element … |
|---|---|
| origin | is where the information first enters the model: a screen the person types it into (`isUserInput`), an external event, any element where it is `isGenerated`. Usually one; more when it can enter in several ways (registration and "change email") |
| stored | is an event that records it: the source of truth. Every event carrying the field is a stop |
| transformed | changes it: a renamed field via a mapping (`email` to `recipient`), a derived value, a normalisation the element's notes or scenarios state ("lower-cased"). Say what in the note |
| used | reads it: a command that carries or checks it, a read model that projects it, a screen that shows it, a scenario that decides on it ("email must be unique"). The note says how |
| sent | hands it outside the model: an automation that passes it to an external service (mail provider, payment, CRM), as the model's names and notes show |

A command that takes the field from its screen is "used" (it carries it), unless the screen isn't modeled: then the command is the origin. Each element appears once: pick the role that says most about it. A read model's scenarios and a command's scenarios (`get_slice_view`) show where a rule decides on the value.

Per stop give: the element name as on the sheet, its type, the field name there, the sheet and slice, and a few business words in the note. No addresses.

**Faithfulness.** Only what the model says. Never invent a stop to make the journey look whole.

## Step 4 — Say what the trace exposes

A trace is worth most for its breaks. Put each in the summary, plainly:

- shown on a screen but no read model carries it, or no origin is marked (`untraceable-field`, `unsourced-fields`);
- stored by an event but nothing ever reads it;
- the same information held under two names with no mapping and no flow between them, so it enters twice;
- an origin that is not marked (`isUserInput` or `isGenerated`) where the trace simply ends;
- a value that is personal data in one place and unmarked in the next (`pii-mismatch`): run `check_model` for the sheets the trace crosses (with `severity: "info"`, which `unsourced-fields` needs) and quote any `untraceable-field`, `unsourced-fields` or `pii-mismatch` finding that falls on a stop.

Keep these as findings in your reply. Record them on the model only if the person asks: a note on the slice or cell (`set_note`, which replaces the text, so read it first), or a question for the human as a pin (`create_pin`).

## Step 5 — Reply

Put it in your reply to the person:

1. One line: the subject and where it originates ("The customer email enters at registration and ends up in five more places").
2. A summary of a few sentences: the origin (personal data, if so), where it is stored, who uses it, and every finding from Step 4.
3. The table, one row per stop in flow order: Role, Element (type), Field, Slice, Note.

```
The customer email enters once, at registration (personal data), is kept in two events and used for login and mails. Nothing shows the changed email after Email Changed.

| Role   | Element (type)                | Field         | Slice             | Note                      |
|--------|-------------------------------|---------------|-------------------|---------------------------|
| origin | Registration (screen)         | email         | Register Customer | typed in by the customer  |
| used   | Register Customer (command)   | email         | Register Customer | must be unique            |
| stored | Customer Registered (event)   | customerEmail | Register Customer | renamed through a mapping |
| stored | Email Changed (event)         | newEmail      | Change Email      | no mapping to email       |
| used   | Customer Profile (read model) | email         | Customer Profile  | shown on the profile page |
| sent   | Send Welcome Mail (automation)| recipient     | Welcome Mail      | to the mail provider      |
```

Keep every stop in the one table, however long the journey.

## Quality Checklist

- [ ] The subject is one piece of information; a vague or double request was asked about, not guessed
- [ ] The trace came from `trace_field`, run again from each origin, not from matching names by eye
- [ ] Look-alike fields that are a different piece of information are left out or marked as such
- [ ] Every stop is something the model contains; none invented
- [ ] Each element appears once, with the role that says most about it
- [ ] Every break (no origin, never read, entering twice, unsourced, personal-data mismatch) is in the summary
- [ ] Nothing on the model was changed
