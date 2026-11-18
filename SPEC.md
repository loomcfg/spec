# The Loom Configuration Language Specification (v0.1.0-draft)

**Status:** Draft Specification  
**File Extension:** `.loom`  
**Media Type:** `application/loom` (provisional)  
**Character Encoding:** UTF-8 (without Byte Order Mark)  

---

## 1. Design Philosophy & Overview

**Loom** is a human-authored configuration and data serialization format designed for application and service configuration. It combines the strict, unambiguous typing of JSON with ergonomic authoring features (comments, optional commas, bare keys, dedented multiline strings) and deterministic, non-Turing-complete composition (`@import` and object spreading).

1. **JSON-Isomorphic Core:** Every Loom document evaluates to a standard JSON data model (`object`, `array`, `string`, `number`, `boolean`, `null`).
2. **Syntax-Driven Typing:** A value's type is strictly determined by its lexical syntax—never by heuristic content guessing.
3. **Deterministic & Bounded Composition:** File imports and object spreads allow modular configuration without loops, arbitrary function execution, or side effects.
4. **Lossless Formatting & CST Support:** Comments, blank lines, and key ordering are preserved in the Concrete Syntax Tree (CST) to enable safe programmatic edits.

---

## 2. Lexical Structure & Encoding

### 2.1 Encoding & Line Endings
* Documents **MUST** be valid UTF-8. A leading UTF-8 Byte Order Mark (`U+FEFF`) is a syntax error.
* Line breaks **MUST** be either Line Feed (`\n`, `U+000A`) or Carriage Return followed by Line Feed (`\r\n`, `U+000D U+000A`). Lone `\r` characters are forbidden.
* Unescaped ASCII control characters (`U+0000` through `U+001F`, except `\t` `U+0009`, `\n` `U+000A`, `\r` `U+000D`) and `DEL` (`U+007F`) are invalid anywhere in a document, including inside comments and strings.

### 2.2 Whitespace & Comments
Whitespace consists of Space (`U+0020`) and Horizontal Tab (`U+0009`). Indentation is never semantically significant except when stripping common leading indentation from block strings (§3.2).

Two comment styles are supported:
* **Line comments:** Begin with `#` and extend to the end of the line.
* **Block comments:** Begin with `/*` and end with `*/`. Block comments **MAY** span multiple lines and **MAY NOT** be nested.

```loom
# Single-line comment
port: 8080 /* Inline block comment */
```

---

## 3. Data Types & Literals

### 3.1 Objects & Root Document
* A Loom document **MUST** evaluate to a JSON Object at the root.
* Top-level outer braces `{` and `}` are **implicit** and **MUST NOT** be written at the root level.
* Nested objects are enclosed in `{` and `}`.
* Key-value pairs are separated by `:`.
* Entries within an object are separated by either a newline (`\n` / `\r\n`) or a comma (`,`). Trailing commas are permitted.
* **Duplicate Keys:** Duplicate keys within the same lexical object are a compile-time error. (Overriding keys via `...spread` is permitted and follows last-writer-wins semantics; see §4.2).

#### Keys
* **Bare keys:** May contain ASCII letters (`A-Z`, `a-z`), digits (`0-9`), underscores (`_`), and hyphens (`-`). A bare key **MUST** start with an ASCII letter or underscore (`[A-Za-z_][A-Za-z0-9_-]*`) and **MUST NOT** be a reserved keyword (`true`, `false`, `null`).
* **Quoted keys:** Any valid single-line escaped string (`"..."`) or literal string (`'...'`) may be used as a key, including empty keys `""`.

### 3.2 Strings
All strings in Loom are explicitly delimited. Unquoted values other than numbers, booleans, and `null` are syntax errors.

1. **Escaped Strings (`"..."`):** Single-line strings supporting standard JSON escape sequences (`\"`, `\\`, `\/`, `\b`, `\f`, `\n`, `\r`, `\t`, `\uXXXX`, plus `\u{X...}` for Unicode scalar values).
2. **Literal Strings (`'...'`):** Single-line strings with no escape processing. Cannot contain unescaped `'` or line breaks.
3. **Multiline Block Strings (`"""..."""` and `'''...'''`):**
   * The opening delimiter **MUST** be followed immediately by a newline (optional trailing whitespace is forbidden).
   * The closing delimiter **MUST** appear on its own line, preceded only by whitespace.
   * **Indentation Stripping (Dedent):** The whitespace prefix of the closing delimiter line defines the baseline indentation. That exact prefix is stripped from every non-empty content line. Any content line with less indentation than the closing delimiter (other than empty/whitespace-only lines) is a syntax error.
   * `"""..."""` processes escape sequences; `'''...'''` is strictly literal.

### 3.3 Numbers
Numbers map to IEEE 754 binary64 floating-point values or signed 64-bit integers (`[-2^63, 2^63 - 1]`):
* **Integers:** Decimal (`42`, `-100`), Hexadecimal (`0xFF_00`), Octal (`0o755`), Binary (`0b1010_0101`).
* **Floats:** Must include a fractional part (`3.14`), an exponent (`1e-6`, `2.5E+8`), or both.
* **Underscores:** Single underscores (`_`) are permitted between digits for readability (`1_000_000`). Leading, trailing, or consecutive underscores are invalid.
* `NaN`, `Infinity`, and `-Infinity` are **NOT** valid numeric literals (preserving strict JSON compatibility).

### 3.4 Booleans & Null
* **Booleans:** Lowercase `true` and `false`.
* **Null:** Lowercase `null`.

### 3.5 Arrays
* Arrays are enclosed in `[` and `]`.
* Elements may be of heterogeneous types and are separated by commas (`,`) or newlines (`\n` / `\r\n`). Trailing commas are permitted.

### 3.6 Type Tags (`@tag "value"`)
Type tags attach semantic domain metadata to string literals without breaking JSON interoperability:
* **Syntax:** `@` followed by a bare identifier (`[a-z][a-z0-9_-]*`), optional whitespace, and a string literal (`"..."` or `'...'`).
* **JSON Compilation:** When compiled to JSON, a tagged value emits its underlying string value directly.
* **SDK Hydration:** Native Loom SDKs (Rust, Python, TypeScript) parse built-in tags into native runtime types and validate their lexical format at compile time:
  * `@dur` — Compound duration (`"15s"`, `"1m30s"`, `"20ms"`, `"7d"`). Units: `ns`, `us`, `ms`, `s`, `m`, `h`, `d`.
  * `@date` — RFC 3339 full-date (`"2026-11-17"`).
  * `@instant` — RFC 3339 date-time with mandatory offset (`"2026-11-17T21:20:00Z"`).
  * `@regex` — Regular expression pattern (`"^[a-z0-9-]+$"`).
  * `@bytes` — Byte size literal (`"512MiB"`, `"10GB"`).
  * Custom user-defined tags (e.g., `@env "PORT"`, `@secret "db_pass"`) are preserved in the AST for host-application resolvers.

---

## 4. Directives & Composition

Directives appear at the top of a `.loom` document before any key-value entries.

### 4.1 `@schema`
Declares a companion JSON Schema file for validation and editor tooling:
```loom
@schema "./schemas/service.schema.json"
```
* At most one `@schema` directive is permitted per document.

### 4.2 `@import` & Object Spreading (`...`)
Loom supports deterministic, acyclic file composition:
```loom
@import "./presets/base.loom" as base
```
* **Import Rules:**
  * Import paths **MUST** be relative POSIX-style string literals starting with `./` or `../`. Network URLs are forbidden.
  * Every `@import` **MUST** bind to an explicit alias (`as <ident>`).
  * Cyclic imports (`A -> B -> A`) and imports exceeding a maximum depth of `32` are compile-time errors.
* **Spreading (`...`):**
  * Inside an object `{ ... }`, `...<expr>` merges all key-value pairs from the referenced object into the current object.
  * `<expr>` may be an imported alias (`...base`) or a dot-path into an imported alias (`...base.server.tls`).
  * **Evaluation Order:** Entries are evaluated top-to-bottom. Later keys or spreads overwrite earlier keys with the same name (shallow merge).

---

## 5. Canonical Formatting & Limits

### 5.1 Canonical Formatting (`loom fmt`)
A compliant formatter produces deterministic output:
* UTF-8 with `\n` line endings and a single trailing newline at EOF.
* 2-space indentation for nested blocks and arrays.
* Single space after `:` (`key: value`).
* Multi-line arrays and objects omit unnecessary commas at line endings; single-line arrays and objects separate items with `, `.
* Comments attached immediately above an entry or inline after an entry remain bound to that CST node during formatting and programmatic `edit` mutations.

### 5.2 Implementation Limits
To prevent denial-of-service on untrusted configuration inputs, compliant parsers **MUST** enforce:
* **Maximum nesting depth:** `128` levels (objects and arrays combined).
* **Maximum `@import` depth:** `32` levels.
* **Maximum document size:** `16 MiB` (configurable by host runtime).
