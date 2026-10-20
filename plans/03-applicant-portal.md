# 03 — Applicant Portal + "Interested" Inbox for TA

## What exists today (and a security problem)
`/api/applicant/*` (`controllers/applicantController.js`) and the frontend page `src/pages/ApplicantPortal.jsx` (`/applicant`).
- 🔴 **`POST /api/applicant/login` takes only an email. There is no password and no OTP.** Anyone who knows or guesses a candidate's email gets a JWT for that profile. With it they can read the profile, documents and resume URLs, edit the profile, and apply to jobs. EcoDash holds about 30k profiles, so this is a real data-exposure risk. **Fix it before anything else, or disable the route until OTP ships.**
- `Profile.findOne({ email })` has **no organizationId filter**, so it breaks tenant isolation in white-label.
- `getOpenJobs` returns **every `positionStatus: 'Open'` requirement in every org** and populates the **client name**. Applicants should not see client identity or budget.
- `applyToJob` drops the candidate straight into the pipeline (`Interview` at `BDE_SCREENING`) with no TA triage, and doesn't record a match score.
- There is also a separate `models/applicant/Applicant.js` (career-site / API-key applications, converted into a Profile later). Keep it as the inbound-form path; the logged-in portal works on `Profile`.

## Target behaviour
1. **Who can log in:** any person whose email **or** phone matches an existing Profile in the org (the 30k), plus new people who sign up and upload a resume. New signups are parsed through the AI server's `/upload-resume` into a new Profile with `candidateSource = Applicant Portal`.
2. **Auth:** OTP over email (and SMS/WhatsApp later). Optionally set a password after the first login. Rate-limit and lock out on repeated failures.
3. **Requirement visibility:** new flag on Requirement, `isOpenToApplicants` (separate from `isPublishedOnWebsite` and `isVisibleToVendors`). The TA or admin toggles it per requirement. Applicants see a sanitized view: title, skills, experience band, mode, location, and a JD **without client name or budget**.
4. **Match gate:** each open requirement shows a **match score** for the logged-in applicant, from the AI server's `/jd-profile-match` (concurrent scoring already exists). The score is cached in `JDMatchRun` per (profile, requirement, JD hash).
   - Score ≥ **apply threshold** (per-org setting, e.g. 70): **Apply** is enabled (and **Interested** too).
   - Below the threshold: only **"I'm interested"**.
   - Compute lazily when the applicant opens the feed: top N by cheap skill overlap first, then AI-score those. Never score 30k × all requirements up front.
5. **Interest record:** a new collection `RequirementInterest`:
```
organizationId, requirement, profile,
sourceType: APPLICANT | VENDOR | INTERNAL,
vendor (if VENDOR), vendorSubmission (link),
action: applied | interested,
matchScore, matchSummary,
status: new | reviewed | shortlisted | moved_to_pipeline | rejected,
assignedTo (TA), reviewedBy, notes, interview (once converted)
```
   This keeps the hiring pipeline (`Interview`) clean. TA chooses **Move to pipeline**, which creates an `Interview` and extends `submittedByType` to `USER | VENDOR | APPLICANT`.
6. **Notify TA:** on create, send a `Notification` (new types `applicant_interested`, `applicant_applied`) and a socket push to the requirement's `assignedUsers`. A daily digest email is optional.
7. **TA inbox screen ("Interest Inbox"):** lists the requirements assigned to me. For each one, it shows all incoming candidates from **three sources with tags**: 🟣 Vendor · 🟢 Applicant · 🔵 Internal (normal sourcing). Columns: match score, action (applied or interested), date, duplicate flag. Bulk shortlist, reject, or move to pipeline.
   - Vendor submissions already live in `VendorSubmission`. Either create a `RequirementInterest` row when a vendor submits, or have the inbox query both. Creating the row is the better choice, because it gives one list, one status, and simpler counts.
8. **Applicant self-service:** My profile (edit + re-upload CV, which triggers a re-parse), My applications and interests with their status, interviews (existing), documents (existing), consent and privacy (DPDP: consent capture, a "delete my data" request).

## Dedupe rules
- Normalise the phone to E.164 and lowercase the email.
- Match on email OR phone within the org. If email and phone point to **different** profiles, don't auto-merge: log the person in to the email match and raise a duplicate flag for the admin.
- The same person submitted by a vendor and also applying directly is shown once in the inbox with both tags. The first-source rule decides ownership (needs a policy decision, see roadmap).

## API (new / changed)
```
POST /api/auth/otp/request   {identifier}          (unified, see 04)
POST /api/auth/otp/verify    {identifier, code}
GET  /api/applicant/requirements                    open-to-applicant reqs + my match score
GET  /api/applicant/requirements/:id
POST /api/applicant/requirements/:id/interest       {action: applied|interested}
GET  /api/applicant/my-interests
GET  /api/interests?requirement=&source=&status=    TA inbox (scoped to assigned reqs unless admin)
PATCH /api/interests/:id                            status / notes / assign
POST /api/interests/:id/move-to-pipeline
PATCH /api/requirements/:id  {isOpenToApplicants}
```

## Waiting on
The **applicant demo** (the user will share it). The UX here is provisional until then.
