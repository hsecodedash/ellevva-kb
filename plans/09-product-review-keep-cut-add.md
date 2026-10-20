# 09 — Product Review: Keep · Fix · Cut · Add (product-owner view, 2026-10-20)

## Positioning first
Ellevva is trying to be three products at once: **staffing CRM/ATS** (Bullhorn, Ceipal, JobDiva, Zoho Recruit, Manatal, Loxo), **HRMS** (Keka, greytHR, Zoho People) and a **vendor/VMS** (Ceipal VMS, SAP Fieldglass-lite).
Recommendation: **sell Ellevva as an AI-first staffing & recruitment CRM for Indian and global staffing agencies.** HRMS and VMS become modules or add-ons, not the headline. Competitors win on focus; Ellevva can win on **AI matching + India workflows (WhatsApp, Naukri, C2H margins) + price**.

## What's good (keep and double down)
| Area | Why it's strong |
|---|---|
| AI resume parsing + JD extraction + JD match with scores | This is the differentiator. Most mid-market tools (Zoho Recruit, Ceipal) still do keyword match |
| Download limits + watermark + hidden trace code | Unusual and valuable for agencies worried about data theft. Market it |
| Vendor module (onboarding, document verification, sharing, KPIs) | Most CRMs lack a real vendor flow, so this is a genuine edge for C2C/C2H agencies |
| Multi-tenant + RBAC permissions | Foundation for SaaS is already there |
| Requirement budget / margin fields | Staffing-specific; generic ATSs don't have them |
| Closure without submission, closure intelligence | Reflects how BD actually works |

## What's not good (fix)
| Problem | Fix |
|---|---|
| Keyword search is substring regex (the "shairpoint" bug) | Skill dictionary + Atlas fuzzy + vector search (plans/07) |
| JD match runs are lost | Saved, named, shareable runs (plans/07) |
| Three logins, email-only applicant login | Unified login + OTP (plans/04) |
| Reports are per user only, and the services are mostly commented out | Team reports, drill-down, scheduled emails (plans/07) |
| Resume export in the browser with fixed design | Server templates + Hide PII (plans/01) |
| **Too many overlapping candidate objects**: `Applicant`, `Profile`, `PublicProfile`, `Submission`, `Interview`, `VendorSubmission`, `PostRequirement`, `Placement`, `Closure` | Settle on one model: **Person (Profile) → Application (person × requirement, any source) → Stages → Placement**. Vendor/applicant/internal is a `source` on the Application, not separate tables. This simplifies the Interest Inbox, reports and dedupe |
| **Too many dashboards**: Dashboard, CRM Dashboard, Admin Analytics, Closure Intelligence, Interview Dashboard, TA Performance, Vendor Performance | One **role-based Home** + one **Reports** area with tabs |
| Duplicate routes (`/jd-match` and `/jd-profile-match`, `/applicant` and `/applicants`) | Merge them |
| No consistent UI system (new UI is 40% done) | Finish the design system first and build all new screens on it |
| Stale branches, ~20 copies of the S3 client, committed test files (`shubham.txt`, `test-*.js`, `fix_passwords.js`) | Repo hygiene sprint |

## What to cut or demote
| Item | Recommendation |
|---|---|
| **HRMS suite** (attendance, shifts, leave, WFH, holidays, assets, timesheets, exit) | Don't delete it; EcoDash uses it internally. Hide it behind `modules.hrms` (Pro and up) and **stop investing in it**. Buyers already own Keka or greytHR. Keep only the **timesheets** that matter for contract staffing billing |
| In-app **Bug Reporting** page | Replace with a small "Feedback" widget (screenshot + text) in the header |
| Internal **chat** | Low value next to Slack, Teams and WhatsApp. Keep it minimal, or replace it with @mentions + comments on records (more useful) |
| **Lock screen** | Keep for compliance customers, but make it an org setting and keep it out of the main UX |
| Separate "Applicants" list (website applications) | Merge into the Application inbox with `source = website` |

## What to add (industry standard and high value)
### Tier 1: needed to compete
1. **Role-based Home ("My Day")**: today's interviews, follow-ups due, requirements nearing deadline, new interest in my requirements, my targets vs achieved. Recruiters live here.
2. **Kanban pipeline per requirement**: drag candidates across stages (Sourced → Screened → Submitted to client → Interview L1/L2 → Offer → Joined / Backout), with stage SLAs. Bullhorn, Manatal and Ashby all have this.
3. **Global search + Cmd/Ctrl+K**: one box for profiles, clients, requirements and people, with fuzzy matching.
4. **Activity timeline on every record**: calls, emails, WhatsApp, notes, stage changes and who did what, in one feed (the base for every report).
5. **Email + calendar sync (Gmail / Outlook)**: auto-log emails to the candidate or client, schedule interviews from the CRM, and send interview invites.
6. **WhatsApp Business integration**: India's real recruiting channel. Templates for "interested?", interview reminders and document requests, with replies logged to the timeline.
7. **Client portal**: the client logs in and sees submitted profiles (rendered with *your* resume template, PII hidden), gives Shortlist / Reject / Feedback, and picks interview slots. This removes most of the BD email back-and-forth and pairs naturally with the template work.
8. **Chrome extension (LinkedIn / Naukri sourcing)**: one click to save a profile into Ellevva with a dedupe check. Manatal, Loxo and Zoho all ship this.
9. **Talent pools + saved searches + alerts**: "Notify me when a new SharePoint profile with 5+ years is added."
10. **Duplicate detection & merge UI** (email, phone, name + company) with a keep-both option.

### Tier 2: differentiators (AI-first)
11. **AI copilot on each record**: summarise the candidate for the client, draft an outreach message, generate screening questions (exists), suggest the next action.
12. **Reverse match**: from a profile, list requirements this person fits (the applicant portal uses the same engine).
13. **Profile freshness score + re-engagement**: flag profiles older than 6 months, auto-nudge via WhatsApp or email ("Update your profile" → applicant OTP login). This turns the 30k profiles into a live database.
14. **Backout risk predictor**: for India staffing, offer-to-join drop-off is the biggest leak. Track offer date, joining date, check-in calls and a risk flag.
15. **Interview self-scheduling links**: the candidate picks a slot from the panel's calendar.
16. **Call logging + AI notes** (later): click-to-call, record (with consent), transcribe, auto-fill fields.

### Tier 3: business and ops
17. **Contract staffing (C2H/C2C) billing**: placements → timesheets → invoices → margin per placement, per client and per recruiter. A huge gap in generic ATSs.
18. **Job multiposting**: publish requirements to the careers page, LinkedIn, Naukri and Indeed, and pull applications back in with parsing.
19. **Careers page builder** per org (elevva-site already has talent / careers pages to reuse).
20. **Targets & incentives**: targets exist; add incentive calculation per closure.
21. **Compliance**: DPDP / GDPR consent capture, data retention rules, "delete my data", audit export.
22. **Public API + webhooks + Zapier** (Enterprise).

## UX principles for the new UI
- **Fewer pages, more drawers.** Open a profile or requirement in a side panel from any list, so the user doesn't lose context.
- **Tables are the product:** saved views, column chooser, filters as chips, bulk actions, inline edit, sticky header, keyboard navigation.
- **Every number is clickable** (dashboards → the underlying records).
- **Empty states teach** (e.g. "No JD matches yet. Paste a JD to try").
- **One primary action per screen**, with consistent placement (top right).
- **Status colours mean the same everywhere** (one stage palette).
- **Mobile-friendly for recruiters on the go:** Home, timeline and quick note on mobile; heavy tables on desktop only.
- **Speed:** sub-second search, optimistic updates, skeleton loaders.

## Suggested overall sequence
1. **Foundation:** security fix, unified login, design system finished, data model consolidation (Application), storage util.
2. **Recruiter daily value:** search fix (normalise → fuzzy → semantic), saved JD matches, paste-JD, Kanban pipeline, Home, timeline.
3. **Output quality:** resume templates + Hide PII → client portal.
4. **Growth channels for EcoDash:** vendor gaps, applicant portal + Interest Inbox, re-engagement of the 30k.
5. **SaaS:** self-serve org signup, plans and billing, super-admin console, modules.
6. **Integrations:** Gmail/Outlook, WhatsApp, Chrome extension, multiposting.
7. **Staffing ops:** C2H billing, backout risk, incentives.
