# TUTUMatch

**TUTUMatch is a free NSW tutor directory** — think Hipages or Indeed, for high-school tutoring. Parents browse listings and contact tutors completely free. Tutors list for free and pay a flat commission ($20, or $15 if they self-report honestly) **only when a real student is confirmed** — and a tutor's first matched student is free. The platform makes no verification claims: listings are tutor-provided and parents verify tutors themselves. Once an introduction is made, the platform is out of the loop.

## New here? Read this first

**What state is the project in?** The codebase is **mid-pivot**. TUTUMatch was originally built as a *verified marketplace* — the platform vetted tutors, and parents paid a $20 "unlock" fee to see a tutor's contact details. It is being rebuilt into the **pure-directory model** described above. The rebuild is split into 5 sessions:

| Session | Scope | Status |
|---|---|---|
| 1 | Remove all verification claims site-wide | ✅ Complete |
| 2 | Remove parent payment; add the free "I want this tutor" Match flow | ✅ Complete |
| 3 | Confirmation flow + strike system + tutor commission (Stripe) | 🚧 Mostly landed — Stripe in simulation; real charges + Permanent Checkout deferred |
| 4 | WWCC reframe + admin pivot + auto-approval | 🚧 Partially landed — OTP/Turnstile deferred; **under-18 gate removed but NOT replaced** (see blockers) |
| 5 | Automation hardening + audit logging + retention | ⬜ Not started |

> ⚠️ **This table drifted from the code.** Sessions 3 and 4 have commits on `main` (`73a83fa`, `b177a76`, `8227c1c`, `f2ef4ff`, …) even though they were previously marked "Not started". **Trust `git log` over this table** until the two are reconciled. The drift is not cosmetic: it is how the under-18 launch blocker below went unnoticed. Verify the current commit yourself rather than relying on a hardcoded hash here.

## 🚨 Legal launch blockers — do NOT deploy publicly until these are cleared

A legal-risk assessment on 2026-07-02 found the pivot's *reasoning* sound but the *current build* unsafe to launch, mainly because code changes outran the legal copy and the auth/privacy surface. These are ranked; 🔴 must be fixed before any public exposure.

| # | Blocker | Where | Level |
|---|---|---|---|
| 1 | **Under-18 tutors can list, but the Terms still promise they're auto-rejected.** Commit `73a83fa` removed the API-layer DOB auto-reject; the planned replacement (age-attestation checkbox, DoD item in Session 4) was never built, so `POST /api/tutor/applications` auto-approves any clean listing regardless of DOB. Meanwhile Terms §11 still says under-18 applications are "automatically rejected" and Privacy says we don't knowingly collect under-18 data. A minor's listing + a promise-we-broke is the single fact that most undermines the directory defence. | `src/app/api/tutor/applications/route.ts`, `src/app/legal/terms/page.tsx` §11, `src/app/legal/privacy/page.tsx` | 🔴 |
| 2 | **PII / WWCC breach vectors.** (a) `ADMIN_EMAILS` auto-promotes at signup with no email-ownership check, and the admin address ships in `.env.example` → first person to register it owns every WWCC/ID doc. (b) `session.ts` falls back to a hardcoded dev secret (public in this repo) if `NEXTAUTH_SECRET` is unset → forgeable admin cookie. (c) `POST /api/matches/request` is unauthenticated, unthrottled, and returns each tutor's full name + phone + email → scrapeable contact base (and silently hides the listing 48h). (d) Privacy says docs are "encrypted at rest" but they sit in plaintext on the local filesystem. | `src/app/api/auth/signup/route.ts`, `src/lib/session.ts`, `src/app/api/matches/request/route.ts`, `src/lib/uploads.ts` | 🔴 |
| 3 | **Legal pages describe systems that don't exist yet.** The audit trail (IP/UA/terms-version) the "self-attestation under audit risk" model relies on is Session 5 (unbuilt); applications store no IP/UA. The strike **reinstatement fees** ($20 after strikes 1–3; perpetual $20 rate after strike 3) are in code + README but appear nowhere in the Terms the tutor accepts. Charging for terms the user never agreed to, plus copy that overstates the service, is ACL exposure. | `src/app/legal/*`, `src/lib/strike.ts` | 🟠 |
| 4 | **Real schools published without permission.** `SEED_SCHOOLS` ships Killara High School and Masada College with `active: true`; the seed writes them live, using real names/brand colours to imply association. The schema even has `permissionEvidenceUrl` and the README warns "get written permission first" — the seeded default ignores it. | `src/lib/schools.ts`, `prisma/seed.ts` | 🟠 |

**Also flagged (lower priority):** unfair-contract-terms review of the tutor indemnity breadth + unilateral variation/revocation clauses (get counsel); wrongful-charge/griefing risk from the unverified match endpoint (ties to blocker 2); this repo is **public** and contains candid defence-cost strategy, the admin email, and exact abuse thresholds ("3 reports in 30 days") — consider making it private; substantiate landing-page figures ("tutors typically $60–80/hr", centres "40–60%") or soften them (ACL/ACCC advertising); add a LICENSE and run a trademark/`.com.au` check on the name.

The full severity × likelihood write-up lives in the assessment that produced this list — the four 🔴/🟠 items map to risks R1–R4 there. Counsel review (already budgeted in *Founder / legal → Phase 2*) should cover blockers 1, 2, and the indemnity terms; **do not self-clear blocker 1.**

➡️ **Step-by-step fix instructions for every blocker are in [Session 0 — Launch blocker remediation](#session-0--launch-blocker-remediation-do-these-before-anything-else), just before the Pivot work plan.** Do that session first; it gates public launch.

**Where do I find things in this README?**

- **The new model** — every design parameter: fees, the 48-hour match flow, the strike system, appeals, WWCC framing. This is the product spec.
- **Pricing & monetization** / **Data model changes** / **API contracts** — the canonical implementation references. Build from these; don't fork the numbers or shapes into individual files.
- **For maintainers — hand-off guide** — read this before writing code. It has the quick state check, how to start a session, code conventions, and the decision log.
- **Pivot work plan** — the session-by-session breakdown, each with a Definition of Done checklist. Start the next session whose DoD isn't satisfied.
- **Local setup** — how to install and run it.

**Picking up the build?** Go straight to *For maintainers — hand-off guide*, then start the next unfinished session in the *Pivot work plan*.

**One caveat:** the **Status** section and parts of **Technical roadmap** further down describe the *pre-pivot* verified-marketplace build. They are kept for historical reference only — the directory model above supersedes them wherever they conflict.

## Why the pivot? (decided 2026-05-21)

The original verified-marketplace model made TUTUMatch a "verified marketplace facilitator" in legal terms — high exposure, needing a Pty Ltd structure and ~$1,400/yr insurance. After working through the legal-exposure analysis (parents-paying vs. tutors-paying-commission), the chosen direction is **a pure directory** (Hipages / Indeed style):

- Parents browse, contact, and use everything completely **free** — no platform transaction at all
- Tutors are charged a **commission per confirmed match** ($20, or $15 with the honesty discount)
- The first matched student is **free for the tutor** — commission starts on the second match onwards (marketing framing: "first student free")
- The platform makes **no verification claims** about tutors anywhere — listings are tutor-provided, parents verify directly

This drops the legal risk profile from "verified marketplace facilitator" down to "classifieds directory" (Hipages-tier exposure — ~$500/yr media liability is enough), and total realistic ongoing cost to **$500–1,000/yr** rather than $2,000+.

---

## The new model — directory + confirmed-match commission

### The headline (canonical phrasing — use everywhere)

> **List free. Pay only when you get a student. First match is on us.**
>
> After that, $20 per student you actually take on. Or skip per-match fees with **TUTUMatch Permanent**: $60 once, list forever, no more commissions (pays for itself at your 4th student).

This is the single-paragraph value prop. Hero, landing page, signup form intro, and tutor dashboard all use this language verbatim or close to it. **Do not** use "free listing first time" — that's ambiguous about what happens later. The above framing is unambiguous: free to list permanently, pay per outcome.

> **Do not put the $15 honesty rate in public/marketing copy** (landing page, Hero, FAQ, how-it-works). Product decision (2026-05-21): the advertised commission is a flat **$20**; the $15 self-report discount is surfaced **in-product only** — at the point a tutor goes to self-report a match — so it lands as a pleasant discount rather than an advertised number. The mechanic itself is unchanged (see Pricing, Match flow, API contracts).

### Money flow (the key change from pre-pivot)

- **Parent pays $0** to the platform — ever, in any flow
- **Tutor pays $20** when a parent confirms a match (or **$15** if the tutor self-reports honestly before the parent confirmation prompt arrives)
- **First confirmed match is free** for every new tutor — commission starts on the second match onwards. Headline marketing hook.
- **TUTUMatch Permanent**: optional $60 one-time upgrade. Tutor never pays per-match commission again. 14-day no-questions refund on the purchase. Forfeited on 3rd-strike permanent suspension. **Non-transferable** — tied to the individual tutor account; it cannot be sold, assigned, or passed to another person, and is revoked (no refund) if the account changes hands. The Terms of Service must state this.
- Future possibility (not for now): bump base commission to **$25 ($20 with honesty)** for *new* tutors only, after the model proves out. Permanent stays at $60.

### The match flow, end to end

1. **Tutor lists for free.** Profile shows their info, subjects, rate, area, etc. Submits WWCC details (see WWCC framing below). Profile goes live after a light spam/abuse-only check by admin — **no credential verification**.
2. **Parent browses for free.** No signup needed to browse. Filter by subject, area, rate.
3. **Parent clicks "I want this tutor".** Free click; reveals the tutor's contact details (email + phone) so the parent can reach out directly. Account creation may be needed at this point for tracking, but no money changes hands.
4. **The 48-hour conversation window opens.** As soon as the parent clicks, the tutor's listing is **temporarily hidden** from public browse (other parents can't simultaneously "claim" them). The tutor is notified immediately: "[Parent name] selected you — self-report within 48 hours if you book a lesson for the $15 honesty rate AND to keep your listing visible."
5. **Tutor either self-reports OR waits.**
   - **Self-reports a confirmed match** within 48h → charged **$15**, listing immediately returns to public browse. Done.
   - **Self-reports no match** within 48h → no charge, listing returns to browse.
   - **Doesn't respond** → goes to step 6 after 48h.
6. **Confirmation prompt sent to parent after 48 hours.** "Did you have a lesson with [tutor]? Yes / No / Not yet." Single-click email links. Multiple reminders at 7, 14, 30 days if no reply.
7. **Resolution paths:**
   - Parent says **Yes** → tutor charged **$20** (no honesty discount; they missed the self-report window). Listing returns to browse.
   - Parent says **No** → no charge. Listing returns to browse.
   - Parent doesn't reply within 30 days → platform asks the tutor directly. If tutor says yes → charged $15 (treated as late self-report). If tutor says no → no charge but logged for audit (random spot-checks happen — see below).
8. **Listing returns to public browse** in all resolved cases.

### The strike system (for tutors who lie about no match when there was one)

A "strike" is triggered when:
- Parent confirms a match happened, AND
- Tutor previously said no, AND
- (After appeal process if used — see below)

| Strike | Penalty | Reinstatement |
|---|---|---|
| **1** | Profile hidden 7 days | Pay the missed $20 to reappear |
| **2** | Profile hidden 30 days | Pay the missed $20 to reappear (no penalty fee) |
| **3** | Profile permanently hidden | Pay $20 to unlock. **All future matches cost $20** (no honesty discount, perpetual) |
| 4+ | Same as 3 — pay $20 per unlock each time | |

The user's explicit reasoning here: keep the strike penalties moderate so honest mistakes can be recovered from, but make a third strike permanently sticky on the $20 rate so consistently dishonest tutors are effectively gating themselves out of the discount.

### Appeal flow (for genuine disputes)

When the parent says no but the tutor insists a match happened, the tutor can **appeal** before the strike is recorded:

- Tutor uploads evidence (bank transfer screenshot, lesson notes, parent's reply mentioning the lesson, etc.)
- Admin manually reviews. Bank transfer with name match = automatic pass.
- If admin agrees → tutor's claim wins, $20 charged, no strike.
- If admin disagrees → strike applied as normal.

This protects honest tutors against parents who lie to avoid the platform seeing the match (which could happen if the tutor and parent took everything off-platform and the parent thinks the tutor should eat the cost).

### WWCC handling — the careful framing

This is the most delicate part of the directory positioning. We need WWCC for child-safety reasons but we **cannot frame ourselves as verifying it** — that's what made the verified-marketplace model legally exposed.

**The platform's framing to tutors at signup:**
> "We ask for your WWCC details so you have them ready to give directly to parents who ask. Parents have the right (and the responsibility) to verify your WWCC directly with the [NSW Office of the Children's Guardian](https://www.kidsguardian.nsw.gov.au/working-with-children/employer-or-agency/how-to-verify-a-wwcc) before any lesson. We don't perform that verification for them and we don't display a 'verified' badge — but having your WWCC ready means a parent can do their own check in 30 seconds."

**The platform's framing to parents:**
> "Tutors on TUTUMatch self-declare a WWCC. **TUTUMatch does not verify WWCCs.** Before booking a lesson, please verify the tutor's WWCC directly at the NSW OCG public lookup [link]. It takes 30 seconds and is free."

**What we collect (same as today):** WWCC number, WWCC full name, WWCC date of birth. These are stored on the application and **shown to the tutor on their own dashboard** so they can copy them when a parent asks. They are **not displayed publicly** and not used to grant any platform-side badge.

**What we DON'T do:** check WWCCs ourselves, display verification badges, claim the tutor is "screened" or "vetted" or "safe", show any platform-stamped trust signal anywhere on the site.

This gives the safety benefit (tutors have WWCCs, parents can verify) without the legal exposure (no verification representation by the platform).

### Free-for-the-first-student framing

This is a real positioning advantage and should be on the landing page front and centre:

> **List free. Your first matched student costs you nothing. From your second student onwards, TUTUMatch charges a flat $20 commission per confirmed match. No subscription. No per-lesson cut. Only pay when a real student commits.**

This is the genuine differentiator vs. tutoring centres (40-60% per lesson, forever) and pay-to-list directories (upfront cost with no guaranteed return).

---

## Pricing & monetization (canonical values — hardcode from here)

These are the numbers the build uses. If they change, the change goes through this section first, then a search/replace through the codebase. Don't fork pricing decisions into individual files.

| Item | Amount | Notes |
|---|---|---|
| **Listing fee** | $0 | No upfront cost. Ever. |
| **First confirmed match per tutor** | $0 | The free-first-match sweetener. Tracked on the tutor's record (`matchesCompletedCount === 0`). |
| **Standard commission per confirmed match** | $20 AUD | Charged via Stripe Connect to tutor's saved card. |
| **Honesty-discount commission** | $15 AUD | Applies when tutor self-reports the match before the parent-confirmation prompt fires (i.e., within the 48-hour window). |
| **TUTUMatch Permanent (one-time)** | $60 AUD | Tutor pays once, never pays per-match commission again. 14-day no-questions refund. Forfeited on 3rd-strike permanent suspension. Non-transferable — tied to the individual tutor account; revoked with no refund if the account changes hands. |
| **Permanent break-even** | 4th student | At standard $20 × 3 = $60, so the 4th confirmed match onwards is pure savings. Mention this in marketing copy. |
| **Future bump (not now)** | $25 / $20 honesty | For *new* tutors only, after model is validated. Permanent stays $60 (or rises in lockstep — decide at activation). |

### What's free vs paid (canonical)

| Action | Cost to user |
|---|---|
| Browse the directory | Free (no account needed) |
| Search / filter / sort tutors | Free |
| Click "I want this tutor" → see contact details | Free (no platform transaction) |
| Tutor signs up + lists a profile | Free |
| Tutor's profile appears in browse | Free (until past first confirmed match, then per-match commission applies) |
| Tutor self-reports a confirmed match honestly | $15 (Stripe charge) |
| Tutor charged by parent confirmation | $20 (Stripe charge) |
| Tutor upgrades to Permanent | $60 one-time (Stripe charge) |
| Re-listing after strike 1 reappearance | $20 to clear missed commission |
| Re-listing after strike 2 reappearance | $20 to clear missed commission (no penalty fee) |
| Re-listing after strike 3 reappearance | $20 (and ALL future matches cost $20 — no honesty discount, perpetual) |
| Appeal evidence upload | Free |

### Refund / write-off policy

| Scenario | Outcome |
|---|---|
| Parent says NO (no match happened) | Tutor not charged. |
| Both parties silent for 30 days | Auto-close, no charge. |
| Tutor wins appeal (parent's NO overturned) | Tutor charged $20 standard (no honesty discount — they didn't self-report in time). |
| Tutor loses appeal | Strike applied (1, 2, or 3). |
| Permanent purchase, refund requested within 14 days | Full refund via Stripe, permanent flag removed, per-match commission resumes. |
| Permanent purchase, after 14 days | No refund. |
| 3rd-strike permanent suspension with Permanent upgrade | Permanent fee forfeited. No refund. (Discourages bad-faith permanent purchases.) |
| Stripe payment fails (declined card, expired etc.) | Tutor's listing hidden pending payment method update; profile reappears once payment succeeds. |

---

## For maintainers (humans and AI agents) — hand-off guide

If you're a future Claude / human picking up this project mid-build, **read this section first**. It's the source of truth for direction, conventions, and "what's about to happen next".

### Quick state check (do this first)

1. **What direction is the project going?** See the **🔄 Direction** banner at the top of this README. Currently: **pivoting from verified-marketplace to pure directory**.
2. **What's the current code state?** Check the most recent commit message. If it ends with "Session X complete", that's the last finished session. If it's docs-only, the pivot rework hasn't started yet.
3. **What's pending?** See the **Pivot work plan** below. Each session has a Definition of Done — start the next session whose DoD isn't yet satisfied.

### How to start the next session

1. Read the README in full, in order. Especially:
   - **The new model** (mechanics)
   - **Pricing & monetization** (canonical $)
   - **Engineering principles** (architectural requirements)
   - **Pivot work plan** session-by-session
   - **Decision log** below (why we chose what we chose)
2. Read the user's most recent message in the chat for any updates not yet in the README. If something they said contradicts the README, *ask before assuming the README is right*.
3. Identify the next unfinished session (check the DoD lists). Confirm with the user before starting if the README's plan vs the user's last instruction don't line up.
4. Use TodoWrite to break the session into todos.
5. After each meaningful change, commit + push to `origin/main` immediately. **The user's memory has an explicit rule for this: "Push every update to GitHub".** Don't batch.
6. When the session is done, update this README:
   - Tick the session's DoD items off
   - Move shipped items in the "Pivot work plan" or "Roadmap" from `[ ]` to `[x]`
   - Update the routes table state icons
   - Update the Status section bullets if a feature is now live

### Conventions used in this codebase

- **File organization:**
  - `src/app/[route]/page.tsx` — server components (default)
  - `src/app/[route]/route.ts` or `src/app/api/[path]/route.ts` — API handlers
  - `src/components/[feature]/ComponentName.tsx` — React components grouped by feature
  - `src/lib/[utility].ts` — shared utilities (db helpers, validators, etc.)
- **Component conventions:**
  - Server components by default; `"use client"` directive only when interactivity is needed
  - PascalCase for component names, camelCase for utilities
  - Each client component imports its own hooks at top of file
- **Styling:**
  - `src/app/globals.css` is the single CSS file. Component-specific classes are namespaced (`thread-row`, `dashboard-card`, etc.). No CSS modules.
  - CSS variables drive theming (`--brand`, `--brand-deep`, `--brand-soft`, etc.). Don't hardcode colours.
  - Mobile-first responsive — breakpoints at 520, 900, 1100.
- **State / data flow:**
  - `useState` for component state; no Redux/Zustand
  - Server components fetch data directly; client components fetch via `fetch('/api/...')`
  - `src/lib/db.ts` is the JSON-store layer that mimics what Prisma+Postgres will look like
- **Forms:**
  - zod for client AND server validation. The schemas live in `src/lib/` so they're shared.
  - Client-side validation is UX; server-side is authoritative
- **Commit messages:**
  - Multi-paragraph descriptive. No Claude attribution footer (user prefers it without).
  - Lead line: short imperative description ("Build X", "Fix Y", "Add Z"). Following paragraphs: what + why.
  - HEREDOC style (see existing commits for examples).
- **Push discipline:**
  - Every commit goes to `origin/main` immediately. No local-only batching.
  - This is in the user's saved memory and is non-negotiable for this project.
- **README maintenance:**
  - When you ship a roadmap item, tick it off in the same commit. Don't separate doc updates from feature commits.
  - This is also in the user's saved memory.
- **Asking the user before changing direction:**
  - The README is the source of truth for the plan
  - If you have a better idea, ask before implementing — don't unilaterally pivot mid-session

### Decision log — why we chose what we chose

Each row is a decision that's already been made. Don't re-litigate unless the user asks.

| Decision | Reasoning |
|---|---|
| Pure directory model (not verified marketplace) | Drops legal-exposure tier from $50k-200k defence-cost range to $10k-30k. Loses some product differentiation but keeps the "pay-per-real-match" economics. |
| No in-platform chat | Removes platform-mediated communications → less duty of care, no "what we knew" liability about parent-tutor conversations |
| Parent pays $0, tutor pays commission | Avoids creating a consumer contract with parents (which would trigger Australian Consumer Law guarantees). Aligns with Hipages / Indeed model. |
| First match free + $20 commission afterward | Aligns cost with value delivered. Lower friction than pay-to-list. |
| $15 honesty discount | The $5 savings rewards self-reporting more than the $5 saves; better than relying on parent prompt response rates |
| $60 permanent option (3-match break-even) | Serves high-volume tutors; predictable revenue for platform; clear value proposition (4th student onwards = savings) |
| No "featured" or editorial sort | Algorithmic ordering only. Editorial selection creates editorial liability. |
| WWCC collected privately, never displayed as "verified" | Avoid verification representation that creates duty of care. Tutor uses their own WWCC to satisfy parents directly. |
| Strike system: 7d / 30d / permanent (no penalty fee on strike 2) | Honest mistakes recoverable. Third strike sticky enough to deter habitual dishonesty. |
| Appeal via evidence + manual admin review (bank-transfer name match = auto-pass) | Protect honest tutors against parents who lie. Auto-pass for unambiguous cases minimises admin work. |
| Auto-approve tutors after phone OTP + email verify + content scan + age tickbox | Removes admin bottleneck. Manual review only for flagged content. |
| Idempotent endpoints + cron-driven scheduling | Reliable automation without manual intervention or worry about retries |
| 7-year audit log retention | Standard for AU legal records + ATO. Append-only for evidence integrity. |
| Self-service data export + deletion | Privacy Act compliance + reduces support workload |

---

## Engineering principles — locked in for the build

These principles drive every design decision in the pivot. They are not aspirations; they're requirements that shape the code.

### Legal-risk minimization (architectural, not just disclaimers)

1. **No editorial judgment.** Listing order is algorithmic only (oldest, score, distance). No human-curated "recommended", "featured", or "best" anywhere. The moment we curate, we accept editorial responsibility.
2. **Minimum platform knowledge.** We track only what's needed to operate (match status, payment status, strike count). No lesson outcomes, no lesson notes, no parent-tutor conversation content. Less knowledge = less implied duty of care.
3. **Audit trail on every action.** Signup, listing, contact request, dispute, payment — each logs IP, user agent, timestamp, terms version accepted, into an append-only audit table. Invisible to users; critical for any future dispute.
4. **No platform-mediated communications.** Once contact details are revealed, parent and tutor talk off-platform. We don't host the conversation so we don't gain knowledge that creates duty.
5. **Listings are user content.** We don't edit them. We don't curate them. We only remove on report, and document the removal.
6. **WWCC + ID stay private to the tutor.** Uploaded for the tutor's own use ("so you have them ready to give parents directly"). Never displayed publicly. Platform never claims they verify anything. Parents are directed to verify with NSW OCG directly.
7. **Self-attestation under audit risk.** Tutors tick checkboxes affirming age, WWCC validity, conduct, etc. False attestation = strike + suspension + records preserved for any authority that asks.
8. **Records retention is automated.** 7 years for transactional records (ATO + legal-defence compliance); shorter for personal data; all auto-purged on schedule.

### Minimum-supervision principles

1. **Default to self-service** everywhere — onboarding, edits, pause, delete, dispute submission.
2. **Auto-approve unless flagged.** New tutors live immediately after passing automated checks (phone OTP, email verify, bio content scan, age tickbox). Manual review only for flagged content. Most signups never touch a human.
3. **Auto-charge via Stripe Connect.** When commission is owed, tutor's saved card is charged automatically. No manual invoicing, no reconciliation.
4. **Auto-suspend on triggers.** Strikes accumulate automatically. Multiple reports about one tutor in a rolling 30-day window = auto-suspended pending review.
5. **Auto-resolve stale disputes.** Match opened, neither party responds within 30 days → auto-close, no charge. Default = no transaction happened.
6. **Cron-driven scheduling.** Every time-based event is a Vercel Cron job. Zero manual triggers.
7. **Three admin queues only:** (a) Spam/abuse moderation, (b) Appeals (tutor disputes a parent's "no" with evidence), (c) Schools CRUD. Realistic admin time at small scale: 15–30 min/day.
8. **Tutor self-reports as the primary path.** System actively encourages it (fast reappearance + $5 discount). Honest tutors typically skip the parent-confirmation loop entirely.
9. **Idempotent endpoints.** Safe to retry. Cron and webhooks can be aggressive without causing duplicate charges or broken state.
10. **Observability over intervention.** Dashboards show patterns (match volume, dispute rate, suspension rate). System runs without watching.

### Implementation choices that follow from these principles

| Choice | Reason |
|---|---|
| **Stripe Connect Express** for tutor accounts | Handles saved cards, charges, refunds, AU tax compliance. No manual money movement. |
| **Phone OTP at signup** (Twilio Verify or similar) | Cheap fraud prevention. Stops automated tutor signup spam. ~$0.07 per verification. |
| **Cloudflare Turnstile** on signup + contact-request forms | Bot prevention without captcha UX friction |
| **Resend for ALL transactional email** | Templated; never manual; includes delivery audit trail |
| **Vercel Cron Jobs** for scheduled tasks | Built into hosting; no separate worker service |
| **Dedicated audit log table** (append-only) | Immutable, easy to query, exportable for compliance |
| **No in-platform messaging** | Directory ≠ communications platform. We reveal contact details and step out. |
| **Algorithmic listing order only** | Removes editorial liability. Sort = function of public data, not human choice. |
| **Content scanning on bios** (regex first, optional ML later) | Catches obvious spam/scam patterns without manual review |
| **Soft-delete with retention** | Removed listings stay queryable for 7 years for audit, hidden from public |

---

## Data model changes (canonical — implement these in session 2 and 3)

The pivot replaces the `Unlock` + `Message` entities with `Match` + `Appeal`. The `TutorApplication` type also grows several fields and loses a few.

```ts
// REMOVE entirely
type Unlock = {...};   // gone — no parent-side payment
type Message = {...};  // gone — no in-platform chat

// ADD: Match (replaces Unlock)
type MatchStatus =
  | "AWAITING_RESOLUTION"        // created when parent clicks "I want this tutor"
  | "RESOLVED_TUTOR_CONFIRMED"   // tutor self-reported within 48h → $15 charged
  | "RESOLVED_PARENT_CONFIRMED"  // parent said YES after 48h → $20 charged
  | "RESOLVED_NO_MATCH"          // parent said NO or both confirmed no-match → no charge
  | "RESOLVED_APPEALED_WON"      // tutor appealed parent's NO and won → $20 charged
  | "RESOLVED_APPEALED_LOST"     // tutor appealed and lost → strike applied
  | "AUTO_CLOSED_NO_RESPONSE";   // 30d silence both sides → no charge

type Match = {
  id: string;                          // app_match_<random>
  parentEmail: string;                 // captured even if parent isn't logged in
  parentUserId?: string;               // set if parent is signed in (rare in directory model)
  tutorApplicationId: string;
  tutorUserId: string;
  status: MatchStatus;
  createdAt: string;                   // ISO timestamp
  tutorHiddenUntil: string;            // createdAt + 48h
  resolvedAt?: string;
  amountChargedCents?: number;         // 1500 or 2000
  stripeChargeId?: string;
  parentConfirmation?: "YES" | "NO" | "NOT_YET";
  parentConfirmedAt?: string;
  tutorSelfReport?: "YES" | "NO";
  tutorSelfReportedAt?: string;
  parentConfirmTokenHash?: string;     // SHA-256 of the email-link token
  parentConfirmRemindersSent: number;  // 0, 1, 2, 3 (at 2d, 7d, 14d, 30d)
  appealId?: string;
  isFreeFirstMatch: boolean;           // true if tutor's matchesCompletedCount was 0
};

// ADD: Appeal
type Appeal = {
  id: string;
  matchId: string;
  tutorUserId: string;                 // the tutor making the appeal
  description: string;                 // tutor's explanation
  evidenceUploadIds: string[];         // refs to /api/uploads/[id] files
  status: "PENDING" | "APPROVED" | "REJECTED";
  reviewerEmail?: string;
  reviewerNotes?: string;
  createdAt: string;
  resolvedAt?: string;
};

// MODIFY: TutorApplication
type TutorApplication = {
  // existing fields kept: id, userId, firstName, lastInitial, fullLastName, publicBio, etc.

  // ADD
  matchesCompletedCount: number;       // for first-match-free logic. 0 until first paid match clears.
  strikeCount: number;                 // 0, 1, 2, 3+
  strikeHistory: {                     // immutable audit
    date: string;
    matchId: string;
    reason: string;
  }[];
  permanentListing: boolean;           // $60 upgrade
  permanentListingPurchasedAt?: string;
  permanentListingStripeChargeId?: string;

  stripeConnectAccountId?: string;     // Stripe Connect Express account
  stripeDefaultPaymentMethodId?: string;

  phoneNumber: string;                 // required, OTP-verified
  phoneVerifiedAt?: string;            // null until OTP completes

  attestations: {
    is18Plus: boolean;
    hasValidWWCC: boolean;
    willProvideWWCCToParents: boolean;
    isAccurateProfile: boolean;
    acceptedAt: string;
    acceptedTermsVersion: string;
  };

  hiddenUntil?: string;                // tutor's listing temp-hidden (during 48h match window OR strike period)

  // REMOVE (was indemnity-specific to pre-pivot model)
  // termsAcceptedVersion, termsAcceptedAt
  // bioFlags (replaced by content scanner output stored on review record)
};

// ADD: AuditLog (append-only)
type AuditLog = {
  id: string;
  userId?: string;                     // null for anonymous parent actions
  ip: string;
  userAgent: string;
  timestamp: string;                   // ISO
  action: string;                      // "TUTOR_SIGNUP", "MATCH_REQUESTED", "PARENT_CONFIRMED_YES", etc.
  targetType?: string;                 // "match", "application", "user", "appeal"
  targetId?: string;
  metadata?: Record<string, unknown>;  // anything else relevant
  termsVersionAccepted?: string;
};
```

## API contracts (canonical — implement these in session 2 and 3)

### Match flow

```http
POST /api/matches/request
Auth: optional (parent may or may not be signed in)
Body: { tutorApplicationId: string, parentEmail: string }
Returns: 201 { matchId: string, tutor: { firstName: string, fullName: string, phone: string, email: string } }
Side effects:
  - Create Match (status AWAITING_RESOLUTION)
  - Set tutorApplication.hiddenUntil = now + 48h
  - Send tutor notification email ("[Parent email] selected you — self-report within 48h for $15")
  - Schedule parent-confirmation cron at +48h
  - Audit log entry

POST /api/matches/[id]/self-report
Auth: required (tutor must own the match)
Body: { confirmed: boolean }
Returns: 200 { ok: true, charged?: { amountCents: 1500 } }
Side effects:
  - confirmed=true: Stripe charge $15 (unless free first match), match.status = RESOLVED_TUTOR_CONFIRMED, clear hiddenUntil
  - confirmed=false: match.status = RESOLVED_NO_MATCH, clear hiddenUntil
  - Increment matchesCompletedCount if confirmed
  - Audit log entry

POST /api/matches/[id]/parent-confirm
Auth: none required (token-based — signed link from email)
Body: { confirmation: "YES" | "NO" | "NOT_YET", token: string }
Returns: 200 { ok: true }
Side effects:
  - Validate token against match.parentConfirmTokenHash
  - YES + tutor hasn't self-reported: Stripe charge $20, match.status = RESOLVED_PARENT_CONFIRMED, increment matchesCompletedCount
  - YES + tutor self-reported NO: open appeal window (24h), notify tutor
  - NO + tutor self-reported YES: open appeal window (24h), notify tutor
  - NO + no tutor input: match.status = RESOLVED_NO_MATCH
  - NOT_YET: increment parentConfirmRemindersSent, schedule next reminder
  - Audit log entry

POST /api/matches/[id]/appeal
Auth: required (tutor)
Body: { description: string, evidenceUploadIds: string[] }
Returns: 201 { appealId: string }
Side effects:
  - Create Appeal record (status PENDING)
  - Match.appealId set
  - Auto-check: if any evidence file is bank-transfer-like AND name match found, suggest auto-approve in admin UI
  - Notify admin via dashboard + weekly digest
  - Audit log entry

PATCH /api/admin/appeals/[id]
Auth: admin only
Body: { decision: "APPROVED" | "REJECTED", notes?: string }
Returns: 200 { ok: true }
Side effects:
  - APPROVED: match.status = RESOLVED_APPEALED_WON, charge $20 to tutor
  - REJECTED: match.status = RESOLVED_APPEALED_LOST, increment tutor.strikeCount
  - Audit log entry

GET /api/cron/match-checkin
Auth: cron secret only
Returns: 200 { processed: number }
Side effects:
  - For matches past 48h without resolution → send parent confirmation email (if reminders sent < max)
  - For matches at day 30 with no resolution → auto-close (AUTO_CLOSED_NO_RESPONSE)
  - For listings past hiddenUntil → ensure they're public again
  - For tutors at end of strike period → restore visibility
```

### Permanent listing

```http
POST /api/tutor/permanent
Auth: required (tutor)
Returns: 200 { stripeCheckoutUrl: string }
Side effects: Create Stripe Checkout session for $60, redirect tutor to pay.

POST /api/stripe/webhook (existing route, extend)
Handles: checkout.session.completed for permanent purchase
Side effects:
  - Set tutor.permanentListing = true
  - Record purchase date + Stripe charge ID
  - Audit log entry

POST /api/tutor/permanent/refund
Auth: required (tutor)
Returns: 200 { ok: true }
Side effects:
  - Validate purchase was within last 14 days
  - Stripe refund the $60
  - Set tutor.permanentListing = false
  - Audit log entry
```

### Auto-approval flow (session 4)

```http
POST /api/auth/signup
(existing, but extend to require phone in body)
Body: { email, password, phone, role: "TUTOR" | "PARENT" }
Returns: 201 { userId: string, otpRequiredForPhone: true }
Side effects:
  - Create user (phoneVerifiedAt = null)
  - Send phone OTP via Twilio Verify
  - Audit log entry

POST /api/auth/verify-phone
Body: { code: string }
Returns: 200 { verified: true }
Side effects:
  - Mark phoneVerifiedAt = now
  - Audit log entry

POST /api/tutor/applications
(existing, but extend to require attestations + run content scanner)
Body: { ...existing fields, attestations: { is18Plus, hasValidWWCC, willProvideWWCCToParents, isAccurateProfile } }
Returns: 201 { applicationId, status: "AUTO_APPROVED" | "PENDING_REVIEW" }
Side effects:
  - Validate all attestations === true (else reject)
  - Run bioContentScanner(publicBio) — if returns flags, status = PENDING_REVIEW
  - If no flags AND phone verified: AUTO_APPROVED, immediately public
  - Audit log entry with attestations payload
```

### Automation routes (session 5)

```http
GET /api/cron/match-checkin              # every 15 min
GET /api/cron/strike-expiry              # daily
GET /api/cron/inactivity-hide            # daily — hide tutors with no activity 90d
GET /api/cron/retention-purge            # weekly — delete records past retention
POST /api/admin/export-audit             # one-off CSV export for compliance
GET /api/admin/health                    # dashboard data (counts, rates)
GET /api/user/data-export                # self-service: returns JSON of all user's data
DELETE /api/user/account                 # self-service: soft-delete account + 30d grace period
```

---

## Session 0 — Launch blocker remediation (do these before anything else)

> This section is the fix plan for the four blockers listed at the top of the README. It is written to be followed by a worker (human or AI) with no other context. Do the 🔴 items before any public deploy. Each work order is self-contained: **Goal → Files → Steps → Acceptance → Guardrails.** After each work order, run `npm run typecheck && npm run build` and commit (see *Push discipline* in the hand-off guide). Do not batch all four into one commit — one commit per blocker so they're reviewable.

**Before you start, one product decision is needed for Blocker 1** (does the platform allow 16–17-year-old tutors at all?). If the answer isn't already recorded in the chat or decision log, **ask the founder before writing code** — it changes which option you implement. Default if no answer: Option A (hard reject under-18), which needs no lawyer.

---

### Blocker 1 🔴 — Under-18 tutors can list, but Terms promise they're rejected

**Goal:** No under-18 applicant can produce a live listing, AND the Terms + Privacy copy exactly match whatever behaviour ships. Today `POST /api/tutor/applications` auto-approves any clean listing regardless of date of birth, while Terms §11 still says under-18 applications are "automatically rejected." Close that gap.

**Files:**
- `src/lib/tutor-form.ts` — the `ageInYears(isoDate)` helper already exists (near the bottom); `tutorApplicationSchema` is the zod schema; `tutorIndemnityAccepted: z.literal(true, …)` is the checkbox-field pattern to copy.
- `src/app/api/tutor/applications/route.ts` — `statusFromScan()` and the POST + PUT handlers (the auto-approve lives here).
- `src/components/tutor/SignupForm.tsx` — the `indemnity-accept` `<label>` block (Section 9 of the form) is the checkbox UI to mirror.
- `src/app/legal/terms/page.tsx` (§11) and `src/app/legal/privacy/page.tsx` (the "Children" paragraph).

**Steps — Option A (hard reject under-18, RECOMMENDED, no counsel needed):**
1. In `src/app/api/tutor/applications/route.ts`, add `import { ageInYears } from "@/lib/tutor-form";`.
2. In **both** the POST and PUT handlers, immediately after `const v = parsed.data;`, add:
   ```ts
   const age = ageInYears(v.dateOfBirth);
   if (age < 18) {
     const now = new Date().toISOString();
     const rejected = { /* build the same application object as the normal path, but: */
       status: "REJECTED" as ApplicationStatus,
       reviewedAt: now,
       reviewerEmail: "auto",
       reviewerNotes: `Auto-rejected: DOB ${v.dateOfBirth} → age ${age} < 18.`,
       // …all the same field mappings (firstName, dateOfBirth, wwcc*, etc.)…
     };
     await upsertApplication(rejected);
     return NextResponse.json({ ok: true, applicationId: rejected.id, status: "REJECTED" }, { status: 200 });
   }
   ```
   Persist the row (don't 400 / silently drop) so there is an audit trail of the attempt, matching what §11 and the audit-trail model promise.
3. Defence-in-depth: add a self-attestation field to the schema. In `tutor-form.ts`, next to `tutorIndemnityAccepted`, add:
   ```ts
   is18PlusAttested: z.literal(true, {
     errorMap: () => ({ message: "You must confirm you are 18 or older to list as a tutor." }),
   }),
   ```
4. In `SignupForm.tsx`, add `const [is18Plus, setIs18Plus] = useState(false);`, render a checkbox by copying the `indemnity-accept` label block (change the copy to "I confirm I am 18 years or older"), and include `is18PlusAttested: is18Plus` in the submit payload alongside `tutorIndemnityAccepted`.
5. Confirm no doc change is needed: Terms §11 already says "automatically rejected" (now true) and Privacy's "Children" paragraph says you don't knowingly collect under-18 data (now true). Leave both as-is.

**Steps — Option B (allow 16–17 with attestation — DO NOT ship without counsel):** do not reject; instead require the age attestation, add parental-consent handling, and **rewrite Terms §11, the Privacy "Children" section, and the Child Safety page** together. Note NSW under-18 workers are generally exempt from holding a WWCC, so the currently-mandatory WWCC fields (`wwccNumber`, `wwccFullName`, `wwccDob`) must become conditional for under-18 applicants. Flag for counsel sign-off before merge.

**Acceptance criteria:**
- [ ] POST and PUT with a DOB implying age < 18 → application stored with `status: "REJECTED"`; no `APPROVED` row is created; the tutor never appears in `/browse`.
- [ ] The response is `200` with `status: "REJECTED"` (not a `400`), so the client shows a rejection banner rather than a form error.
- [ ] POST/PUT with `is18PlusAttested` absent or `false` → `400` validation error.
- [ ] The statement in Terms §11 matches the behaviour that actually shipped (A or B).
- [ ] `npm run typecheck` and `npm run build` pass.

**Guardrails:** the server is authoritative — never rely on the client checkbox alone. Never silently discard an under-18 submission (that destroys the audit trail §11 relies on). Do not ship Option B without recorded counsel sign-off.

---

### Blocker 2 🔴 — PII / WWCC breach vectors (four sub-fixes)

**Goal:** Close the four cheap paths into the tutor identity/WWCC/ID store. Each sub-fix is independent; do all four.

**2a — `ADMIN_EMAILS` self-promotion at signup**
- File: `src/app/api/auth/signup/route.ts` (the `if (isAdminEmail(email)) role = "ADMIN"` line, ~line 30).
- Step: **delete that promotion.** Self-registration must never grant ADMIN. Admin accounts get their role set out-of-band — via a seed/migration script or an existing admin flipping the role in the store. Keep `isAdminEmail` for any server-side gate, but not as a signup side effect.
- Acceptance: registering an address listed in `ADMIN_EMAILS` yields a non-admin role; there is no self-serve path to ADMIN.

**2b — hardcoded session-secret fallback**
- File: `src/lib/session.ts`, the `secret()` function (~line 8).
- Step: make production refuse the public dev fallback:
  ```ts
  function secret(): string {
    const s = process.env.NEXTAUTH_SECRET;
    if (s && s.length >= 16) return s;
    if (process.env.NODE_ENV === "production") {
      throw new Error("NEXTAUTH_SECRET must be set (>=16 chars) in production.");
    }
    return "dev-only-not-secure-replace-me-in-env-local";
  }
  ```
- Acceptance: with `NODE_ENV=production` and no `NEXTAUTH_SECRET`, the app throws instead of signing cookies with the public constant; dev is unchanged.

**2c — unauthenticated, unthrottled contact endpoint**
- File: `src/app/api/matches/request/route.ts`.
- Steps: (i) require a session — call `getSession()` and return `401` if absent (the match flow already anticipates "account creation may be needed at this point"). (ii) Rate-limit per user + IP; implement the Session 5 target of **10 contact-requests/hour/user** with a minimal limiter now (in-memory map or a store-backed counter), return `429` when exceeded. (iii) Only return the tutor's `phone`/`email` in the JSON to an authenticated parent. Preserve the existing dedupe + 48h `hiddenUntil` logic.
- Acceptance: unauthenticated POST → `401`; more than the hourly limit → `429`; tutor phone/email is never returned to an anonymous caller.

**2d — Privacy claims encryption the storage doesn't have**
- Files: `src/app/legal/privacy/page.tsx` (the "stored encrypted at rest" sentence), context in `src/lib/uploads.ts` (local filesystem).
- Step: until real encrypted object storage exists (Technical roadmap — "Real storage (Supabase/S3) with at-rest encryption"), change the copy to describe reality, e.g. "documents are stored with admin-only access via signed URLs; at-rest encryption is being implemented before public launch." **Alternatively**, disable the document-upload UI until encrypted storage lands. Do not claim encryption you don't have.
- Acceptance: the Privacy page contains no unqualified "encrypted at rest" claim while uploads live on local disk.

**Guardrails (all of 2):** these are security fixes — don't "work around" them (e.g. don't leave a debug bypass). If you add a rate limiter, make it fail-closed on its own errors, not fail-open.

---

### Blocker 3 🟠 — Legal pages describe systems that don't exist / fees not in Terms

**Goal:** Make the legal copy true, and make sure every amount the code can charge is in the contract the tutor accepted.

**3a — audit-trail claims**
- Files: `src/app/legal/terms/page.tsx` (§5) and `src/app/legal/child-safety/page.tsx` assert IP / user-agent / audit-log retention; `src/app/api/tutor/applications/route.ts` currently stores none of that.
- Step (recommended): start capturing it now — on application create, read `req.headers.get("x-forwarded-for")` (fall back to the connecting IP) and `req.headers.get("user-agent")`, and persist them plus the already-stored `termsAcceptedVersion` on the application row. That makes the "self-attestation under audit risk" model real. If you'd rather defer, instead soften the copy to claim only what's stored (the accepted Terms version) and drop the IP/UA claims until Session 5.
- Acceptance: the audit-trail language in Terms/Child-Safety matches what a row actually contains.

**3b — strike / reinstatement fees missing from Terms**
- Files: `src/lib/strike.ts` charges a **$20 reinstatement** to reappear after strikes 1–3 and forces the **perpetual $20 rate (no honesty discount)** after strike 3; `src/app/legal/terms/page.tsx` §2/§3 don't mention any of it.
- Step: add the strike schedule and its fees to Terms §2/§3 — the 7-day / 30-day / permanent hide, the $20-to-reappear fee, and the permanent loss of the honesty discount after the third strike. A fee can't be charged unless the tutor agreed to it.
- Acceptance: **every** amount the code can charge appears in the Terms — commission $20/$15, Permanent $60, reinstatement $20, and post-strike-3 $20. Cross-check against `commissionCents()` and `applyStrike()` in `src/lib/strike.ts`.

---

### Blocker 4 🟠 — Real schools published live without permission

**Goal:** No real-named school is publicly active until permission evidence is on file.

**Files:** `src/lib/schools.ts` (`SEED_SCHOOLS` — Killara High School and Masada College, both `active: true`) and `prisma/seed.ts` (writes them live).

**Steps:**
1. In `src/lib/schools.ts`, set `active: false` on both seed schools.
2. In `prisma/seed.ts`, make the upsert refuse to set `active: true` for any school lacking a `permissionEvidenceUrl` (the field already exists on the `School` model). Schools become publicly active only after an admin adds permission evidence via `/admin/schools`.

**Acceptance:**
- [ ] A fresh seed run leaves no real-named school publicly active.
- [ ] Activating a school requires permission evidence recorded on the row.

---

### Definition of Done (Session 0)
- [ ] Blockers 1 and 2 (all 🔴) fixed, each in its own commit, `npm run build` green.
- [ ] Blockers 3 and 4 (🟠) fixed or explicitly deferred **with founder sign-off recorded in the decision log**.
- [ ] Terms §11 and the strike-fee schedule reconciled with the code; Privacy encryption claim reconciled with storage reality.
- [ ] For Blocker 1 Option B only: counsel sign-off recorded before merge.
- [ ] Top-of-README blocker table updated to strike through cleared items (leave the row, mark it ✅ done, so the history is visible).

---

## Pivot work plan — what changes in the code

Realistic estimate: **4–5 focused sessions of work**. Below is the session breakdown. Each session bakes in the engineering principles above — automation and legal-risk minimization are not separate work, they're built into every step.

Each session has a **Definition of Done (DoD)** at the end — tick items off as they ship. **Don't move to the next session until the previous one's DoD is fully satisfied.**

### Session 1 — Content audit + remove all verification claims ✅ COMPLETE

Goal: strip the verified-marketplace language site-wide so the directory framing is consistent.

| File / location | Change |
|---|---|
| `src/components/landing/Hero.tsx` | Remove "verified tutor" framing; reframe as "find a local tutor" |
| `src/components/landing/Trust.tsx` | **Delete** the whole Trust section ("WWCC verified", "Government ID checked", etc.) |
| `src/components/landing/Comparison.tsx` | Remove rows that hinge on verification; replace with "listed locally" angle |
| `src/components/landing/FAQ.tsx` | Rewrite verification-related FAQs into "you verify yourself" answers |
| `src/components/landing/LandingPage.tsx` | Remove Trust from the section order |
| `src/components/school/SchoolBrowse.tsx` | Remove "X verified tutors live" counter; replace with "X tutors listed" |
| `src/app/tutors/[id]/page.tsx` | Remove anything implying TUTUMatch vouches for the tutor |
| `src/app/legal/terms/page.tsx` | Rewrite Sections 1, 5, 7 to reflect directory positioning — no verification claims, no consumer relationship with parents |
| `src/app/legal/child-safety/page.tsx` | Reframe as "what we ask tutors to have" rather than "what we verify" |
| `src/components/landing/Footer.tsx` | Disclaimer block already mostly aligned — small tweaks |
| `src/app/how-it-works/page.tsx` | Rewrite both flows for the new model |
| `src/components/landing/Earnings.tsx` | Reframe "$50 in your pocket" with the "first student free, $20 thereafter" framing |
| New: prominent **"What TUTUMatch is and isn't"** page or banner | "We're a classifieds directory. Listings are tutor-provided. We don't verify anything. Parents verify tutors directly." |

**Definition of Done (Session 1):**
- [ ] `grep -ri "verified|vetted|screened|wwcc verified" src/` returns no marketing/UI uses — _partial: `SignupForm.tsx` still carries verification copy; deferred to Session 4's WWCC reframe_
- [x] Trust component file deleted; LandingPage no longer imports it
- [x] Hero copy uses the canonical headline ("List free. Pay only when you get a student. First match is on us.")
- [x] Earnings card uses the new framing — no "verified" claims
- [x] Comparison table rewritten — no verification-related rows
- [x] FAQ entries rewritten to "you verify yourself" framing
- [x] `/legal/terms` Sections 1, 5, 7 rewritten to directory framing
- [x] `/legal/child-safety` reframed to "what we ask tutors to have"
- [x] `/how-it-works` updated to new flow
- [x] New `/what-we-are` page exists, linked from footer + topnav
- [x] Schools browse hero subtitle removed "X verified tutors live" — replaced with "X tutors listed"
- [x] `npm run build` passes with no warnings about removed components
- [x] **Commit message lead:** "Session 1: strip verification claims site-wide for directory pivot"

### Session 2 — Remove parent payment flow + introduce "I want this tutor" intent capture ✅ COMPLETE

Goal: rip out the unlock-fee transaction; introduce the free-intent-button flow that triggers the 48h hidden window.

| Change | Files |
|---|---|
| **Delete** the unlock confirm page in favour of an "I want this tutor — see contact details" page (no payment, just reveals contact) | `src/app/unlock/[tutorId]/page.tsx` → replace with `src/app/contact/[tutorId]/page.tsx` (or rename route) |
| **Delete** the `ConfirmUnlockButton` payment wiring | `src/components/unlock/ConfirmUnlockButton.tsx` |
| **Delete** parent-side dev unlock shortcut | `src/app/api/unlocks/dev-create/route.ts` |
| **Rewrite** the Unlock data type → `Match` (no money on parent side; tracks the 48h window + later commission flow) | `src/lib/db.ts` |
| **Add** "request contact" API that creates a Match record, hides tutor profile temporarily, notifies tutor | New `src/app/api/matches/request/route.ts` |
| **Add** tutor notification flow ("[Parent] selected you — self-report for $15 + faster profile reappear") | New page + email template stub |
| **Strip** all of `/legal/terms` references to the $20 unlock fee + 5-day parent refund mechanic | `src/app/legal/terms/page.tsx` |
| **Strip** parent-facing refund language site-wide | Multiple files |
| **Remove** the in-platform chat thread feature entirely | Delete `src/app/messages/`, `src/components/messages/Thread.tsx`, related APIs |
| **Remove** the refund + auto-suspension flow (replaced by the strike system in session 3) | Multiple files |

**Definition of Done (Session 2):**
- [x] `/unlock/[tutorId]` route deleted
- [x] `/contact/[tutorId]` route exists; renders tutor's public profile + a "I want this tutor" form (email capture only, no payment)
- [x] `POST /api/matches/request` exists and creates a Match record per the contract above
- [x] On match request, the tutor's listing is hidden from browse for 48h (tutorApplication.hiddenUntil set)
- [x] Tutor receives a notification (placeholder email to console.log for now — wired to Resend in session 5)
- [x] `/messages` + `/messages/[unlockId]` routes deleted; Thread component deleted
- [x] All "$20 unlock fee", "refund", "5-day refund" language stripped from Hero, Terms, FAQ, how-it-works, footer
- [x] `/api/unlock`, `/api/refund`, `/api/cron/refund-flag`, `/api/unlocks/dev-create`, `/api/unlocks/[id]/fast-forward` deleted
- [x] `/admin/refunds` route deleted
- [x] `Unlock`, `Message`, `ConfirmUnlockButton` files deleted
- [x] `Match`, `Appeal` types added to `src/lib/db.ts` per spec
- [x] `grep -ri "Unlock" src/` returns nothing or only legacy/incidental (only the decorative `UnlockIcon`)
- [x] `npm run build` passes
- [x] **Commit message lead:** "Session 2: replace parent-pays unlock with free 'I want this tutor' intent capture + Match data model"

### Session 3 — Confirmation flow + strike system + tutor commission 🟡 MOSTLY COMPLETE

Goal: build the post-match resolution flow (self-report, parent confirmation, strike system, payment).

| Change | Notes |
|---|---|
| **Add** tutor self-report endpoint + dashboard button ("I had a lesson with this parent — confirm match for $15") | Triggers Stripe charge of $15 |
| **Add** scheduled parent-confirmation prompt at 48h after the match request | Email + on-site banner. Single-click "Yes / No / Not yet" buttons. |
| **Add** parent-confirmation API | Triggers $20 charge to tutor on yes; no charge on no. |
| **Add** retry schedule for unresponsive parents | 7 days, 14 days, 30 days. Then auto-ask tutor. |
| **Add** strike system tracking on user record | New fields: `strikeCount`, `strikeHistory`, etc. |
| **Add** strike actions: hide profile for 7 days (strike 1) → 30 days (strike 2) → permanent + $20-per-match perpetual (strike 3+) | Includes a "pay the missed $20 to reappear" mechanism for strike 1/2 |
| **Add** Stripe Connect integration so tutors have a saved payment method (required to list once past their first free match) | Replaces the current Stripe-stub |
| **Add** appeal endpoint for tutors who dispute a no-match parent reply | Upload evidence → admin review |
| **Add** admin view: pending appeals queue with evidence images | Extends `/admin` |
| **Add** permanent listing purchase flow (Stripe Checkout, $60 one-time) | New endpoint + dashboard button |
| **Add** 14-day refund window enforcement for permanent purchases | API + UI |

**Definition of Done (Session 3):**
- [x] Tutor dashboard shows pending matches with a "Self-report — did you book a lesson?" button on each
- [x] `POST /api/matches/[id]/self-report` works, charges (currently via the simulation layer in `src/lib/stripe.ts`), clears hiddenUntil
- [x] Parent receives confirmation email at 48h (`console.log` stub today; wired to a real provider in session 5)
- [x] Parent-confirm email link works without auth — token-validated
- [x] `POST /api/matches/[id]/parent-confirm` charges $20 on YES, no charge on NO, schedules reminder on NOT_YET
- [x] Strike system: `strikeCount` increments correctly; profile hidden for 7d on strike 1, 30d on strike 2, permanent on strike 3+
- [ ] Strike re-listing: paying the missed $20 lifts the temporary hide
- [x] Strike 3+: per-match cost is $20 with no honesty discount (codified via the `noHonestyDiscount` flag)
- [x] `POST /api/matches/[id]/appeal` works; evidence uploads stored; admin queue at `/admin/appeals` displays pending appeals
- [x] Admin can approve/reject appeal; approval charges $20, rejection applies strike
- [ ] Bank-transfer name-match check: simple OCR-or-filename pattern detection that surfaces "looks like a name match" hint to admin (doesn't auto-approve, just suggests)
- [ ] Permanent listing: `POST /api/tutor/permanent` creates Stripe Checkout; webhook handles completion; tutor.permanentListing = true — _deferred until `STRIPE_SIMULATE=false`_
- [ ] Permanent refund within 14 days works via `/api/tutor/permanent/refund` — _deferred with Permanent purchase_
- [ ] Permanent-listing tutors are skipped from commission charges — _goes in the real-Stripe charge path_
- [x] `npm run build` passes
- [x] **Commit message lead:** "Session 3: build the match-confirmation flow, strikes, appeals, and tutor-side Stripe commission" — _shipped across `dac8582` (non-Stripe) + `2c24f0b`/`f2ef4ff` (Stripe scaffold + simulation)_

### Session 4 — WWCC reframe + admin pivot + auto-approval flow 🟡 IN PROGRESS

| Change | Notes |
|---|---|
| **Reframe** tutor signup WWCC section as "we ask so you have it ready for parents" | `src/components/tutor/SignupForm.tsx` |
| **Reframe** WWCC display on the tutor's own dashboard (visible to them, not the public) | New dashboard widget |
| **Remove** WWCC from public profile display (parents see "ask the tutor for their WWCC" prompt instead) | `src/app/tutors/[id]/page.tsx` |
| **Reframe** admin approval queue as **spam/abuse moderation only**, NOT credential check | Admin pages |
| **Auto-approve flow** — new tutor signups go live immediately after passing: phone OTP, email verify, content scan, age tickbox. Only flagged content reaches admin. | New API + scanner |
| **Phone OTP integration** (Twilio Verify or similar) | `src/lib/otp.ts` + signup form |
| **Bot-prevention** — Cloudflare Turnstile on signup + contact-request forms | Layout + forms |
| **Bio content scanner** (regex first; ML later) for obvious scam/spam/contact-info patterns; auto-flags for admin if matched | `src/lib/content-scanner.ts` |
| **Remove** the "auto-reject under 18" auto-mechanism in favour of "you must self-declare 18+ to list" tickbox (we don't verify DOB ourselves) | `src/app/api/tutor/applications/route.ts` |
| **Remove** the document upload + admin verification block from the signup form | These become tutor-private optional uploads, not part of verification |
| **Update** the indemnity clause in Terms to reflect new model (lower platform exposure, narrower indemnity scope) | `src/app/legal/terms/page.tsx` |
| **Update** Privacy Policy to reflect the data we now collect (less, mostly tutor-side) | `src/app/legal/privacy/page.tsx` |
| **Update** `/how-it-works` to match the new flow with diagrams | `src/app/how-it-works/page.tsx` |
| Re-test all the disclaimers, footer copy, and the "What TUTUMatch is and isn't" page | All-pages pass |

**Definition of Done (Session 4):**
- [ ] Phone OTP integration (Twilio Verify or similar) works at signup — _blocked on Twilio account_
- [ ] Tutor cannot list until phone is verified — _depends on OTP_
- [ ] Cloudflare Turnstile (or equivalent) on signup form + contact-request form — _blocked on Cloudflare account_
- [x] Bio content scanner (`src/lib/content-scanner.ts`) flags scam/spam/contact-info patterns
- [x] New tutor signup with no flags: status = AUTO_APPROVED, instantly public
- [x] New tutor signup with flags: status = PENDING_REVIEW, admin sees it in queue
- [x] WWCC info on tutor signup form labelled "we ask so you have it ready for parents — we don't verify"
- [x] Tutor's own dashboard shows their WWCC info privately (for them to copy when a parent asks)
- [x] Public tutor profile page does NOT show WWCC info (or any platform-stamped verification)
- [x] `/tutors/[id]` shows a "Verify this tutor's WWCC yourself with the NSW OCG" link next to where verification was
- [x] Admin approval queue copy reframed to "spam/abuse moderation" — no credential review
- [x] Under-18 auto-reject **removed** (commit `73a83fa`)
- [ ] ⚠️ **NOT DONE — launch blocker 1:** the required age-attestation tickbox that was meant to *replace* the auto-reject was never built. Right now nothing gates DOB at all, yet Terms §11 still promises under-18 applications are "automatically rejected." Either reinstate the API-layer reject or ship the attestation **and** reconcile Terms §11 + Privacy before launch.
- [ ] Verification document upload block removed from signup form — _reframed as optional, tutor-private uploads (the README's softer reading) rather than deleted_
- [ ] Indemnity clause updated in Terms — narrower scope, directory framing — _partial: framing updated in Sessions 1–2; full legal rewrite left for lawyer review_
- [ ] Privacy Policy updated to reflect reduced data collection — _partial: stale verified-marketplace bits cleaned; full rewrite left for lawyer review_
- [x] `npm run build` passes
- [ ] **Commit message lead:** "Session 4: auto-approval flow, phone OTP, WWCC reframe, admin pivot to spam-only" — _multiple commits with subject-specific leads; OTP work deferred_

### Session 5 — Automation hardening + audit logging + retention policies

Goal: bake in the "minimum supervision" principles. After this session, the platform should run with 15–30 min/day of admin time at modest scale.

| Change | Why / Notes |
|---|---|
| **Audit log table** — append-only record of every user action (signup, listing, contact request, dispute, payment, terms acceptance). Captures IP + UA + timestamp + Terms version. | Critical evidence trail for any legal challenge; invisible to users |
| **Audit log middleware** for all API routes that mutate state | Automatic; no per-route work |
| **Rate limiting** on signup, contact request, dispute submission (IP + user-based) | Prevents abuse; upstash redis or Postgres-based |
| **Cron jobs** — at minimum: confirmation-prompt sender, retry scheduler, strike-period expiry checker, listing-inactivity hider, data-retention purger | All Vercel Cron, all idempotent |
| **Auto-suspension triggers** — N reports about same tutor in 30 days = auto-suspend pending review (configurable threshold) | Removes manual trigger work |
| **Auto-resolution timer** — match opened, no resolution in 30 days → auto-close with no charge | Default = no transaction happened |
| **Data retention policies** — 7yr for transactional records, shorter for personal data, automated purge | ATO + legal-defence compliance |
| **Self-service data export + deletion** — user-facing endpoint to download their data (JSON) or request account deletion (with retention exceptions for legal records) | Privacy Act compliance, GDPR-style |
| **Operational dashboard** at `/admin/health` — match volume, dispute rate, suspension rate, payment success rate, no manual reconciliation needed | Pattern observation, not intervention |
| **Compliance auto-report** — generate a periodic CSV of: terms-acceptance log, removals, suspensions, refunds, disputes | For if you ever need to demonstrate good-faith operation to a regulator or court |
| **Stripe Connect tax/GST handling** — once GST threshold is approached, automated alerting + integration with Stripe Tax | Reduces accountant work |
| **Email digest** — weekly automated summary to admin email: counts, anomalies, items needing attention | Replaces daily admin attention with weekly review |
| **Sentry integration** for error monitoring | Optional, ~free tier; spots issues without manual log-checking |

**Definition of Done (Session 5):**
- [ ] Audit log table exists in JSON store + Prisma schema; `appendAudit()` helper used by every API mutation route
- [ ] Audit log middleware records: action, userId (if any), IP, UA, timestamp, terms version
- [ ] Rate limiting in place: 5 signups/hour/IP, 10 contact-requests/hour/user, 3 disputes/day/user
- [ ] Vercel Cron configured for all schedules:
  - `*/15 * * * *` → match-checkin
  - `0 3 * * *` → strike-expiry, inactivity-hide
  - `0 4 * * 0` → retention-purge, admin digest email
- [ ] Auto-suspension trigger: if a tutor receives ≥3 reports in 30 days, status auto-flips to suspended pending review
- [ ] Auto-resolution timer: matches at 30d with no party engagement auto-close as RESOLVED_NO_MATCH
- [ ] `/api/user/data-export` returns JSON dump of user's data
- [ ] `/api/user/account` DELETE soft-deletes account; data purged after 30d grace period (transaction records preserved per retention policy)
- [ ] `/admin/health` shows: 7d match volume, dispute rate, suspension rate, Stripe success rate, open appeals count
- [ ] Weekly admin digest email sent (Sundays via cron) with the above metrics + items needing attention
- [ ] `/api/admin/export-audit` returns CSV of audit log for date range
- [ ] Sentry SDK integrated (optional but recommended)
- [ ] Resend wired for all transactional emails (signup verify, phone OTP fallback, match notification, parent confirmation, refund processed, strike notice, etc.) — templates in `src/lib/email-templates.ts`
- [ ] All emails go through one `sendEmail()` helper that logs to the audit log
- [ ] `npm run build` passes; all crons return 200 in local testing
- [ ] **Commit message lead:** "Session 5: automation hardening — audit logging, retention, cron scheduling, self-service data tools"

---

## Risk profile under the new model

For comparison:

| Model | Legal positioning | Typical defence cost if sued | Insurance | Pty Ltd needed? |
|---|---|---|---|---|
| Original verified-marketplace (current code) | Platform as verifier + payment facilitator | $50k–200k | $1,000–1,400/yr (PI + PL) | Yes |
| Parent-pays + no verification claims | Directory + consumer transaction | $20k–50k | $500–800/yr | Strongly suggested |
| **Pure directory + confirmed-match commission (chosen)** | **Hipages / classifieds tier** | **$10k–30k** | **$300–600/yr (basic media liability)** | **Optional but cheap insurance recommended** |

Realistic ongoing cost in the new model: **~$500–1,000/yr** (insurance + domain + Stripe fees + optional Pty Ltd). Achievable on a small budget.

The bigger one-time Phase 2 cost (lawyer review of Terms ~$1,000–2,000) is still recommended before opening to the general public, but **not the same kind of must-have as it was for the original marketplace model** — directories can credibly launch with carefully-written self-drafted terms and a lawyer review later, once revenue justifies it.

---



---

## Status (verified-marketplace checkpoint — pre-pivot)

⚠️ Below describes the **current code** which is being pivoted away from. Treat as reference for what exists today; the **Pivot work plan** above is the forward direction.

What's working end-to-end right now in the pre-pivot code:

- **Landing page** — full v3 design ported, all sections (Hero with For-parents/For-tutors split, Pitch, Mechanic, How, Comparison, Trust, Guarantee, Earnings, FAQ, Final CTA, Footer + site-wide disclaimer block).
- **School-branded landing pages** at `/schools/[slug]` and an `Other Locations` route at `/schools/other`. Browse tabs switch between areas. Same layout, only the brand colour + content change per area.
- **Browse view** with filters (subject category chips: Math / English / Physics / Chemistry / Biology / Science K-10 — in that order; day-of-week availability chips with a "you might be limiting options" warning; min ATAR; max $/hr; mode) and sort (Oldest profiles first by default, Highest subject result alternative — bands map to marks: B6/E4 → 90, B5/E3 → 80, etc.). Reads **real approved tutor applications** from the JSON store (with `status=APPROVED` and `visibility=true`) and shows them as proper cards linking to `/tutors/[applicationId]`. The "X verified tutors live" counter sits under the filters. Sample tutors have been removed — browse is real-only.
- **Tutor profile page** at `/tutors/[id]` with the refund policy laid out before any payment ask.
- **Unlock confirm page** at `/unlock/[tutorId]` with the same refund explainer — the actual $20 charge is a stub until Stripe is wired.
- **Auth** — sign up, log in, log out. HMAC-signed cookie sessions, `scrypt` password hashing, password show/hide toggle. Admin promotion via `ADMIN_EMAILS` env allowlist. Storage is a local JSON file (`data/users.json`).
- **Tutor signup form** with full validation:
  - 18+ age gate — under-18 submissions are **auto-rejected** at the API layer with reviewer notes recording the date of birth and computed age. The application row is stored (not silently dropped) so there's an audit trail, and the tutor sees an explicit rejection banner in their dashboard. Same logic applies on profile edit. **⚠️ Historical/stale: this reject was removed in commit `73a83fa` and NOT replaced — see launch blocker 1 at the top. The current code does not gate DOB.**
  - WWCC details, ATAR, HSC results
  - "High school attended" with conditional "Other school" free text
  - Per-subject year-level selection + "All years" toggle (subjects offered must be a subset of subjects sat)
  - Multi-slot weekly availability in 15-min increments
  - Tutoring-area dropdown (`Near Killara` / `Near Masada` / `Other location`) that drives which school tab the profile appears under
  - Bio scanned server-side for contact-info bypass (phone/email/social handles)
- **Admin** — list of applications with status pills and bio-scanner flags. Detail page with side-by-side fields, reviewer notes, approve / pause / reject / pending-back actions, and "Test unlock + chat" / "View public profile" shortcuts.
- **Tutor dashboard** — `/dashboard` shows status pill, reviewer notes, edit + view-public-profile buttons, visibility toggle, and conversation count. `/tutor/edit` reuses the signup form pre-filled with the existing submission; saving updates the application and resets status to Pending review (admin re-approves). Visibility toggle is a one-click switch independent of status.
- **Safety, suspension & auto-refund** — the tutor signup form has a prominent safety callout (public libraries recommended; TUTUMatch verifies identity but does not choose lesson locations or take responsibility for what happens at any lesson). The chat thread surfaces the same reminder to the tutor side. If a tutor doesn't reply to an unlocked parent within 5 days, the platform auto-refunds the parent's $20 AND suspends the tutor's account. Suspended users are signed out, can't log back in, and see an explicit appeal-by-email message (`appeals@tutumatch.com.au`). The refund/suspension processor runs lazily on dashboard + messages page loads (and will move to a Vercel cron once production hits). A dev-only "Skip the 5-day wait" button on the parent's side of the chat lets us demo the flow locally.
- **Platform chat** — `/messages` lists threads, `/messages/[unlockId]` is the chat. Post-unlock contact info is revealed inside the thread. Tutor reminder to apply the $20 first-lesson discount is surfaced on their side. First tutor reply records `tutorFirstReplyAt` (used by the 5-day refund auto-flag once Stripe is wired). Local-dev unlock shortcut at `POST /api/unlocks/dev-create` lets us exercise the full sign-up → approve → unlock → chat loop without Stripe.
- **Disclaimer layer** — visible at every decision point: a `BETA` tag in the TopNav, a permanent footer block ("we're an introduction service, not a tutoring provider; lesson locations are not chosen by us; liability is capped at the $20 unlock fee"), an explicit warning card on `/unlock/[tutorId]` right before the pay button, a 7-bullet acknowledgement list on tutor signup, and a "What we are and aren't" section on `/how-it-works`.
- **Tutor indemnity** — a properly drafted Section 13 of the Terms with 10 enumerated triggers (acts/omissions, lesson incidents, misrepresentation, IP claims, $20-discount failure, tax/super, child-safety law breaches, defamation, false statements). Mirror parent clause in Section 14 with full ACL carve-outs preserved. A required checkbox in signup form Section 9 ("I have read and accept the tutor indemnity clause") blocks submission until ticked. Submit button reads "Submit & accept indemnity — list me as a tutor". Server stamps `termsAcceptedVersion` + `termsAcceptedAt` on every application; the admin detail page surfaces this as evidence on file. Bumping `TERMS_VERSION` in `src/lib/legal.ts` re-records acceptance on the next edit.
- **Legal docs** — Terms (with the indemnity, version-stamped), Privacy (APP-aligned), Child Safety drafts at `/legal/*`. All still draft — lawyer review required before public launch.
- **Prisma schema** — full data model (users, tutor profiles, schools, HSC results, subjects, availability, verifications, unlocks, payments, refunds, messages, reports). Not yet connected to a real DB.

The original v3 prototype (HTML + JSX + screenshots) lives in `design-reference/` for visual diffing.

---

## Technical roadmap

Tick items off here as they ship. This list is the canonical source of truth for what's left.

### Critical for first paid launch

- [ ] **Database** — swap the local JSON store for real Postgres
  - [ ] Provision Postgres (Supabase recommended — see Hosting below)
  - [ ] Set `DATABASE_URL` + `DIRECT_URL` in `.env.local` and Vercel
  - [ ] Run `prisma:generate` + `prisma:migrate` against the existing `prisma/schema.prisma`
  - [ ] Replace `src/lib/db.ts` (JSON helpers) with Prisma client calls — keep the function names the same so callers don't change
  - [ ] Seed schools via `prisma/seed.ts`
- [ ] **Stripe payments**
  - [ ] Create Stripe account, verify business (ABN required), get test + live keys
  - [ ] Wire `POST /api/unlock` to create a $20 AUD PaymentIntent + an `Unlock` row
  - [ ] Wire `POST /api/stripe/webhook` to mark the Unlock `PAID`, set `refundEligibleAt = now + 5d`, fire the unlock notification email
  - [x] 5-day auto-refund processor + tutor suspension already wired against the JSON store. Lazy-fires on dashboard + messages page loads. `/api/cron/refund-flag` still needs to hook into the same `processOverdueRefunds()` helper once Stripe is live so production gets a scheduled trigger too.
  - [ ] Wire `POST /api/refund` to actually call Stripe for the money movement (the local processor only flips status + suspends; no real funds move yet)
  - [ ] Configure webhook endpoint in Stripe dashboard pointing at `https://<domain>/api/stripe/webhook`
- [~] **Identity & WWCC verification** (manual review is fine for v1)
  - [x] File-upload UI in `/tutor/signup` and `/tutor/edit` for: government ID, WWCC document, HSC Record of Achievement (PDF / JPG / PNG / HEIC / WEBP, 8 MB max each)
  - [x] Admin verification view shows the uploaded documents inline on `/admin/applications/[id]` — links open the original file in a new tab via the access-checked `/api/uploads/[id]` endpoint
  - [x] Tutor can view their own uploaded docs; admins can view any; everyone else gets 403
  - [ ] Real storage (Supabase Storage / S3) with at-rest encryption + short-lived signed URLs — current implementation is **local filesystem only** (`data/uploads/<userId>/`). DO NOT deploy to multi-user prod hosts yet.
  - [ ] Virus scan on upload (e.g. ClamAV or a SaaS scanner)
  - [ ] Manual WWCC lookup workflow against the NSW OCG public verification (paste number + DOB + name, record the outcome)
  - [ ] Document each verification with reviewer name + date + result on the `Verification` row
- [ ] **Transactional email** (Resend recommended — see Hosting)
  - [ ] Account creation — verify-your-email link (one-time token, 24h TTL)
  - [ ] Password reset — forgot-password link (one-time token, 30min TTL)
  - [ ] Tutor application submitted — confirmation to tutor
  - [ ] Tutor approved / rejected — notification with reviewer notes
  - [ ] Parent unlocks a tutor — notification to the tutor ("$20 collected, remember the first-lesson discount")
  - [ ] $20 unlock receipt — to the parent
  - [ ] First-lesson reminder + the $20 discount instruction — to the tutor
  - [ ] 5-day auto-refund processed — to the parent
  - [ ] Manual refund processed — to the parent + admin audit copy
  - [ ] New in-platform message — to the recipient
- [x] **Live profiles in browse** — `/browse` and `/schools/[slug]` now read approved + visible applications from the JSON store, merge them with the demo samples (real first, deduped), and render them through the same filter/sort pipeline. Cards link to the real `/tutors/[id]` for live tutors; the "Example" badge only renders for samples. A "X verified tutors live" counter sits under the filters.
  - [ ] Pagination + cursor-based loading (deferred — fine without it until a single school has 50+ tutors)
  - [x] Cards link to the real `/tutors/[id]` (not `sample-*`)
- [x] **Tutor dashboard**
  - [x] View profile status with status pill + reviewer notes
  - [x] Edit profile — reuses the same form. Any edit drops status back to PENDING_REVIEW for admin re-approval (child-safety policy).
  - [x] Visibility toggle (`/api/tutor/applications/visibility`) — independent of status; tutor can pause their listing without re-review
  - [x] View public profile shortcut (so the tutor can see what parents see)
  - [x] Inbox surface (count of parents who unlocked) — full chat lives at `/messages`
  - [x] Lifetime unlock stats (count of parents unlocked, replied-to, refunded) shown in a small stat row on the dashboard
  - [ ] Actual dollar earnings (waits on Stripe — once we have real unlock payment records the dashboard can sum those)
- [x] **In-platform messaging** (post-unlock)
  - [x] Chat UI between the parent and the unlocked tutor (`/messages`, `/messages/[unlockId]`)
  - [x] Stores threads + messages in the JSON store; first tutor reply stops the 5-day refund clock (`tutorFirstReplyAt`)
  - [x] Dev shortcut: `POST /api/unlocks/dev-create` creates a PAID Unlock without Stripe so the flow can be demoed end-to-end. Disabled when `NODE_ENV=production`.
  - [ ] Email notification on each new message (waiting on Resend wiring)
- [ ] **Tutor's own profile management** — `/tutor/me` route (or extend dashboard) so they can edit after approval

### Hosting recommendation

Tested combo that fits on free tiers for early launch:

| Service     | What for                       | Free tier covers                          |
|-------------|--------------------------------|-------------------------------------------|
| **Vercel**  | Hosting Next.js + Cron         | 100 GB/mo bandwidth, custom domains, Cron |
| **Supabase** (AU region) | Postgres + Storage (verification docs) | 500 MB DB + 1 GB storage |
| **Resend**  | Transactional email            | 3,000 emails/mo, 100/day                  |
| **Stripe**  | Payments + refunds             | 1.7% + $0.30 per AU card txn (no monthly fee) |
| **Sentry**  | Error monitoring (optional)    | 5k events/mo                              |
| **auDA registrar** | `tutumatch.com.au` domain | ~$15/year                                |

Pick Supabase over Neon because: same provider gives you Postgres + Storage (so verification docs and DB share one auth context) and it has Sydney region.

- [ ] Register `tutumatch.com.au` (or chosen domain)
- [ ] Create Vercel project, connect this GitHub repo
- [ ] Create Supabase project in `ap-southeast-2` (Sydney), grab connection string
- [ ] Create Resend account, add the domain, verify DNS
- [ ] Add all production env vars from `.env.example` to Vercel
- [ ] Configure Vercel Cron to hit `/api/cron/refund-flag` every 15 minutes
- [ ] Optional: Sentry project + DSN in env

### Nice-to-have (post first launch)

- [ ] Sign in with Google (OAuth) on `/login`
- [x] **Schools CRUD admin UI** — `/admin/schools` lets you add/edit/deactivate schools (slug, name, tagline, three brand colours, active toggle). Stored in `data/schools.json`. New schools become tabs on the public site instantly; their brand colour drives the theming via CSS variables.
- [x] **Refund queue UI** in `/admin/refunds` — buckets unlocks into auto-refunded (with tutor suspension status + one-click unsuspend), approaching the 5-day window, and active.
- [x] **Manual unsuspend** — admin button on the refunds page; clears suspension fields so the user can log in again.
- [x] **Reports / dispute resolution UI** — anyone signed in can report a tutor profile (from `/tutors/[id]`) or a chat thread (from `/messages/[unlockId]`) with a reason + description. Admin queue at `/admin/reports` shows open reports first, click to expand → resolve / dismiss with optional notes + action taken (Warned / Suspended user / Rejected application / Refunded parent). Suspending from the report applies the same `suspended` flag the auto-refund flow uses.
- [ ] Tutor photo upload with admin moderation (needs file storage — pair with the WWCC/ID/HSC upload work)
- [ ] Analytics (Plausible — privacy-friendly, cheap)
- [~] **Accessibility baseline** — visible focus rings on all interactive elements, skip-to-content link, `prefers-reduced-motion` respected globally, form errors announced via `role="alert" aria-live="polite"`, larger tap targets (44px min) on touch devices. Still TODO: a proper end-to-end keyboard nav audit, contrast checks against WCAG AA, and screen-reader testing.
- [~] **Mobile polish baseline** — TopNav wraps and truncates on small screens, browse hero / controls / tabs get tighter padding under 520px, admin tables scroll horizontally, tutor form grids collapse to single column under 900px. Still TODO: full mobile pass on /admin/applications/[id] and the chat thread on phones.

### Founder / legal (off-code work)

After the **pure-directory pivot** (see top of README), the legal posture is dramatically simpler than the verified-marketplace model. The protections you still want exist mainly as cheap insurance against the residual operational risk of running any platform that connects strangers — not as essential prerequisites without which you can't launch.

#### Phase 1 — Soft launch in your personal network (~$30 outlay, safe to do this week)

Defensible for testing the directory model with people who know you. No insurance, no Pty Ltd, no lawyer review yet. The whole point is small-scale validation.

- [ ] **ABN** as sole trader (~15 min, free, https://abr.gov.au/)
- [ ] **Domain** (~$15-20/yr — `tutumatch.com.au` if you have an ABN, otherwise `.au` is fine)
- [ ] Set up `hello@tutumatch.com.au` and `appeals@tutumatch.com.au` routed to a real inbox you check daily (safety@ is less critical in the directory model — we're not making safety promises — but worth setting up still)
- [ ] **Only promote within your personal network** — friends, family, school alumni. Don't publicly advertise. The directory framing protects you legally, but Phase 1 caution is still about ensuring problems surface to people who'll talk to you directly rather than sue.
- [ ] Keep volume modest (under ~30 tutors, under ~50 matches)
- [ ] The codebase makes clear we don't verify anything — keep the "What TUTUMatch is and isn't" page (to be built in session 1) prominent

#### Phase 2 — Public launch (~$300-1,000 upfront, ~$500-1,000/yr ongoing)

The pure-directory model makes this **dramatically cheaper** than the original marketplace model required. Order of priority:

- [ ] **Media liability / classifieds insurance** — ~$300-600/yr. Quote from **BizCover** or **Insurance House**. Specify "online classifieds directory for tutoring services, no verification of advertisers". This covers the residual risks (someone sues you anyway, defending against frivolous claims, hosting-related claims). Much cheaper than the PI + PL combo needed for verified marketplaces.
- [ ] **Optional: Pty Ltd** — ~$700 setup, ~$300/yr ongoing. Less urgent for a directory than a marketplace because the exposure is bounded, but the liability shield is cheap insurance for the cost. Set up via **EasyCompanies** or **Lawpath**. **You can defer this until revenue justifies it** — many directory founders operate as sole traders for years without issue.
- [ ] **Lawyer review of Terms** — ~$500-1,000 one-time, less than the marketplace version because the terms are simpler (no consumer relationship with parents, narrower scope of platform service). Useful before public marketing but not blocking for friends-of-friends growth.
- [ ] **Business bank account** (sole trader → eventually a company if you upgrade)
- [ ] **GST registration** trigger reminder (mandatory once turnover ≥ $75k/yr; the directory model with $20 commissions means this kicks in at ~3,750 confirmed matches/yr — that's a real business at that point)
- [ ] **Written permission from each school** before activating their landing page in `/admin/schools` — save the evidence. The directory framing doesn't change school name/logo trademark and defamation exposure.
- [ ] **OAIC data-breach response plan** documented (lighter than the original — we collect less personal info per user in the directory model)

#### Why the directory model lowers the threshold

In the original verified-marketplace model, a child-harm incident triggered an expensive lawsuit that argued "you said you'd verify and you didn't" — that's the high-cost case ($50k-$200k defence, real risk of judgment). 

In the pure-directory model, the same incident still produces a lawsuit attempt, but the defence is structurally cleaner: "We're a classifieds platform. We made no representations about this tutor's safety. The parent had the same access to the tutor's information that anyone in NSW does — the parent had the responsibility to verify WWCC directly. The harm happened in a private arrangement the parent and tutor made without our involvement."

That defence wins most of the time, and even when it doesn't go to verdict the case settles cheaply. The realistic worst case in the directory model is **~$10k-30k in defence costs over 3-6 months** — well within what a $300-600 insurance policy covers.

**Bankruptcy-level outcomes that loomed in the marketplace model are unlikely under the directory model.** That's the actual reason this pivot matters: not just lower probability of getting sued, but bounded downside even if you are.

---

## Local setup

```powershell
# 1. install
npm install

# 2. env
Copy-Item .env.example .env.local
# at minimum set NEXTAUTH_SECRET (32+ chars) and ADMIN_EMAILS

# 3. dev server
npm run dev
# → http://localhost:3000
```

Once you have a Postgres URL set in `.env.local`:

```powershell
npm run prisma:generate
npm run prisma:migrate     # creates the initial migration
npm run seed               # inserts the two demo schools
```

Type-check: `npm run typecheck`. Build: `npm run build`.

For the **production setup** (Stripe, Supabase, Resend, domain, Vercel), see **[SETUP.md](./SETUP.md)** — a step-by-step walkthrough of every external service signup, what to copy where, and what costs what.

## Adding a new school

Sign in as an admin (email in `ADMIN_EMAILS`) and go to **`/admin/schools`**. Fill the form: slug auto-generates from the name, three brand colours drive the theming via CSS variables, and the `active` toggle controls whether the school appears publicly. **Get written permission from the school first** before flipping `active` to true — using their name and colours without consent risks defamation and trademark issues.

Schools are stored in `data/schools.json` (will move to Postgres once Supabase is wired). The two seeded schools (Killara, Masada) live in `src/lib/schools.ts` as `SEED_SCHOOLS` — used to bootstrap the JSON store on first run.

## Routes

State legend: ✅ Done · 🚧 Stub · 📝 Draft · ♻ Refactor in pivot · ❌ Remove in pivot · ✨ New in pivot

| Path                                | What it is                                                                | State        |
|-------------------------------------|----------------------------------------------------------------------------|--------------|
| `/`                                 | Default landing page                                                       | ♻ Refactor — strip verification claims, reframe Earnings/Comparison/Hero |
| `/schools/[slug]`                   | School-branded browse (Killara / Masada / Other)                          | ♻ Refactor — remove "X verified" counter, reframe |
| `/browse`                           | All-tutors browse with filters                                            | ♻ Refactor — same |
| `/tutors/[id]`                      | Tutor profile page                                                        | ♻ Refactor — remove verification badges, refund explainer, contact-CTA framing |
| `/unlock/[tutorId]`                 | $20 confirm page                                                          | ❌ Remove — replaced by free `/contact/[tutorId]` reveal page |
| `/contact/[tutorId]`                | NEW: free intent capture + contact reveal + 48h hidden window trigger     | ✨ Build in session 2 |
| `/tutor/signup`                     | Tutor application form                                                    | ♻ Refactor — reframe WWCC as "have it ready", remove platform-verification language |
| `/dashboard`                        | Tutor hub                                                                 | ♻ Refactor — replace unlock stats with match stats, add self-report buttons, show pending matches |
| `/tutor/edit`                       | Edit tutor profile                                                        | ♻ Refactor — minor copy updates |
| `/messages`                         | Chat thread list                                                          | ❌ Remove — no platform-mediated chat in directory model |
| `/messages/[unlockId]`              | Chat thread                                                               | ❌ Remove — same |
| `/admin`                            | Admin: applications queue                                                 | ♻ Refactor — repurpose as spam/abuse moderation only, no credential check |
| `/admin/applications/[id]`          | Application detail                                                        | ♻ Refactor — same |
| `/admin/matches`                    | NEW: matches queue with pending self-reports, parent confirmations        | ✨ Build in session 3 |
| `/admin/appeals`                    | NEW: tutor appeals queue (when parent says no but tutor disputes)         | ✨ Build in session 3 |
| `/admin/strikes`                    | NEW: strike audit log + override controls                                 | ✨ Build in session 3 |
| `/login`                            | Email + password auth                                                     | ✅ Keep (minor copy tweaks) |
| `/legal/terms`                      | Terms of Service                                                          | ♻ Rewrite — narrower scope, directory framing, lighter indemnity |
| `/legal/privacy`                    | Privacy Policy                                                            | ♻ Update — less data collected per user under new model |
| `/legal/child-safety`               | Child Safety Policy                                                       | ♻ Reframe — "what we ask tutors to have" not "what we verify" |
| `/how-it-works`                     | Explainer                                                                 | ♻ Rewrite — new model end-to-end |
| `/contact`                          | Routed contact emails                                                     | ✅ Keep |
| `/what-we-are`                      | NEW: prominent "we don't verify anything" statement page                  | ✨ Build in session 1 |
| `POST /api/auth/{signup,login,logout,me}` | Cookie-based auth                                                  | ✅ Keep |
| `POST /api/tutor/applications`      | Submit application                                                        | ♻ Refactor — strip indemnity acceptance (different shape), remove document review |
| `PUT  /api/tutor/applications`      | Update application                                                        | ♻ Refactor — same |
| `PATCH /api/tutor/applications/visibility` | Toggle profile visibility                                          | ✅ Keep |
| `GET/PATCH /api/admin/applications` | List + approve/reject                                                     | ♻ Refactor — approve = spam-check only |
| `/admin/schools`                    | Schools CRUD                                                              | ✅ Keep |
| `/admin/refunds`                    | Refund queue                                                              | ❌ Remove — no parent refunds in new model |
| `/admin/reports`                    | Reports queue                                                             | ✅ Keep |
| `POST /api/reports`                 | Submit a report                                                           | ✅ Keep |
| `PATCH /api/admin/reports/[id]`     | Admin resolve / dismiss + optional suspend                                | ✅ Keep |
| `POST /api/uploads`                 | Upload a doc                                                              | ♻ Refactor — uploads become tutor-private (not for verification) |
| `GET  /api/uploads/[id]`            | Fetch a doc                                                               | ♻ Refactor — only owner can fetch (no admin verification view) |
| `GET/POST /api/admin/schools`       | List + create school                                                      | ✅ Keep |
| `PATCH/DELETE /api/admin/schools/[id]` | Update / delete school                                                 | ✅ Keep |
| `POST /api/admin/users/[id]/unsuspend` | Admin: clear suspension                                                | ♻ Refactor — applies to strike-suspended users not refund-suspended |
| `POST /api/unlocks/dev-create`      | Dev-only: create a PAID Unlock                                            | ❌ Remove — no Unlock entity in new model |
| `POST /api/unlocks/[id]/fast-forward` | Dev-only fast-forward                                                   | ❌ Remove — same |
| `GET /api/threads`                  | Chat threads                                                              | ❌ Remove — no chat |
| `GET/POST /api/threads/[unlockId]/messages` | Chat messages                                                     | ❌ Remove — no chat |
| `POST /api/matches/request`         | NEW: parent expresses interest, triggers 48h tutor-hidden window          | ✨ Build in session 2 |
| `POST /api/matches/[id]/self-report` | NEW: tutor self-reports match (charges $15)                              | ✨ Build in session 3 |
| `POST /api/matches/[id]/parent-confirm` | NEW: parent confirms via email link                                  | ✨ Build in session 3 |
| `POST /api/matches/[id]/appeal`     | NEW: tutor disputes a "no" with evidence upload                          | ✨ Build in session 3 |
| `POST /api/stripe/webhook`          | Stripe webhook                                                            | 🚧 Stub — to be wired in session 3 |
| `GET  /api/cron/match-checkin`      | NEW: cron to send parent confirmation prompts at 2 days / 7 / 14 / 30    | ✨ Build in session 3 |
| `POST /api/unlock`                  | Original parent-pays unlock                                               | ❌ Remove |
| `POST /api/refund`                  | Original refund flow                                                      | ❌ Remove |
| `GET  /api/cron/refund-flag`        | Original 5-day refund flagger                                             | ❌ Remove |

## Tech

Next.js 14 (App Router) · TypeScript · Tailwind · CSS-variable theming · Prisma + Postgres (schema written) · `scrypt`-hashed passwords + HMAC cookie sessions · Stripe (planned) · Resend (planned) · Supabase Storage (planned).

---

## Design reference

`design-reference/` contains the original v3 HTML + JSX + screenshots that this codebase ports from. Useful for visual diffing if you tweak the landing layout.

## Contact

`hello@tutumatch.com.au` — change to whichever inbox the founder uses. Reflected in `src/components/landing/Footer.tsx` and the legal stubs.
