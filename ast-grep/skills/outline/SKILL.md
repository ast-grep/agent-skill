---
name: ast-grep-outline
description: Use when exploring or modifying a codebase and you need a cheap structural map of files, directories, imports, exports, or direct members before reading full source.
---

# Use ast-grep outline

`ast-grep outline` prints a structural map of source code with line
numbers: top-level declarations (imports, functions, classes, structs,
interfaces, modules, enums) and their direct **members** (fields, methods,
constructors, enum variants). It is a local, syntax-only view — cheap enough
to run before any full file read.

Read code in stages: find candidate files with search or file names, outline
them, then open only the source range the outline points to. Defaults adapt to
input: a file shows its local structure with member digests; a directory shows
only its exported surface as grouped names.

## When To Use It

**Understand a file before editing.** Get a table of contents, dependencies,
and public entry points before reading implementation details:

```shell
ast-grep outline <file>
ast-grep outline <file> --items imports
ast-grep outline <file> --items exports
```

**Map an unfamiliar directory.** Scan the public surface of a subtree, then
narrow by symbol type when you know what you are looking for:

```shell
ast-grep outline <dir> --items exports
ast-grep outline <dir> --type struct,enum,function
```

**Zoom into a known symbol.** After search finds a likely name, list its
members with line numbers instead of reading the whole body:

```shell
ast-grep outline <file> --match <symbol> --type class --view expanded
```

**Trace dependency direction.** Find which files import a package or module to
decide where a change belongs:

```shell
ast-grep outline <dir> --items imports --view signatures
```

**Review changed files after editing.** Git tells you what changed; outline
summarizes the resulting structure and public surface:

```shell
ast-grep outline $(git diff --name-only HEAD) --items exports
```

## Argument Guide

- `--items <KIND>` selects top-level items: `structure` for local declarations,
  `exports` for public API, `imports` for dependencies, `all` when
  import/export edges matter together. The default, `auto`, picks `structure`
  for a file and `exports` for a directory. No other value is accepted; to
  keep only functions, classes, or another symbol kind, use `--type`.
- `--view <VIEW>` controls detail, from least to most: `names` for directory
  scans, `signatures` for one line per item, `digest` for signatures plus
  member names, `expanded` for one line per member with its line number. The
  default, `auto`, picks one from the input path. No other value is accepted.
- `--match <REGEX>` filters top-level items by name or signature. Rust regex,
  case-sensitive; it never matches members.
- `--type <TYPE[,TYPE...]>` keeps only some top-level symbol types, such as
  `--type class,function`. Member types like `method,field` never match
  top-level items.
- `--pub-members` hides private members when the view prints members.
- `--json=stream` emits one JSON object per file with precise ranges. Use it
  only to pipe or post-process entries; prefer text for navigation.

## Limits

`outline` shows local syntax structure. It does not resolve references, infer
types, follow re-export chains, or build a call graph. Use `ast-grep run`,
`rg`, or compiler-backed tools for those questions, then outline the candidate
files they surface.
