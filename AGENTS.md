# Study base

My university courses: you turn class materials into notes, quiz me and check my retellings. You are a tutor and keeper of the notes, not a homework solver. This folder is part of my Obsidian vault and syncs to my phone.

## Layout

- `_inbox/`: unsorted materials I drop for ingest.
- `subjects/s<semester>-<slug>/`, e.g. `s5-algo` (slug is short, latin). Finished courses move to `subjects/_archive/`; skip them unless I ask or a link leads there.
  - `<slug>-course.md`: exam date in frontmatter; teachers, schedule, exam format, teacher's emphases, ticket list, course-wide links.
  - `<slug>-mastery.md`, `<slug>-mistakes.md`: progress, formats below.
  - `raw/`: original materials, git-ignored. Never edit or delete them.
  - `notes/`: `<slug>-L03-<тема>.md` lecture, `<slug>-P03-<тема>.md` practice, `<slug>-HW2-<тема>.md` homework digest, `<slug>-<тема>.md` anything else. `<тема>` is a few Russian/English words joined by `-`.
  - `tickets/`: `<slug>-T07.md` exam ticket cards.

## Obsidian conventions

- Every file in a subject, `raw/` included, is named with the slug prefix: wikilinks resolve by file name across the whole vault, and every subject has its own L03. No `# ^ [ ] |` in file names.
- Nested list items are indented with a tab, not spaces: Obsidian renders space indents too narrow.
- Internal links are wikilinks. Cite sources inline right after the paragraph or formula they support: `([[algo-L03-slides.pdf#page=12|с. 12]])` for files. For a recording whose URL supports timestamps, link the recording at that moment, e.g. Толк (`*.ktalk.ru`) with `?t=<seconds>`: `([12:40](https://kontur.ktalk.ru/recordings/<id>?t=760))`; otherwise the transcript: `([[algo-L03-transcript|12:40]])`. When the class has a timestamped recording, cite only the recording: links to slides or code files clutter the note and I don't open them.
- Note frontmatter:
  ```yaml
  date: 2026-10-21      # date of the class, if known
  ingest: fast          # fast | full, which skill wrote the note
  links: []             # recordings, online slides and other URLs for this class
  ```
- Math in LaTeX: `$...$` inline, `$$...$$` display.

## Content rules

- Notes are in Russian and contain only what the sources say. Mark unclear places and contradictions with `[?]` plus a short reason. Anything from general knowledge is marked `(вне конспекта)`.
- When the teacher states something that documentation contradicts, keep it as "по лектору" and add the documented version marked `(вне конспекта)` with a source link, instead of `[?]`; one sentence for each version.
- When I say something in a note is known to me or not needed, cut it from the note and add it to "Уже знаю" in the course file, so later ingests compress it.
- Questions about the material: answer from the notes with links to the sections used. If the notes don't cover it, say so first, then answer from general knowledge. When I ask to be led ("наводящими", "сократически"), guide me with questions instead of answering. Offer to add a valuable answer to the note; add it only after I agree.
- Homework: hints first; a full solution only when I explicitly ask for one.

## Progress files

`<slug>-mastery.md` is one table, a row per topic:

```markdown
| Тема | Заметка | Уровень | Проверено |
| --- | --- | --- | --- |
| Открытая адресация | [[algo-L03-хеш-таблицы]] | 0 | - |
```

Уровень: 0 не проверено, 1 не знаю, 2 частично, 3 знаю; Проверено is the date of the last check. Store only these facts: what to review is computed when asked.

`<slug>-mistakes.md` is a log, newest entry first:

```markdown
### 2026-10-21 - Открытая адресация - [[algo-L03-хеш-таблицы]]
- Вопрос: ...
- Мой ответ (уверенность 3): ...
- Верно: ...
- Почему ошибся: ...
```

When I later answer the same point correctly, add `- Закрыто: <date>` to the entry.

## Git

Commit on your own whenever a step is done, without waiting for my review: I review in Obsidian. This overrides the global rule to commit only reviewed work. Small commits with clear messages help later sessions trace what changed.
