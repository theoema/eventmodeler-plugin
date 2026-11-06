---
name: eventmodeling-workshop-questions
description: "The questions an event modeling facilitator asks about a screen, a button or a field (what happens, what data is stored, what comes next, which screen follows, which business rules apply, a real example) and how to put them to the people who know the business: in what order, how many, in what words, and what each answer becomes in the model. Use when walking people through a model or a set of screens to draw out what is missing, when preparing questions for a domain expert, or when turning their answers into model changes. Do not use for: deciding whether to ask before a modeling step at all, or recording that interview (use eventmodeling-interview-protocol), or writing the scenarios an answer leads to (use eventmodeling-elaborating-scenarios)."
---

# Workshop Questions

What a facilitator asks in an event modeling workshop when pointing at a screen: **"what happens here?"** Asked of the people who know the business, one element at a time, in the order a user would meet them. The answers are what turn a screen into an event model.

This step applies the rules of **`eventmodeling-core-rules`** (read it once per session if you haven't); nothing is restated here. Tool names and conventions are in `eventmodeling-core-rules` → `../eventmodeling-core-rules/references/tools.md`.

It is not a step of its own. Use it wherever the people who know the business are asked about the model: live in a conversation, or as open questions left on the model for them to answer later.

---

## The Catalog

Ask only what the model does **not** already answer. A question about something the model states (a field, an event, a scenario) is noise. Before asking, read what is there:

- `get_slice_view` for the slice: its fields, flows, scenarios and notes;
- `check_model` with `severity: "info"` (its default lists warnings and errors only) for the holes it already knows: `unsourced-fields` and `untraceable-field` (a field with no known origin), `slice-without-scenario` and `no-error-scenario` (nothing says when a command is refused), `todo-list-without-storyline`;
- `list_pins` with `status: "waiting"`: a question already open on that area is not asked twice.

Each question has a category, which decides what it asks about:

| Category | Asked about | Examples |
|---|---|---|
| action | a button or link | What happens when someone clicks this? What do they need to have filled in first? |
| data | a field or a form | What is stored when this is submitted? Is this entered or derived? Which of it is needed later, and where? |
| next | the result of an action | What happens after this: which screen is shown, what is the person told? What else starts because of it (a mail, a hand-over to someone)? |
| rule | a command that could refuse | What business rules apply here? When must this be refused, and what does the person see then? |
| example | any rule or number | Give a real example: a situation where this goes through, and one where it doesn't. (An answer becomes a Given/When/Then.) |
| view | a read model, a list, a total | Where does this come from? What does the list show when it is empty? Who sees which rows? |
| who | an action or a screen | Who does this, and who must not? |
| time | anything with a date or a wait | When does this happen, what if it is late, what if it is repeated? |

**Wording.** One question, in plain business language, in the screen's own words ("What happens when someone saves a note?"), never a model term (command, event, read model, slice). Pick the **one question per element that matters most**. A screen with ten questions is not guided, it is a form.

---

## Putting Them in Order: the Guided Tour

Walk the questions **along the timeline of the event model, left to right, as strictly as it can be**: the story the model tells is the story the person is guided through. Take the sheets top to bottom (the order the story is told in), and on each sheet the slices left to right. Inside a slice, take its elements in the order they sit on the timeline: the screen the person starts on, the command they fire, the event that follows, the read model and screen that show the result, then the next slice's screen. The next question's screen is the screen the timeline leads to next, so **the next question never jumps back**; a question about an earlier part of the story is asked when the tour passes it, not later.

Depart from the timeline only when it is unavoidable: a rule question that needs the situation a later slice creates (give it a set-up, below, instead of moving it), or two lanes whose slices share a column (the lane whose action comes first goes first). Within one screen, top to bottom, actions before the consequences they have. Each answer should make the next question easier; at each point on the timeline ask the happy-path questions (action, data, next) before rule, example and time for the elements the path showed to be risky.

### Keep the tour moving forward: always ask what comes next

A tour that circles the same few screens teaches nothing new. Drive it **forward**:

- **At most two questions per screen**, the ones that matter most. A third question about the same screen waits until the tour has been through the rest.
- **Every action gets a "next" question**, asked right after its action or data question: "What happens after this: what does the person see, what else starts, who is told?" The answer is the next screen, event or automation on the timeline, so the tour has somewhere to go.
- **Ask where the story ends.** On the last screen of a chain, and on the last slice of the sheet, ask what happens after it: "What does the person do next? What happens a day later? Who takes over from here?" Ask it **again for every screen a "next" answer named that the model doesn't have yet**: a screen that doesn't exist has nothing to hang a question on, so the question goes on the element that leads there, the last screen's button or result. That open end is how the model grows; the tour never just stops at the last modeled screen.
- **A later round moves on.** Questions already answered are not asked again, so the next round starts at the first screen that still has an open question, and the frontier of the tour keeps advancing along the timeline instead of staying where the first answers were given.
- Rule, example and time questions are the second pass: only after the happy path has been walked to its end, and only for the elements it showed to be risky.

---

## A Question Can Set the Scene

A question about a rule is best asked **in the situation it is about**: the order already shipped, the email already taken. That situation is a scenario's Given.

- **From the model:** if a scenario already puts the model in that situation, name it in the question by its title ("Take the case 'Reject a reservation for a date in the past': ...").
- **New:** if the model has no scenario for it yet, say the situation in a sentence with only the values the question turns on ("Mia registered on 12 October and confirms on 14 October: is that too late?"). Leave every other value out; the relevance is clear only when the decisive values are the ones stated.

The "When" is not part of the set-up: it is what the person does next.

---

## Putting Them to People

**In a conversation:** ask the questions of the current screen (at most two) in the order above, name the screen, and wait for the answer before moving on. Do not paste the whole tour; the next question depends on the last answer.

**For people who answer later:** leave each question as a pin (`create_pin`), one question per pin, worded as above and anchored on the cells of the screen, slice or lane it is about. A pin waits on its area until someone replies, and resolving it means answering it, never deleting it. A decided case is not a pin (`eventmodeling-core-rules`, Open Questions vs. Decided Failures): once an answer settles a rejection, it becomes a scenario, not a lingering question.

Questions are for **asking**. Do not fill a gap with a guess and call it answered; where something is undecided and nobody can answer yet, say so in the reply, or note it (`set_note`) so it is seen.

---

## What the Answers Become

An answer is a modeling decision; apply it with the step that owns it, on the **original** element (never on a linked copy).

| Answer to | Becomes | See |
|---|---|---|
| rule, example | Given/When/Then scenarios, with the values the answer gave; a refusal is a scenario with an expected failure | `eventmodeling-elaborating-scenarios` |
| data | fields on the element, each with a source or marked where the information starts (`update_element`) | `eventmodeling-information-completeness` |
| next | the following screen, command, event or automation on the timeline | `eventmodeling-identifying-inputs`, `eventmodeling-designing-automation-chains` |
| view | a read model, with its source events and the screen that shows it | `eventmodeling-identifying-outputs` |
| who | the role on the actor lane, and which lane issues the command | `eventmodeling-identifying-inputs` |
| time | a field for the date, a scenario for the late or repeated case, an automation where the system must act at a time | `eventmodeling-designing-automation-chains` |

An answer that **contradicts** the model is a question back (`reply_pin`, or in the conversation), not a silent change. Read the model's side of it first (`get_slice_view`, `get_history`), put the two next to each other and ask which holds. Run `check_model` after applying a batch of answers.

---

## Quality Checklist

- [ ] Nothing was asked that the model, its scenarios or a waiting pin already answers
- [ ] The questions follow the timeline left to right and never jump back
- [ ] No more than two questions per screen per round, and every action has its "next" question
- [ ] The tour asks where the story ends, and again for every screen an answer named that doesn't exist yet
- [ ] Each question is one sentence in the screen's own words, with no model terms
- [ ] A rule question names its situation (an existing scenario's title, or the decisive values) rather than leaving it to the reader
- [ ] Every answer was applied with the step that owns it, to the original element, and a contradiction was asked about, not overwritten
