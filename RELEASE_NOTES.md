## v0.1.9

### Parser

- Ported four SDBL parse-gap fixes from upstream
  [PR #13](https://github.com/alkoleft/tree-sitter-bsl/pull/13): field
  access after `ВЫРАЗИТЬ(...)`, tuple left-hand sides of `В`,
  `УНИЧТОЖИТЬ` inside query packages, and nested joins with deferred
  `ПО`. Official `develop` had no new commits. Fork packaging, bindings
  and the `tree-sitter-bsl` name are unchanged.

## v0.1.8

### Packaging

- Aligned the package version across every manifest, including the
  self-version in `Cargo.lock` and `package-lock.json`, which had been left
  behind by previous bumps. `Makefile` and `CMakeLists.txt` had drifted further
  still, to 0.1.6, and are dropped entirely below.
- Pointed package metadata at this fork: `pyproject.toml`, `package.json`,
  `Cargo.toml`, `tree-sitter.json`, `Makefile` and `CMakeLists.txt` referenced
  either the upstream repository or a non-existent `tree-sitter/tree-sitter-bsl`.
- Renamed the Go module path to `github.com/cujoko/tree-sitter-bsl`. Breaking
  for Go consumers, but the previous path never resolved to this project.
- Removed the tag-triggered workflow that published to npm, crates.io and PyPI.
  Those names belong to the upstream project, so the jobs could not have
  succeeded.
- Documented installation from the repository and dropped the badges and
  playground link, which reported upstream's packages and grammar.
- Removed the Zed dev extension (`editors/zed-bsl`), including its
  `sdbl_embedded` carrier grammar. It pointed at the upstream author's local
  checkout and is unused here; ADR-0002 keeps the record of the approach.
- Dropped the unused binding scaffolding inherited from upstream: the Go, C,
  Java and Kotlin bindings, `go.mod`, `Package.swift`, `Makefile` and
  `CMakeLists.txt`. The package ships the Python, Rust and Node bindings, which
  are the ones this project and its consumers build against. `tree-sitter.json`,
  `.gitattributes`, `.editorconfig` and the GitHub CI matrix were updated to
  match.

### Packaged Query Files

- The wheel now ships the SDBL highlight queries. Only `grammars/bsl/queries`
  was copied into the package before, so `grammars/sdbl/queries/highlights.scm`
  never reached consumers despite being documented as part of the package.
- Query files moved from `tree_sitter_bsl/queries/` to
  `tree_sitter_bsl/queries/<grammar>/`, because both grammars use the file name
  `highlights.scm`. Breaking for anything reading the old path; nothing in this
  workspace reads these files today.
- `MANIFEST.in` and the `include` list in `Cargo.toml` were extended to cover
  the SDBL queries as well.

No parser or grammar changes.

## v0.1.7

### Parser

- Added the standalone `sdbl` grammar for 1C query texts, including query
  packages, temporary-table statements, source joins, top-level sections,
  literals, functions, operators and real-query acceptance coverage.
- Expanded BSL grammar coverage for imported Lezer gaps, date literals,
  repeated omitted arguments, keyword-like member names, string literals,
  `Выполнить` chains and exception rethrow statements.
- Moved BSL and SDBL into explicit `grammars/<name>/` directories while keeping
  public binding entry points stable.

### Bindings and Editor Integration

- Exposed SDBL through Node.js, Rust, Python, Go and C bindings while keeping
  BSL as the default package language.
- Added tree-sitter highlight queries for BSL/SDBL and BSL string injections
  for static query texts.
- Added a local Zed dev extension, including `sdbl_embedded` for raw BSL string
  injection highlighting.

### Tooling and Docs

- Added per-grammar playground commands and quick parse scripts for BSL/SDBL.
- Rewrote the user-facing README in Russian and documented the current
  BSL/SDBL usage, playground, validation and editor-integration flow.
- Updated parser-work specs, ADR implementation status, release notes and the
  active/archive task ledgers.

**Full Changelog**: https://github.com/alkoleft/tree-sitter-bsl/compare/v0.1.6...v0.1.7
