---
name: ingest
description: Turn materials of a class into a note the same way ingest-fast does, then ask me about the places that stayed unclear; also finishes a note made by ingest-fast. Use when I hand over a new class or ask to finish a fast note.
argument-hint: "[subject, class, files]"
---

Follow `.claude/skills/ingest-fast/SKILL.md` and write the note the same way, with `ingest: full`. The one difference: resolve doubts with me instead of only reporting them.

- After writing, ask me about the `[?]` places in one message. I usually answer from the recording or slides: write my answers in as lecture facts with a citation to that moment. Mark `(со слов)` only what I say I recall from memory; leave `[?]` where I don't know either.
- A fast note: re-read its `raw/` sources, fix the note against them, then ask about its `[?]` the same way. Keep `## Мой пересказ` and anything I wrote by hand.
- At the end offer `retell` for this class; don't push it.
