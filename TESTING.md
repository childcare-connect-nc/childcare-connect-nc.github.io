# ChildCare Connect — Testing Guide

How to test this site, split by who (or what) can run each check. Live site only:
**https://childcareconnectnc.com** — never test via `file://` (module imports break).

## How AI-assisted testing works

Claude can run the **[AI]** checks below on request ("run the test pass") by fetching
live pages. Its limits, so results are read correctly:

- **No JavaScript execution.** Fetches see raw HTML only. Anything rendered client-side
  (provider cards, modals, star ratings) can't be verified remotely — only that the
  markup, script, and copy shipped correctly.
- **No logins.** Anything behind provider/admin auth is human-only.
- **No direct database access.** RLS/schema checks are done by Claude writing SQL
  introspection queries and Nikieta running them in the Supabase SQL Editor.
- **Screenshots you paste ARE readable.** For visual bugs, a screenshot in chat is a
  first-class test artifact.

Legend: **[AI]** Claude can verify remotely · **[H]** human in a browser ·
**[SQL]** run provided SQL in Supabase, paste results back

---

## 1. Deploy smoke test — after every push

- [ ] **[AI]** All pages return 200 and contain their expected `<title>`:
  index, browse, browse-preview, provider, admin, terms, privacy (all `.html`)
- [ ] **[AI]** The change just deployed is actually present in the served HTML
  (GitHub Pages takes ~1–2 min; a stale fetch means wait, not a bug)
- [ ] **[AI]** OG meta tags + `og-image.png` still present on index, provider,
  browse-preview

## 2. browse.html — public search

- [ ] **[H]** Cards render for approved providers only; each card shows type pill,
  "✓ Personally Approved" badge, photo banner (if photo set), rating line or
  "No reviews yet"
- [ ] **[H]** City + care-type filters narrow results; both empty-state messages
  reachable (no providers at all vs. no filter matches)
- [ ] **[H]** All four sort options reorder correctly. Price sort: providers with no
  starting_price go last. Rating sort: unrated providers go last.
- [ ] **[H]** Inquiry modal: required-field validation, bad-email validation, success
  message, inquiry appears in the provider's inbox
- [ ] **[H]** Review modal: submitting with no stars is blocked; success message says
  review is checked before appearing; review does NOT appear publicly before approval
- [ ] **[H]** ⚠️ Test public visibility in an **incognito window** — an admin session in
  the same browser sees pending listings too (by design; bit us once already)
- [ ] **[H]** Mobile: grid collapses to one column; modals usable on a phone

## 3. provider.html — portal

- [ ] **[H]** Sign-up: requires T&C checkbox, 8+ char password; lands in edit mode
  with empty form ("No listing yet" status)
- [ ] **[H]** Sign-in with existing account: compact summary shown (not full form),
  Edit profile expands it, Cancel discards unsaved edits, Save persists + re-collapses
- [ ] **[H]** Required-field validation on save (business name, contact, city, type, bio)
- [ ] **[H]** Starting price: accepts plain numbers only, persists, drives price sort
  on browse
- [ ] **[H]** Photo upload — a profile must be saved first:
  - brand-new account, no saved profile → friendly "save your profile first" message
  - saved profile → upload succeeds, thumbnail appears, persists on refresh,
    banner appears on browse card
  - oversized (>5MB) file rejected with clear message; non-image rejected
- [ ] **[H]** Status badge matches reality: pending → "● Pending review" + welcome note;
  approved → "● Live"; **rejected → still shows "● Pending review"** (deliberate)
- [ ] **[H]** Inquiries inbox lists messages newest-first with parent contact info

## 4. admin.html — admin portal

- [ ] **[H]** Non-admin account → "Not authorized" (and confirm via RLS below that this
  is enforced server-side, not just UI)
- [ ] **[H]** Three provider sections populate correctly; Approve/Reject/Revoke/
  Move-to-pending each move the listing to the right section AND change public
  visibility on browse (verify in incognito)
- [ ] **[H]** Reject/Revoke: note prompt — **Cancel aborts entirely** (regression:
  once rejected anyway); saved notes show only on rejected cards
- [ ] **[H]** Pending reviews: grouped by provider with count badge; Approve makes the
  review public + updates the card's average; Delete confirms first, then removes

## 5. Security / RLS — [SQL] + [H]

- [ ] **[SQL]** Policy audit — paste results back to Claude to diff against expected:
  ```sql
  select tablename, policyname, cmd, qual, with_check
  from pg_policies
  where schemaname in ('public','storage')
  order by tablename, policyname;
  ```
  Expected highlights: providers public SELECT gates on `review_status = 'approved'`
  (NOT the legacy `is_approved`); admin_notes admin-only; reviews public SELECT gates
  on `is_approved = true`; storage has all four cmds for provider-photos scoped to
  the uploader's folder (SELECT policy is required for upsert to work).
- [ ] **[SQL]** Status-desync scan (should return zero rows):
  ```sql
  select id, business_name, review_status, is_approved from providers
  where (review_status = 'approved') <> is_approved;
  ```
  Drift is harmless to the public site (review_status wins everywhere) but means
  someone edited is_approved directly — worth knowing.
- [ ] **[H]** Logged out entirely: browse shows only approved; admin.html shows only
  the sign-in form; pending provider data not present in the network tab
- [ ] **[AI]** No secrets in the repo: only the publishable `sb_publishable_…` key
  appears; nothing matching service_role or database passwords, in any file or
  commit message

## 6. Regression list — past real bugs, retest after related changes

| Bug | Where | Guard |
|---|---|---|
| Cancel on reject-note prompt still rejected | admin.html | §4 |
| is_approved / review_status drift leaked a pending listing publicly | RLS + provider.html | §5 SQL scan |
| Photo upload: missing storage SELECT policy broke upsert | storage RLS | §5 audit |
| Photo upload before first profile save → NOT NULL crash | provider.html | §3 |
| "Preview" ribbon overlapped card badges | browse-preview.html | §7 |
| Supabase free tier auto-pauses after ~1 week idle → every query fails ("Failed to fetch") | whole site | if all data loading dies at once, check dashboard for "Paused" **before** debugging code |

## 7. browse-preview.html + static pages

- [ ] **[AI]** 4 sample providers in script; banner copy says "Preview — these are
  sample listings"; disabled message buttons; sample ratings labeled "(sample rating)"
- [ ] **[H]** Filters work over sample data; badges flow without overlapping (the
  ribbon regression); page never touches Supabase
- [ ] **[AI]** index.html: hero pill lists the 3 active cities; expansion copy
  (Greensboro/High Point/Durham) in waitlist + FAQ; parent CTA → Tally; provider
  CTA → provider.html; "See what it'll look like →" link present
- [ ] **[AI]** terms/privacy reachable from every page footer

## 8. Performance & accessibility — periodic

- [ ] **[H]** Lighthouse (incognito, mobile) on index + browse: Performance ≥ 90,
  Accessibility ≥ 90 — plain static pages should score high; a drop is a regression
- [ ] **[H]** Provider photos: card banners load reasonably on throttled "Fast 3G"
  (5MB originals shown at ~180px — if this drags, consider client-side resize
  before upload as a future improvement)
- [ ] **[H]** Keyboard-only: tab through filters, open/close modals (Esc), submit forms
- [ ] **[AI]** Static a11y sweep of the HTML: form inputs labeled, images have alt
  text, aria-labels on icon-only elements, heading order sane

## Reporting

For each failure: page URL, steps, expected vs. actual, screenshot if visual,
browser + device. Claude files fixes; **nothing is "done" until Nikieta re-verifies
on the live site** (per CLAUDE.md workflow).
