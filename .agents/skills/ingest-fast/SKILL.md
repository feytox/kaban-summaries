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
6. Write the note `notes/<slug>-L03-<тема>.md` with frontmatter `ingest: fast` by `note.md` in this folder; read it before the first note.
7. Update the `Фокус` line of the course file when the class shows the course teaches more or something else than it says: the focus is unknown at the start and grows over the semester. Keep it one sentence.
8. Add the class's topics to mastery (format: `.agents/progress-formats.md`): 2-5 per class, each narrow enough for one quiz question set, level 0, checked `-`.
9. Final report in chat: one line per class (file, number of `[?]`, word count and number of ideas), then guesses you made about subject, number or date, and a changed course focus.

## Course file template

Sections in this order; leave out a section until there is something to put in it. Teachers are a table; emphases are bullets grouped under `###` per teacher when there are several. `Фокус` is one sentence on what the course teaches to understand.

```markdown
---
exam:
---
# <Название курса>

Фокус: <чему учит курс>

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
