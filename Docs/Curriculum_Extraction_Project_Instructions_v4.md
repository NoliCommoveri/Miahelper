# Curriculum Extraction Project Instructions v4

## Purpose

This specification defines the required workflow for extracting curriculum data
and generating Miacademy activity count workbooks. Follow these instructions
exactly.

> **What changed from v3:** the counts sheet you produce is now joined downstream
> against a **PDF page map** on the `Unit Name` + `Lesson Name` pair. Those names
> are a **join key**, so exact, consistent spelling matters more than ever
> (see NAME-001). Also: **do not add a PDF row** — a later app inserts it
> automatically from the page map (see OUT-002). The schema, counts, and order
> rules are otherwise unchanged.

------------------------------------------------------------------------

# REQ-001 Required Inputs

Before generating any workbook, obtain:

-   Course Name
-   Video activity count
-   Practice Level activity count
-   Quiz activity count

Never infer required inputs.

## Default Activity Counts

Unless the user explicitly overrides them:

-   Video = 1
-   Practice Level = 4
-   Quiz = 1

------------------------------------------------------------------------

# WF-001 Workflow

Follow these steps in order.

1.  Obtain required inputs.
2.  Extract every visible unit.
3.  Extract every visible lesson.
4.  Validate the extraction.
5.  Generate the workbook.
6.  Validate the workbook.

Do not combine or skip steps.

If any required information is missing, stop and ask one concise question.

------------------------------------------------------------------------

# EXT-001 Extraction Rules

Extract every visible lesson regardless of status.

Include:

-   Completed lessons
-   Unchecked lessons
-   Locked lessons
-   Assessments
-   End-of-course assessments
-   Final exams

Completion status is ignored.

Maintain the exact order shown in the curriculum.

Never alphabetize or reorder lessons.

> Order is load-bearing downstream: the PDF page map is reconciled to your sheet
> **by position**, so a reordered lesson misaligns the page ranges. Preserve the
> on-screen order precisely.

Extraction sources may include:

-   Screenshots
-   Screen recordings

Lessons may span multiple screenshots.

Merge screenshots before extracting.

Never duplicate lessons.

------------------------------------------------------------------------

# NAME-001 Lesson-Name Consistency (join key)

`Unit Name` and `Lesson Name` are used downstream as a **join key** against a
separate PDF page map. To keep the join reliable:

-   Use the **exact on-screen spelling, casing, and punctuation** of each unit
    and lesson. Do not abbreviate, expand, or "clean up" names.
-   Preserve internal punctuation exactly, including colons and commas
    (e.g. `Egypt: Old Kingdom`, `Summarizing Our Learning, Part I`).
-   The `Unit Name` and `Lesson Name` values must be **byte-for-byte identical**
    across all three rows of a lesson. No trailing spaces, no stray variants.
-   Name assessments consistently with what is shown (e.g. `Unit 3: Assessment`).
    Do not invent a unit prefix that is not on screen.

If a name is genuinely illegible in the source, stop and ask rather than
guessing — a wrong name breaks the downstream page-range match silently.

------------------------------------------------------------------------

# OUT-001 Workbook Schema

Filename:

`Mia_[Course Name]_Counts.xlsx`

Columns (exact order):

1.  Unit Name
2.  Lesson Name
3.  Activity Type
4.  Activity Count

Do not add additional columns.

------------------------------------------------------------------------

# OUT-002 Activity Types

Each lesson produces exactly three rows.

The order is fixed.

1.  Video
2.  Practice Level
3.  Quiz

The Activity Type column is an enumeration. Only these values are permitted:

-   Video
-   Practice Level
-   Quiz

Do not place lesson names into the Activity Type column.

Do not create an Activity Name column.

**Do NOT add a `PDF` row or a PDF placeholder row.** The PDF row (with its page
range) is inserted automatically by the downstream app from the page map. Adding
one here would create a duplicate. Emit only the three activity rows above.

------------------------------------------------------------------------

# VAL-001 Validation Checklist

Before returning the workbook verify:

-   Every lesson has exactly three rows.
-   Activity order is Video → Practice Level → Quiz.
-   Activity counts match the requested/default values.
-   Activity Type contains only Video, Practice Level, or Quiz (no `PDF`).
-   `Unit Name` / `Lesson Name` are identical across a lesson's three rows and
    match the on-screen spelling (NAME-001).
-   No duplicate lessons exist.
-   All visible lessons were extracted.
-   Column headers exactly match specification.
-   Filename follows `Mia_[Course Name]_Counts.xlsx`.

------------------------------------------------------------------------

# FAIL-001 Common Failure Modes

Do NOT:

-   Infer the course name.
-   Infer activity counts (except documented defaults).
-   Omit unchecked lessons.
-   Omit assessments.
-   Reorder lessons.
-   Create an Activity Name column.
-   Add a PDF row or PDF placeholder.
-   Abbreviate or alter unit/lesson names.
-   Populate Activity Type with lesson names.
-   Generate output before required inputs are known.
-   Rename the output file.
-   Skip validation.

------------------------------------------------------------------------

# GUIDING PRINCIPLE

When the instructions and the user request appear to conflict:

-   Follow this specification.
-   Do not optimize.
-   Do not simplify.
-   Do not infer missing information.
-   Produce deterministic output.
