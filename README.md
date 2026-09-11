# table-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`. Installing this package works;
calling it panics with `not implemented`.

## What this is

Text tables that line up. A table is a value; measuring it, laying it
out, drawing it and exporting it are functions of that value, and none
of them touches a terminal.

- `tblmodel` — the table, its rows, its cells and its columns;
- `tblwidth` — how wide a cell is, and how wide a column should be;
- `tblstyle` — the lines between the cells, as presets and as values;
- `tblrender` — the table as lines;
- `tblexport` — CSV, TSV and markdown out of the same columns.

```
novo pkg add table-nv
novo pkg build
novo test
```

## The one example that will work

```novo ignore
use tblmodel
use tblrender
use tblwidth

// A listing, sized to a width the program asked its terminal for.
fn listing(rows: [[Str]], columns: Int) -> Str
    var t = tblmodel.with_headers(["name", "size", "modified"])
    for r in rows
        match tblmodel.push_row(t, r)
            Ok(next) => t = next
            Err(_)   => return "a row does not match the columns"
    let w = tblwidth.rule()
    match tblwidth.solve(t, columns, w)
        Ok(solved) => tblrender.render(t, solved, w)
        Err(e)     => e.message()
```

## The load-bearing interface: the width is a rule, and it is unicode-nv's

```novo ignore
pub fn rule() -> widths.WrapWidth      // of_char = uwidth.char_width
pub fn measure(text: Str, w: widths.WrapWidth) -> Int
```

**Columns are counted in cells, not in bytes and not in codepoints.**
`str.len` of `"世界"` is six; its codepoint count is two; a terminal
gives it **four** cells. A table sized by either of the first two has a
column that is wrong for every CJK string, every emoji and every
combining mark — and the wrongness is not subtle, it is a row that does
not line up with the one above it.

The rule itself is unicode-nv's: UAX #11's East Asian Width for the
wide forms, the Mn/Me/Cf categories for the zero-width ones, and the
emoji presentation rules for the sequences that join. `tblwidth.rule()`
is a `widths.WrapWidth` built from `uwidth.char_width`, which is
exactly the composition textwrap-nv's own manifest anticipates — *the
width arrives as a rule the caller supplies* — and this package is that
caller. **No signature in either package changes for it to work.**

Two rules, one for measuring and one for wrapping, would be two
answers, and a cell wrapped by one and measured by the other is a cell
that overflows its column. There is one here, and `rule_cjk()` is the
second only in the sense that a terminal configured East Asian is a
different terminal.

## An ANSI escape costs no width

`"\e[31mERR\e[0m"` is fourteen bytes, three cells and one word. A table
that measured the bytes would indent every column after it by eleven,
on every coloured row.

`measure` feeds the text through ansi-nv's `vtparse` and counts only
what the parser reports as printable — a state machine rather than a
regular expression, so an OSC string with a semicolon in it, a CSI with
intermediates and a bare `ESC` are all measured as what a terminal
would actually show.

Two consequences are published rather than left to be discovered:

- **A truncation closes what it cut open.** Cutting `"\e[31mERROR"` at
  three cells leaves the terminal red for the rest of the line and
  every row after it, so `truncate` appends the reset.
- **A wrap re-opens the style on each line**, so a wrapped coloured
  cell is coloured on every line rather than on the first.
- **An export strips them.** A colour sequence in a CSV field is a
  field that breaks whatever reads it, and there is no terminal on the
  other end.

## The solver is an allocation, not a constraint system

Fixed columns take what they ask for; minimums are met; percentages are
computed against the total; what is left is shared among the automatic
columns in proportion to their natural widths, with the remainder given
to the widest.

That is comfy-table's arrangement and tabulate's, and it is
deliberately not more clever: a solver nobody can predict by reading it
is a solver whose output nobody can explain when a column comes out
narrow.

It can refuse. `TableTooNarrow` names what the fixed and minimum
columns plus the borders already need and what the caller offered —
because the alternative is squeezing a column to zero, and a zero-wide
column renders as a stack of ellipses that tells a reader nothing.

**The total width is an argument.** Asking the terminal how wide it is
takes `[io]`; it is the program's question rather than the library's,
and keeping it out here is what lets the same table be solved for a
file, a web page and a test fixture. A caller with no opinion passes
`0` and gets the content's natural width.

## Borders are eleven characters and four flags

Every table anybody draws is the same layout with a different set of
glyphs and a different answer to *which rules are drawn*. So a preset
is a value:

| | what it is for |
| --- | --- |
| `ascii()` | a log file, and a terminal that predates UTF-8 |
| `unicode_box()` `rounded()` `heavy()` | a modern terminal; every glyph is one cell |
| `markdown()` | a README — a real output format, not a decoration |
| `plain()` | a listing. `ls -l` is a table with this style |
| `header_rule()` | `--help` output and package listings |

**Markdown is not a border preset dressed up.** It has three rules the
others do not: the header separator carries the alignment (`:---`,
`---:`, `:---:`), a `|` inside a cell must be escaped, and there is no
bottom rule at all. Those are fields — `escape_pipes`,
`align_in_rule` — so the renderer reads them rather than asking which
preset it was handed.

**Padding is part of the width and the solver knows it.** A column
solved to eight cells inside a border with one space each side occupies
ten. Getting that wrong is how a table comes out one character wider
than the terminal and wraps every line, which is the commonest bug in
this kind of package; `overhead_total` is the arithmetic, published
once.

## What the `novo` CLI's own tables would move onto it

`compiler/bin/novo.ml` prints about twenty tables today, each as a
`Printf.printf` format string with hand-counted column widths —
`"%-26s %-10s %-12s %-6s %s"` for `novo pkg search`, `"%-16s %s"` for
the REPL commands, `"%-12s %-40s %s"` for the requirement listings,
`"%-30s %10s %10s %9s %s"` for the benchmark comparison. Each one has
its widths written twice, once in the header line and once in the row
line, and they are kept in step by hand.

Four things would change, and three of them are bugs that exist today:

- **The widths would be computed rather than guessed.** A package name
  longer than 26 characters currently pushes the whole row right and
  every column after it stops lining up. `TableWidthMax` and the
  solver's truncation are the fix, and nobody has to pick 26.
- **CJK and emoji would line up.** `%-26s` pads by bytes. A package
  description with a single ideograph in it is already misaligned in
  `novo pkg search`, and `measure` is the whole of the fix.
- **The colour would stop costing width.** Anywhere the CLI wants to
  colour a cell today it must either not pad it or count the escape
  bytes by hand.
- **`--format=csv` would be one call rather than a second code path.**
  Today an export means writing the loop again with different
  separators, which is how the display and the export drift apart.

What it would **not** need: spans, styling per cell beyond what the
caller already put in the string, or any terminal query — the CLI
already knows its width and would pass it in.

## What is not here

**No spans.** A cell that spans columns makes the width solver a
constraint system instead of an allocation, makes CSV export ambiguous,
and is wanted by roughly one caller in fifty. comfy-table has them;
this does not, and this paragraph is where somebody finds that out
before they depend on the package.

**No terminal query, no colour decisions, no printing.** The width, the
colour and the writer are all the caller's. A library does not print.

**No JSON or HTML export.** JSON is a shape decision — an array of
objects, or of arrays, and where does the header go? — and belongs to
the caller with std.json; HTML is html-nv's escaping and a caller's own
template.

## The layer, and the three dependencies

`core`. Every value here is arithmetic over text the caller already
holds, and the two things that would make it `host` — asking the
terminal its width, and writing the output — are both the caller's.

Each dependency is a table that should exist once on this registry
rather than three times: **unicode-nv** is the display width,
**textwrap-nv** is the wrap inside a cell, and **ansi-nv** is the
escape scan. All three are `core`.

**No device claim.** Every value here is a list — of rows, of cells, of
lines — and a table built without a heap is not a table. numfmt-nv and
template-nv are the packages a firmware formats a column with.

## The reference implementations

comfy-table and tabled for the model and the width solver, Python's
`tabulate` and `rich.table` for the presets and the alignment
defaults. The expected lines in `tests/` are whole rendered lines,
because a renderer's bugs are off-by-one bugs and an assertion about a
length would pass for a table with the padding on the wrong side.

## Status

**NOT IMPLEMENTED — interface only.** `0.0.1`, `stability = "draft"`,
recorded `implemented = false` on the registry. The first
implementation is the `0.1.0` published over it.

```
novo pkg build     # clean: the signatures type-check and the rows fit
novo test          # red: every body is a todo()
```
