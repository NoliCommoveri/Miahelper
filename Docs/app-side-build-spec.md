# Build Spec: PDF-Row Excel Updater (consumes map-csv-v1)

**Contract version:** map-csv-v1 (must match the producer file
`session-side-trim-instructions.md`)
**For:** a Claude Code build.
**What this builds:** a small, deterministic command-line tool that reads a
lesson→page map (produced by the session side) and an existing curriculum Excel
workbook, and inserts one **PDF** row under each lesson's **Video** row, giving
that lesson's page range and page count in the trimmed PDF. Output is an updated
copy of the workbook. Nothing else changes.

This is the "everything else" half of a two-part workflow. The hard/semantic
work (footer reading, lesson reconciliation, deciding what to delete) already
happened session-side and is baked into the map. This half is pure, testable
bookkeeping. Keep it that way — do not add intelligence here.

---

## Key simplification: this tool does NOT open the PDF

All page information the tool needs is already in the map CSV (`trimmed_page`
per lesson). The trimmed PDF is a deliverable for the parent to read; this tool
never parses it. **No PDF library dependency.** (Optional: a `--verify-pdf`
flag may open the trimmed PDF solely to assert `max(trimmed_page) == page
count`; keep this optional and off by default so the core tool stays
dependency-light.)

---

## Inputs / Outputs

**Inputs**
- `--map PATH` — the `*-pdf_page_map.csv` file (map-csv-v1).
- `--xlsx PATH` — the reference workbook to update (`.xlsx`; support `.xlsm`
  via `keep_vba=True` if encountered).

**Output**
- A **new** workbook written to `--out PATH` (default:
  `<xlsx-stem>-with-pdf-rows.xlsx`). Never edit the input workbook in place —
  always write a copy, so a bad run is non-destructive.

---

## Tech constraints

- Python 3.11+.
- **Excel: `openpyxl` only.** Do NOT round-trip the workbook through pandas or
  any tool that rewrites the file wholesale — it will destroy the existing
  formatting, merged cells, and styling this spec is required to preserve. Load
  with `openpyxl.load_workbook(path)`, edit the worksheet objects, save.
- Map CSV: standard `csv` module or pandas is fine (read-only, small).
- No network. Fully offline.
- Deterministic: same inputs → byte-stable logical output (ordering fixed).

**Known openpyxl caveats to handle, not ignore:**
- `ws.insert_rows(idx)` shifts rows down but does **not** copy styles into the
  new blank row, and does **not** fix references inside formulas or update
  merged-cell ranges reliably. You must copy styling yourself (see §Row
  insertion) and handle merged cells in the affected block.
- Saving can drop charts/images and some conditional formatting even on an
  edit-only pass. Emit a warning telling the user to eyeball the output if the
  source workbook contained charts/images.

---

## map-csv-v1 consumer contract

The tool must accept exactly the file the producer emits:

- One row **per surviving page** of the trimmed PDF. Header row present.
- Fixed leading columns, in order:
  - `trimmed_page` — 1-based page number in the trimmed PDF.
  - `original_page` — 1-based page in the original PDF (audit only; the tool
    does not use it for logic, but must not choke on it).
- Followed by the **lesson-identifier columns**, whose header names and values
  are copied verbatim from the workbook's own lesson-block identifier columns
  (commonly `course`, `unit`, `lesson`, but read them from the CSV header — do
  not hardcode the names).
- `trimmed_page` values are contiguous 1..N across the whole file with no gaps.

**Validation on load (fail loud, see §Errors):**
- Header has `trimmed_page` and `original_page` as the first two columns.
- At least one identifier column follows them.
- `trimmed_page` is 1..N contiguous, no duplicates, no gaps.
- Every row has non-empty identifier values (a blank identifier means the
  producer flagged an unresolved page; stop and report it rather than guessing).

The set of identifier column names in the CSV is the **join key**. The tool must
find those same-named identifier columns in the workbook (case-insensitive,
trimmed header text) and match on the **full tuple of their values**, never on a
single bare name. If it can't locate all identifier columns in the workbook,
fail loud.

---

## Workbook structure assumptions + required inspection

The workbook already contains, per lesson, a block of rows including a row
labelled **Video** in some label column (e.g. a `Type` column). Layout varies
between workbooks, so the tool must **inspect and confirm, not assume**:

1. **Label column** — the column that contains the literal value `Video`
   (case-insensitive) once per lesson block. Detect it by scanning columns for
   `Video` occurrences; the column with one `Video` per lesson block is it. If
   ambiguous, require `--label-col` and error asking for it.
2. **Identifier columns** — matched by header name to the CSV's identifier
   columns.
3. **Range/count target columns** — where the PDF row writes its payload.
   Default: mirror the columns the `Video` row uses for its own payload. If that
   can't be inferred unambiguously, require `--range-col` and `--count-col`.

The tool must have a **`--dry-run` mode (make it the recommended first pass)**
that prints what it detected — label column, identifier columns, range/count
target columns, the number of lesson blocks found, and the full list of rows it
*would* insert (lesson tuple → range, count, target row index) — and writes
nothing. Only a real run mutates.

---

## Core algorithm

1. Load and validate the map CSV (§consumer contract).
2. Load the workbook; select the target worksheet (default active; allow
   `--sheet`).
3. Detect columns and lesson blocks (§inspection). Build, for each lesson block,
   its identifier tuple and the row index of its `Video` row.
4. **Group the map by identifier tuple.** For each group:
   - Sort its `trimmed_page` values ascending.
   - Build the **page range string** (§range building).
   - **Page count = number of pages in the group** (`len(pages)`), NOT
     `end − start + 1`. These differ when a lesson is non-contiguous; the count
     of actual pages is the correct value.
   - Record a **non-contiguous flag** if the pages have any gap.
5. **Match** each map group to its workbook lesson block by full identifier
   tuple. Handle mismatches per §Edge cases.
6. For each matched lesson, **insert a PDF row immediately below its Video row**
   (§row insertion), setting:
   - identifier columns = copied from the Video row's own cells (same values),
   - label column = `PDF` (mirroring however `Video` is labelled),
   - range column = the page range string,
   - count column = the page count integer.
   Insertions shift subsequent row indices — process blocks **bottom-to-top**
   (descending Video-row index) so earlier insertions don't invalidate the
   indices of blocks not yet processed.
7. Save to `--out`. Print a summary (§Summary output).

---

## Range building (reference logic — adapt as needed)

Input: sorted unique ints. Output: `"12–14, 17"` style string.
Separator between endpoints: en dash `–` (U+2013). Join between disjoint runs:
`, `. A single page renders as just that number (`"17"`).

```python
def build_range(pages: list[int], dash: str = "–") -> tuple[str, bool]:
    pages = sorted(set(pages))
    runs, start, prev = [], pages[0], pages[0]
    for p in pages[1:]:
        if p == prev + 1:
            prev = p
        else:
            runs.append((start, prev)); start = prev = p
    runs.append((start, prev))
    noncontig = len(runs) > 1
    parts = [str(a) if a == b else f"{a}{dash}{b}" for a, b in runs]
    return ", ".join(parts), noncontig
```

Make `dash` configurable via `--range-sep` (default `–`) in case the workbook
convention differs, but keep it consistent within a run.

---

## Row insertion (the fiddly part — get styling right)

`insert_rows` alone leaves an unstyled blank row. Use the **Video row as a
structural template**: insert, then copy styles cell-by-cell from the Video row
to the new row, then overwrite the specific cells (label, identifiers, range,
count). Copy row height. Check merged cells in the block.

```python
from copy import copy

def insert_pdf_row(ws, video_row_idx, col_map, id_values, range_str, count):
    new_idx = video_row_idx + 1
    ws.insert_rows(new_idx)
    # copy formatting + row height from the Video row
    for col in range(1, ws.max_column + 1):
        src = ws.cell(row=video_row_idx, column=col)
        dst = ws.cell(row=new_idx, column=col)
        if src.has_style:
            dst.font          = copy(src.font)
            dst.border        = copy(src.border)
            dst.fill          = copy(src.fill)
            dst.number_format = src.number_format
            dst.protection    = copy(src.protection)
            dst.alignment     = copy(src.alignment)
    if ws.row_dimensions[video_row_idx].height is not None:
        ws.row_dimensions[new_idx].height = ws.row_dimensions[video_row_idx].height
    # write values
    for col_name, col_idx in col_map["identifiers"].items():
        ws.cell(row=new_idx, column=col_idx, value=id_values[col_name])
    ws.cell(row=new_idx, column=col_map["label"], value="PDF")
    ws.cell(row=new_idx, column=col_map["range"], value=range_str)
    ws.cell(row=new_idx, column=col_map["count"], value=count)
```

**Merged cells:** if the Video row (or the block) contains horizontally merged
ranges that should also apply to the PDF row, replicate the merge on the new row
after insertion. If merges span vertically across the Video row, inserting below
it is usually safe, but detect any merged range that intersects the insertion
point and log a warning rather than silently corrupting it. Do not assume no
merges exist — inspect.

---

## Edge cases (handle each explicitly)

| Case | Handling |
|------|----------|
| Map group whose tuple matches **no** workbook block | Collect all; fail loud at end, listing every unmatched tuple. Do not partially write. |
| Workbook lesson block with **no** map pages (e.g. video-only lesson, or fully-trimmed) | Insert no PDF row; add to a "no-PDF-pages" flag list in the summary. Do not insert an empty row by default. |
| **Duplicate** identifier tuple across two workbook blocks (ambiguous match) | Fail loud, name the colliding tuple. This is the contract's core assumption; surface it, don't guess. |
| Lesson **non-contiguous** in the map | Allowed. Emit the multi-run range (e.g. `12–14, 17`) and count = actual page count. Add to the summary's non-contiguous flag list. |
| A block has **no Video row** but has map pages | Fail loud, name the lesson. The insertion anchor is missing; a human must resolve. |
| Identifier column present in CSV but **absent** in workbook | Fail loud before any write. |
| `.xlsm` input | Load with `keep_vba=True`; preserve macros on save. |
| Workbook contains charts/images | Proceed, but warn that openpyxl may drop them; advise checking the output. |

**Failure policy:** any fatal condition aborts the entire run **before writing
output**. Never produce a half-updated workbook.

---

## Validation / errors

- Validate the map fully before touching the workbook.
- Do all matching and detection, and collect *all* fatal issues, before any
  mutation. Report them together (don't fail on the first and hide the rest).
- On success, before saving, assert: exactly one PDF row inserted per matched
  lesson; each inserted row's identifier cells equal its Video row's; count
  equals `len(group)`.
- Exit non-zero on any fatal error; print actionable messages (which lesson,
  which column, what was expected).

---

## Summary output (print on every run, dry or real)

- Detected: label column, identifier columns, range/count columns, sheet, N
  lesson blocks, M map groups.
- Inserted (or would-insert, in dry-run): count of PDF rows.
- **Flags to spot-check**, itemised:
  - Non-contiguous lessons (name + range).
  - Lessons with no PDF pages (skipped).
  - Any charts/images-present warning.
- Output path.

---

## CLI

```
pdfrows --map MAP.csv --xlsx BOOK.xlsx [--out OUT.xlsx] [--sheet NAME]
        [--label-col NAME] [--range-col NAME] [--count-col NAME]
        [--range-sep "–"] [--dry-run] [--verify-pdf TRIMMED.pdf]
```

`--dry-run` is the recommended first pass and mutates nothing.

---

## Testing (build these fixtures)

Create a synthetic workbook fixture and a matching map, and assert end-to-end.
Minimum fixture: 1 course, 2 units, 2 lessons each (4 lessons), each lesson
block having identifier columns + a `Video` row with some payload + at least one
other row, plus surrounding formatting (a fill color, a border, a set row
height) so style-preservation is actually testable.

Required test cases:
1. **Happy path** — every lesson contiguous; assert one PDF row under each Video
   row, correct range, correct count, and that the new row inherited the Video
   row's fill/border/height.
2. **Non-contiguous lesson** — map gives a lesson pages `12,13,14,17`; assert
   range `"12–14, 17"`, count `4` (not `6`), and non-contiguous flag raised.
3. **Single-page lesson** — assert range renders as bare `"17"`, count `1`.
4. **Unmatched map tuple** — assert the run aborts before writing and names the
   tuple.
5. **Duplicate workbook tuple** — assert abort + report.
6. **Lesson with no map pages** — assert no row inserted + flagged in summary.
7. **Bottom-to-top insertion** — assert that inserting into an earlier block
   didn't shift a later block's target (i.e. all 4 rows land correctly in one
   run).
8. **Idempotence guard (optional)** — running twice on the same output should
   either detect existing PDF rows and refuse, or be explicitly documented as
   not-idempotent. Pick one and test it.

---

## Scope boundaries — do NOT

- Do not open, trim, or renumber the PDF (the map already carries page numbers).
- Do not read PDF footers or do any lesson-name reconciliation — the map's
  identifier values are authoritative; use them verbatim.
- Do not recompute or "correct" the trimmed page numbers.
- Do not edit the input workbook in place; always write a copy.
- Do not reorder, restyle, or otherwise touch rows other than the PDF rows you
  insert.
- Do not add columns beyond writing into the detected label/range/count columns.

Deliverable: one updated workbook copy with exactly the PDF rows added.
