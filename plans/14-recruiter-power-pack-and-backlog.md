# 14 — Recruiter power pack, SaaS growth, AI, engineering + backlog (2026-10-21)

User decisions (2026-10-21): build now → global search (most important), semantic search, pipeline board, My Day, activity timeline, duplicate merge, company setup wizard, upgrade requests + invoice log, AI copilot, backout risk + interview self-booking, fix known gaps, tests in repos + CI, repo clean-up, Sentry. Direct to prod (no staging). Everything else → backlog.

## Workstreams (one agent each, own git worktree + branch `feat/<stream>`; merged by the lead)
### A. Global search (Ctrl+K) + semantic search — `feat/search`
- **Command palette** on every staff page (Ctrl/Cmd+K, also a header search box). One box searches:
  - **Screens and features**: a registry of every page AND the notable features inside pages (e.g. "Paste JD" → /jd-match tab, "Hide PII" → download modal, "Plan & usage", "Re-engage profiles", "Discovery call" …), each with many synonyms/keywords (Hindi-English and common misspellings welcome). Typo-tolerant (fuzzy). Selecting a feature navigates to the page **and scrolls to/highlights the feature** (anchor ids `data-feature="…"` + `?focus=` param).
  - **Records**: profiles, requirements, clients, users, vendors, interviews, JD match runs — via `GET /api/search?q=&types=&limit=` (tenant + permission aware, typo-tolerant using skill aliases/regex + Atlas text where available), grouped results, keyboard navigation, recent searches.
  - Respects permissions and plan features (hidden screens don't appear).
- **Semantic search** for profiles: embeddings via AI `/embed` stored on Profile (`embedding`, `embeddingModel`, `embeddedAt`, text hash), Atlas Vector Search index `profile_embedding_v1` (script creates it; falls back to keyword search if missing), re-embed on profile create/update (async, non-blocking), backfill script (batched, resumable, `--dry-run` shows count + cost estimate). Used by: profile search ("Smart search" toggle), JD match pre-filter, palette "profiles like …".
### B. Recruiter daily — `feat/recruiter-daily`
- **Pipeline board per requirement** (Kanban): columns Sourced → Screened → Submitted to client → L1 → L2 → Offer → Joined / Backout / Rejected; drag-and-drop updates the existing Interview/stage data (no new candidate model yet — backlog 5.1), stage SLA badge (days in stage), filters by owner/source.
- **My Day** home for recruiters/BD: today's interviews, follow-ups due, requirements near deadline, new interest on my requirements, my targets vs achieved, recently viewed.
- **Activity timeline** on profile, requirement and client pages: notes, stage changes, interviews, interests, downloads, emails sent, edits — one feed with filters.
### C. Data quality & hiring ops — `feat/hiring-ops`
- **Duplicate detection & merge**: candidates by email / phone / name+company; review screen with side-by-side diff, choose fields, merge (moves notes, interviews, interests, CVs, analyses to the kept profile; keeps an audit record; reversible info kept), "not a duplicate" mark. Handles the existing `DUP-` profiles.
- **Backout risk**: on offer/selected candidates — offer date, joining date, check-in calls log, risk score (rule-based: days to join, counter-offer flag, missed check-ins, notice period) + list "At-risk joiners".
- **Interview self-booking links**: recruiter offers slots → public link → candidate picks one → interview created, both notified; reschedule/cancel link.
### D. SaaS growth + known gaps — `feat/saas-growth`
- **Company setup wizard** after first login of a new company: company profile/logo → invite team → import profiles (CSV/Excel with column mapping, uses profile limit) → first requirement → done; skippable, resumable.
- **Upgrade requests**: "Request upgrade" on Plan & usage → stored request → super-admin console queue (approve = set plan) + email to SALES_CONTACT_EMAIL.
- **Invoice log** (manual invoicing) in the console per company: number, amount, currency, period, paid date, status, notes; "paid until" can be set from an invoice.
- **Known gaps**: welcome note uses real trial days/plan; console drawer shows plan/module/status history; Interest Inbox summary shows withdrawn count; plan-change history kept beyond ActivityLog TTL (store on org or separate collection); fix outdated `otp_test` / `vendor_test` doubles (moved into repo tests by F).
### E. AI copilot — `feat/copilot`
- AI server endpoints (internal key): `/copilot/candidate-summary` (client-ready summary, PII-free option), `/copilot/outreach` (email/WhatsApp draft for a requirement), `/copilot/next-actions` (suggestions from record state), `/copilot/screening` reuse.
- Server: `/api/copilot/*` with tenant + permission checks, plan feature `copilot` (Starter+), per-month usage counter, caching by input hash.
- UI: "Copilot" side panel on profile and requirement pages (summary → copy / insert into submission email; outreach draft; next actions with one-click links).
### F. Engineering — `feat/engineering`
- **Tests into repos**: move the scratchpad suites into `elevva-server/tests/` (unit + API e2e against a Mongo service container, emails stubbed), `elevva-ai-server/tests/` already there, `elevva-frontend/tests/` (Playwright mocked UI). `npm test` / `pytest`.
- **GitHub Actions** in each repo: install, lint (changed files), build, tests on push/PR to main.
- **Repo clean-up**: remove committed junk (`shubham.txt`, `test-*.js`, `fix_passwords.js`, etc. after checking nothing imports them), merge duplicate routes (`/jd-profile-match`→`/jd-match`, `/applicant` vs `/applicants` naming), list stale branches for the user. **S3 client dedupe is done by the lead after merging A–E** (touches many files).

## Backlog (not now)
- 5.1 Single **Application** record (person × requirement, any source) replacing Applicant/Submission/Interview/VendorSubmission overlap.
- Usage emails at 80%/100% and before trial/paid end · "View as company" impersonation · white-label branding per company.
- WhatsApp Business · Gmail/Outlook sync · Client portal · Chrome extension · Job multiposting.
- Contract staffing billing · incentives · DPDP "delete my data"/retention/audit export · public API, webhooks, SSO.
- Staging environment (user chose direct to prod) · Brevo SPF/DKIM before bulk invites · Render AI 1 GB.

## Status (2026-10-20, evening)
All six workstreams (A–F) built, merged to local `main` in elevva-server / elevva-frontend / elevva-ai-server, plus the S3 client dedupe (`utils/s3Client.js`). Feature branches kept; worktrees removed.

**Verified on merged main (local DB only):** server `npm test` 25/25, `npm run test:e2e` 211/211 (smoke 48, saas 75, bulk 12, applicant 76); feature API suites A 29, B 66, C 83, D 89 + saas 75, E 31 — all pass; frontend lint 0 errors, build OK, UI pytest 2/2 (33 + 46 checks); AI server pytest 93/93. S3 paths are not exercised locally (no AWS keys in tests).

**Deploy steps (after push):**
- Server env: `BOOKING_TOKEN_SECRET` (long random; else falls back to JWT_SECRET). Optional: `COPILOT_CACHE_HOURS`, `COPILOT_TIMEOUT_MS`, `SEMANTIC_SEARCH_ENABLED`, `EMBEDDING_MODEL`, `PROFILE_VECTOR_INDEX`, etc. AI server optional: `COPILOT_MODEL`.
- Sentry: `SENTRY_DSN` + `SENTRY_ENVIRONMENT=production` on both Render services (if not done).
- One-off scripts on prod server: `node scripts/createProfileVectorIndex.js --dry-run` then without; `node scripts/backfillProfileEmbeddings.js --dry-run` (shows cost) then without; `node scripts/scanDuplicateProfiles.js --dry-run` then without.

**Open items found during the build:**
- Vendor submission emails are broken: `services/vendor/vendorNotifyService.js` uses undefined `getAdmins`/`runWithTenant`/`User`/`sendEmail` → needs a decision on which org admins receive them.
- Staff Applicants "Stats" button → `/applicants/stats` has no page.
- `elevva-server/uploads/` has 91 committed files incl. 5 resumes + 11 user documents (PII in git); check DB references before removing.
- Pipeline board moves send no notifications (existing stage endpoint doesn't); My Day not the landing page; no plan gate on board/My Day.
- Backout high-risk alert only fires when a tracker is viewed/updated (no cron); tracker "Joined" doesn't move the pipeline stage.
- No "Import" button on the Profiles list (route `/profiles/import` + wizard work).
- Search: semantic pre-filter not used on the JD Match page itself (only SubmitProfileModal).
- Stale remote branches with unmerged commits: frontend `prod`, `rel/anuj`, `rel/parth`; ai-server `rel/v2`. Fully merged ones can be deleted (list in agent F report).

## Status update (2026-10-20, night) — open items closed
Done and pushed: vendor submission/stage emails fixed (admins + requirement owner + assigned users); Applicant Stats page + `GET /api/applicants/stats`; stage-change notifications (single, board, bulk); My Day is the landing page for non-admins who work with interviews/requirements; tracker "Joined" moves the pipeline stage (project dates asked); daily 09:00 IST backout-risk sweep (`JOINING_RISK_CRON=off` disables); Import CSV/Excel button on Profiles; Smart search adds up to 50 meaning-based fits to the JD Match pool; all 9 API suites now in `elevva-server/tests/e2e` (CI uses a replica-set Mongo); server CI lints the whole repo. Plan gate on board/My Day: decided not to gate (core recruiter screens, all plans).

**Still waiting on the user (prod access is blocked for Claude by auto mode):**
1. Render env: server `BOOKING_TOKEN_SECRET`, `SENTRY_DSN`, `SENTRY_ENVIRONMENT=production`, `SALES_CONTACT_EMAIL`; AI server `SENTRY_DSN`, `SENTRY_ENVIRONMENT=production`.
2. Prod scripts: vector index, embeddings backfill, duplicate scan, uploads → S3 copy (`scripts/migrateLocalUploadsToS3.js`).
3. After the uploads copy succeeds: Claude removes `uploads/` from git (`git rm -r --cached uploads` + .gitignore). Old copies stay in git history.
4. Optional: delete fully merged remote branches (not deploy branches).

## Incident + cleanup (2026-10-20, late night)
- The 1536-dim embeddings backfill pushed the free 512 MB Atlas cluster over quota (writes blocked app-wide). Fixed by: archiving + dropping `talent-document` (raw CV text copies, 242 MB, write-only; archive at `s3://elevva-resume-storage/archives/talent-document-2026-10-20-35590.jsonl.gz`), AI server no longer writes it (`SAVE_RAW_RESUME_TEXT=true` re-enables), embeddings now 512-dim packed float32 (~2 KB/profile). DB after full backfill: ~293 MB of 512 MB.
- All 91 `uploads/` files copied to S3 and untracked from git (still in git history). `/uploads/...` falls back to S3.
- Lesson: on the free cluster, every backfill/dry-run must show storage, not just cost (`scripts/dbSizeReport.js` lists sizes).
