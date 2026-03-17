---
name: executiveboard-app-library-doc-uploader
description: Creates or updates Implementation Kits and other content documents in the Executive Board™ app library (Supabase-backed), ensuring they are published, discoverable, and correctly linked from the marketing site.
---

# Skill: Executive Board™ App Library Doc Uploader

## Purpose

Provide a **repeatable, safe, database-aware** way for Manus to create or update
content documents (Implementation Kits, guides, playbooks) in the Executive Board™
web app at `https://board.sightlineaisolutions.com`, using the existing Supabase
backend.

This Skill replaces ad‑hoc Supabase exploration each time with a consistent process
that:

- Writes to the correct Supabase table (`kits` or successor).  
- Sets required fields (`slug`, `title`, `content`, `metadata`, `published`).  
- Returns stable app URLs for cross‑linking from blog posts and other surfaces.

---

## Inputs

When this Skill is called, Manus receives:

- **DocType** (string, required)  
  Logical type of document. One of:
  - `ImplementationKit`
  - `Guide`
  - `Playbook`
  - `InternalDoc`  
  (These map into `metadata` fields; the physical table remains `kits` unless changed.)

- **Title** (string, required)  
  Human-readable title shown in the Executive Board™ UI, e.g.:  
  `Executive Board™ Implementation Kit – How an Independent Optometry Practice Can Create an SOP for Every Aspect of the Office Like a Pro — Without Burning Out the Team`

- **Slug** (string, required)  
  URL-safe identifier, lowercase, hyphenated, no spaces, e.g.:  
  `implementation-kit-create-sop-every-aspect-office`  
  This must be unique per kit and becomes the URL segment:  
  `https://board.sightlineaisolutions.com/kits/{Slug}`

- **MarkdownBody** (string, required)  
  Full markdown content of the document, including section headings, diagnostic
  checklists, phases, and tool mapping tables.

- **Summary** (string, optional)  
  1–3 sentence summary shown in cards or previews.

- **PrimaryLane** (string, optional but strongly recommended)  
  Primary Executive Board lane for this document, e.g.:
  - `Marketing Governance & Campaign Operations`
  - `Operations Governance & Documentation Infrastructure`
  - `Communication Governance & Recall`
  - `Staff Capacity & Onboarding`

- **PracticeSizeRange** (string, optional)  
  Description of target practice size; examples:
  - `1–2 locations`
  - `2–6 locations`
  - `All independent practices`

- **Tags** (array of strings, optional)  
  Internal tags for filtering/search, e.g.:
  - `"ImplementationKit"`
  - `"SocialMedia"`
  - `"SOPs"`
  - `"Recall"`
  - `"Multi-Location"`

- **EstimatedImplementationWindowWeeks** (integer, optional)  
  Typical weeks required to run the full roadmap (e.g., `8`, `10`, `12`).

- **UpdateStrategy** (string, optional, default `"upsert-by-slug"`)  
  Controls how existing records are handled:
  - `"upsert-by-slug"` – if a row with this `slug` exists, update it; otherwise create.  
  - `"create-only"` – create a new row; error if `slug` already exists.  
  - `"update-only"` – update existing row; error if `slug` does not exist.

---

## Required Database Behavior

This Skill assumes content is stored in a Supabase Postgres table, currently:

- Table: `kits`  
- Key columns (as observed in the existing app):
  - `slug` (text, primary unique identifier per kit)  
  - `title` (text)  
  - `content` (text, markdown)  
  - `metadata` (jsonb)  
  - `published` (boolean)  
  - `created_at` / `updated_at` (timestamps, defaulted by DB)  

If the schema evolves, this Skill must be updated accordingly.

When Manus runs this Skill, it must:

1. **Authenticate to Supabase**

   - Use the **existing Supabase project** backing `board.sightlineaisolutions.com`
     (e.g., `https://qmzusqgmrceozcdipqwi.supabase.co` with the configured service key).  
   - Use a role/key with permission to read and write the `kits` table, respecting
     Row-Level Security (RLS) policies.

2. **Normalize inputs**

   - Trim whitespace from `Title`, `Slug`, `Summary`.  
   - Enforce `Slug` format:
     - Lowercase.  
     - Spaces and special characters replaced with hyphens.  
     - No leading/trailing hyphen.

3. **Resolve `metadata`**

   Build a `metadata` JSON object that minimally includes:

   - `doc_type`: `DocType`  
   - `primary_lane`: `PrimaryLane` (if provided)  
   - `practice_size_range`: `PracticeSizeRange` (if provided)  
   - `tags`: `Tags` (array, may be empty)  
   - `estimated_implementation_weeks`: `EstimatedImplementationWindowWeeks` (if provided)  

   If an existing row is being updated, merge the new metadata onto existing metadata,
   preserving fields that are not explicitly overwritten, unless the caller intends
   a full overwrite.

4. **Apply `UpdateStrategy`**

   - `"upsert-by-slug"` (default):
     - `SELECT` by `slug`.  
     - If found, `UPDATE content`, `title`, `metadata`, set `published = true`,
       and update `updated_at`.  
     - If not found, `INSERT` a new row with `slug`, `title`, `content`, `metadata`,
       `published = true`.

   - `"create-only"`:
     - `SELECT` by `slug`; if exists, **fail** with a clear error.  
     - If not exists, `INSERT` as above.

   - `"update-only"`:
     - `SELECT` by `slug`; if not exists, **fail** with a clear error.  
     - If exists, `UPDATE` as above.

5. **Ensure published visibility**

   - Always set `published = true` for created/updated rows unless explicitly told
     otherwise by a future extension of this Skill.  
   - Do not modify any other rows.

---

## Outputs

After successful completion, the Skill must return a concise status object (plain text
or JSON, depending on Manus conventions), including:

- `Status`: `"created"` or `"updated"`.  
- `DocType`.  
- `Title`.  
- `Slug`.  
- `PrimaryLane` (as stored in metadata).  
- `PracticeSizeRange` (if present).  
- `EstimatedImplementationWindowWeeks` (if present).  
- `DocumentID` (internal DB id if available).  
- `LibraryURL`: `https://board.sightlineaisolutions.com/kits/{Slug}`  

### Example Output

```json
{
  "Status": "created",
  "DocType": "ImplementationKit",
  "Title": "Executive Board™ Implementation Kit – How an Independent Optometry Practice Can Create an SOP for Every Aspect of the Office Like a Pro — Without Burning Out the Team",
  "Slug": "implementation-kit-create-sop-every-aspect-office",
  "PrimaryLane": "Operations Governance & Documentation Infrastructure",
  "PracticeSizeRange": "1–6 locations",
  "EstimatedImplementationWindowWeeks": 12,
  "DocumentID": "kit_abc123",
  "LibraryURL": "https://board.sightlineaisolutions.com/kits/implementation-kit-create-sop-every-aspect-office"
}
