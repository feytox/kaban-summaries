---
name: retell
description: Check my from-memory retelling of a note or an exam ticket against the notes, explain only the gaps and save the result; for tickets also ask examiner follow-ups. Use when I want to retell a class or topic, or rehearse an exam ticket.
argument-hint: "[note or ticket, e.g. algo L03 | algo T07]"
---

1. Find the target: a note, or for a ticket its wording in the course file and the notes covering it. Don't show their content. If my retelling isn't in the message, ask for it and wait.
2. Compare my text with the notes: one line on what's right, then what's missing (most important first) and what's wrong. Don't retell the note back to me.
3. Explain only the gaps, briefly, with links to note sections.
4. For a note: append to `## Мой пересказ` at the end of the note (create it if absent) the date, my text verbatim and the list of gaps. Update mastery for the topics of that note.
5. For a ticket, continue:
   - ask me to retell the gaps again and check that
   - ask 2-4 examiner follow-ups one at a time: definitions the answer relies on, proof steps, an example, links to other tickets; lean on the teacher's emphases from the course file
   - create or update `tickets/<slug>-T07.md` from the template below
   - update mastery for its topics; log wrong answers to mistakes

## Ticket card template

```markdown
---
rehearsed: <date>
---
# Билет 7. <формулировка из файла курса>

## План ответа

<outline from the notes, each point with a wikilink to its section>

## Слабые места

- <date>: <what was missing or wrong>
```
