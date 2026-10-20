# 10 — Decisions (defaults accepted 2026-10-20) & Progress

## Decisions
| Question | Decision |
|---|---|
| Hide PII scope | Default policy from plans/01: surname, email, phones, URLs, location, company names, CTC, education years, cert issuers, photo. Configurable per org |
| Who can uncheck Hide PII | Permission `profiles:export_with_pii` |
| Who creates templates | Users with `resume_templates:write`; publishing needs `resume_templates:publish` |
| Template authoring v1 | Code + live preview; drag-and-drop later |
| Apply threshold | 70% default, per-org setting |
| Public job board without login | No; applicants see requirements only after login |
| Vendor signup | Invite-only by default (`vendorSignupMode`), open mode available |
| Discovery call | Manual date + meeting link in v1 |
| Duplicate ownership | First source wins; later sources are tagged |
| Storage | Stay on AWS S3; consolidate into one storage util (R2-ready) |
| HRMS | Demote to a module; no new investment |
| Data model | Consolidate into Person → Application → Stages → Placement (phase 1 foundation) |
| Client portal | In scope, after resume templates |
| Plans | Free / Starter / Pro / Enterprise with the placeholder limits in plans/08 |

## Progress
### P0 — Applicant portal security (branch `feat/applicant-otp-security` in elevva-server + elevva-frontend; not committed yet)
- [x] Email-only `POST /api/applicant/login` retired (410)
- [x] Email OTP: `POST /api/applicant/otp/request`, `/otp/verify`. 6-digit code, HMAC-hashed, 10-min TTL, 5 attempts, single use, 60 s cooldown, rate-limited, generic response (no email enumeration)
- [x] Applicant JWT now carries `organizationId` + `principalType`; `protectApplicant` runs requests inside the tenant context. Old tokens are rejected and the portal falls back to login
- [x] Org gating: `Organization.modules.applicantPortal` / `vendorPortal` (default off) + `scripts/enableOrgModules.js`
- [x] `Requirement.isOpenToApplicants` + a toggle in the requirement form; the applicant jobs API only lists those requirements, without client or budget fields; search is regex-escaped
- [x] Fixed the jobs list using the non-existent `jobTitle` field (now `workRole`)
- [x] Frontend: two-step OTP login screen
- [ ] **Deploy steps:** set `APPLICANT_PORTAL_ORG_SLUG=<ecodash slug>` on the server (optionally `VITE_APPLICANT_ORG_SLUG` on the frontend), then run `node scripts/enableOrgModules.js <slug> applicantPortal=true vendorPortal=true`
- [ ] Existing requirements default to *not* open to applicants. TA must tick the toggle

### Found during P0
- Frontend main (PR #107) sends a `recall` param plus vendor and source filters that only the **unmerged server `origin/vendor` commit** handles. Merge that server commit.
- `getMyProfile` / `updateMyProfile` in the applicant controller use field names the Profile model doesn't have (`currentJobTitle`, `totalExperience`; the model has `currentRole`, `totalExp`). Fix this in the applicant portal v2 work.

### Round 2 — committed on local main (2026-10-21)
- [x] **Unified login (A):** `/api/auth/identify`, shared `utils/otpService.js`, `principalType` JWT claims, single `/login` page, `/vendor/login` redirects. Also fixed the applicant portal not unwrapping the `{data}` response envelope
- [x] **Skill search (B):** `utils/skillAliases.js` (233 groups), `Profile.skillsNormalized` + hooks, typo `didYouMean`, `skillMatchCount` ranking. Security: no more populated password hashes. **Run `scripts/backfillSkillsNormalized.js`**
- [x] **Saved JD matches + Paste JD (C):** JDMatchRun extended + tenant-scoped, runs list/reopen/rename/star/archive/share/stats, `/api/jd-match/extract`, drawer UI, did-you-mean chip. **Run `scripts/backfillJDMatchRunOrg.js`**
- [x] **Team reports (D):** `/api/reports/{ta,bd}/team/summary|download`, Reports page at `/reports`. Fixed tenantScope aggregate ObjectId casting
- [x] **Vendor gaps (E):** signup fields, invite-only (`VENDOR_SIGNUP_MODE`), VendorInvite, discovery_scheduled/done, approval gate, funnel counts, `/vendor/register` route
- [x] **AI server (F):** routers, retries/timeouts, error envelope, `/render-pdf` (Playwright), `/embed`, `/redact-pii`, `/match/batch`, `/health/ready`; Docker deploy with Chromium, tesseract and fonts; pinned deps; 73 pytest tests
- [x] **Resume templates + Hide PII (G):** template builder with live preview, 4 starter templates, export with watermark and download limit. **Run `scripts/seedResumeTemplates.js <slug> --publish`**
- [x] **Applicant v2 + Interest Inbox (H):** match-gated Apply/Interested, RequirementInterest, TA inbox at `/interests`, vendor submissions feed the inbox

### Verification (2026-10-21)
- Unit/stub tests: all agent suites pass
- API smoke test on a throwaway local Mongo + AI container: **48/48**
- Headless browser smoke test (staff + applicant flows, 6 new pages): **14/14**
- Bugs found and fixed by the smoke tests:
  - **ActivityLog writes failed silently since the multi-tenant change**, so the daily download limit was never enforced (tenant plugin filled organizationId after validation)
  - Lazy queries passed to `runWithTenant` ran outside the context (applicant OTP verify, identify, vendor notify)
  - Permission resource enum missing `interests` / `resume_templates`
  - Requirement titles showed employment type; editor preview clipped; vendor count NaN; Header React warning

### Next
- Run `node scripts/seedPermissions.js`, then give roles `interests:*`, `resume_templates:*`, `profiles:export_with_pii`
- Phase 5: self-serve org signup + Free/Starter/Pro/Enterprise plans
- Candidate data-model consolidation (Application)
- Notification settings per user; vendor submission backfill into the inbox

Env to set: `APPLICANT_PORTAL_ORG_SLUG`, `VENDOR_SIGNUP_MODE`, `VENDOR_PORTAL_URL`, `JD_MATCH_MINUTES_PER_PROFILE` (optional).
Later model fields: `Organization.vendorSignupMode`, `Organization.jdMatchMinutesPerProfile`; Notification types `vendor_*`, `applicant_*`.
