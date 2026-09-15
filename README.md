# table-nv

A text table lays rows of already-formatted text out in columns that line up,
for a terminal, a log file or a markdown document. This package brings that to
novo-lang. It follows [comfy-table](https://docs.rs/comfy-table) and
[tabled](https://docs.rs/tabled) for the model and the column-width solver, and
Python's [tabulate](https://pypi.org/project/tabulate/) and
[rich](https://rich.readthedocs.io/en/stable/tables.html) for the border presets
and the alignment defaults.

**Status: NOT IMPLEMENTED — interface only.** Every function is declared with
its full signature, but every body is a `todo()` that panics when called. The
package is published so its design can be reviewed and depended on before it is
implemented. Version 0.1.0 will be the first working release.

## What it is

A **table** is a list of columns and a list of rows. A **row** holds one
**cell** per column. A cell holds text the caller has already formatted, plus a
**kind** saying what that text was formatted from: plain text, a number, a
timestamp, or nothing at all. The kind decides the default alignment and gives a
sort something to compare. Nothing here reformats a number.

A **column** carries a header, an alignment, and a **width constraint**: as wide
as its widest cell, exactly so many cells, at least so many, at most so many, a
range, or a percentage of the total.

Width is counted in **cells**, the columns a character occupies on a terminal.
This is neither the byte count nor the character count.

| Text | Bytes | Characters | Cells |
| --- | --- | --- | --- |
| `hello` | 5 | 5 | 5 |
| `世界` | 6 | 2 | 4 |
| `"\e[31mERR\e[0m"` | 14 | 14 | 3 |

**Solving** a table means turning the constraints and a total width into one
number per column. The solver is an allocation, not a constraint system. Fixed
columns take what they ask for. Minimums are met. Percentages are computed
against the total. What is left is shared among the automatic columns in
proportion to their natural widths, and the remainder goes to the widest.

A **style** is the glyphs drawn between the cells, plus the answers to which
rules are drawn and how much padding a cell gets. Seven presets are supplied and
a caller can build its own.

| Preset | What it is for |
| --- | --- |
| `ascii` | A log file, or a terminal that predates UTF-8 |
| `unicode_box`, `rounded`, `heavy` | A modern terminal; every glyph is one cell |
| `markdown` | A markdown document |
| `plain` | A bare listing, like `ls -l` |
| `header_rule` | Help output and package listings |

## Install

```
novo pkg add table-nv
```

## Example

```novo
use tblmodel
use tblstyle
use tblwidth
use tblrender
use tblexport

fn main() [io]
    // Three columns, each as wide as its content until a constraint says otherwise.
    let empty = tblmodel.with_headers(["package", "version", "status"])

    // One row of already-formatted text. It must hold one cell per column.
    match tblmodel.push_row(empty, ["table-nv", "0.0.2", "interface"])
        Err(e)   => println(e.message())
        Ok(rows) =>
            // Unicode box drawing instead of the ASCII default.
            let t = tblmodel.styled(rows, tblstyle.unicode_box())

            // The rule that counts a character's width in terminal cells.
            let w = tblwidth.rule()

            // One width per column, for a terminal 80 cells wide.
            match tblwidth.solve(t, 80, w)
                Err(e)     => println(e.message())
                Ok(solved) =>
                    // The table as a person sees it.
                    println(tblrender.render(t, solved, w))

                    // The same columns again, quoted for a spreadsheet.
                    println(tblexport.to_csv(t))
```

Build and test with `novo pkg build` and `novo test`. Today `novo test` fails on
purpose: every test reaches a `not implemented` panic.

## What the package contains

| Module | Contents |
| --- | --- |
| `tblmodel` | The table as a value: its columns, its rows, its typed cells, its width constraints, and the refusals it answers. |
| `tblwidth` | How wide a piece of text is in cells, how a cell is padded, truncated and wrapped, and the solver that turns constraints into one width per column. |
| `tblstyle` | The eleven glyphs and the flags that make up a border, with seven presets built from them. |
| `tblrender` | The table as lines: the whole document, a list of lines, one line by index, or bytes into a caller's buffer. |
| `tblexport` | The same columns as comma-separated values, tab-separated values, a caller-chosen delimiter, or a markdown table. |

## How to choose an entry point

**A program printing one table calls `tblrender.render`.** Build the table,
solve it, render it, and write the string wherever the output goes.

**A program printing a very large table calls `tblrender.render_lines` or
`tblrender.render_line`.** `render` builds the whole document before the caller
sees a byte of it. `render_line` answers line 400 without the 399 before it,
which is what a pager and a scrolling full-screen program need.

**A program that redraws often calls `tblrender.render_into`.** It appends into
a buffer the caller owns and sizes with `tblrender.render_len`, so a table
redrawn many times allocates once.

**A program drawing its own frame calls the parts.** `tblrender.top_rule`,
`.header_rule`, `.mid_rule`, `.bottom_rule`, `.render_row` and `.render_cell`
are each public, so a full-screen layout can place them itself.

**A program writing a file calls `tblexport`.** Use `to_csv` for a whole small
table, and `header_record` with `csv_record` per row for a large one, so nothing
holds the whole document.

## The rules a user needs

1. **Columns are counted in cells.** The rule is unicode-nv's character width:
   East Asian Width for the wide forms, the Mn, Me and Cf categories for the
   zero-width ones, and the emoji presentation rules for sequences that join.
   Unicode Standard Annex #11.
2. **`tblwidth.rule_cjk` counts East Asian Ambiguous characters as two cells.**
   That is a terminal the user configured for East Asian text. It is a start-up
   decision and never changes mid-line.
3. **An ANSI escape costs no width.** `tblwidth.measure` recognises escapes with
   ansi-nv's parser rather than a pattern, so an OSC string containing a
   semicolon, a CSI with intermediates and a bare escape byte are each measured
   as what a terminal would show.
4. **A truncation closes the styles it cut open.** Cutting `"\e[31mERROR"` at
   three cells would otherwise leave the terminal red for the rest of the line
   and every row after it. `tblwidth.truncate` appends the reset.
5. **A wrapped cell re-opens its style on each line**, so a wrapped coloured
   cell is coloured on every line rather than only the first.
6. **An export strips every escape.** A colour sequence in a CSV field breaks
   whatever reads the file, and there is no terminal at the other end.
7. **Every row holds one cell per column.** `tblmodel.push_row` refuses a row
   that does not, naming the expected count and the count found. A ragged table
   otherwise renders with a hole in it, three calls away from the mistake.
8. **The total width is an argument, never a terminal query.** Read the real
   width once in your program and pass it in. Passing `0`, or calling
   `tblwidth.solve_natural`, gives the content's natural width.
9. **Padding and rules count toward the total.** A column solved to eight cells
   inside a border with one space each side occupies ten.
   `tblstyle.overhead_total` is that arithmetic, and getting it wrong is how a
   table comes out one character too wide and wraps every line.
10. **The solve can refuse.** `TableTooNarrow` carries what the fixed and
    minimum columns plus the borders already need, and what the caller offered.
    Nothing is squeezed to zero, because a zero-wide column renders as a stack
    of ellipses.
11. **A percentage constraint must be between 1 and 100**, and it needs a total
    to be a percentage of. `TableBadPercent` is the first refusal and a total of
    `0` is the second.
12. **Solve once, render many times.** `tblrender` takes the solved widths
    rather than computing them, so rendering the same table twice costs one
    solve, and `render_line` is cheap.
13. **A cell's text is never reinterpreted.** The kind sets the default
    alignment and gives `tblmodel.sorted_by` something to compare.
    `sorted_by` compares numbers and times on their carried values and text
    bytewise. It is not a locale-aware collation.
14. **A cell marked no-wrap overflows its column rather than wrapping.** That is
    what a URL or a hash wants, because breaking one across two lines makes it
    unselectable. `tblmodel.cell_nowrap` sets it.
15. **A hidden column keeps its cells.** It is excluded from the solve, from the
    render and from `tblexport.to_markdown`, and toggling it back needs no
    rebuild.
16. **Separator rows are skipped by every export.** A rule is a visual device,
    and a blank line in a CSV is a record somebody's parser will invent.
17. **CSV quoting follows RFC 4180.** A field is quoted when it holds the
    delimiter, a quote, a carriage return or a newline, and a quote inside a
    quoted field is doubled. RFC 4180 section 2.
18. **TSV has no quoting at all.** `tblexport.to_tsv` replaces a tab or a
    newline inside a cell with a space. Call `tblexport.tsv_is_lossless` first
    to find out whether that will happen.
19. **`tblrender.render_len` is in bytes, not cells.** A table eighty cells wide
    with CJK in it is more than eighty bytes per line.
20. **`tblrender.render_line` answers nothing past the last line**, so a caller
    looping until it runs out has a terminator, and a blank row is
    distinguishable from the end.

## What is not included

- **Cells that span columns.** A span turns the width solver into a constraint
  system and makes a CSV export ambiguous. comfy-table has spans and this
  package does not, which is worth knowing before you depend on it.
- **Asking the terminal how wide it is.** That performs input and output, and it
  is the program's question. Keeping it out is what lets one table be solved for
  a screen, a file and a test fixture.
- **Printing.** Every call answers text or writes into a buffer the caller owns.
- **Choosing colours.** A cell's text arrives with whatever escapes the caller
  put in it, and they are measured as nothing and carried through unchanged.
- **Writing a file.** Writing performs input and output, and every function here
  declares no effects.
- **JSON and HTML export.** JSON is a shape decision, an array of objects or an
  array of arrays, with the header somewhere. That belongs to the caller with
  `std.json`. HTML is an escaping problem and a template.
- **Locale-aware sorting.** `tblmodel.sorted_by` compares text bytewise. A
  caller that needs a collation sorts its own data before building the table.
- **Running on a microcontroller.** No module claims it. Every value here is a
  list of rows, cells or lines, and a table built with no heap is not a table.

## Related packages

- [unicode-nv](https://novo-lang.org/packages/unicode-nv) supplies the character
  width this package measures with. The data is the Unicode character database,
  which nothing should transcribe twice.
- [textwrap-nv](https://novo-lang.org/packages/textwrap-nv) wraps the text
  inside a cell. Its width rule is one function from a character to a cell
  count, and `tblwidth.rule` builds one from unicode-nv's.
- [ansi-nv](https://novo-lang.org/packages/ansi-nv) supplies the escape-sequence
  parser that tells bytes and cells apart in a coloured string.
- [progress-nv](https://novo-lang.org/packages/progress-nv) draws progress bars
  on the same terminal under the same width rule, so a bar and a table cannot
  come to different answers about how wide a string is.
- `std.csv` in the standard library reads and writes comma-separated values.
  `tblexport` uses the same RFC 4180 quoting, so a table exported here reads
  back there.
- `std.cli` in the standard library parses command-line arguments and prints
  help. Its listings are the shape `tblstyle.header_rule` draws.

## Tests

```bash
novo test tests                          # every suite
novo test tests/tblmodel_tests.nv        # the table as a value, and its refusals
novo test tests/tblwidth_tests.nv        # cells, escapes and the solver
novo test tests/tblrender_tests.nv       # the lines, the presets and the exports
```

`novo test` fails on purpose today. Every assertion reaches a `not implemented:
table-nv.<module>.<fn>` panic, because every body is a `todo()`. The tests are
the specification the implementation will have to satisfy.

The expected cell widths come from the Unicode character database through
unicode-nv. The expected CSV records follow RFC 4180. The border glyphs and the
alignment defaults are comfy-table's, tabled's, tabulate's and rich's.

The rendering assertions are whole lines, as a person would see them. A
renderer's mistakes are off-by-one mistakes, and an assertion about a line's
length would pass for a table with the padding on the wrong side. The first
assertion in the suite is that `世界` measures four cells, which is the reason
the package exists.

## Implementation status

| Item | Implemented |
| --- | --- |
| `tblmodel.TableCellKind`, `.TableCell`, `.TableRow`, `.TableAlign`, `.TableVAlign` | declared |
| `tblmodel.TableColumn`, `.TableWidth`, `.Table`, `.TableError`, `impl Error for TableError` | declared |
| `tblmodel.table`, `.with_headers`, `.push_column`, `.column`, `.aligned`, `.constrained` | no |
| `tblmodel.push_row`, `.push_cells`, `.push_separator` | no |
| `tblmodel.cell`, `.number_cell`, `.time_cell`, `.empty_cell`, `.cell_aligned`, `.cell_nowrap` | no |
| `tblmodel.column_count`, `.row_count`, `.cell_at`, `.set_cell`, `.set_column`, `.set_hidden` | no |
| `tblmodel.styled`, `.sorted_by`, `.check` | no |
| `tblwidth.TableSolved` | declared |
| `tblwidth.rule`, `.rule_cjk` | no |
| `tblwidth.measure`, `.measure_plain`, `.has_escapes`, `.strip_escapes` | no |
| `tblwidth.fit_prefix`, `.truncate`, `.pad`, `.wrap_cell`, `.cell_height` | no |
| `tblwidth.natural_width`, `.minimum_width`, `.minimum_total` | no |
| `tblwidth.solve`, `.solve_natural`, `.row_height`, `.total_height` | no |
| `tblstyle.TableBorder`, `.TableStyle` | declared |
| `tblstyle.ascii`, `.unicode_box`, `.rounded`, `.heavy`, `.markdown`, `.plain`, `.header_rule` | no |
| `tblstyle.padded`, `.with_row_rules`, `.with_outer`, `.with_empty`, `.with_ellipsis`, `.border` | no |
| `tblstyle.overhead_per_column`, `.overhead_outer`, `.overhead_total`, `.draws_rules` | no |
| `tblrender.render`, `.render_lines`, `.render_line`, `.line_count` | no |
| `tblrender.render_into`, `.render_len` | no |
| `tblrender.top_rule`, `.header_rule`, `.mid_rule`, `.bottom_rule` | no |
| `tblrender.render_row`, `.render_header`, `.render_cell`, `.effective_align` | no |
| `tblexport.to_csv`, `.to_tsv`, `.tsv_is_lossless`, `.to_delimited`, `.to_markdown` | no |
| `tblexport.to_csv_into`, `.csv_len`, `.csv_record`, `.header_record` | no |
| `tblexport.quote_field`, `.needs_quoting` | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
