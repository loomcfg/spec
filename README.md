# Loom Configuration Language Specification (`loomcfg/spec`)

Official formal specification, EBNF grammar, and language-agnostic conformance test suite for **Loom** (`.loom`).

## Overview

**Loom** is a human-authored configuration format designed for application and service configuration:
- **JSON-Isomorphic Core:** Every valid `.loom` file compiles deterministically to standard JSON.
- **Syntax-Driven Typing:** Eliminates implicit type coercion (such as YAML's `on`/`off`/`no` pitfalls).
- **Bounded Composition:** Supports relative `@import` directives and `...spread` merging without Turing-complete loops or side effects.
- **Lossless CST Editing:** Standardized formatting and comment-preserving mutation rules.

## Repository Structure

- [`SPEC.md`](./SPEC.md) — Formal language specification (v0.1.0-draft).
- [`grammar.ebnf`](./grammar.ebnf) — ISO/IEC 14977 EBNF formal grammar.
- [`conformance/`](./conformance/) — Language-neutral test fixtures (`valid/`, `invalid/`, `edit/`) for verifying parser and formatter implementations.

## License

MIT License — Copyright (c) 2026 Brandon Himpfen / LoomCFG
