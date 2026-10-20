# 01 — Resume Template Builder + Export (with Hide PII)

## Problem today
There are two export paths, and they don't agree:
1. **Frontend** `elevva-frontend/src/utils/generateEcodashPdf.js` uses hard-coded jsPDF drawing (EcoDash hero, pills, sections). Its privacy rules are baked in: first name only; no phone, email or location; no company names; no education years; no cert issuers. It runs **in the browser** (`ProfileList.jsx` → `onExport`), so the download limit and audit log are skipped and the PII rules could be bypassed.
2. **Server** `GET /api/profiles/:id/cvs/:cvId/download?format=ecodedash` (`controllers/profileCVController.js`) only stamps a "Downloaded by…" watermark and a hidden trace code on the **original** uploaded PDF. The original PDF still carries the candidate's phone and email.

Neither path supports a custom design, and adding a new look means writing new jsPDF code.

## Goal
- Admins or recruiters design **resume templates** in HTML/CSS and save them.
- On the Profile page, **Download** offers: Original CV · any saved template (e.g. "EcoDash Classic", "Client-X Minimal").
- Every templated export has a **"Hide PII"** checkbox.
- The output is a PDF.

## Design

### Data model — `ResumeTemplate` (new, tenant-scoped)
```
organizationId, name, description,
html, css,                 // Handlebars template + stylesheet
pageSize: A4|Letter, margins,
status: draft|published, isDefault,
version (int), previousVersions[] (or ResumeTemplateVersion collection),
thumbnailKey (S3), assets[] (S3 keys for logos/images),
createdBy, updatedBy, timestamps
```

### Template language
Use **Handlebars** (logic-less, so it is safe for non-dev authors), with a fixed, documented data contract:
```
{{candidate.firstName}} {{candidate.lastName}} {{candidate.fullName}}
{{candidate.email}} {{candidate.phone}} {{candidate.location}} {{candidate.linkedin}}
{{candidate.currentRole}} {{candidate.totalExp}} {{candidate.noticePeriod}}
{{#each candidate.skills}} {{#each candidate.techStack}}
{{#each candidate.experience}} {{title}} {{company}} {{startDate}} {{endDate}} {{description}} {{/each}}
{{#each candidate.education}} … {{#each candidate.certifications}} …
{{candidate.summary}}            // optional AI summary (ai-server /generate-profile-content)
{{org.name}} {{org.logoUrl}}     // white-label branding
{{meta.refId}} {{meta.downloadedBy}} {{meta.date}} {{meta.piiHidden}}
helpers: {{#if}}, {{formatDate}}, {{initials}}, {{join}}
```
Templates can use `{{#unless meta.piiHidden}}` to change the layout (for example, drop the contact bar).

### PII masking: enforced in data, not in the template
A server-side `buildResumeViewModel(profile, { hidePII, policy })` **removes** fields before the template ever sees them. If PII were only hidden with CSS, it would still be in the PDF text layer.

Default PII policy, configurable per org in Settings:
| Field | Hide PII = ON |
|---|---|
| Surname | removed (first name only) |
| Email, phone, alternate phone | removed |
| LinkedIn / GitHub / portfolio URLs | removed |
| Current location / address | removed (keep city? → open question) |
| Company names in experience | removed (matches current jsPDF rule; see open questions) |
| CTC current/expected | always removed from client-facing exports |
| Education years, cert issuers | removed (matches current rule; configurable) |
| Photo | removed |

The free-text sections (summary, experience descriptions) can still mention emails or phone numbers. Run a regex scrubber for email, phone and URL patterns over all text fields when Hide PII is ON.

- Checkbox **defaults to ON**. Unchecking it requires a new permission `profiles:export_with_pii`. Users without the permission see the box checked and disabled.

### Rendering
- **Server-side HTML → PDF with headless Chromium.** pdf-lib, pdfkit and jsPDF can't render designer HTML/CSS.
- Options, in order of preference:
  1. A **Gotenberg** container (Chromium-as-a-service, a single Docker image). It keeps Chromium out of the Express process.
  2. Playwright inside `elevva-ai-server` (it's already Python and already deployed on Render) as a `/render-pdf` endpoint.
  3. Puppeteer inside elevva-server. Simplest, but Chromium adds about 300–500 MB of RAM on Render.
- After rendering, reuse the existing pdf-lib step to stamp the watermark and hidden trace code, so leak tracing keeps working.
- Reuse the existing **daily download limit** (`DownloadLimit`) and the `ActivityLog` `profile_download`. Add `templateId`, `templateVersion` and `hidePII` to the log.

### API
```
GET    /api/resume-templates                 list (published + my drafts)
POST   /api/resume-templates                 create
GET    /api/resume-templates/:id
PUT    /api/resume-templates/:id             save (bumps version)
POST   /api/resume-templates/:id/publish
POST   /api/resume-templates/:id/duplicate
DELETE /api/resume-templates/:id             soft delete
POST   /api/resume-templates/preview         {html, css, profileId?, hidePII} → PDF/HTML (sample data if no profile)
POST   /api/resume-templates/assets          upload image → S3 → URL
GET    /api/profiles/:id/export?templateId=&hidePII=true → PDF
POST   /api/profiles/export-bulk             {profileIds[], templateId, hidePII} → ZIP (phase 2)
```
Permissions: `resume_templates:read|write|publish`, `profiles:export_with_pii`.

### Editor screen (v1 = code + live preview, not drag-and-drop)
- Left: tabs **HTML | CSS | Settings** (CodeMirror 6 or Monaco).
- Right: **live preview** in a sandboxed iframe, rendered with sample data or a picked real profile, plus a **Hide PII toggle** for preview.
- Sidebar: a **variable palette** (click to insert `{{candidate.firstName}}`) and asset upload (logo, background).
- Ships with 3–4 **starter templates**: port the current EcoDash jsPDF design to HTML as "EcoDash Classic", plus Minimal, Two-column and Modern.
- "Import HTML" lets a designer build the template in Figma, Canva or any tool, export HTML, and paste it.
- v2: a WYSIWYG block builder (GrapesJS) for non-technical users.
- Security: sanitize templates (no `<script>`, no external fetch except S3 assets). The renderer runs with JavaScript disabled and network restricted.

### Profile page change
Replace the current "Export" button with a **Download ▾** menu: `Original CV` / `<Template A>` / `<Template B>` … / plus a `☑ Hide PII` checkbox and a "Preview" option. Remember the last-used template per user.

## Effort estimate
Backend model + API + renderer + PII builder: about 5–6 days. Editor screen + preview + starter templates: about 5 days. Profile download menu: 1 day.
