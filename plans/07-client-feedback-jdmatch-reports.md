# 07 — Client Feedback: JD Match + HR/BD Reports (2026-10-20)

## A. JD Match history (feedback from Jyoti)
> "Save the history of previous JD match. Right now it's not saving, so the recruiter has to run it again if he loses it. Save by user, name it, share with in-house TA colleagues, rename. Don't give a delete option; instead count: in a month they ran X JD matches and saved Y hours."

### What exists
`models/JDMatchRun.js` already stores every run: the ranked candidate pool, batches, scores, matched and missing skills. **There is no UI or API to list, reopen, name or share runs.** It's an audit log only. It also *requires* a `requirement`, so a pasted JD can't be saved.

### Changes
- Add to `JDMatchRun`: `name` (default "<Req code> · <date>"), `source: requirement | pasted_jd | uploaded_jd`, `jdText` / `jdHash`, `requirement` becomes optional, `sharedWith[] (User)`, `sharedWithTeam (bool)`, `lastOpenedAt`, `openCount`, `starred`, `archived` (soft hide only, **no delete**), `shortlist[] (profileIds the recruiter ticked)`, `notes`.
- API: `GET /api/jd-match/runs?mine|shared|requirement=` · `GET /runs/:id` (reopen with no AI cost; scores come from the cache) · `PATCH /runs/:id` (rename, star, archive, notes) · `POST /runs/:id/share {userIds | team}` · `POST /runs/:id/rerun` (same JD against the latest profiles, with a diff showing *new* matches since the last run).
- Sharing is limited to users in the same org who hold `jd_match:read`. Shared runs open read-only, with "Duplicate to my runs".
- **Time-saved metric:** `estimatedMinutesSaved = profilesScored × minutesPerManualScreen` (org setting, default 3 min/profile) + reopen count × run time. Shown as "This month: 42 JD matches · ~31 hrs saved" on the user's dashboard and in the TA report.
- UX: a **"Recent matches"** drawer on the JD Match page and a **"Matches" tab on each requirement**. Auto-save every run, so it can never be lost.

## B. JD Match: paste a requirement (demo account)
- Add a **"Paste JD"** mode on the JD Match page: textarea or file upload → AI server `/extract-requirement` (already exists) → show the extracted skills, experience and location as editable chips → run the match. A requirement record is not needed.
- "Save as requirement" button converts a good pasted JD into a real Requirement.
- In the **demo org**, preload sample profiles so the match always returns results.

## C. JD Match search is not semantic ("shairpoint" / "share point" / "sharepoint")
### Root cause
`getProfilesInJD` (`controllers/profileController.js` ~L1382) runs `techStack: { $regex: term, $options: 'i' }`. That's a plain case-insensitive substring match:
- `shairpoint` (typo) → 0 results
- `share point` (space) → 0 results, because the stored value is "SharePoint"
- `sharepoint` → matches everything that contains the substring, unranked, so "all results"

### Fix, in layers (the DB is MongoDB Atlas, so Atlas Search is available)
1. **Normalise:** lowercase, strip spaces, dots and hyphens on both the query and the stored skills (`share point` = `sharepoint` = `Share-Point`). Add a `skillsNormalized[]` field, kept up to date in a pre-save hook. This is quick and fixes the space problem today.
2. **Skill dictionary and synonyms:** a `Skill` master with canonical name + aliases (`SharePoint` ← sharepoint, share point, SPO, SharePoint Online; `React` ← reactjs, react.js; `K8s` ↔ Kubernetes). Map parsed resumes and JDs onto canonical skills. Use it as an Atlas Search **synonyms** collection.
3. **Fuzzy:** an Atlas Search index on skills, techStack, currentRole and experience titles with `fuzzy: { maxEdits: 2 }`, which handles `shairpoint`. Results are relevance-ranked, not "all".
4. **Semantic:** store an embedding per profile (skills + roles + summary) and per JD. Use **Atlas Vector Search** for "SharePoint developer" ≈ "SPFx / Power Automate / M365 developer". Combine vector and keyword scores (hybrid), then send only the top N to the AI server for detailed scoring, which also cuts OpenAI cost.
5. **"Did you mean SharePoint?"** suggestion chip when results are 0.

## D. HR (TA) report and BD report
### What exists
`services/taReportService.js` and `services/bdReportService.js` exist, but **almost all of the code is commented out**. Routes: `/api/reports/ta/:userId/summary|download` and `/bd/:userId/...`, **per user only**. Vendor reports exist.

### Requested metrics → source
**TA / HR report** (per user + **whole TA team combined**)
| Metric | Source |
|---|---|
| New profiles added | `Profile.createdAt`, `submittedBy` |
| Profiles converted from website applications into submissions | `Applicant.status=converted` → `convertedToProfile` → Interview/Submission |
| Submissions per requirement | `Interview` / `Submission` grouped by requirement |
| Interview updates per user | Interview `statusHistory` / ActivityLog grouped by user |
| JD matches run, hours saved | `JDMatchRun` (section A) |
| Team total | same aggregations without a user filter, plus a per-user breakdown table |

**BD report** (per user + **whole BD team combined**)
| Metric | Source |
|---|---|
| Clients added | `Client.createdAt`, `createdBy` |
| Requirements added / on hold / closed | `Requirement.statusHistory` |
| CVs submitted to client | Interview stage → client submission |
| Interviews updated | Interview `statusHistory` |
| Joinings | `Placement` / `Closure` |
| Onboarding | onboarding records / Closure stage |
| Other activity | `ActivityLog` grouped by action |

### UX
- **Reports page** with: report type (TA / BD), scope (**Me / Pick user / Whole team**), date range presets (Today, This week, This month, Last month, Custom), and compare with previous period (▲▼ %).
- KPI tiles, then a per-user leaderboard table, then a trend chart, then drill-down (click a number to see the actual records).
- Export Excel (the ExcelJS code exists) and PDF.
- **Scheduled email:** weekly team report to managers every Monday, monthly to admins.
- Permissions: users see their own; team leads see their team (`reports:team`); admins see all.
