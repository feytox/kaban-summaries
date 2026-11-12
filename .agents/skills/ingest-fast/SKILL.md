---
name: ingest-fast
description: Quickly turn class materials (files in _inbox/, paths, pasted text, links) into compact notes, many classes in one run, without questions. Use for the backlog of accumulated lectures or when I ask for a fast ingest.
argument-hint: "[subject, class type and numbers, files]"
---

Work without asking me anything: record doubts as `[?]` in the note and in the final report. The one exception is material that fits no existing subject: ask for its name, slug and semester.

1. Inputs: everything in `_inbox/`, or only what my message names, plus pasted text and links from it.
2. Match each input to a subject (from my message, else by content against the `<slug>-course.md` files) and to a class: type and number from my message or the material, else the next free number. Materials of one class go into one note.
3. New subject: create `subjects/s<N>-<slug>/` with `<slug>-course.md` from the template below, filling only what I told you, and in `_private/subjects/s<N>-<slug>/` a mastery file with just the table header and an empty mistakes file.
4. Put each input into the subject's `raw/` as `<slug>-L03-<kind>.<ext>` (kind: `transcript`, `slides`, `handout`, `text`, etc.):
	- move files from `_inbox/`, copy files from elsewhere;
	- pasted text, transcripts and other plain text get `.md`, so citations to them open in Obsidian;
	- URLs go into the note's `links`; don't try to fetch videos;
	- a Толк recording (`https://<host>.ktalk.ru/recordings/<id>`) without a transcript among the inputs: download it with `curl -sS 'https://<host>.ktalk.ru/api/recordings/<id>/transcript/file?type=Text'` (no login needed) into `raw/<slug>-L03-transcript.md`.
5. Process classes one at a time: read the class's sources in full (PDFs 20 pages per Read), write its note, then move to the next.
6. Write the note `notes/<slug>-L03-<тема>.md` with frontmatter `ingest: fast` by `note.md` in this folder; read it before the first note.
7. A class that had a note before, even one I deleted to redo: keep its file name unless the topic changed, then fix links to it and its headings that no longer resolve: `grep -rn '<old-name>' subjects _private`.
	- A note I deleted to redo is written from the sources only: don't read the old version, git history included.
8. Update the `Фокус` line of the course file when the class shows the course teaches more or something else than it says: the focus is unknown at the start and grows over the semester. Keep it one sentence.
9. Add the class's topics to mastery (format: `.agents/progress-formats.md`): 2-5 per class, each narrow enough for one quiz question set, level 0, checked `-`. A class that already has rows: rewrite them to fit the new note, keeping levels and dates.
10. Remove the processed entries from `_inbox/inbox.md` with Edit right after re-reading it: I add entries during the session.
11. Final report in chat: one line per class (file, number of `[?]`, word count and number of ideas), then guesses you made about subject, number or date, and a changed course focus.

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
