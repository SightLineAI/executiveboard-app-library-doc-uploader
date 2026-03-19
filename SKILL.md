---
name: executiveboard-app-library-doc-uploader
description: Creates or updates Executive Board™ Implementation Kits in the Supabase-backed library, including week metadata and symptom-action mappings.
---

# Skill: Executive Board™ App Library Doc Uploader (Updated)

## Purpose

Provide a **single, safe entry point** for Manus to write Implementation Kits and
related metadata into the Executive Board™ app backend, including:

- Rewritten kit markdown.  
- Week‑by‑week metadata for the UI stepper.  
- Symptom → action rows for the playbook table.  
- Time estimates and multi‑location flags.

All kit content changes should go through this Skill.

---

## Data Model Assumptions

Supabase table:

- **Table:** `kits`

Columns:

- `slug` (text, unique, primary key)  
- `title` (text)  
- `content` (text, markdown)  
- `metadata` (jsonb)  
- `published` (boolean)  
- `created_at` (timestamp)  
- `updated_at` (timestamp)

`metadata` fields used by this Skill:

- `doc_type` (e.g., `ImplementationKit`)  
- `primary_lane`  
- `practice_size_range`  
- `tags` (array of strings)  
- `estimated_time_first_day` (integer, minutes)  
- `total_weeks` (integer)  
- `weeks` (array; see format below)  
- `symptom_action_rows` (array)  
- `has_multi_location_section` (boolean)

Other metadata keys must be preserved if already present.

---

## Inputs

Manus must supply:

- **DocType** (string, required)  
  e.g. `ImplementationKit`.

- **Title** (string, required)  
  Display title for the kit.

- **Slug** (string, required)  
  URL slug for the kit. Lowercase, hyphenated, unique.

- **RewrittenKitMarkdown** (string, required)  
  Full markdown from the Implementation Kit Rewriter.

- **PrimaryLane** (string, required)  
  e.g. `Marketing Governance & Campaign Operations`.

- **PracticeSizeRange** (string, optional)  
  e.g. `1–6 locations`.

- **Tags** (array of strings, optional)  
  e.g. `["ImplementationKit", "Social Media"]`.

- **EstimatedTimeFirstDayMinutes** (integer, required)  
  e.g. `30`.

- **TotalWeeks** (integer, required)  
  e.g. `10`.

- **WeekMetadataJSON** (string, required)  
  JSON string conforming to the `weeks` specification from the rewriter Skill.

- **SymptomActionRowsJSON** (string, required)  
  JSON string with `symptom_action_rows` entries.

- **HasMultiLocationSection** (boolean, optional; default `false`)  

- **UpdateStrategy** (string, optional; default `"upsert-by-slug"`)  
  - `"upsert-by-slug"` – update if slug exists; create if not.  
  - `"create-only"` – create only; error if slug exists.  
  - `"update-only"` – update only; error if slug does not exist.

---

## Behavior

### 1. Authenticate to Supabase

- Use the existing Supabase project backing `board.sightlineaisolutions.com`.  
- Use a key/role that can read and write the `kits` table and obey RLS.

### 2. Normalize and Validate Inputs

- Trim whitespace from `Title` and `Slug`.  
- Ensure `Slug` is lowercase, hyphenated, with no spaces.  
- Parse `WeekMetadataJSON` and `SymptomActionRowsJSON` as JSON.  
- Validate that `weeks` is an array and `symptom_action_rows` is an array; if not, fail with a clear error.

### 3. Construct Metadata

Build a new metadata object:

```json
{
  "doc_type": "ImplementationKit",
  "primary_lane": "... from PrimaryLane ...",
  "practice_size_range": "... from PracticeSizeRange ...",
  "tags": ["ImplementationKit", "..."],
  "estimated_time_first_day": 30,
  "total_weeks": 10,
  "weeks": [...],
  "symptom_action_rows": [...],
  "has_multi_location_section": true
}
```
If an existing metadata JSON already exists for this slug:
Load the current metadata.
Overwrite the keys listed above.
Preserve any other keys to avoid losing unrelated data.
4. Apply UpdateStrategy
Look up any existing row with this slug.
"upsert-by-slug" (default):
If row exists:
UPDATE that row:
title ← Title
content ← RewrittenKitMarkdown
metadata ← merged metadata
published ← true
updated_at ← now
If no row:
INSERT a new row with:
slug, title, content, metadata, published = true.
"create-only":
If row exists → return an error (no changes).
Else perform the same insert as above.
"update-only":
If no row exists → return an error.
Else perform the same update as in upsert.
All operations must be limited to the single matching row.
5. Ensure Published Visibility
Always set published = true on the target row.
Do not change published values on any other row.

Outputs
Return a JSON status object like:
```json
{
  "Status": "created",
  "DocType": "ImplementationKit",
  "Title": "Executive Board™ Implementation Kit – How an Independent Optometry Practice Can Run Social Media Like a Pro — Without Burning Out the Team",
  "Slug": "implementation-kit-run-social-media-like-a-pro",
  "PrimaryLane": "Marketing Governance & Campaign Operations",
  "PracticeSizeRange": "1–6 locations",
  "EstimatedTimeFirstDayMinutes": 30,
  "TotalWeeks": 10,
  "HasMultiLocationSection": true,
  "DocumentID": "kit_abc123",
  "LibraryURL": "https://board.sightlineaisolutions.com/kits/implementation-kit-run-social-media-like-a-pro"
}
```
Use "Status": "updated" when appropriate.

Safety Rules
Never delete rows.
Never alter Supabase config, RLS, or unrelated tables.
Do not silently change slugs; if a slug must be altered to meet format rules, fail and report instead.
On any constraint or RLS error, return a clear error message and leave data unchanged.

