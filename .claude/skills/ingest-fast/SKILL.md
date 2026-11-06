---
name: ingest-fast
description: Quickly turn class materials (files in _inbox/, paths, pasted text, links) into compact notes, many classes in one run, without questions. Use for the backlog of accumulated lectures or when I ask for a fast ingest.
argument-hint: "[subject, class type and numbers, files]"
---

Work without asking me anything: record doubts as `[?]` in the note and in the final report. The one exception is material that fits no existing subject: ask for its name, slug and semester.

1. Inputs: everything in `_inbox/`, or only what $ARGUMENTS names, plus pasted text and links from my message.
2. Match each input to a subject (from my message, else by content against the `<slug>-course.md` files) and to a class: type and number from my message or the material, else the next free number. Materials of one class go into one note.
3. New subject: create `subjects/s<N>-<slug>/` with `<slug>-course.md` from the template below, filling only what I told you, and in `_private/subjects/s<N>-<slug>/` a mastery file with just the table header and an empty mistakes file.
4. Move inputs into the subject's `raw/` as `<slug>-L03-<kind>.<ext>` (kind: `transcript`, `slides`, `handout`, `text`...). Copy instead of moving files from outside `_inbox/`. Save pasted text as a `.md` file in `raw/`. Transcripts and other plain-text files also get `.md`, so citations to them open in Obsidian. URLs go into the note's `links`; don't try to fetch videos. A Толк recording (`https://<host>.ktalk.ru/recordings/<id>`) without a transcript among the inputs: download it with `curl -sS 'https://<host>.ktalk.ru/api/recordings/<id>/transcript/file?type=Text'` (no login needed) into `raw/<slug>-L03-transcript.md`.
5. Process classes one at a time: read the class's sources in full (PDFs 20 pages per Read), write its note, then move to the next.
6. Note `notes/<slug>-L03-<тема>.md`, frontmatter `ingest: fast`. It is a skeleton for revision that I read instead of the slides and the recording, not a retelling: conclusions without the reasoning that led to them; any class fits in 800 words of text, code blocks not counted: a longer class means cutting harder, not a longer note. Machine transcripts mishear terms: fix obvious ones silently ("кубер нейтс" -> Kubernetes, "Газул" -> Guzzle); use `[?]` only when the meaning itself is in doubt. Write for skimming on a phone; pick the form by content:
   - a config, manifest, command, query, log line or API call: a code block with short comments instead of a description in words; my own example is fine when the class had none, with one line per note saying some code is mine; no code that only decorates a topic without code
   - things compared by the same attributes, or many items of one kind (12 factors, Service types): a table, split by theme with subheadings when long; remarks go below as short bullets keyed to the row; an item that gets a row gets no separate definition line or bullet repeating it
   - a term worth recalling: its own line `**Термин** - одна фраза`; a problem, a cause or a position is not a term
   - cause and effect: 1-2 sentences
   - everything else: bullets, one fact each, about 100 characters; a point with parts is a short parent bullet with nested ones; bold the key word when it helps scanning

   What to keep and how to write it:
   - keep, most important first: definitions and distinctions between close terms, in the teacher's words; the teacher's own positions, especially where they go against common practice; practical rules for real work (e.g. "migrations run as a Job before the new version starts"); an example or number only when the idea rests on it, in a few words ("changes cause 70%+ of failures")
   - a list from a slide or a standard source (e.g. the 12 factors) keeps every item, as table rows or bullets; from the talk add only what the slide lacks
   - chat Q&A: only an answer that adds a definition or a rule
   - a linked article: only the part the teacher pointed to
   - wikilinks wherever a concept from another note is used, in this subject or another one, `_archive/` included
   - general basics: drop; a bare term in one line if the rest builds on it
   - citations: one per section, where it starts; with timestamped recording links, also one per definition and per paragraph that draws on another moment of the class, never per bullet
   - Russian that reads like a good cheat sheet, neither essay nor telegraph: cut filler, lead-ins ("Ниже...", "X различаются тем, что...") and the obvious, never the subject or verb a sentence needs; check cases and agreement; the line after a heading never continues the heading's phrase; in text, an arrow or a lone bold word never replaces a verb or a clause ("стажёр сломал прод-БД -> выясняют", a bullet that is only a bold phrase), arrows are fine in code comments and tables; no "является", "осуществляется", "обеспечивает", "в рамках"
   - teacher's stories: drop; a story with a real-world lesson becomes that lesson in one line; hints on what matters or will be on the exam go to "Акценты преподавателей" in the course file
   - demos, live coding and exercises: what was done and the key commands, API or code; prefer a code block plus one line over a paragraph that describes code; reconstruct code the teacher read aloud, noting once per note that code is reconstructed; no step-by-step retelling, no task solutions; a demo that only illustrates a conclusion already in the note gets one line, no code
   - proofs and derivations: a one-line idea
   - homework given in a class: skip, just `ДЗ` in the Пропущено callout; I track homework separately
   - sections follow the meaning blocks of the class, not slide boundaries or the order of talk
   - end the note with a collapsed callout listing what was cut, e.g.
     ```markdown
     > [!note]- Пропущено
     > байка про инцидент X, демо установки Y, ДЗ.
     ```
   - an existing note: merge into its sections instead of appending a duplicate
   - after writing the note, revise it in place: count the words of its text without frontmatter, code blocks and citations with a quick script; while over 800, cut the lines I'd miss least when revising for the exam; then go line by line and fix every line that lacks a subject or verb it needs, uses an arrow or a lone bold word instead of a verb, or breaks case or agreement
7. Add the class's topics to mastery (format: `.claude/progress-formats.md`): 2-5 per class, each narrow enough for one quiz question set, level 0, checked `-`.
8. Final report in chat: one line per class (file, number of `[?]`, word count if over 800 and why), then guesses you made about subject, number or date.

## Course file template

Sections in this order; leave out a section until there is something to put in it. Teachers are a table; emphases are bullets grouped under `###` per teacher when there are several.

```markdown
---
exam:
---
# <Название курса>

## Преподаватели

| Кто | Что ведёт | Где работает |
| --- | --------- | ------------ |

## Расписание
## Оценка
## План курса
## Акценты преподавателей
## Ссылки

## Билеты

1.
```
