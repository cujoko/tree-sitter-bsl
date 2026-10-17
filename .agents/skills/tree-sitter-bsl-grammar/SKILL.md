---
name: tree-sitter-bsl-grammar
description: >-
  Grammar authoring for tree-sitter-bsl: layout of the BSL and SDBL grammars, grammar.js
  structure, adding BSL constructs, corpus test format, generate/test commands, bindings,
  and version bumping. Use when editing grammars/*/grammar.js, adding corpus tests,
  debugging parse trees, or releasing a new grammar version.
---

# tree-sitter-bsl: грамматика и рабочий процесс

## Project Context

`tree-sitter-bsl` is a tree-sitter grammar repository for 1C BSL and, by
`docs/decisions/0001-add-sdbl-query-language-grammar.md`, a separate `sdbl`
grammar for the 1C query language.

The repository owns grammar behavior and parser-facing contracts:

- `grammars/bsl/grammar.js` as the BSL source grammar.
- `grammars/bsl/test/corpus/*.bsl` as the BSL behavioral regression contract.
- `grammars/bsl/src/grammar.json`, `grammars/bsl/src/node-types.json`,
  `grammars/bsl/src/parser.c` and BSL
  binding-facing generated artifacts when BSL grammar generation is part of the
  change.
- `grammars/sdbl/grammar.js` as the SDBL source grammar.
- `grammars/sdbl/test/corpus/*.sdbl` as the SDBL behavioral regression contract.
- `grammars/sdbl/src/grammar.json`, `grammars/sdbl/src/node-types.json` and
  `grammars/sdbl/src/parser.c` as SDBL generated artifacts when SDBL
  grammar generation is part of the change.
- `spec/sdbl-syntax.zip` as the packed vendored source snapshot for 1C
  query-language syntax. Unpack to `spec/sdbl-syntax/` when reading pages.
- Node, Rust and Python bindings only as parser package integration surfaces.

The repository does not own analyzer facts, diagnostics, metadata models, HBK
facts, query tools, report formats or downstream product behavior. Keep those
concerns out of grammar changes unless a later accepted project decision adds a
local contract for them.

Use `spec/IMPLEMENTATION_TODO.md` as the active parser-work ledger. Keep
completed parser-work history under `spec/archive/`. Use
`spec/sdbl-query-language.md` and `spec/sdbl-source-evidence.md` for the durable
SDBL grammar contract and source evidence. `README.md` is user-facing
orientation and package usage documentation, not the implementation ledger. When
README, chat notes, comments or task text conflict with the ledger, reconcile
`spec/IMPLEMENTATION_TODO.md` before implementation.

## Implementation Order

For non-trivial grammar work, follow this order:

1. Read `spec/IMPLEMENTATION_TODO.md` and the relevant
   `grammars/bsl/grammar.js` rules.
   For SDBL work, also read `spec/sdbl-query-language.md`,
   `spec/sdbl-source-evidence.md` and the relevant `grammars/sdbl/grammar.js`
   rules.
2. Add or update focused corpus cases before changing grammar behavior.
3. Implement only the active syntax behavior and its direct verification.
4. Regenerate parser artifacts when the grammar changes.
5. Update `spec/IMPLEMENTATION_TODO.md`, README or release notes when the
   durable parser contract, validation command or public node shape changed.

Keep implementation small and grammar-specific. Do not introduce broad
compatibility layers, hidden fallbacks, generic parser pipelines or downstream
consumer adapters just to make a syntax case pass.

## Grammar Rules

Test and implement concrete BSL syntax behavior, not broad approximations.

- Prefer precise grammar rules over catch-all tokens.
- Do not accept invalid BSL only to avoid `ERROR` nodes.
- Do not accept invalid SDBL only to avoid `ERROR` nodes.
- Keep BSL and SDBL grammar behavior separate until a later accepted decision
  defines embedded query parsing for BSL strings.
- Keep structured preprocessor parsing for `#Если` / `#Область`; do not replace
  it with a generic skipped-line token.
- Use `lezer-bsl` snippets only as candidate input examples. Do not copy Lezer
  AST node names, visitors or failing expectations as the tree-sitter contract.
- Preserve existing node shapes where practical. If a node-shape migration is
  necessary, document it in tests and release notes.
- Keep unknown or intentionally unsupported syntax explicit in the ledger
  instead of hiding it in ad-hoc probes.

## Testing Rules

Test observable parser behavior.

- Corpus tests should describe the syntax contract being protected.
- Expected trees must use the current tree-sitter node style for this project.
- Do not test private helper order or incidental `grammar.js` decomposition.
- Prefer small corpus sections grouped by syntax feature over one large imported
  dump.
- Use deterministic BSL snippets for focused syntax coverage.
- Use real project files only as acceptance corpus inputs and never mutate those
  external checkouts during parser validation.

Normal validation:

- `npm test` verifies that the Node binding loads.
- `tree-sitter test -p grammars/bsl` validates BSL corpus expectations when the
  local CLI works.
- `tree-sitter test -p grammars/sdbl` validates SDBL corpus expectations.
- Targeted Node binding probes are acceptable only as temporary diagnostics when
  the tree-sitter CLI is blocked on the current host.

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
