# tablecat Product Requirements

## Purpose

tablecat **extracts tables from any source, independent of file extension or
vendor, and reshapes them into clean forms that people and LLMs can use.**

The primary value is **ingest (extraction)**: pulling tables out of messy,
heterogeneous sources and rendering them — by default to Markdown, and on
demand to CSV/TSV/JSON. It ships first as a CLI, with an interactive
**GUI as a formal near-future goal** for the extraction and join workflows.

As the name suggests (`table` + `cat`), **concatenation** of multiple tables is
a first-class operation.

## MVP (v1) Boundary

| Area | v1 requirement |
|---|---|
| Format detection | Detect input format by **content sniffing**, not by extension |
| Input | CSV / TSV / JSON / Markdown table / fixed-width & whitespace-aligned text (fwf) / HTML table |
| Internal model | A lightweight **typed table** `{ columns, rows }`; type inference is best-effort |
| Conversion | Convert input → output format through the internal model |
| Join | **Vertical concat** of multiple inputs (union by column name) |
| Output | Markdown (default) / CSV / TSV / JSON |
| Interface | A single-command CLI with stdin/stdout and pipe support |
| Architecture | **Separate core (lib) from the CLI frontend from day one** |

v1 succeeds when a user can, without specifying an extension, extract a table
from any supported source and render it (Markdown by default), and can
vertically concatenate multiple tables into one — all through the CLI.

## Primary Users

- Developers and writers who **extract tables from web pages, docs, or logs to
  paste into LLM prompts or Markdown documents.**
- Data practitioners who **bundle multiple mixed-format files (CSV/TSV/JSON)
  into a single table.**
- Operators who **pull columns out of aligned terminal or log text** (later via
  GridSelect).

## Core Workflows

1. Copy a web page's HTML, pipe it through `tablecat`, get the Nth table as
   Markdown, and paste it into a document.
2. Concatenate several different-format files (`a.csv b.tsv c.md`) in one
   command into a single table aligned by column name.
3. Rectangular-copy terminal output (later via GridSelect) and run
   `pbpaste | tablecat -f fwf` to infer column boundaries and structure it.
4. Convert a JSON array to spreadsheet-friendly CSV with `tablecat data.json -o csv`.

## Functional Requirements

| ID | Requirement |
|---|---|
| FR1 | **Auto-detect input format by content** (extension-independent). `-f/--from` allows explicit override. |
| FR2 | **Parse** CSV / TSV / JSON / Markdown table / fwf / HTML table into the internal model. |
| FR3 | **Render** the internal model to Markdown / CSV / TSV / JSON. Markdown is the default. |
| FR4 | **Infer whether a header row exists**; when absent, auto-name columns. `--no-header` overrides. |
| FR5 | **Infer each column's type** best-effort (distinguish string / number / boolean / null); use it for shaping and joins. |
| FR6 | When a source contains multiple tables (e.g. HTML), **select the target table** via `--select <n\|all>`. Default is the first table. |
| FR7 | **Vertically concatenate** multiple inputs aligned by column name; fill missing columns with null (union). |
| FR8 | With no input arguments, **read from stdin** and **write to stdout** (pipe-friendly). |
| FR9 | On detection/parse failure, print the **reason and remedy** to stderr and exit non-zero. |
| FR10 | Provide core functionality as a **CLI-independent library API**; the CLI is a thin frontend over it. |

## Internal Model

The canonical representation is **not** Markdown but a **typed, structured table.**

```
Table = {
  columns: [{ name: string, type?: 'string'|'number'|'boolean' }],
  rows:    Array<Array<value | null>>   // distinguishes null from empty string
}
```

- Markdown / CSV / TSV / JSON are each **one rendering** of this model.
- Retaining type information makes FR5 (shaping) and FR7 (column matching on
  concat) work.
- v1 may tolerate lossiness (the consumer is a human/LLM; Parquet-grade
  strictness is not a goal).

## Join / Concat Specification

v1's "join" is limited to **vertical concat**. True to the name `tablecat`,
`cat` = row concatenation is first-class.

| Item | v1 behavior |
|---|---|
| Default action | Concatenate inputs top-to-bottom into a single table |
| Column matching | Match by **column name** (union); columns present on only one side are null-filled on the other |
| Column order | Base on the first input's column order; append newly seen columns at the end |
| Header mismatch | If column names do not overlap at all, warn and still widen (continue union) |
| Type mismatch | If a column's inferred types conflict, fall back to string |

**Horizontal (keyed) join** is out of scope for v1; add `--join-on <key>` post-v1.

## v1 Input / Output List

| Input format | v1 | Main sniffing cues |
|---|---|---|
| CSV | ✅ | `,` delimiter, quoting |
| TSV | ✅ | `\t` delimiter |
| JSON | ✅ | Leading `[`/`{`, records or arrays |
| Markdown table | ✅ | `|` delimiters + `---` separator row |
| Fixed-width / whitespace-aligned (fwf) | ✅ | Column boundaries from runs of spaces |
| HTML table | ✅ | `<table>` elements |
| PDF | ❌ post-v1 | Layout analysis is a separate problem |
| XLSX / Parquet | ❌ post-v1 | Binary, strict-typing formats |
| On-screen text | ❌ GUI/GridSelect | Requires UI to capture |

| Output format | v1 |
|---|---|
| Markdown (default) | ✅ |
| CSV | ✅ |
| TSV | ✅ |
| JSON (records) | ✅ |
| HTML / XLSX / Parquet | ❌ post-v1 |

## CLI Design

A **single command with flags** (faithful to the README's "one command",
pipe-friendly). No subcommands.

```
tablecat [inputs...] [flags]
```

| Flag | Description | Default |
|---|---|---|
| `(positional)` | Input files; multiple inputs are vertically concatenated. Omit to read stdin | stdin |
| `-o, --to <fmt>` | Output format (md/csv/tsv/json) | `md` |
| `-f, --from <fmt>` | Force input format (override auto-detection) | auto |
| `--select <n\|all>` | Target table when a source contains multiple | first table |
| `--no-header` | Do not treat the first row as a header | auto-infer |
| `--help / --version` | Help / version | — |

Examples:

```sh
cat data.csv | tablecat                 # → Markdown to stdout
tablecat report.html --select 2 -o csv  # 2nd table of the HTML as CSV
tablecat a.csv b.tsv c.md               # concatenate 3 files into Markdown
pbpaste | tablecat -f fwf               # structure a rectangular copy (later GridSelect)
tablecat data.json -o tsv               # JSON array → TSV
```

## Architecture

Because a GUI is a formal goal, **core/frontend separation is a precondition,
not an aspiration.**

```
input(any) ──extract──▶ [core: typed table] ──render──▶ output(md/csv/tsv/json)
                            ▲
        multiple capture paths ──┘
        (CLI stdin / future tablecat-GUI / future GridSelect)
```

- **Core (lib)**: sniffing, parsing, internal model, concat, rendering. UI-independent.
- **CLI**: a thin layer calling the core; the only frontend in v1.
- **GUI (post-v1)**: interactive table selection, join-key mapping, preview.
  A separate frontend sharing the same core.

## Relationship to GridSelect

- GridSelect is a **capture frontend** that rectangular-selects on-screen
  monospace text; it does not infer structure (an explicit GridSelect non-goal).
- tablecat provides the **shared engine** that structures the raw text coming
  out of such capture.
- The two remain **separate repositories**, loosely coupled at the boundary
  (rectangular text → structuring). The only contact points are "GridSelect
  emits rectangular text" and "tablecat accepts fwf on stdin".
- No integration code in v1. The design only guarantees that a future
  tablecat-GUI and GridSelect can converge on "multiple capture paths → one
  tablecat core."

## v1 Assumptions

- Inputs are mostly regular tables; extremely malformed HTML/text is best-effort.
- fwf column-boundary inference is heuristic and may require `-f` or later
  adjustment by the user.
- v1 does not target streaming of huge data (in-memory processing is fine).

## Post-MVP Candidates

- Horizontal join (`--join-on <key>`), filtering, column selection, sorting.
- Inputs: PDF, XLSX, Parquet. Outputs: HTML, XLSX, Parquet.
- GUI frontend (interactive extraction, join-key mapping, preview).
- Deeper GridSelect integration (end-to-end rectangular capture → structuring).
- Richer type inference, including semantic types like dates and currency.

## Explicit Non-Goals

| Non-goal | Reason |
|---|---|
| OCR / image / screenshot recognition | tablecat is text-first; screen capture is GridSelect/GUI territory |
| Parquet-grade lossless data-pipeline fidelity | v1 consumers are humans/LLMs; strict type preservation is not the goal |
| DB connectivity / becoming a SQL engine | The goal is conversion and concat, not a query engine |
| Horizontal (keyed) JOIN in v1 | Get vertical concat solid first; JOIN is post-v1 |
| Extension-based format decisions | Content sniffing is core to the product |
| Styling / decoration | Beyond alignment, cosmetic tuning is out of scope |

## Verification

- Confirm v1 scope stays within "extraction + vertical concat + format
  conversion," with horizontal join, GUI, and PDF deferred or out of scope.
- Confirm non-goals have not leaked into v1 functional requirements.
- Confirm at least three concrete user workflows are described.
- Confirm the CLI satisfies the main workflows without specifying an extension
  (sniffing).
- Confirm core/CLI separation is required so a future GUI/GridSelect can attach
  to the same core.
