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

## Contract changes (server agent, as built)
All responses keep the usual `{success, message, data, errors}` envelope (payload keys below are inside `data`).

**Registration**
- `POST /api/applicant/register/request`: `phone` required (10–15 digits after stripping spaces, `+`, `-`, `()`); `consent` must be `true`. Invalid input → **400** `{errors:{fullName?, email?, phone?, consent?}}`. Otherwise always **200** with the same generic message (new email, existing profile, or no portal profile).
- `POST /api/applicant/register/verify`: new profile → **201**, existing profile (login code) → **200**; body = `/otp/verify` shape plus `isNew` (`token, principalType:"applicant", isNew, applicant:{id,name,email,code}`). Profile limit reached → **403** `code: PLAN_LIMIT_REACHED` with a candidate-friendly message (plan not mentioned). Wrong/expired/used code → 401.
- New applicant profiles start with `applicant.availability = "active"`, `candidateSource: "Applicant Portal"`, tag `source:applicant`, resume placeholder `pending-upload` (counts as "no resume" in strength).

**Resume** `POST /api/applicant/resume` (multipart `file`, PDF/DOC/DOCX, ≤10 MB; wrong type/size → 400) → `{resumeUrl, suggested, parsed, parseError?, message}`. `resumeUrl` is a 15-min presigned URL, `null` when S3 isn't configured. `suggested` keys are PATCH-able profile fields (`fullName, phone, currentLocation, preferredLocation, currentRole, currentCompany, totalExp, noticePeriod, expectedCTC, linkedinUrl, githubUrl, portfolioUrl, techStack, description`) plus `skills` as **string[]**. AI down → `parsed:false, suggested:{}` (still 200, resume saved).

**Home** `GET /api/applicant/home` → `{strength:{score, missing:[{key,label}]}, availability, topMatches, applications:{total, byStage:{<stage label>: count}}, nextInterview}`. `topMatches` items have the same shape as `/requirements` items. `nextInterview` = `null` or `{id, stage, stageLabel, at, requirement:{id, requirementCode, workRole, techStack}}`. Strength keys: `resume, resumeFresh, skills, totalExp, currentRole, noticePeriod, expectedCTC, preferredLocations, workModes, linkedinUrl`.

**Preferences** `PATCH /api/applicant/preferences` → `{preferences:{availability, preferredRoles, preferredLocations, workModes, expectedCTC, noticePeriod, jobAlerts:{enabled}}, strength}`. Arrays accept arrays or comma strings (max 10, de-duplicated). `workModes` must be Profile work-mode values (`Remote, Hybrid, On-site, Permanent, C2H, Freelancer-8hrs, Freelancer-4hrs, Part-Time`). Errors → 400 `{errors:{availability?|workModes?|jobAlerts?}}`. Side effects: `preferredLocation` = locations joined by ", ", `workMode` = first work mode; turning job alerts on clears an earlier unsubscribe.
- `GET /api/applicant/me` profile now also has `expectedCTC, workMode, resume.{originalName,uploadedAt}, applicant.{availability, availabilityUpdatedAt, preferredRoles, preferredLocations, workModes, jobAlerts.enabled, updatedByApplicantAt}`; `PATCH /api/applicant/profile` also accepts `expectedCTC`.

**Opportunities** `GET /api/applicant/requirements` accepts `q, location, workMode, minMatch, sort=match|newest, page, limit` (old `search`, `minScore` still work); response adds `sort`. `sort=match` (default) ranks across all open roles (up to 200), not just the page. `GET /api/applicant/requirements/:id` returns `why:{matched, missing}` both at top level and inside `requirement`.

**Applications** `GET /api/applicant/my-interests` items add `stage, stageLabel, canWithdraw, timeline:[{stage,label,at}]`. Stage keys: `interested, applied, under_review, shortlisted, in_process, submitted_to_client, interview, offer, joined, not_selected, withdrawn`. `statusLabel` can be `Withdrawn`. `POST /api/applicant/my-interests/:id/withdraw` → 200 `{interest:{id,status:"withdrawn",statusLabel}}` (idempotent); after a recruiter shortlisted/moved it → **409** `code: CANNOT_WITHDRAW`; not mine → 404. Applying/interest again after withdrawing reopens the same application (status `new`).
- Staff side: `RequirementInterest.status` can now be `withdrawn` (candidate only). Staff can't change a withdrawn application (409; bulk skips with reason `withdrawn`); `/api/interests/summary` rows include `withdrawn` count.

**Unsubscribe** `GET /api/public/applicant/unsubscribe?token=` → HTML page when `Accept: text/html`, else JSON. Link base: `PUBLIC_API_URL`, else Render's `RENDER_EXTERNAL_URL`, else the request host.

**Re-engage (staff)** — admin role (or super admin) + `profiles:update`, company must have `applicantPortal` (else 403 `code: MODULE_NOT_ENABLED`).
- `GET /api/applicant-reengage/stats` → `{stats:{withEmail, neverInvited, invitedRecently, signedIn, optedOut, readyToInvite}, batchMax, reinviteDays}`.
- `POST /api/applicant-reengage/invite {limit?, dryRun?}` → `{message, result:{dryRun, requested, selected, sent, failed, batchMax, reinviteDays, sample:[{id,name,email}]}}`. `limit` defaults to 100, capped at `batchMax`. Skips: opted out, placeholder emails, duplicate copies, invited or signed in within `reinviteDays`, repeated emails. Rate limit: 20 runs/hour.

**TA profile list** `GET /api/profiles?availability=active,open,not_looking` filters; profile documents (list and detail) include the `applicant` sub-document (`availability`, `updatedByApplicantAt`, ...).

**Env** `APPLICANT_ALERTS_ENABLED` (false), `APPLICANT_INVITE_BATCH_MAX` (300), `APPLICANT_REINVITE_DAYS` (90), `PUBLIC_API_URL` (optional, for unsubscribe links). Job alerts run Mondays 09:00 IST.
