# tablecat Technical Design (v1)

Implementation target for v1. Detailed enough to delegate to an implementation
agent. Derived from [product-requirements.md](./product-requirements.md); when
in doubt, the PRD wins.

## 1. Language, Runtime, Distribution

| Item | Decision |
|---|---|
| Language | **Go** (>= 1.22) |
| Module path | `github.com/Saber5656/tablecat` (adjust if the remote differs) |
| CLI flags | `github.com/spf13/pflag` (GNU-style, interspersed args; no subcommands) |
| HTML parsing | `golang.org/x/net/html` |
| CSV/TSV | stdlib `encoding/csv` |
| JSON | stdlib `encoding/json` |
| Distribution | Single static binary; cross-compile via `GOOS/GOARCH` |

Rationale: single-binary distribution, strong stdlib for csv/json, easy
cross-compile. A future macOS-native GUI (aligned with GridSelect's Swift
direction) attaches as a **separate frontend that invokes this binary or the
Go library** — no coupling required.

## 2. Package Layout

Core is a **CLI-independent importable library** (PRD FR10). CLI is a thin
frontend under `cmd/`.

```
tablecat/
├── go.mod
├── cmd/tablecat/
│   └── main.go            # flag parsing, IO, wiring; exit codes
├── table/                 # public core library (import: .../tablecat/table)
│   ├── model.go           # Table, Column, Type, Format
│   ├── sniff.go           # Sniff(data) Format
│   ├── parse.go           # Parse(data, opts); parser dispatch + interface
│   ├── parse_csv.go       # CSV + TSV (delimiter param)
│   ├── parse_json.go
│   ├── parse_markdown.go
│   ├── parse_fwf.go
│   ├── parse_html.go
│   ├── render.go          # Render(t, format); renderer dispatch
│   ├── render_markdown.go
│   ├── render_delimited.go# CSV + TSV
│   ├── render_json.go
│   ├── concat.go          # Concat(tables)
│   └── infer.go           # InferTypes, header inference helpers
├── table/testdata/        # fixtures per format + golden outputs
└── docs/
```

## 3. Internal Model (`model.go`)

Canonical representation is a **typed table**, not Markdown. Cells are stored as
`*string` so **null is distinguished from empty string** (`nil` = null).

```go
package table

type Type int

const (
    TypeString Type = iota // default / fallback
    TypeNumber
    TypeBool
)

type Column struct {
    Name string
    Type Type
}

type Table struct {
    Columns []Column
    Rows    [][]*string // len(row) == len(Columns); nil element = null
}

type Format int

const (
    FormatUnknown Format = iota
    FormatCSV
    FormatTSV
    FormatJSON
    FormatMarkdown
    FormatFWF
    FormatHTML
)
```

- `Column.Type` is best-effort metadata used by concat matching and by the JSON
  renderer (to emit numbers/bools unquoted). Cell storage stays textual.
- Invariant: every row has exactly `len(Columns)` entries. Parsers must pad
  short rows with `nil` and never emit ragged rows.

## 4. Public Core API

```go
// Sniff detects the input format from content. Never returns an error;
// falls back to FormatFWF for tabular-looking text, FormatUnknown only for
// clearly non-tabular input (empty / binary).
func Sniff(data []byte) Format

type ParseOptions struct {
    Format   Format // FormatUnknown → sniff internally
    NoHeader bool   // true → do not treat first row as header; auto-name columns
    Select   string // "" or "1".."n" (1-based) or "all"; applies to multi-table sources
}

// Parse turns raw bytes into a single Table. For multi-table sources (HTML),
// Select picks one ("n") or unions all ("all"). Runs InferTypes before return.
func Parse(data []byte, opts ParseOptions) (*Table, error)

// Render serializes a Table to the given output format.
func Render(t *Table, format Format) ([]byte, error)

// Concat vertically concatenates tables, unioning columns by name (see §9).
// Never returns an error; a nil/empty slice yields an empty Table.
func Concat(tables []*Table) *Table

// InferTypes fills Column.Type best-effort from the current rows (§8).
func InferTypes(t *Table)
```

Internal parser contract (not exported):

```go
// parseX returns one Table per table found in the source (HTML may be >1;
// all others return exactly one). Selection/type-inference happen in Parse.
type parserFunc func(data []byte, opts ParseOptions) ([]*Table, error)
```

`Parse` flow: resolve format (opts.Format or `Sniff`) → dispatch to parserFunc →
apply `Select` (default first; `"n"` 1-based; `"all"` → `Concat`) → `InferTypes`
→ return.

## 5. Format Sniffing (`sniff.go`)

Deterministic, ordered checks on a leading sample (first ~64 KiB). First match wins.

| Order | Format | Rule |
|---|---|---|
| 1 | HTML | After trimming leading space, starts with `<` **and** a case-insensitive match for `<table` exists |
| 2 | JSON | First non-space byte is `[` or `{` **and** `json.Valid(data)` is true |
| 3 | Markdown | Some line contains `\|` **and** a separator line matches `^\s*\|?[\s:]*-{1,}[\s:|-]*\|?\s*$` (a dashed rule) |
| 4 | TSV | Sample lines each contain `\t`, and tab count per line is consistent (mode ≥ 1) |
| 5 | CSV | Sample lines contain `,` with consistent field counts (mode ≥ 1) after CSV quoting |
| 6 | FWF | Fallback for any remaining multi-line, space-containing text |
| — | Unknown | Empty input or no printable tabular structure |

Notes:
- "Consistent" = the modal delimiter count across sampled non-empty lines appears
  in ≥ 70% of lines. Prefer TSV over CSV when both look consistent and tabs exist.
- Sniff must never panic on binary; guard with `utf8.Valid` / printable checks →
  `FormatUnknown`.

## 6. Parsers

### 6.1 CSV / TSV (`parse_csv.go`)
- Use `encoding/csv` with `Comma = ','` (CSV) or `'\t'` (TSV), `FieldsPerRecord = -1`
  (tolerate ragged input), `LazyQuotes = true`.
- Header handling per §8.2. Pad short records to column count with `nil`;
  empty field → `*string` to `""` (not nil). Truly missing trailing fields → `nil`.

### 6.2 JSON (`parse_json.go`)
Accept two shapes:
- **Array of objects** `[{...}, ...]`: columns = union of keys in first-seen order
  across all objects; missing keys → `nil`; nested object/array values → compact
  JSON string; JSON `null` → `nil`. No header inference (keys are names).
- **Array of arrays** `[[...], ...]`: rows as-is; header handling per §8.2.
- A single object `{...}` → one-row table (array-of-objects with one element).
- Reject other shapes with a clear error.

### 6.3 Markdown (`parse_markdown.go`)
- Parse GFM pipe tables: split on unescaped `|`, trim cells, drop the separator
  (`---`) row. First content row is the header (structural). Unescape `\|`.
- Ignore non-table lines around the table; parse the first contiguous table
  block (further tables are out of scope for MD in v1; `Select` ignored).

### 6.4 Fixed-width / whitespace-aligned (`parse_fwf.go`) — heuristic
Column-boundary inference (best-effort, documented as such):
1. Take sample lines (all, capped at ~1000). Let `W = max line width`.
2. For each column index `c` in `[0, W)`, mark `sep[c] = true` if **every**
   sampled line has a space (or is shorter than `c+1`) at `c`.
3. Column ranges = maximal runs where `sep[c] == false`. Slicing boundaries fall
   in the `sep == true` gaps.
4. For each line, slice by ranges and `strings.TrimSpace` each cell; short line →
   trailing cells `nil`.
5. Fallbacks: if `< 2` ranges found, emit a single-column table (whole line
   trimmed). Header handling per §8.2.

### 6.5 HTML (`parse_html.go`) — multi-table
- Parse with `golang.org/x/net/html`; find all `<table>` elements in document order.
- Per table: rows from `<tr>`; cells from `<th>`/`<td>` (text content, whitespace-
  collapsed, tags stripped). If the first row is all `<th>`, it is the header;
  else header handling per §8.2. `colspan`/`rowspan` in v1: treat each cell once
  (no expansion) and pad ragged rows with `nil` — documented limitation.
- `Parse` applies `Select` across the returned tables.

## 7. Renderers

### 7.1 Markdown (`render_markdown.go`) — default
- Header row from `Columns[].Name`, then a `---` separator per column (left align).
- Cell escaping: replace `\n`/`\r` with a single space (MD tables are single-line);
  escape `|` as `\|`. `nil` → empty cell.
- Pad columns to a consistent width is **not** required (renderers may emit compact
  pipes); golden tests define the exact expected form.

### 7.2 CSV / TSV (`render_delimited.go`)
- `encoding/csv` with the matching `Comma`. `nil` → empty field (CSV/TSV cannot
  distinguish null from empty — documented lossiness). Always write a header row
  from `Columns[].Name`.

### 7.3 JSON (`render_json.go`)
- Emit an **array of objects**, keys = column names, in column order.
- Value by `Column.Type`: `TypeNumber` → numeric literal if the cell parses,
  else string; `TypeBool` → `true`/`false` if the cell parses, else string;
  `TypeString` → string. `nil` → JSON `null`.
- Pretty-print with 2-space indent.

## 8. Inference (`infer.go`)

### 8.1 Type inference (PRD FR5)
For each column, scan non-nil cells:
- All parse as int/float (`strconv.ParseFloat`) → `TypeNumber`.
- Else all in `{true,false}` case-insensitively → `TypeBool`.
- Else → `TypeString`. All-null or empty column → `TypeString`.

### 8.2 Header inference (PRD FR4)
- If `opts.NoHeader` → no header; name columns `col1..colN`.
- HTML: `<th>` row is header when present (§6.5). Markdown: first row is header.
- CSV/TSV/FWF/JSON-arrays: apply heuristic — treat row 0 as header when
  **every** cell in row 0 is non-numeric **and** at least one later row has a
  numeric cell in some column. Otherwise default header = true (most tabular
  input has headers). When header = false, auto-name `col1..colN` and keep row 0
  as data.
- JSON array-of-objects: keys are the header; the heuristic does not apply.

## 9. Concat (`concat.go`, PRD FR7)

```
Concat(tables):
  unified columns = first table's columns (order preserved)
  for each later table:
    for each column not already present (match by Name):
      append it
  for each table, for each row:
    build a new row over the unified columns; value taken by column Name,
    missing columns → nil; append to result.Rows
```

- Column type of a unified column: if all contributing tables agree on the type →
  that type; on conflict → `TypeString` (fallback).
- Header mismatch: if a table shares **zero** column names with the accumulated
  set, the CLI prints a warning to stderr (see §10) but concat still widens.
- `Concat` itself never errors; empty input → empty `*Table`.

## 10. CLI (`cmd/tablecat/main.go`)

Single command, no subcommands. Flags via pflag (interspersed args allowed).

| Flag | Type | Default | Maps to |
|---|---|---|---|
| positional args | `[]string` | — | input files; multiple → `Concat`; none → stdin |
| `-o, --to` | string | `md` | output `Format` (md/csv/tsv/json) |
| `-f, --from` | string | `auto` | forced input `Format` (auto → sniff) |
| `--select` | string | `""` | `ParseOptions.Select` (`n`/`all`) |
| `--no-header` | bool | `false` | `ParseOptions.NoHeader` |
| `--help / --version` | bool | — | usage / version string |

Flow:
1. Resolve inputs: positional files, else read all of stdin.
2. For each input: read bytes → `Parse(data, opts)` → `*Table`. On error, wrap
   with the source name.
3. `Concat` all tables (single input → itself). Emit header-mismatch warnings to
   stderr.
4. `Render(final, outFormat)` → stdout.
5. Exit codes: **0** success; **1** on any detect/parse/render/IO error, with a
   message to **stderr** in the form `tablecat: <source>: <reason>; <remedy>`
   (PRD FR9). Example remedy: "could not detect format; pass -f csv|tsv|json|md|fwf|html".

Format name parsing (`--to`/`--from`): accept `md`/`markdown`, `csv`, `tsv`,
`json`, `fwf`, `html`; unknown → error listing valid names.

## 11. Errors

- Library returns `error` (wrapped with `%w`); never `os.Exit` or print inside
  `table/`. All user-facing messaging and exit codes live in `cmd/tablecat`.
- Parsers fail fast with actionable messages (what was expected, how to override
  with `-f`).

## 12. QA Gate / Acceptance Criteria

Every FR must have a test. Ship only when all pass.

| FR | Test |
|---|---|
| FR1 | Sniff unit tests: one fixture per format detected correctly with no extension; ambiguous cases resolve per §5 order |
| FR2 | Parse golden tests: fixture → expected `Table` for CSV/TSV/JSON(both shapes)/MD/fwf/HTML |
| FR3 | Render golden tests: `Table` → expected bytes for md/csv/tsv/json |
| FR4 | Header inference: with-header, no-header (`--no-header`), and heuristic (numeric-diff) cases |
| FR5 | Type inference: number / bool / string / mixed→string / all-null→string columns |
| FR6 | HTML with 3 tables: `--select 2` picks the 2nd; `--select all` unions all |
| FR7 | Concat: same schema; disjoint columns (null-fill); partial overlap; type conflict→string; zero-overlap warning |
| FR8 | stdin→stdout round trip (`cat x.csv \| tablecat`) |
| FR9 | Undetectable/garbage input exits 1 with a stderr message naming `-f` |
| FR10 | `table` package has zero imports from `cmd/` and builds/tests standalone |

Additional:
- **Round-trip**: `csv → Table → csv` and `md → Table → md` stable on canonical fixtures.
- **Null vs empty**: JSON `null` survives to JSON output as `null`, empty string as `""`.
- Race/`go vet`/`gofmt` clean; `go test ./...` green.

## 13. Suggested Implementation Order (for delegation)

1. `model.go` + `go.mod` + skeleton packages.
2. `render_delimited.go` + `render_markdown.go` + `render_json.go` (renderers first
   — they define golden output and are dependency-free).
3. `parse_csv.go` (CSV/TSV) → smallest end-to-end path with the renderers.
4. `infer.go` (types + header) and wire into `Parse`.
5. `sniff.go`.
6. `parse_json.go`, `parse_markdown.go`.
7. `parse_fwf.go` (heuristic; most test-heavy).
8. `parse_html.go` (+ `golang.org/x/net/html`).
9. `concat.go`.
10. `cmd/tablecat/main.go` (flags, IO, exit codes, warnings).
11. Fill `testdata/` and the QA-gate tests throughout.

Each step compiles and its tests pass before the next.
