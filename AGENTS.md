# Agent Notes

Shared agent guidance lives in `AGENTS.md` (this file) and `.agents/skills/`.
`CLAUDE.md` imports this file. Cursor, Codex, and Claude Code all read it.

## Before Making Changes

- Files under `.cursor/rules/` are project-only extras (globs, 1C, terminals), not a second copy of the shared rules.
- For grammar, parser, or architecture changes, read the relevant `.agents/skills/*/SKILL.md`.
- Follow the managed sections below; they apply to every agent.

## Skills To Check

- `.agents/skills/tree-sitter-bsl-grammar/SKILL.md` for grammar, parser, generated parser files, and corpus/test changes.

# Project Rules for Agents

These rules apply to the whole repository.

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
- `grammars/sdbl/grammar.js` as the planned SDBL source grammar.
- `grammars/sdbl/test/corpus/*.sdbl` as the planned SDBL behavioral regression
  contract.
- `grammars/sdbl/src/grammar.json`, `grammars/sdbl/src/node-types.json` and
  `grammars/sdbl/src/parser.c` as planned SDBL generated artifacts when SDBL
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
   rules once that file exists.
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
- `tree-sitter test -p grammars/sdbl` validates SDBL corpus expectations after
  the SDBL grammar scaffold exists.
- Targeted Node binding probes are acceptable only as temporary diagnostics when
  the tree-sitter CLI is blocked on the current host.

## Worktree Discipline

Generated artifacts, dependency directories and unrelated local changes may
exist in the checkout. Inspect scope before editing or committing, and do not
revert user-owned changes. Keep commits narrow: grammar changes, corpus changes,
generated parser updates and documentation updates should be grouped only when
they belong to the same parser behavior.

<!-- agent-rules:begin | управляется sync-agent-rules.py, правьте dev-utils/agent-rules/ -->

## External project notes

This project may have a `.notes` directory that points to external working notes.

Rules for using `.notes`:

- `.notes` is not automatically authoritative.
- Prefer `.notes/_current.md` as the curated current context.
- Treat other notes as non-authoritative unless they have explicit metadata such as `status: active` or `status: reference`.
- Treat `.notes/00-inbox/`, `.notes/30-someday/`, `.notes/80-completed/`, `.notes/90-archive/`, old plans, drafts and raw imported notes as historical or unprocessed context only.
- Folders `.notes/10-urgent/` and `.notes/20-active/` may hold current task notes; still verify them against the repository before acting.
- Closed tasks live in `.notes/80-completed/`; reference, dumps and historical material live in `.notes/90-archive/`.
- Source code, tests, configs, migrations, build scripts and repository files override external notes.
- If an external note conflicts with repository files, do not silently follow the note. Mention the conflict and prefer the repository.
- Do not perform large changes based only on old notes. First verify against current code and current project instructions.

## Local junction directories

The project root on a developer machine may contain **junction** directories
(not in git; they may be missing on other machines).

### `.temp/`

- Temporary files **for this project**: debug output, intermediate artifacts,
  manual experiments.
- The junction points outside the repository (typically `D:\Temp\<project>`).
- If `.temp/` exists, prefer it over `tmp`, `temp`, `test_output`, and similar
  directories inside the tracked tree.
- If the junction is absent, use the system temp directory (Python:
  `tempfile.mkdtemp()`, `tempfile.TemporaryDirectory()`; PowerShell: `$env:TEMP`,
  `[System.IO.Path]::GetTempPath()`).
- Output of a transformation or build — obfuscation, parsing, conversion,
  normalization — goes there too. Never write it into tracked test data or
  fixtures.
- Do not commit `.temp/` contents.

### `.notes/`

- Local external working notes. The junction typically points outside the
  repository, for example `D:\Notes\Work\_Dev\...\<project>`.
- Policy for those notes is in the "External project notes" section.
- If `.notes/` is absent, do not require it in CI or on other machines.

## Python package manager: PDM

This project uses **PDM** for dependencies and virtual environments.

- Install or sync with `pdm install` or `pdm sync`.
- Add or remove packages with `pdm add` / `pdm remove`.
- Run tools and scripts with `pdm run -p .dev …` when they are configured in
  `pyproject.toml`.

Do **not** use `uv`, `pip install` (for project lockfiles), or Poetry unless the
user explicitly asks for an exception.

`PDM_USE_UV` must stay **unset** in every environment — Windows, WSL and the
dev container alike. The uv resolver does not support PDM's `inherit_metadata`
lock strategy and silently discards it. When that happens, `requires_python` and
`groups` disappear from every entry in `pdm.lock`, so the lock no longer records
which group a package belongs to. Updating a single package rewrites roughly
600 lines.

A mixed setup is the worst case: with uv enabled on one machine and disabled on
another, `pdm.lock` flips between `strategy = ["inherit_metadata"]` and
`strategy = []` on every update, producing conflicts across the whole file.

`PDM_USE_UV` is an environment variable and overrides a per-project `pdm.toml`,
so the setting cannot be pinned inside the repository. Check before locking:

```sh
pdm config use_uv   # must report False
```

## Python environment safety

These rules apply to **every** Python invocation: tests, apps, helpers,
migrations, generators, one-off scripts, and `python -c`.

- Resolve the repository environment first and check the real interpreter with
  `sys.executable`.
- Prefer the nested `.dev` project (typically `.dev/.venv`) through the project
  manager: `pdm run -p .dev ...`. If `.dev` is absent, use the root `.venv`.
- Do not run task logic with system/base Python or user-site, even for a
  temporary script that only uses the standard library.
- Base Python is allowed only to discover interpreters and verify the
  environment (`py -0p`, `python --version`, printing `sys.executable`). After
  that, run further Python through the project environment.
- Do not install dependencies into system/base Python or user-site
  (`pip install`, `python -m pip install`, and equivalents aimed there).
- If there is no suitable venv, or a dependency is missing from it, stop and
  tell the user. Do not "fix" that with a global install.
- Installing or upgrading anything in base Python needs an explicit user OK for
  that exact action.
- Use `pipx` only for a planned user-facing CLI install or parity check, never
  as a substitute for the repository dev environment.

## Reading files: keep the context small

Whatever you read stays in the conversation and is sent again with every later
request. A large file read twice costs you twice on every request that follows.
Russian text also costs more tokens per character than English.

- **Do not re-read a file already in this conversation.** Use the content you
  already have. Re-read only when the file may have changed since:
  - you edited it;
  - a command or formatter rewrote it;
  - the checkout moved.

  After the context has been compacted, the earlier content is gone, so reading
  it again is correct. The same applies to skills, `AGENTS.md`, and docs.
- **Read large files in parts.** A large file is roughly 10 KB or more: a source
  module, a long doc, a log.
  - First find the place: `rg -n` for a symbol or phrase, or an outline such as
    `rg -n '^(def |class |Процедура |Функция )'`.
  - Then read only the line ranges you need, for example
    `Get-Content <file> | Select-Object -Skip N -First M` or `sed -n 'N,Mp'`.
  - Read a large file whole only when the task needs all of it, such as a
    rewrite or a full review.

These rules are about repeated and oversized reads, not about skipping context.
The start-up route (`AGENTS.md`, `.ai/*`, the relevant skills) is still read
once.

## Commit message format

When the agent stops and has changed files, it must proactively suggest commit
messages.

For each affected project/repository with file changes, provide exactly one
ready-to-use message.

If no files were changed, do not suggest a commit message.

Each message must be concise, imperative, and aligned with repository style.
This is a suggestion only. Creating a commit is a separate explicit request
(`/cm` or `$cm`).

<!-- agent-rules:end -->
