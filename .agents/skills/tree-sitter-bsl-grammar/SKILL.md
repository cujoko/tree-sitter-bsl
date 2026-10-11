---
name: tree-sitter-bsl-grammar
description: >-
  Grammar authoring for tree-sitter-bsl: layout of the BSL and SDBL grammars, grammar.js
  structure, adding BSL constructs, corpus test format, generate/test commands, bindings,
  and version bumping. Use when editing grammars/*/grammar.js, adding corpus tests,
  debugging parse trees, or releasing a new grammar version.
---

# tree-sitter-bsl: грамматика и рабочий процесс

Порядок работы, правила грамматики и тестов — в `AGENTS.md` (разделы «Implementation
Order», «Grammar Rules», «Testing Rules»). Здесь — устройство репозитория и команды.

## Две грамматики

Репозиторий держит две независимые грамматики (`docs/decisions/0003-use-per-grammar-directories.md`):

- **`bsl`** — 1C:Enterprise BSL (рус./англ. ключевые слова, case-insensitive), файлы `bsl`, `osl`;
- **`sdbl`** — язык запросов 1С (`docs/decisions/0001-add-sdbl-query-language-grammar.md`),
  контракт — `spec/sdbl-query-language.md`, источники — `spec/sdbl-source-evidence.md`.

Внедрение SDBL в строки BSL — только через injection-запросы
(`docs/decisions/0002-define-bsl-string-sdbl-injection-contract.md`), не через саму грамматику BSL.

## Ключевые файлы

| Область | Путь | Роль |
|---------|------|------|
| **Грамматика** | `grammars/<g>/grammar.js` | Единственное место для правки конструкций |
| **Генерированный парсер** | `grammars/<g>/src/parser.c`, `grammar.json`, `node-types.json` | Результат `tree-sitter generate`, коммитится вместе с `grammar.js` |
| **Corpus-тесты** | `grammars/<g>/test/corpus/*.bsl` / `*.sdbl` | Эталонные S-expression деревья |
| **Queries** | `grammars/<g>/queries/*.scm` | Подсветка и injections для редакторов |
| **Конфиг** | `tree-sitter.json` | Обе грамматики, версия, включённые bindings |
| **Bindings** | `bindings/python/`, `bindings/node/`, `bindings/rust/` | Интеграция пакета, не место для логики грамматики |
| **Журнал работ** | `spec/IMPLEMENTATION_TODO.md` | Активный план; история — `spec/archive/` |

`<g>` — `bsl` или `sdbl`. `src/parser.c` вручную не редактируется.

## Структура `grammars/bsl/grammar.js`

```js
module.exports = grammar({
  name: 'bsl',
  extras: ($) => [/\s/, $.line_comment],
  conflicts: ($) => [[$._plain_variable_spec, $._exported_variable_spec]],
  word: ($) => $.identifier,
  reserved: { global: ($) => reservedKeywords($) },  // ключевые слова ≠ identifier
  rules: { source_file: ($) => repeat($._definition), ... }
})
```

- `PREC` — числовые приоритеты выражений.
- `CORE_KEYWORDS` — пары `[русский, английский]`; `buildKeywords()` создаёт правила
  `IF_KEYWORD`, `WHILE_KEYWORD` и т.д.
- `PREPROC_KEYWORDS` — препроцессор (`#Если`/`#If`, `#Область`) и аннотации.
- Всё, что попало в `reservedKeywords`, не может быть `identifier`.

## Добавление конструкции BSL

1. Corpus-случай в `grammars/bsl/test/corpus/*.bsl` — до правки грамматики.
2. Ключевое слово → `CORE_KEYWORDS` / `PREPROC_KEYWORDS`; правило → `rules`
   (`seq`, `choice`, `repeat`, `optional`, `field`, `alias`, `prec`).
3. Если токен не должен быть идентификатором — проверить, что он попадает в `reservedKeywords`.
4. `npm run generate:bsl` → `npm run test:corpus:bsl`.
5. Закоммитить `grammar.js` вместе с `src/*`.

## Формат corpus-тестов

```
================
Присвоение переменной
================

А = 1;

---

(source_file
  (assignment_statement ...))
```

- Заголовок теста обрамлён строками из `=`; исходник и дерево разделены `---`.
- Незавершённый ввод: `(MISSING ...)` / `(ERROR ...)` в ожидаемом дереве.
- Ожидаемое дерево копировать из вывода `tree-sitter test`/`parse`, а не писать по памяти.

## Команды

| Что | Команда |
|-----|---------|
| Регенерировать парсер | `npm run generate` (или `generate:bsl` / `generate:sdbl`) |
| Corpus-тесты | `npm run test:corpus` (или `test:corpus:bsl` / `test:corpus:sdbl`) |
| Системный CLI | `tree-sitter test -p grammars/bsl`, `tree-sitter test -p grammars/sdbl` |
| Разобрать файл | `npm run parse:bsl -- <file>` / `npm run parse:sdbl -- <file>` |
| Node binding | `npm test` (собирает addon и грузит обе грамматики) |
| Python binding | `python -m unittest bindings/python/tests/test_binding.py` в `.dev`-окружении |
| Lint | `npm run lint` |
| Всё сразу | `npm run test:all` |
| Playground | `npm start` (BSL), `npm run start:sdbl` |

## Python binding

```python
import tree_sitter
import tree_sitter_bsl

bsl = tree_sitter.Parser(tree_sitter_bsl.Language())
sdbl = tree_sitter.Parser(tree_sitter_bsl.SDBLLanguage())
```

Потребитель в workspace — `codemask-1c-core` (`tree-sitter-bsl>=0.1.8`; в `.dev` —
editable `file:///` на этот checkout). Пакет не публикуется в публичные реестры; установка —
из git (см. `README.md`).

## Версия и бамп

Версия одна на весь пакет: `bindings/python/tree_sitter_bsl/_version.py` (источник для
`pyproject.toml`), `package.json`, `Cargo.toml`, `tree-sitter.json` (`metadata.version`).
При релизе менять все четыре и дописывать `RELEASE_NOTES.md`.
