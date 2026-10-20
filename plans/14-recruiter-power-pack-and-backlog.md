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
