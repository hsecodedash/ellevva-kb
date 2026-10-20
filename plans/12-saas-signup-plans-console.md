# 12 — SaaS: self-signup, plans & limits, super-admin console, org settings (2026-10-20)

Billing is **out of scope**: invoices are manual. The super-admin sets each company's plan and "paid until" date by hand.

## Decisions (defaults chosen as product owner)
- Anyone can sign up a company at `/signup`. Switch off with `ORG_SIGNUP_ENABLED=false`.
- A new company gets a **14-day Pro trial**, then drops to **Free** automatically (`SIGNUP_TRIAL_DAYS`, `SIGNUP_TRIAL_PLAN`).
- The signup admin must **verify their email** before first login. Existing users are unaffected (no verification flag = verified).
- Vendor portal and applicant portal are **never part of a plan**. Only the super-admin can switch them on (EcoDash).
- Existing companies (EcodeDash, The Premier Consultants, DEMO) are migrated to **Enterprise** so nothing changes for them.
- Paid plan with `validUntil` in the past: **7-day grace**, then effective plan = Free. Data is never deleted; over-limit companies just can't add more.
- Limits block **adding** (new user, new profile, new JD match run, new export). Reading/editing existing data is never blocked.

## Plan catalogue (`constants/plans.js`, single source of truth)
| | Free | Starter | Pro | Enterprise |
|---|---|---|---|---|
| Users (active) | 3 | 10 | 50 | unlimited |
| Profiles | 500 | 5,000 | 50,000 | unlimited |
| JD match runs / month | 20 | 200 | 2,000 | unlimited |
| Resume exports / month | 50 | 1,000 | 10,000 | unlimited |
| Custom resume templates (edit/publish) | — (starter designs only) | ✓ | ✓ | ✓ |
| Team reports (TA/BD) | — | ✓ | ✓ | ✓ |
| HRMS (attendance, shifts, leave, WFH, holidays, assets, exit) | — | — | ✓ | ✓ |
| Timesheets | ✓ | ✓ | ✓ | ✓ |

`-1` = unlimited in code. Super-admin can override any single limit per company (`limitsOverride`).

## Data model
`Organization` adds:
```
plan: {
  code: "free"|"starter"|"pro"|"enterprise",
  trialEndsAt: Date|null,       // trial of plan.code until this date
  validUntil: Date|null,        // paid until (manual invoice); null = no expiry
  notes: String,                // invoice refs etc. (super-admin only)
  updatedAt: Date, updatedBy: ObjectId(User)
}
limitsOverride: { users?, profiles?, jdMatchesPerMonth?, exportsPerMonth? }   // numbers, -1 unlimited
features override: modules.{hrms?, reports?, customTemplates?} (Boolean|undefined → plan default)
modules.vendorPortal / modules.applicantPortal (existing, super-admin only)
settings: {
  hidePiiDefault: Boolean (true),
  vendorSignupMode: "invite_only"|"open" (env fallback),
  applyMatchThreshold: Number 0-100 (env fallback 70),
  jdMatchMinutesPerProfile: Number (env fallback 3)
}
signup: { source: "self"|"admin", createdAt }
```
`models/OrgUsage.js`: `{ organizationId, period: "YYYY-MM", jdMatches, exports }`, unique (org, period), atomic `$inc`.
`User` adds: `emailVerification: { required: Boolean, tokenHash, expiresAt, verifiedAt }`.

## Server API (all responses `{success, message, data, errors}`)
### Public
- `POST /api/signup` `{companyName, name, email, password, phone?, teamSize?, website? , hp?}` → 201 `{data:{email}}`. Rate-limited per IP; `hp` honeypot must be empty; email globally unique (409 `EMAIL_TAKEN`); slug auto-unique. Creates org (+roles via `seedRolesForOrganization`), Admin user, sends verification email (Brevo). 403 `SIGNUP_DISABLED` when off.
- `POST /api/signup/verify` `{token}` → 200 `{data:{token: <jwt>, user}}` (logs in directly) or 400 `TOKEN_INVALID`.
- `POST /api/signup/resend` `{email}` → always 200 (no enumeration), rate-limited.
- `POST /api/auth/login`: unverified signup admin → 403 `{code:"EMAIL_NOT_VERIFIED"}`.

### Company (logged-in staff)
- `GET /api/org/plan` → `{plan:{code,name,effectiveCode,isTrial,trialEndsAt,validUntil,inGrace,expired}, limits:{...}, features:{...}, usage:{users,profiles,jdMatches,exports, period}}` (any staff user).
- `GET /api/org/settings`, `PATCH /api/org/settings` (`settings:read` / `settings:update`; admin role).
- `GET /api/auth/me` additionally returns `user.organization = {_id,name,slug,plan:{effectiveCode,isTrial,trialEndsAt}, features, modules}` so the sidebar can hide modules.
- Limit hit anywhere → **403** `{success:false, code:"PLAN_LIMIT_REACHED", message:"Your Free plan allows 3 users. Contact us to upgrade.", errors:{limit:"users", max:3, used:3}}`.
- Module off → **403** `{code:"MODULE_NOT_IN_PLAN", errors:{module:"hrms"}}`.

### Super-admin (`protect` + `requireSuperAdmin`), mounted `/api/platform`
- `GET /api/platform/overview` → counts: orgs by plan & status, signups last 7/30 days, trials ending in 7 days, paid expiring in 14 days.
- `GET /api/platform/orgs?search&plan&status&page&limit` → rows with plan, effective plan, users, profiles, this-month usage, createdAt, signup source, last activity.
- `GET /api/platform/orgs/:id` → full org + limits + usage + admins list.
- `PATCH /api/platform/orgs/:id/plan` `{code, trialEndsAt?, validUntil?, notes?, limitsOverride?}` → logs to ActivityLog.
- `PATCH /api/platform/orgs/:id/modules` `{vendorPortal?, applicantPortal?, hrms?, reports?, customTemplates?}` (null = back to plan default).
- `PATCH /api/platform/orgs/:id/status` `{status:"active"|"suspended"|"inactive", reason?}` (existing auth middleware already blocks non-active orgs).

## Frontend
- `/signup` (public): company + admin form, password rules, success screen "check your email"; link from `/login`.
- `/verify-email?token=` → verifies, stores session, goes to dashboard with a welcome banner.
- Login: `EMAIL_NOT_VERIFIED` → message + "Resend email".
- Settings → **Plan & usage** (`/settings/plan`): plan card, trial/expiry banner, usage bars vs limits, feature list per plan, "Contact us to upgrade" (mailto, no payment).
- Settings → **Company settings** (`/settings/organization`): Hide PII default, vendor signup mode (only if vendorPortal), apply threshold (only if applicantPortal), minutes per profile.
- Global: `PLAN_LIMIT_REACHED` / `MODULE_NOT_IN_PLAN` show one consistent "upgrade" toast/modal; trial banner in header when ≤ 5 days left or in grace.
- Sidebar hides HRMS items when `features.hrms` false, Team Reports when `features.reports` false; template editor read-only when `customTemplates` false.
- Super-admin **Platform console** (`/platform`, super-admin only): overview cards; companies table (plan badge, effective plan, users/profiles, usage this month, trial/expiry, status); company drawer with plan editor (plan, trial end, paid until, notes, per-limit overrides), module switches (incl. vendor/applicant portals), suspend/activate.

## Scripts
- `scripts/migrateOrgPlans.js` — sets every org without `plan.code` to Enterprise (idempotent).

## Env (server)
`ORG_SIGNUP_ENABLED=true`, `SIGNUP_TRIAL_DAYS=14`, `SIGNUP_TRIAL_PLAN=pro`, `PLAN_GRACE_DAYS=7`, `SALES_CONTACT_EMAIL` (shown in upgrade messages), `FRONTEND_URL` (verification link, existing).
