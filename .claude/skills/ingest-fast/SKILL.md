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
6. Note `notes/<slug>-L03-<тема>.md`, frontmatter `ingest: fast`. It is a skeleton for revision that I read instead of the slides and the recording, not a retelling: conclusions without the reasoning that led to them; a 90-minute class usually fits in 400-800 words. Write for skimming on a phone. A defined term is its own paragraph starting with the term in bold. A connected explanation (why a rule exists, how A leads to B) is a paragraph of 1-3 short sentences. Bullets only for parallel items (kinds, properties, steps, anti-patterns), one fact each, about 100 characters, never several facts joined by `;`; a point with parts is a short parent bullet with nested ones. A comparison of two or three things is a table. Machine transcripts mishear terms: fix obvious ones silently ("кубер нейтс" -> Kubernetes, "Газул" -> Guzzle); use `[?]` only when the meaning itself is in doubt.
   - keep, most important first: definitions and distinctions between close terms, in the teacher's words; the teacher's own positions, especially where they go against common practice; practical rules for real work (e.g. "migrations run as a Job before the new version starts"); an example or number only when the idea rests on it, in a few words ("changes cause 70%+ of failures")
   - a list from a slide or a standard source (e.g. the 12 factors) becomes one line per item; an item under "Уже знаю" gets just its name; from the talk add only what the slide lacks
   - chat Q&A: only an answer that adds a definition or a rule
   - items under "Уже знаю" in the course file and general basics: drop; a bare term in one line if the rest builds on it
   - one citation per section, pointing where the section starts
   - plain Russian: no "является", "осуществляется", "обеспечивает", "в рамках"
   - teacher's stories: drop; a story with a real-world lesson becomes that lesson in one line; hints on what matters or will be on the exam go to "Акценты преподавателя" in the course file
   - demos, live coding and exercises: what was done and the key commands, API or short code fragment that shows the idea; no step-by-step retelling, no task solutions
   - proofs and derivations: a one-line idea
   - homework: the concepts and methods the tasks use, not task solutions
   - sections follow the meaning blocks of the class, not slide boundaries or the order of talk
   - last line `Пропущено: ...` listing what was cut, e.g. "байка про инцидент X, демо установки Y"
   - an existing note: merge into its sections instead of appending a duplicate; never touch `## Мой пересказ`
   - before saving, reread the note as me revising for the exam and cut every line I wouldn't miss
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
