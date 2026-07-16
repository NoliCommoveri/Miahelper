# Session Instructions: Trim Curriculum PDF + Emit Page Map

**Contract version:** map-csv-v1
**Your job:** the *session-side half* of a two-part workflow. You take a
curriculum PDF and its reference Excel, remove non-content pages from the PDF,
and emit two things: the **trimmed PDF** and a **page-map CSV**. A separate app
consumes those later. You do NOT edit the Excel. You do NOT build page ranges.
Stay inside the boundaries below.

Follow these steps literally. Do not improvise, optimize, or add steps. When a
step says stop and ask, stop and ask.

---

## HARD RULES (read first, these are the ones that get broken)

1. **Never delete a page without explicit user confirmation.** You propose a
   deletion list; a human approves it; only then do you delete. No silent
   deletion, ever.
2. **When unsure whether a page is content or not, KEEP it and flag it.**
   Deletion is destructive; a wrongly-kept page is a minor annoyance, a wrongly
   deleted page is lost work. Bias to keep.
3. **Lesson names in the map come from the EXCEL, never from the PDF footer.**
   Footers are abbreviated and inconsistent. The Excel is the source of truth
   for spelling and for lesson identity.
4. **Trimmed page numbers are 1-based in the NEW (post-deletion) file.** Page 1
   is the first surviving page. Do not use original page numbers as the
   `trimmed_page` value.
5. **Write the CSV with a library (Python `csv` or pandas), not by hand.**
   Lesson names contain commas and colons; hand-typed CSV will corrupt. 
6. **If either input file is missing, stop and ask for it.** Do not proceed with
   one file.
7. **Build the map only AFTER deletions are confirmed and applied**, off the
   final trimmed file.

---

## INPUTS (both required at session start)

- **PDF** — the curriculum download to trim. Lesson names appear in page footers
  (informal, may be abbreviated).
- **Excel** — already contains the Course / Unit / Lesson hierarchy with a
  "Video" row per lesson. This is the authoritative source for lesson
  names/spelling and for the order and identity of lessons.

## SKILLS TO LOAD

- PDF reading/editing: `/mnt/skills/public/pdf/SKILL.md` and, since reading is
  heavier than writing here, `/mnt/skills/public/pdf-reading/SKILL.md`.
- Excel reading: `/mnt/skills/public/xlsx/SKILL.md`.
  (You only READ the Excel. You never write to it.)

---

## STEP 1 — Inspect both files (mandatory every run; do not assume layout)

1. Read every page of the PDF and record its footer text. Build an internal
   table: `original_page → raw footer text`.
2. Open the Excel. Identify the **leading identifier columns that define a
   lesson block** — typically Course, Unit, Lesson, but confirm the actual
   column headers in this workbook, because they may differ. Record their exact
   header text; you will reuse those exact header names in the CSV.
3. Read the ordered list of lessons from the Excel (top to bottom), each with
   its full identifier tuple (e.g. Course="…", Unit="…", Lesson="…").
4. **Reconcile footer → Excel lesson by ORDER, not by string match.** Lessons
   appear in the PDF in the same order as in the Excel. Walk the PDF footers in
   order and the Excel lessons in order together, so footer "Sun & Stars"
   aligns to Excel "Lesson 1: The Sun" by position. Use fuzzy text similarity
   only as a tie-breaker, never as the primary key.
5. Produce an internal map: `original_page → (excel identifier tuple)`, using
   the Excel's spelling. If a page's lesson genuinely can't be determined,
   carry forward the previous page's lesson (lessons are contiguous blocks) and
   add it to the flag list for the human.

Do not show this raw map to the user unless asked; it is scaffolding.

---

## STEP 2 — Propose pages to delete (the one semantic judgment)

A page is a **delete candidate** only if it has no content the child uses.
Delete candidates are limited to these three kinds:

- **Cover / title pages.**
- **Section-divider pages** — just a unit name and an icon/graphic, no activity.
- **Educator-info pages** — addressed to the parent/teacher, not the child:
  teacher notes, lesson instructions to the educator, materials lists, prep
  steps, standards/objectives references, answer keys.

**Everything the child reads, looks at, or works on stays** — activities,
readings, worksheets, lesson images, questions — *even if it is the first page
of a lesson right after the educator pages.*

Signals that a page is educator-info (signals, not proof — a human confirms):
words like "Materials," "Objectives," "Standards," "Teacher," "Prep,"
"Instructions," "Answer Key," "Lesson Overview"; a page that talks *about* the
lesson rather than *presenting* it.

**Tie-breaker: if a page mixes educator text and child content, KEEP it and
flag it.** Only delete pages that are clearly and wholly non-content.

Build a candidate list. For each candidate record: `original page number`,
`kind` (cover / divider / educator), and a one-line reason.

### Confirmation checkpoint (mandatory, this is the only required human step)

Present the candidate list to the user in this format and then STOP and wait for
approval:

```
Proposed deletions (please confirm, edit, or reject):
  p3   — divider    — "Unit 2" title page, icon only, no activity
  p4   — educator   — materials list + teacher prep notes
  p9   — cover      — course cover page
  ...
Uncertain / flagged for your call:
  p12  — KEPT       — mixes a standards note with a child warm-up; kept by default
Reply with approval, or tell me which to add/remove.
```

Do not delete anything until the user replies with an approved set.

---

## STEP 3 — Build the trimmed PDF

1. Delete exactly the approved pages — no more, no fewer.
2. Save as `<original-name>-trimmed.pdf`.
3. The surviving pages are now renumbered 1..N in their existing order. This new
   numbering is what the map uses.

---

## STEP 4 — Emit the page map (map-csv-v1)

Build one CSV with **one row per surviving page** in the trimmed PDF.

**Filename:** `<original-name>-pdf_page_map.csv`

**Columns, in this exact order, with a header row:**

| column          | meaning                                                        |
|-----------------|----------------------------------------------------------------|
| `trimmed_page`  | 1-based page number in the trimmed PDF                          |
| `original_page` | 1-based page number in the original PDF (for auditing)         |
| *identifier cols* | the SAME lesson-identifier columns as the Excel, same header text, same spelling (e.g. `course`, `unit`, `lesson`) |

- The identifier columns must carry the **Excel's** values (Step 1), never
  footer text.
- Emit rows sorted by `trimmed_page` ascending, contiguous 1..N with no gaps.
- Write via a CSV library so commas/colons in names are quoted correctly.
- Do NOT compute page ranges, page counts, or contiguity here. That is the
  app's job. One flat row per page only.

**Example (columns Course/Unit/Lesson; values illustrative):**

```csv
trimmed_page,original_page,course,unit,lesson
1,5,"Astronomy","Unit 1: The Solar System","Lesson 1: The Sun"
2,6,"Astronomy","Unit 1: The Solar System","Lesson 1: The Sun"
3,7,"Astronomy","Unit 1: The Solar System","Lesson 1: The Sun"
4,10,"Astronomy","Unit 1: The Solar System","Lesson 2: The Inner Planets"
5,11,"Astronomy","Unit 1: The Solar System","Lesson 2: The Inner Planets"
```

(Note the original_page jump 7→10: pages 8–9 were deleted. The trimmed_page
stays contiguous. That is correct.)

---

## STEP 5 — Deliver

Provide both output files: the trimmed PDF and the page-map CSV.

Then give the user a short plain-language summary that flags anything they
should spot-check specifically:

- Any lesson whose pages ended up **non-contiguous** in the trimmed file (a
  lesson's pages interrupted by another lesson's) — name the lesson and the
  trimmed pages, so they know before the app flags it.
- Any page you **kept but were unsure about** (the flagged set from Step 2).
- Any lesson where the **footer→Excel match was ambiguous** and you resolved it
  by order.
- Any surviving page whose lesson you had to **carry forward** because its
  footer was unreadable.

Keep the summary to the exceptions only. Do not recap the whole document.

---

## SCOPE BOUNDARIES — do NOT do these (they belong to the app)

- Do not open, edit, or write to the Excel. You only read it.
- Do not insert the "PDF" row, compute page ranges, or count pages.
- Do not merge non-contiguous pages into a single span or "fix" ordering.
- Do not rename lessons, correct the Excel's spelling, or reorder anything.
- Do not add columns to the CSV beyond the contract above.

Your deliverables are exactly two files: the trimmed PDF and the page-map CSV.
Nothing else changes.
