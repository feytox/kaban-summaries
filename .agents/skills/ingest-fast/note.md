# Class note

The note is what I read instead of the slides and the recording, and the base I later memorize from in quiz and retell. It explains how the subject works and why, in a few connected ideas; it is not a list of everything the teacher said.

## Selection

- The `Фокус` line of `<slug>-course.md` says what the course teaches. If it is missing, write one from the course plan and the material, mark it `[?]` and mention it in the report.
- Keep what explains the subject of the course: definitions and distinctions between close terms in the teacher's words, mechanisms, trade-offs and their reasons, the teacher's positions (especially against common practice), rules for real work with the reason behind them.
- Drop how to operate a particular tool on this class's setup: logins, click paths, editor plugins and settings, syntax trivia of a format, packages needed to make a demo run, etc. Name them in Пропущено.
- Commands, configs and APIs of the technology the class is about (SQL queries in a databases course, git commands in a version control class) are the subject itself, not tool trivia: keep the main ones as code blocks.
- A convention the teacher insists on ("always indent with 2 spaces", "always name the file in lowercase") is still tool trivia: Пропущено, not the note or the course file. I watch the recordings anyway; the note is for revising what is in focus.
- Kept is what a student of this course should be able to explain on the exam; a fact that only matters to someone typing this exact config is dropped.
- The teacher's hints on what in the subject matters or will be on the exam go to "Акценты преподавателей" in the course file.
- The rule on contradicting documentation (AGENTS.md) applies only to facts kept in the note.
- A list from a slide or a standard source (e.g. the 12 factors) keeps every item; from the talk add only what the slide lacks.
- An example or number only when the idea rests on it, in a few words ("changes cause 70%+ of failures").
- Chat Q&A: only an answer that adds a definition or a rule.
- A linked article: only the part the teacher pointed to.
- General basics: drop, or a bare term in one line if the rest builds on it.
- Teacher's stories: drop; a story with a real-world lesson becomes that lesson in one line.
- Demos, live coding and exercises: the code that carries the idea, with what was done and the outcome in its comments; no step-by-step retelling or task solutions.
- Code the teacher read aloud is reconstructed, with one line per note saying so.
- A demo that only illustrates a conclusion already in the note gets one line, no code.
- Proofs and derivations: a one-line idea.
- Homework: just `ДЗ` in Пропущено; I track homework separately.
- What the sources don't cover is left out, not noted ("не обсуждали", a table row of dashes).

## Structure

- Before writing, name the 3-6 main ideas of the class: claims about the subject with their reason, not topic titles. Sections follow these ideas, not slide boundaries or the order of talk.
- A section opens with 1-2 sentences: the idea and why it holds. Then its support (a code block, a table, a definition), then a few remarks on that support.
- A fact or rule goes into the section of the idea it serves; one that serves none goes to Пропущено, not into a catch-all section like "Основы", "Правила" or "Разное".
- A term is defined at its first use in the note, not in a later section.
- When a section has no idea to state (a run of separate terms each with its code, a comparison the table says in full, a list from a slide), start with the support itself. An opening sentence that only restates the heading, or unrelated facts glued into a paragraph, is worse than no opening.
- Size the note by how many ideas the class has, not by the length of the transcript: about 100-150 words of text per idea, code blocks not counted, so a class with 3 ideas gives about 400 words. 800 words is a hard limit, not a size to fill.
- End with a collapsed callout listing what was cut:
  ```markdown
  > [!note]- Пропущено
  > байка про инцидент X, демо установки Y, ДЗ.
  ```
- An existing note: merge into its sections instead of appending a duplicate.

## Form

- A config, manifest, command, query, log line, API call, etc.: a code block with short comments instead of a description in words.
- Right and wrong variants of the same code: one code block with a comment per variant, not a table describing them in words.
- Where the class had no code but the idea needs some, my own example is fine, with one line per note saying some code is mine.
- Things compared by the same attributes, or many items of one kind: a table; remarks go below as short bullets keyed to the row; an item that gets a row gets no separate line repeating it.
- A term worth recalling: its own line `**Термин** - одна фраза`. A problem, a cause or a position is not a term. Bold only terms being defined.
- A paragraph makes one point in 1-2 sentences; a third sentence usually is a second point or a detail to cut.
- A remark: a bullet with one fact, about 100 characters; two facts joined by `;` become two bullets or a sentence.
- A point with parts is a short parent bullet with nested ones.
- Machine transcripts mishear terms: fix obvious ones silently ("кубер нейтс" -> Kubernetes); `[?]` only when the meaning itself is in doubt.
- Russian that reads like a good cheat sheet, neither essay nor telegraph: cut filler, lead-ins ("Ниже...", "X различаются тем, что...") and the obvious, never the subject or verb a sentence needs.
- The line after a heading starts its own sentence, not a continuation of the heading's phrase.
- In text, a verb carries the claim: no arrow or lone bold word in its place.
- No officialese: "является", "осуществляется", "обеспечивает", "в рамках", etc.
- Wikilinks wherever a concept from another note is used, in this subject or another one, `_archive/` included.
- Citations: one where a section starts, plus one per definition and per paragraph that draws on another moment of a timestamped recording. Bullets carry none: a fact that needs a citation of its own becomes a paragraph.

## Check

After writing, revise the note in place:
- Ask of every fact whether the sources say it as written: a paraphrase must not add a cause or detail the teacher didn't state. Fix such lines; general knowledge gets `(вне конспекта)`.
- Ask of every fact whether it helps explain something in `Фокус`, else move it to Пропущено.
- Count the words of its text without frontmatter, code blocks and citations with a quick script; while over the size its ideas justify, cut the lines I'd miss least when revising for the exam.
- Every section that has an idea opens with it.
- A list longer than about 8 bullets is split by the ideas it serves or partly cut.
- No bullet carries two facts.
- Find bullets with citations: `grep -nE '^\s*- .*(\?t=[0-9]|#page=|-transcript\|)' <note>`. Drop a citation when the section's own citation already leads to that part of the source; otherwise turn the bullet into a paragraph that keeps it.
- Fix every line that lacks a subject or verb it needs, uses an arrow or a lone bold word instead of a verb, or breaks case or agreement.

## Examples

A class with code: the idea, then the code it rests on, then remarks.

````markdown
## Дедлок

**Дедлок** - вечная взаимная блокировка потоков, при которой программа зависает. Разбудить такие потоки снаружи нельзя, поможет только перезапуск программы. ([00:24](https://kontur.ktalk.ru/recordings/ApfZoVY7uNlZXrUh9YzR?t=24))

Классический пример - вложенные блокировки в разном порядке. Это та же ситуация, что в задаче об обедающих философах. ([04:45](https://kontur.ktalk.ru/recordings/ApfZoVY7uNlZXrUh9YzR?t=285))

```csharp
// поток 1: Петя -> Вася
lock (petya)
{
    Thread.Sleep(100);   // даём второму потоку взять свой лок
    lock (vasya) { ... }
}

// поток 2: Вася -> Петя
lock (vasya)
{
    Thread.Sleep(100);
    lock (petya) { ... }
}
// поток 1 держит Петю и ждёт Васю, поток 2 держит Васю и ждёт Петю
```

"Молитва против дедлоков" работает в любых языках: ([32:10](https://kontur.ktalk.ru/recordings/ApfZoVY7uNlZXrUh9YzR?t=1930))
1. Вложенные блокировки везде брать в одном порядке.
	- порядок задаёт неизменный ключ: ID или имя, но не баланс
2. Отпускать блокировки в обратном порядке, как стек.
	- `lock` делает это сам, следить надо при ручных `Mutex` и `Monitor`

Можно взять один глобальный лок на все переводы. Дедлоков не будет, но в каждый момент пройдёт только один перевод, а банку с миллионами операций это не подходит. ([11:43](https://kontur.ktalk.ru/recordings/ApfZoVY7uNlZXrUh9YzR?t=703))
````

An overview class without code: the idea in prose, details that support it, the rest cut.

```markdown
## Предсказуемость поставки

Главная задача CI/CD - предсказуемость: на проде ровно то, что прошло тесты. Окружения теста и прода готовит одна автоматика: руками даже по чеклисту они разъедутся. ([37:33](https://fiit.ktalk.ru/recordings/7Nty0Rsp2R27Jn3Cay5Q?t=2253))

Дистрибутив собирают только в CI и из общего хранилища отдают и на тест, и на прод: пакет с ноутбука может подхватить из системы другую версию библиотеки. ([31:12](https://fiit.ktalk.ru/recordings/7Nty0Rsp2R27Jn3Cay5Q?t=1872))

**Координаты дистрибутива** - репозиторий, ветка и коммит сборки в label образа. По ним автоматика проверяет, что на проде то, что тестировали. ([38:21](https://fiit.ktalk.ru/recordings/7Nty0Rsp2R27Jn3Cay5Q?t=2301))
```
