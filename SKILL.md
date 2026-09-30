---
name: 1901-read-idea-queue
description: Reads one design record from the live 1901 Idea Queue sheet; read-only, exact match.
---

# 1901 Read Idea Queue

Reads exactly one design record from the authoritative live 1901 Main Street
Idea Queue and returns it as one deterministic JSON object with its source
provenance. It only reads. It never writes, never interprets, and never decides
whether a design may proceed. Readiness belongs to `1901-validate-readiness`,
which may consume this skill's output; this skill never invokes it.

Core rule: **VERIFY, DON'T ASSUME.** Return only what was actually read from
the live Idea Queue in this run. Never reconstruct a value from memory, chat
history, a prior run, a filename, or inference. A value you did not read does
not exist.

## When to Use

Trigger on requests such as:

- "Read the Idea Queue record for 1901-003"
- "What does the queue say about design 1901-017?"
- "Pull the live row for <design_id>"
- "Get the queue evidence for a readiness check"
- Any time Walter needs the current queue values for one design before doing
  anything else with it.

Do not use it to look up several designs at once, to search by concept or
status, or to answer whether a design is ready.

## Authoritative Source

There is exactly one authoritative live Idea Queue:

| Field | Value |
|---|---|
| Spreadsheet ID | `1UxnZsA9aWlxZHqMAt17_7HAe86w_cHcMik3xpXQrfq0` |
| Worksheet | `Idea Queue` |
| Sheet ID (gid) | `1283408381` |
| URL | `https://docs.google.com/spreadsheets/d/1UxnZsA9aWlxZHqMAt17_7HAe86w_cHcMik3xpXQrfq0/edit#gid=1283408381` |

Historical, duplicate, exported, or cached copies of the Idea Queue are never
substituted, whatever they are called. If this exact spreadsheet and this exact
worksheet cannot be read in this run, the result is `SOURCE_UNAVAILABLE` and
nothing else is tried.

Read it with the read-only Google Sheets access this bot actually has (a Sheets
read tool, a Sheets connector's read call, or a read-only view of the URL
above). Use read operations only. If the bot has no way to read it, that is
`SOURCE_UNAVAILABLE`, not a reason to find another way.

## Input

Exactly one `design_id`, for example `1901-003`.

- Trim leading and trailing whitespace from the user's supplied id before
  anything else. ` 1901-003 ` is looked up as `1901-003`. That is the only
  normalisation allowed, and it applies to the input only.
- After trimming, a missing, empty, or multi-valued id is `INVALID_REQUEST`.
  Do not guess what was meant. Anything else goes to lookup as-is; an id
  that matches no stored cell exactly is simply `NOT_FOUND`.
- Matching against the sheet is exact string equality between the trimmed
  input and the stored cell value: no trimming or altering of the cell, no
  case folding, no fuzzy match, no "closest" id, no normalising `1901-3` into
  `1901-003`.

## Known Queue Fields

The standard record has these expected headers. Map them by the header text
you read, never by remembered column positions.

| Group | Expected headers |
|---|---|
| Queue (historically A:N) | `id`, `concept`, `season`, `style`, `vibe`, `status`, `idea_notes`, `image_prompt`, `art_path`, `art_gens`, `printify_id`, `etsy_url`, `notes`, `removed_reason` |
| Render workflow (historically Q:W) | `human_decision`, `render_status`, `render_source_path`, `render_output_folder`, `render_notes`, `render_updated_at`, `render_qa` |

Columns O and P are outside the standard record. Column P holds pre-existing
values in some rows, such as `simple`. Its meaning is not confirmed. Do not
invent a meaning for it, or for any other column you do not recognise; preserve
it under `unknown_columns` exactly as read.

## Header Handling

1. Read the live header row first, in full.
2. Compare each header, with surrounding whitespace removed, against the
   expected names by exact, case-sensitive equality. `ID` is not `id`.
3. An expected header that is absent is reported in `schema_warnings`, and its
   field is returned as `null`. An expected header that appears more than once
   is reported, and its field is returned as `null` because the true column
   cannot be determined.
4. A header you do not recognise is an unknown column. Keep it.
5. Never repair, rename, trim in place, reorder, or "fix" a header. Report and
   move on.

## Lookup Procedure

Given a `design_id`:

1. Trim surrounding whitespace from the supplied id, then validate it.
   Anything but one usable id: `INVALID_REQUEST`.
2. Read the exact authoritative spreadsheet and the exact `Idea Queue`
   worksheet. Not readable: `SOURCE_UNAVAILABLE`.
3. Read the header row and map fields by name (Header Handling above).
4. If the `id` header is missing or duplicated, no lookup is possible:
   `SCHEMA_WARNING`.
5. Read the **entire** `id` column and collect every row whose cell equals the
   requested id exactly. Do not stop at the first match; a second match must
   be detected.
6. Zero matches: `NOT_FOUND`. More than one: `DUPLICATE_ID` with every
   matching row number. Do not choose one.
7. Exactly one match: read that complete row, build `record` from the mapped
   headers, and collect populated unknown columns.
8. Return the JSON object below. Make no change anywhere.

## Output

Return exactly one JSON object and nothing that contradicts it in prose:

```json
{
  "design_id": "the supplied id after trimming surrounding whitespace",
  "result": "FOUND | NOT_FOUND | DUPLICATE_ID | INVALID_REQUEST | SOURCE_UNAVAILABLE | SCHEMA_WARNING",
  "source": {
    "spreadsheet_id": "1UxnZsA9aWlxZHqMAt17_7HAe86w_cHcMik3xpXQrfq0",
    "sheet_name": "Idea Queue",
    "sheet_id": "1283408381",
    "row_number": "sheet row of the single match, else null",
    "matching_rows": "every matching sheet row number, [] when none"
  },
  "record": "the 21 expected fields when FOUND, else null",
  "unknown_columns": [ { "column": "P", "header": "", "value": "as read" } ],
  "schema_warnings": [ "one plain sentence per problem" ]
}
```

Rules that make the shape deterministic:

- Every result carries `source` with the three identifiers, so provenance is
  never missing. `row_number` is the row as Google Sheets numbers it.
- `record` is present only for `FOUND`, with all 21 expected keys in the
  order listed above. Never hard-code a value; every value is read live.
- A field whose column exists but whose cell is blank is `""`. A field whose
  column does not exist in the live schema is `null`. Keep that distinction.
- `unknown_columns` lists every populated cell in the matched row whose
  header is not an expected name, as `{column, header, value}`. `header` is
  `""` when the header cell is blank. Empty unknown cells are not listed.
- `schema_warnings` is `[]` when the header row matched expectations.
  Warnings can accompany `FOUND`, `NOT_FOUND`, and `DUPLICATE_ID`. The result
  is `SCHEMA_WARNING` only when the `id` header itself is missing or
  duplicated.
- No `message`, no verdict, no recommendation.

## Result Values

| result | meaning |
|---|---|
| `FOUND` | Exactly one row matches the requested id |
| `NOT_FOUND` | The live queue was read and no row matches exactly |
| `DUPLICATE_ID` | More than one row carries the exact id; none chosen |
| `INVALID_REQUEST` | No usable single design id was supplied |
| `SOURCE_UNAVAILABLE` | The exact spreadsheet or worksheet could not be read; nothing substituted |
| `SCHEMA_WARNING` | Source readable, but the `id` header is missing or duplicated, so no lookup is possible |

## No Interpretation

This skill retrieves queue evidence. It does not determine whether `status`
Approved is sufficient, whether `human_decision` authorises production, whether
`render_source_path` is valid, whether the artwork is the correct master,
whether soft-IP is clear, whether Open Items block production, whether budget
is available, whether Printify or Etsy is ready, or whether the design may
proceed. Those belong to other skills. Report `human_decision = ""` as `""`,
not as "unapproved".

## Never Do

This skill is read-only. Never:

- write to Google Sheets: no cell change, no added or deleted row, no header
  change, no sort, no created or renamed sheet
- write to Google Drive: no upload, move, copy, rename, or delete
- modify `human_decision`, `status`, or any render field
- call Printify or Etsy
- render artwork
- infer approval or infer a missing value
- substitute a historical or duplicate Idea Queue
- clean up, normalise, or repair data or headers
- invoke `1901-validate-readiness` or any other skill, or return a readiness
  verdict; this skill is not an orchestrator
- use memory, chat history, or a prior run as evidence

If completing the read would require any of the above, stop and return
`SOURCE_UNAVAILABLE` for access problems or `INVALID_REQUEST` for input
problems. There is no other exit.

## Pitfalls

- **The first match looks right, so stop there.** Read the whole id column.
  A second `1901-003` lower down makes the answer `DUPLICATE_ID`.
- **Header says `ID` or `Id`.** That is not `id`. Report the missing header;
  do not map it anyway.
- **Column P says `simple`.** Preserve it under `unknown_columns` with
  `header: ""`. Do not call it "complexity" or anything else.
- **`human_decision` is blank.** Return `""`. Do not write "not approved" or
  guess from `status`.
- **The sheet cannot be reached, but you remember the row.** Memory is not the
  live queue. `SOURCE_UNAVAILABLE`, record `null`.
- **A near-miss id such as `1901-03`.** `NOT_FOUND`. Do not pick the closest.
- **Input arrives as ` 1901-003 ` or with a trailing newline.** Trim it and
  look up `1901-003`. A stored cell that itself carries stray whitespace is a
  different string and stays `NOT_FOUND`; the cell is never trimmed.

## Examples

Values below are fixtures to show shape and logic. Live output always carries
the values actually read.

### 1. FOUND

```json
{ "design_id": "1901-003", "result": "FOUND",
  "source": { "spreadsheet_id": "1UxnZsA9aWlxZHqMAt17_7HAe86w_cHcMik3xpXQrfq0", "sheet_name": "Idea Queue", "sheet_id": "1283408381", "row_number": 4, "matching_rows": [4] },
  "record": { "id": "1901-003", "concept": "Autumn Porch Cat", "season": "Fall", "style": "Vintage", "vibe": "Cozy", "status": "Approved",
    "idea_notes": "", "image_prompt": "", "art_path": "", "art_gens": "", "printify_id": "", "etsy_url": "", "notes": "", "removed_reason": "",
    "human_decision": "APPROVE", "render_status": "", "render_source_path": "Masters/1901-003_master.png", "render_output_folder": "",
    "render_notes": "", "render_updated_at": "", "render_qa": "" },
  "unknown_columns": [ { "column": "P", "header": "", "value": "simple" } ],
  "schema_warnings": [] }
```

### 2. NOT_FOUND

```json
{ "design_id": "1901-999", "result": "NOT_FOUND",
  "source": { "spreadsheet_id": "1UxnZsA9aWlxZHqMAt17_7HAe86w_cHcMik3xpXQrfq0", "sheet_name": "Idea Queue", "sheet_id": "1283408381", "row_number": null, "matching_rows": [] },
  "record": null, "unknown_columns": [], "schema_warnings": [] }
```

### 3. DUPLICATE_ID

```json
{ "design_id": "1901-003", "result": "DUPLICATE_ID",
  "source": { "spreadsheet_id": "1UxnZsA9aWlxZHqMAt17_7HAe86w_cHcMik3xpXQrfq0", "sheet_name": "Idea Queue", "sheet_id": "1283408381", "row_number": null, "matching_rows": [2, 4] },
  "record": null, "unknown_columns": [], "schema_warnings": [] }
```

### 4. FOUND with a blank `human_decision`

Same shape as Example 1 with `"human_decision": ""`. Nothing else changes, and
no interpretation is added.

### 5. FOUND with an expected header missing

The live header row has no `render_qa` column.

```json
{ "design_id": "1901-006", "result": "FOUND",
  "source": { "spreadsheet_id": "1UxnZsA9aWlxZHqMAt17_7HAe86w_cHcMik3xpXQrfq0", "sheet_name": "Idea Queue", "sheet_id": "1283408381", "row_number": 2, "matching_rows": [2] },
  "record": { "id": "1901-006", "concept": "Autumn Porch Cat", "season": "Fall", "style": "Vintage", "vibe": "Cozy", "status": "Approved",
    "idea_notes": "", "image_prompt": "", "art_path": "", "art_gens": "", "printify_id": "", "etsy_url": "", "notes": "", "removed_reason": "",
    "human_decision": "APPROVE", "render_status": "", "render_source_path": "Masters/1901-006_master.png", "render_output_folder": "",
    "render_notes": "", "render_updated_at": "", "render_qa": null },
  "unknown_columns": [ { "column": "P", "header": "", "value": "simple" } ],
  "schema_warnings": [ "expected header 'render_qa' is missing from the live header row" ] }
```

### 6. SCHEMA_WARNING: the `id` header is missing

The first header cell reads `ID`, so no `id` column can be identified.

```json
{ "design_id": "1901-006", "result": "SCHEMA_WARNING",
  "source": { "spreadsheet_id": "1UxnZsA9aWlxZHqMAt17_7HAe86w_cHcMik3xpXQrfq0", "sheet_name": "Idea Queue", "sheet_id": "1283408381", "row_number": null, "matching_rows": [] },
  "record": null, "unknown_columns": [],
  "schema_warnings": [ "expected header 'id' is missing from the live header row" ] }
```

### 7. SOURCE_UNAVAILABLE

```json
{ "design_id": "1901-003", "result": "SOURCE_UNAVAILABLE",
  "source": { "spreadsheet_id": "1UxnZsA9aWlxZHqMAt17_7HAe86w_cHcMik3xpXQrfq0", "sheet_name": "Idea Queue", "sheet_id": "1283408381", "row_number": null, "matching_rows": [] },
  "record": null, "unknown_columns": [], "schema_warnings": [] }
```

### 8. INVALID_REQUEST

```json
{ "design_id": "", "result": "INVALID_REQUEST",
  "source": { "spreadsheet_id": "1UxnZsA9aWlxZHqMAt17_7HAe86w_cHcMik3xpXQrfq0", "sheet_name": "Idea Queue", "sheet_id": "1283408381", "row_number": null, "matching_rows": [] },
  "record": null, "unknown_columns": [], "schema_warnings": [] }
```

## Verification

The skill worked if the reply is one JSON object in the shape above, `source`
names the exact authoritative spreadsheet and worksheet, `design_id` is the
supplied id with only surrounding whitespace removed, every value in `record`
was read from the live sheet in this run, blank cells are `""` and absent
columns are `null`, and no cell, row, header, sheet, file, or external system
changed during the run.
