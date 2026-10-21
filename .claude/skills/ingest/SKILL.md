---
name: ingest
description: Turn materials of a class into a full note with proofs, examples and links to related notes, then clarify open points with me; also upgrades notes made by ingest-fast. Use when I hand over a new class or ask to expand a fast note.
argument-hint: "[subject, class, files]"
---

Follow `.claude/skills/ingest-fast/SKILL.md` with these changes.

- Upgrading a fast note: re-read its `raw/` sources and rewrite it to this level, keeping `## Мой пересказ` and anything I wrote by hand.
- The note gets `ingest: full` and contains:
  - proofs and derivations in full as given
  - demos and live coding as a skeleton of steps with the key commands and code; still no step-by-step retelling of the talk
  - wikilinks wherever a concept from another note is used, in this subject or another one, `_archive/` included
- After writing, ask me about the `[?]` places in one message: I may remember the class. Fix the note from my answers, marking them `(со слов)`; leave `[?]` where I don't know either.
- At the end offer `retell` for this class; don't push it.
