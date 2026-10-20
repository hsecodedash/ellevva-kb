# 13 — Applicant portal v3 (designed in-house; no external demo) — 2026-10-20

Goal: turn the ~31k EcoDash profiles into a live, self-updating talent pool, and let new people join. The applicant portal stays **EcoDash-only** (org module `applicantPortal`).

## Experience (screens)
1. **Join / sign in** (`/login`, plus `/applicant/join`)
   - Existing profile: email → 6-digit code (exists).
   - New person: "New here? Create your profile" → name, email, phone, consent tick (DPDP: "EcoDash may store my details and contact me about jobs") → email code → **upload resume** → AI pre-fills the form → applicant checks and saves. Profile source = "Applicant Portal".
   - Same response whether an email exists or not (no account enumeration).
2. **Home** (new default tab)
   - Greeting + **profile strength** meter (0–100) with a checklist of what's missing (resume, skills, experience, current role, notice period, expected CTC, preferred locations, work mode, LinkedIn; resume older than 6 months counts as "update your resume").
   - **Availability** switch: Actively looking / Open to offers / Not looking (feeds TA filters).
   - **Top matches for you**: best 3 open roles with match %, Apply / I'm interested.
   - **My applications** summary (counts by stage) and **next interview** card.
3. **Opportunities**: search, filters (location, work mode, min match %), sort by match or newest; role drawer shows **why you match** (matched skills ✓, missing skills) and Apply/Interested.
4. **My applications**: per application a **timeline** (Interested/Applied → Under review → Shortlisted → Submitted to client → Interview → Offer / Not selected) and **Withdraw** while still Interested/Applied.
5. **Profile & preferences**: existing profile edit + preferences (preferred roles, locations, work modes, expected CTC, notice period) + **job alerts** (weekly email, on/off).
6. Interviews, Documents, Onboarding: keep as is.
7. Navigation: tab bar scrolls horizontally on small screens; nothing cut off at any width.

## TA / admin side
- Profile list + profile view: **availability badge** ("Actively looking") and an availability filter; "updated by candidate on <date>".
- **Re-engage existing profiles** (Settings → Applicant portal, admins only): counts of profiles never invited / invited / logged in; "Send invite batch" (max 300 per batch, default 100) emailing "Update your profile and see matching roles" with a sign-in link; never re-invites the same profile within 90 days; skips profiles without a valid email or with an opt-out. Dry-run shows how many would be sent. **Never triggered automatically.**
- Unsubscribe link in every alert/invite email (signed token) → sets opt-out.

## Server API (elevva-server, all under applicant auth unless noted)
- `POST /api/applicant/register/request` (public, rate-limited) `{fullName, email, phone, consent:true, org?}` → always 200 generic; new email → stores a pending registration + sends code; existing profile → sends normal login code.
- `POST /api/applicant/register/verify` (public) `{email, code, org?}` → creates the Profile for a pending registration (candidateSource "Applicant Portal", tag `source:applicant`, consentAt) or logs in an existing one → same response shape as `/otp/verify`.
- `POST /api/applicant/resume` multipart `file` → stores in S3, asks AI server `/parse-resume`, returns `{resumeUrl, suggested:{...profile fields}}`; does not overwrite fields until applicant saves via `PATCH /api/applicant/profile`.
- `GET /api/applicant/home` → `{strength:{score, missing:[{key,label}]}, availability, topMatches:[...3], applications:{byStage}, nextInterview}`.
- `PATCH /api/applicant/preferences` `{availability?, preferredRoles?, preferredLocations?, workModes?, expectedCTC?, noticePeriod?, jobAlerts?:{enabled}}`.
- `GET /api/applicant/requirements` gains filters `q, location, workMode, minMatch, sort=match|newest`; `GET /api/applicant/requirements/:id` gains `why:{matched:[], missing:[]}`.
- `GET /api/applicant/my-interests` items gain `timeline:[{stage,label,at}]`; `POST /api/applicant/my-interests/:id/withdraw`.
- `GET /api/public/applicant/unsubscribe?token=` (public) → opt-out page/JSON.
- Staff: `GET /api/applicant-reengage/stats`, `POST /api/applicant-reengage/invite {limit, dryRun}` (permission `profiles:update` + admin; org must have applicantPortal).
- Weekly job-alert job (in-process cron, same style as existing crons): for applicants with alerts on, email up to 5 new open roles (match ≥ 60) since their last alert. Disabled unless `APPLICANT_ALERTS_ENABLED=true`.

Profile additions: `applicant: { availability: "active"|"open"|"not_looking", availabilityUpdatedAt, preferredRoles[], preferredLocations[], workModes[], jobAlerts:{enabled, lastSentAt}, consentAt, optOutAt, invitedAt, lastLoginAt, updatedByApplicantAt }` (all optional; existing fields expectedCTC / noticePeriod / preferredLocation reused where they exist).

## AI server
- `POST /parse-resume` (requires X-Internal-Key) multipart `file` → `{profile:{fullName, email, phone, currentRole, currentCompany, totalExp, skills[], education[], experience[], currentLocation, linkedinUrl, ...}}`; parse only, no CRM write, no S3. Reuses the existing extraction pipeline; same error envelope.

## Env
`APPLICANT_ALERTS_ENABLED` (default false), `APPLICANT_INVITE_BATCH_MAX` (300), `APPLICANT_REINVITE_DAYS` (90).
