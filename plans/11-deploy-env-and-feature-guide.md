# 11 — Deploy env + feature guide (2026-10-20)

## 1. Environment variables

### elevva-server (main API)
| Variable | Value | Required |
|---|---|---|
| `AI_SERVER_URL` | Public URL of the AI server, e.g. `https://<ai-service>.onrender.com` (no trailing slash) | Yes — PDF export, applicant match % and JD match use it |
| `AI_INTERNAL_KEY` | Random secret, `openssl rand -hex 32`. **Same value on the AI server** | Yes |
| `APPLICANT_PORTAL_ORG_SLUG` | `ecode-dash` | Yes for applicant login |
| `VENDOR_SIGNUP_MODE` | `invite_only` (default) | Recommended |
| `VENDOR_PORTAL_URL` | `https://app.ellevva.com` | Yes — links in vendor emails |
| `APPLY_MATCH_THRESHOLD` | `70` (default) — match % at which "Apply" appears | Optional |
| `JD_MATCH_MINUTES_PER_PROFILE` | `3` (default) — minutes saved per screened profile | Optional |

Already required and unchanged: `JWT_SECRET` (also signs applicant OTP codes), `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_REGION`, `AWS_S3_BUCKET` (template assets), `FRONTEND_URL`, Brevo mail settings.

### elevva-ai-server (Render)
Switch the service runtime to **Docker** (render.yaml already does this).
| Variable | Value | Required |
|---|---|---|
| `AI_INTERNAL_KEY` | Same value as on elevva-server | Yes |
| `CORS_ORIGINS` | `https://app.ellevva.com` (`*` still works) | Recommended |
| `RENDER_ALLOWED_HOSTS` | Hosts template images/fonts may load from, e.g. `fonts.googleapis.com,fonts.gstatic.com`. S3 bucket is added automatically | Optional |
| `RENDER_CONCURRENCY` | `1` on 512 MB plan, `2` on 1 GB+ | Recommended |
| `HEAVY_CONCURRENCY` | `1` on 512 MB plan, `2` on 1 GB+ | Recommended |
| `LOG_FORMAT` | `json` | Optional |

Unchanged: `OPENAI_API_KEY`, `MONGODB_URI`, `AWS_*`, `CRM_BACKEND_URL`, `BULK_INGEST_API_KEY`.

### elevva-frontend (Vercel)
| Variable | Value | Required |
|---|---|---|
| `VITE_APPLICANT_ORG_SLUG` | Same slug as `APPLICANT_PORTAL_ORG_SLUG` | Optional (server default is used) |

Redeploy the frontend after changing `VITE_*` values (they are baked in at build time).

## 2. One-time scripts (run on elevva-server with the prod `MONGO_URI`)
```
node scripts/seedPermissions.js
node scripts/enableOrgModules.js <ecodash-slug> applicantPortal=true vendorPortal=true
node scripts/seedResumeTemplates.js <ecodash-slug> --publish
node scripts/backfillSkillsNormalized.js
node scripts/backfillJDMatchRunOrg.js
```
Then in **Roles & Permissions** give roles: `interests:read/update` (TA), `resume_templates:read/write/publish` (admin), `profiles:export_with_pii` (only who may download with contact details).

## 3. Features and how to use them


| Feature | Where | How |
|---|---|---|
| One login | `/login` | Enter email → app shows password (staff/vendor) or "send code" (applicant). Several accounts → pick one |
| Applicant portal | `/applicant` | Applicant logs in with email code; Opportunities tab shows open requirements with match %; Apply (≥ threshold) or I'm interested; My Applications tab tracks status |
| Open a requirement to applicants | Requirement form | Tick "Open to applicants" — client and budget stay hidden |
| Interest Inbox | Sidebar → Interest Inbox (`/interests`) | Per requirement: vendor / applicant / internal candidates; shortlist, reject, move to pipeline. TA gets a notification on each new interest |
| Vendor invite + signup | Vendors → Invite vendor | Vendor gets link `/vendor/register`, fills name, POC, website, LinkedIn, bench strength, bench tech |
| Vendor approval | Vendor detail | Schedule discovery call → mark done → approve → vendor logs in, uploads documents → verify → onboarded |
| Smart search | JD match / profile search | Spelling and spacing variants ("shairpoint", "share point") resolve to SharePoint, with "Showing results for" chip |
| Saved JD matches | JD Match (`/jd-match`) | Every run auto-saved; History drawer: rename, share with colleagues, archive (no delete); monthly "X matches · ~Y hrs saved" |
| Paste JD | JD Match | "Paste JD" tab — paste text instead of uploading a file |
| Team reports | Sidebar → Team Reports (`/reports`) | TA or BD tab, one person or whole team, date range, compare with previous period, Excel export |
| Resume templates | Settings → Resume Templates (`/settings/resume-templates`) | Duplicate a starter, edit HTML/CSS with live preview, publish, set default |
| Export with template + Hide PII | Profiles list / profile view → Download | Choose template, keep "Hide PII" ticked (first name only, no phone/email/links). Unticking needs `profiles:export_with_pii`. PDFs are watermarked and count toward the daily limit |

## 4. Seed log (2026-10-20, DB `Saas-Elevva-Mern`)
- Permissions: added 10 (`submissions:*` ×4, `interests:*` ×2, `resume_templates:*` ×3, `profiles:export_with_pii`). Needed `submissions` added to the Permission resource enum.
- `ecode-dash`: applicantPortal + vendorPortal enabled; 4 resume templates created and published.
- JD match runs: organizationId backfilled on 271.
- Profiles: skillsNormalized on 30,979 / 30,980; index `organizationId_1_skillsNormalized_1` built.
- Duplicate codes fixed: DUP-Mohammad PRO-A004 → PRO-A577, DUP-Siddharth PRO-A570 → PRO-A578; unique index `org_profileCode_unique` built.
- Local `.env` MONGODB_URI updated to the Render value (same cluster/DB, newer password).
- Pending (user, via Roles & Permissions UI): Admin → all new; TA Manager → interests + export_with_pii; TA Executive → interests; BD Manager → export_with_pii.
