# SEDMAT Conformance Test Vectors

This directory holds language-agnostic JSON test suites for verifying SEDMAT implementations (Go, TypeScript, Python, Rust, etc.) against the specification.

## Suites

- [`v3.6.json`](./v3.6.json) — Canonical SEDMAT v3.6 parse and execution vectors covering:
  - Substitution flags (`g`, `i`, `m`, `N`, `gi`, `gim`)
  - 1-based paragraph addressing (`N`, `N,M`, `$`, `N,$`), validation errors (`5`, `1,3`, `$`, `$,2d`, `$,$d`, `0d`, `2,0d`, `3,d`, `5,2d`), and terminal deletion semantics (`$d`, `N,$d`, sole-element deletion)
  - Addressed `d`, `d/pat/`, `s///`, `a/text/`, `i/text/` (including `\n` expansion) and `y/src/dst/` transliteration
  - Canonical `{flags}` brace DSL: flexible placement (`{b}text`, `text{b}`, `{c=red}text{b}`), implicit `t=$0`, inline scoping (`{b=text}`), reset (`{0}`), additive no-reset (`{!0}`, bare `{!0}`, multi-span `{!0}text{b}`), breaks (`{+=p|c|s}`), columns (`{cols=N}`), checkboxes (`{check}`), TOC (`{toc}`), comments (`{"=...}`), bookmarks (`{@=...}`), smart chips (`{u=chip://...}`), tables (`{T=...}`), and images (`{img=...}`)

## Schema Overview

Each JSON file contains:

- `spec`: `"SEDMAT"`
- `version`: Spec version string (e.g., `"3.6"`)
- `parse_vectors`: Array of parser + semantic enrichment test cases (`input`, `want_error`, `command`, `pattern`, `replacement`, `global`, `nth`, `address`, `expect_brace`, `expect_span_count`, `expect_table_create`)
- `execution_vectors`: Array of end-to-end document execution cases (`paragraphs`, `script`, `expected_paragraphs`, `want_error`)

## Consumers

- **Go**: [`github.com/visionik/libsedmat-go`](https://github.com/visionik/libsedmat-go) (`conformance_test.go`)
- **TypeScript**: `@voxio/sedmat` / `libsedmat-ts`
