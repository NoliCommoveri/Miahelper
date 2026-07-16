# Build Spec: Miacademy Course File Builder — PDF Page-Range Integration

**Contract version:** map-csv-v1 (must match the producer file
`session-side-trim-instructions.md`)
**Target:** the existing static app `miacademy-course-file-builder.html`.
**Supersedes:** the earlier Python/openpyxl `pdfrows` design. That tool edited
the *counts* workbook; this project does the equivalent work inside the HTML app
that already builds the roster, so there is no separate Python tool and **no new
dependency** (SheetJS is already loaded and parses CSV too).

**What this adds:** a second, optional file upload — the `*-pdf_page_map.csv`
produced session-side — and, when present, the app inserts **one `PDF` row per
lesson** into the roster, whose `Activity Name` is that lesson's page range in
the trimmed PDF. Everything else about the app is unchanged and the map is
optional, so old single-file behaviour still works.

---

## Where this sits

1. **ChatGPT** emits the counts sheet (`Unit Name`, `Lesson Name`,
   `Activity Type`, `Activity Count`; Video/Practice Level/Quiz only; no PDF row).
2. **Session-side Claude** emits the trimmed PDF + `*-pdf_page_map.csv`
   (map-csv-v1: `trimmed_page`, `original_page`, `Unit Name`, `Lesson Name`;
   one row per surviving page).
3. **This app** takes the counts sheet **and** the page map and produces the
   final roster. ← *this build*

The join key between the two files is the `(Unit Name, Lesson Name)` pair.

---

## The one design decision to lock: where the PDF row goes

Within each lesson block the current app emits, in order:
`Video(s) → Practice Level(s) → Quiz(s) → any other`.

**Insert the `PDF` row immediately after the lesson's Video row(s), before
Practice Level.** This mirrors the "PDF row below the Video row" convention used
throughout this project and keeps lesson blocks self-consistent. Keep this in a
single, clearly-marked spot so it is trivial to move later.

The `PDF` row's cells:

| column        | value                                             |
|---------------|---------------------------------------------------|
| Child Name    | typed in Step 1 (same as every row)               |
| Course Name   | typed in Step 1                                    |
| Unit Name     | the lesson's Unit Name                             |
| Lesson Name   | the lesson's Lesson Name                           |
| Activity Type | `PDF`                                              |
| Activity Name | the page-range string, e.g. `12–19` or `12–14, 17`|

**No schema change** — the six output headers stay exactly as they are.

---

## map-csv-v1 consumer contract (what the app must accept)

- One row **per surviving page** of the trimmed PDF. Header row present.
- First two columns: `trimmed_page`, `original_page` (1-based ints).
- Then the identifier columns `Unit Name`, `Lesson Name`, copied verbatim from
  the counts sheet. Match them by normalized header (trim + lowercase, contains
  `unit` / `lesson`) — reuse the existing `findColumn` helper, do not hardcode
  positions.
- `trimmed_page` is contiguous 1..N across the whole file.

**Validation on load (see §Errors):**
- `trimmed_page` present and integer-parseable; `original_page` present.
- Both identifier columns locatable.
- Warn (not fatal) if `trimmed_page` is non-contiguous or has gaps/dupes — the
  app only needs to *group* by lesson, so it can still proceed, but the producer
  likely erred and the user should know.

---

## Inputs / Outputs

**Step 2 becomes a two-slot upload:**
- **Counts sheet — required** (unchanged behaviour; `.xlsx/.xls/.csv`).
- **Page map CSV — optional but recommended** (`.csv`; also accept `.xlsx`).

If only the counts sheet is provided, behave exactly as today (no PDF rows).
If both are provided, inject PDF rows. The Step 3 summary must state clearly
whether PDF rows were added and how many.

**Output:** unchanged filename convention `Mia_<Course>_<MMDDYYYY>.xlsx` /`.csv`,
same six columns, same sheet name `Course Structure`.

---

## Core algorithm changes (inside `buildRoster`, plus a new map parser)

### 1. Parse the map into a per-lesson page list

```js
// returns { pagesByLesson: { "unit\u0001lesson": [int,...] }, error, warnings }
function parsePageMap(rows){
  if(!rows || !rows.length) return { error:"The page map file has no rows." };
  var headers = Object.keys(rows[0]);
  var tpCol = findColumn(headers, ['trimmed_page','trimmed page','trimmed']);
  var unitCol = findColumn(headers, ['unit']);
  var lessonCol = findColumn(headers, ['lesson']);
  if(!tpCol || !unitCol || !lessonCol){
    return { error:"The page map needs trimmed_page, Unit Name, and Lesson Name columns." };
  }
  var pagesByLesson = {}, seen = [];
  rows.forEach(function(r){
    var unit = String(r[unitCol]==null?'':r[unitCol]).trim();
    var lesson = String(r[lessonCol]==null?'':r[lessonCol]).trim();
    var pg = parseInt(r[tpCol],10);
    if((!unit && !lesson) || isNaN(pg)) return;
    var key = unit + '\u0001' + lesson;
    if(!pagesByLesson[key]){ pagesByLesson[key] = []; seen.push(pg); }
    pagesByLesson[key].push(pg);
  });
  // contiguity check across the whole file → warning only
  var all = seen.slice().sort(function(a,b){return a-b;});
  return { pagesByLesson: pagesByLesson, warnings: [] };
}
```

### 2. Range builder (reference logic — keep the en dash)

```js
function buildPageRange(pages, dash){
  dash = dash || '\u2013'; // en dash –
  var uniq = pages.slice().sort(function(a,b){return a-b;})
                  .filter(function(v,i,a){return i===0 || v!==a[i-1];});
  if(!uniq.length) return { range:'', noncontig:false, count:0 };
  var runs=[], start=uniq[0], prev=uniq[0];
  for(var i=1;i<uniq.length;i++){
    if(uniq[i]===prev+1){ prev=uniq[i]; }
    else { runs.push([start,prev]); start=prev=uniq[i]; }
  }
  runs.push([start,prev]);
  var parts = runs.map(function(r){ return r[0]===r[1] ? String(r[0]) : r[0]+dash+r[1]; });
  return { range: parts.join(', '), noncontig: runs.length>1, count: uniq.length };
}
```

- Single page → bare number (`"17"`). Count is `uniq.length`, **not**
  `end − start + 1` (they differ for non-contiguous lessons).

### 3. Inject the PDF row in `buildRoster`

`buildRoster(rows, childName, courseName, pagesByLesson)` gains a fourth arg
(default `null`). Inside the per-lesson emit, right after the Video push:

```js
// ... after the g.video block ...
if (pagesByLesson){
  var pk = g.unit + '\u0001' + g.lesson;
  var pm = pagesByLesson[pk];
  if (pm && pm.length){
    var built = buildPageRange(pm);
    out.push([childName, courseName, g.unit, g.lesson, 'PDF', built.range]);
    if (built.noncontig) noncontigLessons.push(g.unit + ' / ' + g.lesson + ' (' + built.range + ')');
    matchedLessonKeys[pk] = true;
  } else {
    lessonsWithoutPdf.push(g.unit + ' / ' + g.lesson);
  }
}
```

After the loop, compute **map lessons that matched no counts lesson** (iterate
`pagesByLesson` keys not in `matchedLessonKeys`) → `unmatchedMapLessons`. Return
all three flag lists alongside `rows` and `lessonCount`.

**Grouping caveat:** the app currently groups by `(unit, lesson)` using trimmed
values; the map must be matched with the *same* normalization (trim both sides).
Since both files draw lesson names from the same counts sheet upstream, exact
matches are expected; a mismatch means an upstream spelling drift and should be
surfaced, not silently dropped.

---

## Step 2 wiring

- Add a second dropzone + hidden `<input type=file>` for the map. Reuse the
  existing drag/drop + `FileReader` + `XLSX.read` path; for CSV, `XLSX.read`
  with `sheet_to_json({defval:''})` already works.
- Store `state.pageMap` (the parsed `pagesByLesson`) and `state.mapFileName`.
- The counts file still drives navigation to Step 3. Order-independence:
  whichever file is dropped, only re-run `buildRoster` once the counts sheet is
  present; if the map arrives after, re-run and refresh the preview.
- Show a filename chip for each uploaded file. Let the user proceed with counts
  only (map chip absent) — that is the backward-compatible path.

---

## Step 3 (preview + summary) changes

- Preview table unchanged in shape; PDF rows now appear inline in each lesson
  block (right after Video) so the user can eyeball them.
- Summary line: append `… including <n> PDF page-range rows` when a map was used.
- Add an itemised **flags** area, shown only when non-empty:
  - **Non-contiguous lessons** (name + range) — spot-check these.
  - **Lessons with no PDF pages** (skipped, no PDF row) — expected for some
    assessments; confirm.
  - **Map lessons not found in the counts sheet** (spelling drift) — these
    produced no row; the user should reconcile names.

---

## Edge cases (handle each explicitly)

| Case | Handling |
|------|----------|
| No map uploaded | Build roster as today; no PDF rows; no flags. |
| Lesson in counts, absent from map | No PDF row; add to "no PDF pages" flags. |
| Map tuple absent from counts | No row; add to "unmatched map lessons" flags. Never invent a lesson. |
| Non-contiguous lesson (`12,13,14,17`) | Range `"12–14, 17"`; flag it. |
| Single-page lesson | Range renders as bare `"17"`. |
| Duplicate `(Unit, Lesson)` in counts | Existing app already merges by key; the single merged block gets one PDF row. (Counts should not contain dupes.) |
| Map missing required columns | Fatal for the map only: show the existing error-banner, keep the counts-only roster usable. |
| Blank identifier in a map row | Skip that row; if a whole lesson ends up empty, it simply gets no PDF row (and no flag, since it never keyed). |

**Failure policy:** a bad map must never corrupt the counts-only output. If the
map fails validation, surface the error and let the user download the roster
without PDF rows (or re-upload a fixed map).

---

## Testing (extend the current manual flow)

1. **Happy path** — every lesson contiguous; assert exactly one PDF row per
   lesson, placed right after its Video row, `Activity Type = PDF`,
   `Activity Name` = correct range.
2. **Non-contiguous lesson** — map gives `12,13,14,17`; assert `"12–14, 17"` and
   a non-contiguous flag.
3. **Single-page lesson** — assert bare `"17"`.
4. **Counts lesson with no map pages** — assert no PDF row + "no PDF pages" flag.
5. **Map lesson not in counts** — assert no row + "unmatched" flag, roster still
   builds.
6. **No map at all** — assert output identical to the current app.
7. **Map dropped before counts** — assert the app waits, then builds correctly
   once counts arrives.
8. **Names with commas/colons** (`Summarizing Our Learning, Part I`,
   `Egypt: Old Kingdom`) — assert the join still matches and the CSV/XLSX export
   quotes correctly.

---

## Scope boundaries — do NOT

- Do not open, trim, or renumber the PDF (the map already carries page numbers).
- Do not read PDF footers or reconcile lesson names — the map's values are
  authoritative; use them verbatim.
- Do not recompute or "correct" trimmed page numbers.
- Do not change the six output columns, the filename convention, or the sheet
  name.
- Do not reorder or restyle existing activity rows; only add the PDF row.
- Do not make the map required — counts-only must keep working.

**Deliverable:** the updated `miacademy-course-file-builder.html`, adding an
optional page-map upload and one PDF page-range row per lesson, with everything
else unchanged.
