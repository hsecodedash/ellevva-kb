# 06 — Roadmap, Risks, Open Questions

## Phasing (recommended order)
| Phase | Scope | Why this order | Rough size |
|---|---|---|---|
| **P0 (now)** | Disable or lock the email-only applicant login; add the org filter; strip the client name from the applicant jobs API. Check whether the unmerged server `vendor` commit is needed by frontend #107. | Live data-exposure risk on about 30k profiles | 0.5–1 day |
| **P1** | Unified login + OTP service + principalType JWT | Vendor and applicant work both build on it | 4–5 days |
| **P2** | Resume templates + PII view model + Chromium renderer + profile download menu | Independent; highest day-to-day value for recruiters | 10–12 days |
| **P3** | Vendor gaps: signup fields, discovery call stage, feature flag / invite-only, funnel view | Vendor module is mostly built; these are additions | 4–5 days |
| **P4** | Applicant portal v2: Open-to-applicants flag, match-gated Apply/Interested, `RequirementInterest`, TA Interest Inbox, notifications | Waiting on the applicant demo | 8–10 days |
| Parallel | New UI (about 40% pending): new screens above use the new UI kit from the start | Avoid building the same screens twice | — |
| Hygiene | Single `utils/storage.js` S3 client (R2-ready); merge or delete stale branches; pull elevva-site | Lowers risk on every phase | 1 day |

## Ideas worth adding
- **AI summary in templates:** `{{candidate.summary}}` from the AI server's `/generate-profile-content`, written to be client-ready with PII removed.
- **Client-specific default template:** a client record can pin the template used when submitting profiles to that client.
- **Share link instead of PDF:** an expiring, watermarked web view of a templated resume, with view tracking.
- **Re-engagement campaign:** email the 30k existing profiles: "Update your profile & see matching roles" → OTP login. This is the main growth lever for the applicant portal (needs consent and an unsubscribe).
- **Vendor bench upload:** verified vendors bulk-upload bench resumes that are auto-matched to open requirements.
- **Ownership / first-source rule** for duplicates across vendor, applicant and internal (needed before vendor payouts).
- **Vendor SLA metrics** feed the existing `VendorKpiSnapshot` (submission-to-shortlist ratio, response time).

## Risks
- Chromium memory on Render: use a separate render service (Gotenberg) rather than running it in the API process.
- AI scoring cost and latency for applicant feeds: lazy scoring, prefilter, cache.
- PII leakage through free text in uploaded resumes: regex scrubber, and possibly an AI-based redaction pass later.
- Tenant isolation: every new collection carries `organizationId` and goes through `protectTenant` / tenant context.

## Open questions for the product owner
1. Hide PII: should company names, education years and cert issuers also be removed (current jsPDF behaviour), or only contact details and surname? Keep the city?
2. Who can uncheck Hide PII: admins only, or a role permission?
3. Who can create templates: admins only, or any recruiter (with admin publish)?
4. Template authoring: is code + preview fine for v1, or is drag-and-drop needed from day one?
5. Applicant apply threshold: what default %? Can TA override per requirement?
6. Can applicants see requirements without logging in (public job board on elevva-site), or only after login?
7. Vendor signup: open (anyone with the link) or invite-only?
8. Discovery call: manual scheduling, or a Calendly / Google Calendar integration?
9. Same person via vendor and applicant: who owns the candidate?
10. Storage: stay on AWS S3 or move to Cloudflare R2? (Today it's S3; R2 is not integrated.)
