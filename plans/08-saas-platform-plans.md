# 08 — Ellevva as a SaaS: Self-Serve Orgs, Account Types, Plans

## The model (from the 2026-10-20 notes)
- **Ellevva** = the product. **EcoDash** = one tenant (org) using it.
- Four kinds of login after this build:

| Account | Who | Where they sign up | What they get |
|---|---|---|---|
| **Org user (Ellevva customer)** | Anyone signing up on ellevva.com | `app.ellevva.com/signup` | Creates **their own org**, becomes Org Admin, gets the full CRM (profiles, clients, requirements, pipeline, JD match, HRMS, reports). **No vendor or applicant modules.** |
| **Staff user of an org** | Invited by an org admin | invite link | Role-based access within that org |
| **Vendor** | Staffing partners of **EcoDash only** | EcoDash's vendor signup page | Sees only requirements EcoDash admin shares |
| **Applicant** | Candidates of **EcoDash only** | EcoDash's careers / applicant page | Sees only requirements EcoDash admin opens to applicants |

How it is decided which account type someone is creating: by **where** they sign up, not by a dropdown. A signup on ellevva.com always creates an org. Vendor and applicant signup exists only on EcoDash-branded pages (`ecodedash.ellevva.com/vendor/signup` or `careers.ecodedash.com`). Login stays unified (plans/04).

## Requirement visibility rule (vendor + applicant)
A requirement is visible to vendors or applicants **only if an EcoDash admin turns it on**:
- `isVisibleToVendors` (exists) + `vendorAccessMode all|selected` (exists)
- `isOpenToApplicants` (new)
- Toggling these needs a new permission `requirements:publish_external`. Every toggle is logged. There's an optional auto-close date for external visibility.
- External views never show client name, budget, margin or internal notes.

## Build vendor/applicant as "modules", not EcoDash hard-coding
Store `organization.modules = { vendorPortal: bool, applicantPortal: bool, hrms: bool, ... }` and gate routes, sidebar items and signup pages on it. Turn both portals **on for EcoDash only** today. Later they can be sold as an Enterprise add-on without a rewrite. Never check `org.name === 'EcoDash'` in code.

## Self-serve org onboarding (new Ellevva customer)
1. Sign up: name, work email, password (or Google). Verify the email.
2. **Setup wizard:** company name, logo, brand colour, team size, industry (staffing agency / in-house HR), time zone and currency. Then invite teammates (with roles), import profiles (CSV / bulk resume upload → AI parse), add the first client and requirement. Optional sample data ("Try with demo data").
3. Land on a dashboard with a **getting-started checklist** (5 steps, progress bar).
4. Default roles are seeded per org: Org Admin, BD, TA, Recruiter, Viewer (`seed.js` permissions already exist).

## Plans (Free · Starter · Pro · Enterprise)
Suggested entitlements. The numbers are placeholders for a business decision.

| | **Free** | **Starter** | **Pro** | **Enterprise** |
|---|---|---|---|---|
| Users | 2 | 5 | 25 | Unlimited / custom |
| Profiles stored | 500 | 5,000 | 50,000 | Unlimited |
| Clients & requirements | 3 active reqs | 25 active | Unlimited | Unlimited |
| Resume parsing (AI) / month | 50 | 500 | 5,000 | Custom |
| JD match runs / month | 10 | 100 | 1,000 | Unlimited |
| Semantic (vector) search | – | ✓ | ✓ | ✓ |
| Resume templates | 1 default | 3 | Unlimited + Hide PII policy | Unlimited |
| Pipeline, interviews, closures | ✓ | ✓ | ✓ | ✓ |
| Reports | basic | TA/BD per user | + team, scheduled email | + custom reports, API |
| HRMS (attendance, leave, WFH, assets) | – | – | ✓ | ✓ |
| Email integration / chat | – | ✓ | ✓ | ✓ |
| Custom domain & white-label | – | – | logo/colours | full (domain, emails) |
| Vendor portal / applicant portal | – | – | – | add-on (EcoDash today) |
| SSO, audit export, data residency | – | – | – | ✓ |
| Support | community | email | priority | dedicated CSM |

### Implementation
- `Plan` collection (code, limits, features, price monthly/yearly) + `organization.subscription { plan, status: trialing|active|past_due|canceled, trialEndsAt, currentPeriodEnd, seats, provider ids }`.
- `requireFeature('hrms')` and `checkLimit('profiles')` middleware. Monthly usage counters in a `UsageCounter` collection: AI parses, JD matches, storage.
- Payments: **Razorpay** (India, INR) and/or **Stripe** (international). Self-serve upgrade for Starter and Pro; Enterprise is "Talk to sales" (form on elevva-site → lead).
- 14-day **Pro trial** on signup, then fall back to Free. Data is never deleted on downgrade; limits only block new creates.
- Soft limits: warn at 80%, block at 100% with an upgrade CTA.
- **Super Admin console** (Ellevva internal): all orgs, plan, usage, MRR, trial conversions, impersonate (with audit), enable modules.

## Things to check in code before building
- `routes/organization` POST and GET are currently **not auth-protected** in the route file. Verify whether middleware is applied upstream. Self-serve signup needs its own public endpoint; org CRUD must be super-admin only.
- Tenant isolation: every query goes through tenant context (`utils/tenantContext`, `protectTenant`). Audit the older controllers (applicantController has none).
