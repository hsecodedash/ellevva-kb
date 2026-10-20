# 02 — Vendor Onboarding (EcoDash-only feature)

## What already exists (built, in main)
- **Model** `models/vendor/Vendor.js`: `status` pending → approved | rejected | disabled; `onboardingStatus` not_started → in_progress → submitted → changes_required → verified; profile steps company_details → addresses → contacts → references → documents. Also `VendorDocument`, `VendorContact`, `VendorAddress`, `VendorReference`, `VendorStatusHistory`, `AuditLog`, `VendorKpiSnapshot`.
- **Signup** `POST /api/vendor-auth/register` takes organizationId, companyName, contactName, email, mobile, website. The frontend `VendorRegister.jsx` is **mostly commented out**. The org was meant to come from `?org=<id>` in the link.
- **Approve** `PUT /api/vendors/:id/approve` generates a temp password, emails credentials, and sets `mustResetPassword`.
- **Vendor portal**: onboarding wizard, verification status, requirements shared with them (`Requirement.isVisibleToVendors`, `vendorAccessMode all|selected`, `sharedVendors`, `RequirementVendorShare`), profile submission, duplicate check, submissions list.
- **Admin**: Vendor Management list/detail/review, Add vendor, Vendor submissions review, Vendor performance dashboard, permission-based sidebar.
- **Documents** go to AWS S3 (`utils/vendor/vendorDocumentStorage.js`, presigned URLs), the same bucket and approach as resumes. ✅ Matches the "use the same storage as resumes" requirement.

## Target flow (from the 2026-10-20 notes)
```
Signup (basic) → Discovery call → Approve → Vendor logs in → Completes profile + docs → Verify → Active (can submit)
```

## Gaps → changes
| # | Gap | Change |
|---|---|---|
| 1 | Signup lacks **LinkedIn, bench strength, bench tech** (they only exist later in `profile`) | Add them to the register payload and validation: `linkedinUrl`, `benchStrength` (number), `benchTech[]` (tag input from the skills master). Write them into `profile.*` so the onboarding wizard is pre-filled. |
| 2 | No **discovery call** stage | Add `status` values: `registered` → `discovery_scheduled` → `discovery_done` → `approved` / `rejected`. Add a `discoveryCall { scheduledAt, scheduledBy, meetingLink, notes, outcome: go/no_go/hold, completedAt }` subdocument and log every change to `VendorStatusHistory`. Approval is only allowed from `discovery_done`. |
| 3 | Register page commented out; org passed by raw ObjectId | Public page `/vendor/signup` on the org's own domain or subdomain (resolve org from host or slug, not the ObjectId). Add hCaptcha or Turnstile plus the existing rate limiter. |
| 4 | **"Should not be open for all"** | (a) Feature flag `organization.features.vendorManagement` (on only for EcoDash). The routes, sidebar and signup page 404 when it is off. (b) Optional `vendorSignupMode: open | invite_only`. In invite_only mode, admins send a signed invite link and signup requires the token. |
| 5 | Separate vendor login (`/vendor/login`, own JWT) | Fold into unified login (plans/04). |
| 6 | Admin needs a funnel view | Vendor Management header shows status counts (Registered · Call scheduled · Call done · Approved · Onboarding · Submitted · Verified · Rejected), with filters by status, bench tech and bench size, and an age-in-stage column. |
| 7 | Notifications | Add `vendor_registered`, `vendor_discovery_scheduled`, `vendor_onboarding_submitted`, `vendor_verified` to the `Notification.type` enum. Send email to the vendor at each step (templates exist in `utils/vendor/vendorEmailTemplates.js`). |
| 8 | Calendar | v1: a manual date/time field plus a meeting link. v2: Google Calendar invite creation. |

## Admin screens touched
- Vendor list (funnel counts + filters)
- Vendor detail → new **Discovery call** card (schedule / mark done / notes / outcome) above the Approve / Reject buttons
- Onboarding review (exists): verify documents, request changes, verify

## Open
- Should a rejected vendor be allowed to reapply after N days?
- Is the bench tech list free text or tied to the skills master?
