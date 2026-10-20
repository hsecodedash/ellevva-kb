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
