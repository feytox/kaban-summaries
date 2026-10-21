---
name: ingest-fast
description: Quickly turn class materials (files in _inbox/, paths, pasted text, links) into compact notes, many classes in one run, without questions. Use for the backlog of accumulated lectures or when I ask for a fast ingest.
argument-hint: "[subject, class type and numbers, files]"
---

Work without asking me anything: record doubts as `[?]` in the note and in the final report. The one exception is material that fits no existing subject: ask for its name, slug and semester.

1. Inputs: everything in `_inbox/`, or only what $ARGUMENTS names, plus pasted text and links from my message.
2. Match each input to a subject (from my message, else by content against the `<slug>-course.md` files) and to a class: type and number from my message or the material, else the next free number. Materials of one class go into one note.
3. New subject: create `subjects/s<N>-<slug>/` with `<slug>-course.md` from the template below, filling only what I told you, a mastery file with just the table header and an empty mistakes file.
4. Move inputs into the subject's `raw/` as `<slug>-L03-<kind>.<ext>` (kind: `transcript`, `slides`, `handout`, `text`...). Copy instead of moving files from outside `_inbox/`. Save pasted text as a `.md` file in `raw/`. URLs go into the note's `links`; don't try to fetch videos.
5. Process classes one at a time: read the class's sources in full (PDFs 20 pages per Read), write its note, then move to the next.
6. Note `notes/<slug>-L03-<тема>.md`, frontmatter `ingest: fast`. It keeps what the exam or real work needs and I don't know yet:
   - concepts, definitions, principles and trade-offs, in the lecturer's terms
   - items under "Уже знаю" in the course file, and basics the rest builds on: one line with a citation
   - teacher's stories: drop; a story with a real-world lesson becomes that lesson in one line; hints on what matters or will be on the exam go to "Акценты преподавателя" in the course file
   - demos, live coding and exercises: what was done and the key commands, API or short code fragment that shows the idea, with a citation; no step-by-step retelling, no task solutions
   - proofs and derivations: a one-line idea with a citation
   - homework: the concepts and methods the tasks use, not task solutions
   - sections follow the meaning blocks of the class, not slide boundaries or the order of talk
   - last line `Пропущено: ...` listing what was cut, e.g. "байка про инцидент X, демо установки Y"
   - an existing note: merge into its sections instead of appending a duplicate; never touch `## Мой пересказ`
7. Add the class's topics to mastery: 2-5 per class, each narrow enough for one quiz question set, level 0, checked `-`.
8. Final report in chat: one line per class (file, number of `[?]`), then guesses you made about subject, number or date.

## Course file template

```markdown
---
exam:
---
# <Название курса>

- Семестр:
- Уже знаю:
- Преподаватели:
- Расписание:
- Формат экзамена:
- Акценты преподавателя:
- Ссылки:

## Билеты

1.
```
