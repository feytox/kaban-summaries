# Progress files

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

`<slug>-retell.md` is a log of retellings, newest entry first:

```markdown
### 2026-10-21 - [[algo-L03-хеш-таблицы]]
<my text verbatim>

Пробелы:
- ...
```
