# 05 — Screen Inventory

Legend: 🆕 new · ✏️ change existing · ✅ exists, no change

## Public / auth
| Screen | Route | Status | Notes |
|---|---|---|---|
| Unified login | `/login` | ✏️ | identifier → password / OTP / account chooser; org branding |
| OTP verify | `/login/otp` | 🆕 | 6-digit code, resend timer |
| Candidate signup | `/signup` | 🆕 | name, email, phone, resume upload → AI parse → confirm details |
| Vendor signup | `/vendor/signup` | ✏️ (rebuild `VendorRegister`) | Company, POC name, email, mobile, website, LinkedIn, bench strength, bench tech; captcha; only if the org has vendors enabled (or via invite token) |
| Vendor signup thanks | `/vendor/signup/done` | 🆕 | "We'll schedule a discovery call" |
| Forgot / reset password | existing | ✅ | shared across types |

## Staff — Resume templates
| Screen | Route | Status | Notes |
|---|---|---|---|
| Template gallery | `/settings/resume-templates` | 🆕 | cards with thumbnail, default badge, duplicate / publish / delete |
| Template editor | `/settings/resume-templates/:id` | 🆕 | HTML / CSS / Settings tabs · live preview · variable palette · asset upload · sample-or-real profile picker · Hide PII preview toggle · version history |
| PII policy | `/settings/resume-templates/pii-policy` | 🆕 | checkboxes for which fields Hide PII removes |
| Profile download menu | `/profiles/:id` + profile list row | ✏️ | Download ▾ (Original / templates) · ☑ Hide PII (locked without permission) · Preview |
| Bulk export | profile list selection | 🆕 (phase 2) | ZIP of PDFs |

## Staff — Vendors (EcoDash only)
| Screen | Route | Status | Notes |
|---|---|---|---|
| Vendor management | `/vendors` | ✏️ | funnel counts header, filters (status, bench tech, bench size), age in stage |
| Vendor detail | `/vendors/:id` | ✏️ | **Discovery call card** (schedule, link, notes, outcome) · approve / reject gated on call done · status timeline |
| Onboarding review | `/vendors/:id/review` | ✅ | |
| Vendor invite | modal on `/vendors` | 🆕 | send an invite link (invite_only mode) |
| Vendor submissions / performance | existing | ✅ | |

## Staff — Applicants & TA
| Screen | Route | Status | Notes |
|---|---|---|---|
| Requirement form | `/clients/requirements/:id/edit` | ✏️ | new toggle **Open to applicants** next to vendor visibility |
| **Interest Inbox** | `/interests` | 🆕 | left: my assigned requirements with new-count badges; right: candidates tagged Vendor / Applicant / Internal, match score, action, bulk shortlist / reject / move to pipeline |
| Requirement detail → Interests tab | `/clients/requirements/:id` | ✏️ | same list scoped to one requirement |
| Notifications | bell | ✏️ | new types: applicant_interested, applicant_applied, vendor_* |
| Applicant settings | `/settings/applicant-portal` | 🆕 | apply threshold %, OTP channels, consent text |

## Applicant portal
| Screen | Route | Status | Notes |
|---|---|---|---|
| Opportunities feed | `/applicant/opportunities` | ✏️ | open-to-applicant reqs, **match %**, Apply (≥ threshold) / I'm interested; filters (skill, mode, location) |
| Opportunity detail | `/applicant/opportunities/:id` | 🆕 | sanitized JD, match breakdown (matched / missing skills) |
| My applications | `/applicant/applications` | 🆕 | applied / interested with status |
| My profile | `/applicant/profile` | ✏️ | edit, re-upload CV → re-parse |
| Interviews, documents, onboarding | existing tabs | ✅ | |
| Privacy & consent | `/applicant/privacy` | 🆕 | consent record, delete-my-data request |

## Vendor portal
| Screen | Route | Status | Notes |
|---|---|---|---|
| Awaiting discovery call | `/vendor/onboarding/status` | ✏️ | shows the new pre-approval states |
| Others (onboarding, requirements, submissions, account) | existing | ✅ | vendor submissions also feed the Interest Inbox |
