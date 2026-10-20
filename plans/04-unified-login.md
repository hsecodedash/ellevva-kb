# 04 — Same Login for All (Staff · Vendor · Applicant)

## Today: three separate systems
| Who | Endpoint | Collection | Credential | Frontend |
|---|---|---|---|---|
| Staff | `POST /api/auth/login` | `User` (RBAC roles/permissions) | password | `/login` |
| Vendor | `POST /api/vendor-auth/login` | `Vendor` | password (temp → reset) | `/vendor/login` |
| Applicant | `POST /api/applicant/login` | `Profile` | **email only** 🔴 | `/applicant` |

## Proposal
**One login screen. Keep the three collections separate** (merging vendors and candidates into `User` would tangle RBAC).

1. `/login` asks for **email or phone**, then `POST /api/auth/identify` resolves the identifier within the current org (resolved from the domain or subdomain) against User → Vendor → Profile.
   - Staff or vendor: show the password field (plus "Login with OTP").
   - Applicant only: send an OTP.
   - Match in more than one collection (e.g. a vendor POC who is also a candidate): show an account chooser ("Continue as Vendor — Acme / as Candidate").
   - Not found: offer "Sign up as candidate" or "Register as vendor" (vendor only if the feature is enabled for the org).
   - Enumeration protection: use a generic response with a uniform delay. `identify` returns only the allowed methods, never whether an account exists for a given type, when called unauthenticated more than N times.
2. **JWT claims:** `{ sub, principalType: 'user'|'vendor'|'applicant', organizationId }`. A single middleware `authenticate()` loads the principal by type. The existing `protectTenant`, `protectVendor` and `protectApplicant` become thin guards on `principalType`.
3. **Frontend:** a single AuthContext (the recent "auth context refactor" is a good base). After login, route by `principalType`: staff → `/dashboard`, vendor → `/vendor/dashboard` or onboarding, applicant → `/applicant`. Remove `/vendor/login`, or keep it as a redirect.
4. **Shared:** forgot or reset password, OTP service (email now; SMS/WhatsApp provider later), session limits (`SessionLimit` / `UserSession` already exist for staff; extend them to the other types), lock screen for staff only.
5. **White-label:** org from host (`acme.ellevva.com` or a custom domain → `Organization.slug`) drives the logo and colours on the login page. EcoDash gets the vendor signup link; other orgs don't.

## Migration
- Ship `identify` + OTP alongside the old endpoints. Switch the frontend over. Keep the old endpoints for one release, then remove them. **Exception:** the applicant email-only login is removed immediately.
