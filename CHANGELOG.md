# Changelog

All notable changes to table-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.0.2 — 2026-09-15

README rewritten to the package README style guide (docs/writing-a-readme.md); no change to the interface.

## 0.0.1 — 2026-09-11

The **interface**: every signature and every effect row, and no bodies.
`stability = "draft"`, and the release is recorded `implemented = false`.

### Added

- `tblmodel` — the table as a value: typed cells that carry what their
  text was formatted from, columns with width constraints as values,
  separator rows, hidden columns that keep their cells, and refusals
  that name both numbers.
- `tblwidth` — `rule()`, the display-width rule built from unicode-nv's
  `char_width`; `measure`, which counts ANSI escapes as nothing; and
  the solver, with `TableTooNarrow` rather than a column squeezed to
  zero.
- `tblstyle` — eleven glyphs and four flags, with seven presets and
  markdown's three rules as fields rather than a branch.
- `tblrender` — lines, one line at a time for a pager, and a render
  into a caller's buffer.
- `tblexport` — CSV with RFC 4180 quoting, TSV with `tsv_is_lossless`
  beside it, a caller-chosen delimiter, and markdown.

### Known

- **Columns are counted in cells.** `"世界"` is six bytes, two
  codepoints and four cells, and a table sized by either of the first
  two does not line up.
- **The width rule is unicode-nv's, supplied to textwrap-nv**, which is
  the composition textwrap-nv's own manifest anticipated; no signature
  in either package changed for it.
- **An ANSI escape costs no width**, a truncation closes what it cut
  open, a wrap re-opens the style on each line, and an export strips
  them.
- **The total width is an argument**, never a terminal query, which is
  what keeps this `core` and what lets one table go to a screen, a file
  and a test.
- **The solver is an allocation rather than a constraint system**, and
  it refuses rather than squeezing a column to zero.
- **No spans**, and the README says so where a reader will see it
  before depending on the package.
- **No device claim**: a table is lists, and a table without a heap is
  not a table.
- **Three `core` dependencies** — unicode-nv, textwrap-nv, ansi-nv —
  each a table that should exist once on this registry.
