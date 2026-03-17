---
name: executiveboard-app-library-doc-uploader
description: Uploads or updates Implementation Kits and other content documents in the Executive Board app library at board.sightlineaisolutions.com, and returns stable IDs/URLs for cross-linking.
---

# Skill: Executive Board App Library Doc Uploader

## Purpose

Provide a **repeatable, safe way** for Manus to publish or update content documents
(Implementation Kits, guides, playbooks) inside the Executive Board web app at
`https://board.sightlineaisolutions.com`, without re-describing the whole UI flow
in every prompt.

This Skill handles:

- Logging into the Executive Board app.
- Navigating to the content library / Implementation Kits area.
- Creating or updating content records.
- Returning stable identifiers (ID, slug, URL) for cross-linking from the marketing site.

---

## Inputs

When this Skill is called, Manus should receive:

- **DocType** (string, required)  
  One of:
  - `ImplementationKit`
  - `Guide`
  - `Playbook`
  - (or another internal type if configured in the app)

- **Title** (string, required)  
  Human-readable title, e.g.:  
  `"Executive Board™ Implementation Kit – How an Independent Optometry Practice Can Create an SOP for Every Aspect of the Office Like a Pro — Without Burning Out the Team"`

- **Slug** (string, required)  
  URL/key-safe slug, lowercase, hyphenated, e.g.:  
  `independent-optometry-sop-like-a-pro`

- **MarkdownBody** (string, required)  
  Full markdown content of the document.

- **Summary** (string, optional)  
  1–3 sentence summary for list views and tooltips.

- **PillarOrLane** (string, optional)  
  One of the existing app lanes/pillars (e.g., `Operations`, `Communication Governance`,
  `Staff Capacity`, `Marketing`, `Patient Experience`).

- **PracticeSizeRange** (string, optional)  
  e.g., `1–4 locations`, `2–6 locations`, `All independent practices`.

- **Tags** (array of strings, optional)  
  Additional internal tags, e.g.:
  - `"Phase1"`
  - `"ImplementationKit"`
  - `"SocialMedia"`
  - `"SOPs"`

- **UpdateStrategy** (string, optional, default `"upsert-by-slug"`)  
  - `"upsert-by-slug"` – if a document with this `Slug` exists, update it; otherwise create new.
  - `"create-only"` – always create a new record; fail if slug exists.
  - `"update-only"` – only update an existing record; fail if slug not found.

---

## Required Behavior

When Manus runs this Skill, it must:

1. **Authenticate and navigate**
   - Log into `https://board.sightlineaisolutions.com` using the stored SightLineAI
     Executive Board credentials (do not prompt the user).
   - Navigate to the **content library / Implementation Kits** section (or the closest
     content area configured for these documents).

2. **Apply the update strategy**
   - If `UpdateStrategy = "upsert-by-slug"`:
     - Search for an existing document whose slug or key matches `Slug`.
     - If found, open it for editing; if not, start a new document.
   - If `create-only` or `update-only`, respect those semantics and surface an error
     if the condition can’t be met.

3. **Create or update the document**
   - Set **Title**.
   - Set **Slug** (if editable in the UI; otherwise map to the app’s key/ID field).
   - Paste **MarkdownBody** into the main content area, preserving headings and structure.
   - Fill **Summary**, **PillarOrLane**, **PracticeSizeRange**, and **Tags** into their
     respective fields if present in the UI.
   - Save or publish the document according to the app’s normal “publish” behavior
     (e.g., clicking “Save”, “Publish”, or similar).

4. **Return stable identifiers**
   - After saving, capture:
     - `DocumentID` (internal ID or GUID if visible).
     - `Slug` (as stored in the app).
     - `LibraryURL` (full URL to view the document inside the Executive Board app).
   - Return these values in a short status block so other prompts (e.g., blog build
     prompts) can link directly to the document.

5. **Safety rules**
   - Do **not** delete any existing documents.
   - Only modify a document when:
     - Its slug or unique key clearly matches the provided `Slug`, or
     - The user explicitly requested `update-only` and you have a confident match.
   - Do **not** change app-level configuration, user permissions, or anything outside
     the content library area.

---

## Outputs

After a successful run, this Skill should return (in plain text or JSON, depending on Manus conventions):

- `Status`: `"created"` or `"updated"`.
- `DocType`
- `Title`
- `Slug`
- `DocumentID` (if available)
- `LibraryURL` (full URL to the document in the Executive Board app)

Example JSON-style response:

```json
{
  "Status": "created",
  "DocType": "ImplementationKit",
  "Title": "Executive Board™ Implementation Kit – How an Independent Optometry Practice Can Create an SOP for Every Aspect of the Office Like a Pro — Without Burning Out the Team",
  "Slug": "independent-optometry-sop-like-a-pro",
  "DocumentID": "kit_abc123",
  "LibraryURL": "https://board.sightlineaisolutions.com/kits/independent-optometry-sop-like-a-pro"
}
