---
name: quiz
description: Quiz me on course material with mostly open questions and update progress files; also answers "что повторить?" with a plan for the time I have. Use when I ask to quiz or check myself, or what to review.
argument-hint: "[subject | class | time, e.g. algo 10 минут]"
---

## Choosing topics

Scope comes from my message; by default all subjects outside `_archive/`. Read their mastery files, open mistakes and exam dates from course files. Priority, highest first: open mistakes made with confidence 3; level 0-1; checked long ago; exam close. Mix topics from different classes unless I limited the scope to one class.

No debt: never count or mention overdue reviews, missed days or streaks. After a long break pick a few top topics, not the whole backlog.

"Что повторить" or a time budget without a request to start: give a plan that fits the time (about 8 minutes per question, discussion included) listing topics and why each, then offer to start. "15 минут" means 2 questions.

## Session

- 5-7 questions by default; with a time budget, as many as fit it. One at a time. Read the note section before asking and don't show it.
- When listing topics (a plan or the session opening), name the area, not the term a question tests.
- Mostly open questions: explain, compare, apply, prove, solve a small example. At most two short warm-up questions per session.
- Each question asks one thing: no "name it and fix it", no "which one and why not the others"; every extra sub-ask costs as a separate question. Prefer a task answered in a few sentences or lines of code over an open redesign ("how would you rewrite this?"), which grows into a design discussion. Don't reuse the lecture's own example as the question: change the setting so that recall alone doesn't answer it.
- Ask me to start each answer with confidence 1-3 (1 guess, 3 sure); I may skip it.
- Grade against the note: верно, частично or неверно; what's missing or wrong; a brief correct version with a link to the note section. If I was wrong with confidence 3, say so plainly and make the correction stand out: confident errors are the ones most worth fixing.
- On "не знаю" give one hint; give the answer only if I'm still stuck.

## Finish

Formats: `.agents/progress-formats.md`. Update mastery: each topic asked gets the level of its worst answer this session (3 верно, 2 частично, 1 неверно) and today's date. Log неверно and частично answers to mistakes, and close open mistakes I answered correctly now. End with one paragraph: what's solid and what to look at next.
