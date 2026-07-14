# ChildCare Connect — Project Brief

## What this is
A childcare marketplace for North Carolina (Wilmington → Raleigh → Charlotte) connecting families with local childcare providers. Built and run by Nikieta, a solo founder (QA engineer by trade — fintech/banking compliance background). Budget-conscious: the entire stack must stay free or near-free. Do not introduce paid services without asking.

- Live site: https://childcareconnectnc.com (custom domain on GitHub Pages, HTTPS enforced)
- Repo doubles as the host: this is a GitHub Pages user site (repo name childcare-connect-nc.github.io)
- Business email: childcareconnectnc@gmail.com

## Stack
- **Frontend:** plain HTML/CSS/JS, one file per page, no build step, no frameworks. Google Fonts: Fraunces (headings) + Nunito Sans (body). Supabase JS v2 imported via CDN ESM (`https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2/+esm`).
- **Backend:** Supabase free tier (project: childcare-connect, region US East).
  - URL: `https://jcacedyzmsowclexnxvu.supabase.co`
  - Publishable (anon) key is embedded in provider.html and browse.html — this is intentional and safe; security lives in RLS. NEVER put the service_role key or DB password anywhere in this repo.
- **Waitlist forms:** Tally — parents https://tally.so/r/7RPNO2, providers https://tally.so/r/7RPNY2
- **Hosting quirk:** pages must be tested on the live site, not by opening files locally (file:// breaks module imports). Deploy = commit to main, GitHub Pages auto-publishes in ~1–2 min.

## Pages
- `index.html` — landing page. Parent CTA → Tally waitlist (intentional until enough listings exist to make browsing worthwhile), plus a "See what it'll look like →" link to browse-preview.html. Provider CTA → provider.html directly (no Tally form in this flow despite the provider Tally link existing — see note below). Footer links to legal pages. Hero pill names the active launch cities (Wilmington/Raleigh/Charlotte); waitlist copy and FAQ also mention Greensboro/High Point/Durham as planned next expansion. Has OG/Twitter social-preview meta tags + og-image.png.
- `provider.html` — provider portal: email/password auth (Supabase), profile create/edit, approval-status badge (pending/live), inbox of parent inquiries. Signup requires a T&C agreement checkbox. **No photo upload UI exists yet** — see Parked checklist. Has OG/Twitter meta tags (robots: noindex, so not search-indexed, but still gets a social preview card when shared).
- `browse.html` — public parent-facing search: filter by city + care type, provider cards showing a care-type pill and a green "✓ Personally Approved" badge, inquiry modal that inserts into `inquiries`. **Does not render any provider photo** — the `photo_url` column isn't even selected in the query yet. Not yet linked from the homepage nav (deliberate soft launch).
- `browse-preview.html` — preview page for waitlisted parents: 4 static sample provider cards (initials avatars, star ratings, "Personally Approved" + "Preview" badges), working client-side city/type filters over the sample data only. No Supabase calls. Linked from index.html's waitlist section.
- `terms.html`, `privacy.html` — legal pages, styled to match. Effective June 12, 2026.

**Note on Tally provider link:** both this file and prior notes reference a Tally provider waitlist form (`7RPNY2`), but neither index.html nor provider.html currently links to it — the provider CTA goes straight to the real provider.html signup portal. Confirm with Nikieta whether that Tally form is still meant to be live anywhere before assuming it's part of the funnel.

## Database (Supabase)
**providers** — id (uuid, = auth.users id), business_name, contact_name, email, phone, city, care_type, ages_served (text), price_range (text, free-form), bio, photo_url, is_approved (boolean, default false), created_at.

**inquiries** — id, provider_id (FK), parent_name, parent_email, child_age_range, message, is_read, created_at.

**RLS (all enforced, do not weaken):**
- Public can SELECT providers only where is_approved = true; providers can SELECT/INSERT/UPDATE only their own row; nobody can set is_approved through the client — approval is done manually in the Supabase Table Editor (that's the admin workflow).
- Anyone can INSERT inquiries (parents have no accounts); providers can SELECT/UPDATE only inquiries where provider_id = their auth.uid().

**Storage:** public bucket `provider-photos`, 5 MB limit, image/jpeg|png|webp only. Policies restrict insert/update/delete to a folder named with the uploader's auth.uid(); path convention `{user_id}/listing.{ext}`, upsert + cache-busting `?v=` query param. **This exists on the Supabase side but is not yet wired to any frontend page** — provider.html has no upload input, and browse.html doesn't select or render `photo_url`. Treat as backlog, not shipped.

## Product principles (non-negotiable)
1. **Data minimization** — never collect children's names, birthdates, or photos. Inquiries use age ranges only. Photo-upload UI tells providers not to include children's faces.
2. **Honest claims** — listing review is a basic screening, not verification. Marketing copy must never promise more than terms.html disclaims (see FAQ wording on index.html). No fake stats or invented user counts.
3. **Approval gate** — every listing starts is_approved = false and is invisible until Nikieta approves it (her vetting includes checking NC DCDEE public records).
4. **Free tier first** — GitHub Pages + Supabase free tier + Tally free. Founding providers get 6 months free; no payment processing exists yet.

## Design system
Cream background #FDFAF5; ink #11302E; body text #4F5D58; teal (trust) #0E4D4A with soft #E3EFEA; coral (warmth/CTAs) #E76F51, hover #C9543A, soft #FBE6DE; lines #E8E0D3; success green #1D9E75. Rounded (10–26px radii), pill buttons, warm + personable small-business tone — never corporate. Headings Fraunces, everything else Nunito Sans.

## Parked checklist (the backlog)
- Wire photo upload: add a file input to provider.html's profile form, upload to the existing `provider-photos` bucket, save `photo_url` on the provider row, and render it on browse.html cards. The bucket + storage policies already exist in Supabase — only the frontend is missing.
- Replace hero illustration on index.html with a real photo (instructions are in a comment in the file)
- Graphics fine-tuning pass
- Re-enable Supabase email confirmation before parent-facing launch (Authentication → set Site URL to https://childcareconnectnc.com first; it was disabled deliberately to reduce founding-provider signup friction)
- Browse sorting: name A–Z/Z–A (easy); by price (requires adding a numeric starting_price column — price_range is free text); by rating (requires building a reviews system: table + RLS + moderation + display)
- Reviews system (prerequisite for rating sort)
- Eventually: link browse.html from the homepage and switch the parent CTA from the Tally waitlist to browse (when there are enough live listings)
- Eventually: paid provider subscriptions via Stripe (terms.html already covers this in "when paid subscriptions launch" language)
- Decide whether the provider Tally waitlist form (7RPNY2) is still part of the funnel anywhere, or should be considered retired now that provider.html is a full self-serve signup portal
- Consider replacing the Tally parent/provider forms with a native Supabase-backed waitlist form embedded on-site (proposed, not yet approved — would need a new `waitlist` table + insert-only RLS policy)

## Working with Nikieta (required workflow)
- **Explain before you act.** Before making changes, state in plain English what you're about to change and why. After changes, summarize exactly what was modified and how to test it on the live site.
- **She asks questions — answer them properly.** She's technical (QA engineer) but new to web development. When she asks how or why something works, explain it clearly without jargon-dumping and without condescension. Teaching her the system is part of the job.
- **Security changes require her explicit approval first.** Anything touching RLS policies, storage policies, auth settings, or Supabase configuration: propose it, explain the risk/benefit, and wait for her yes before applying or committing.
- **She is the QA gate.** After every deploy, give her a short test checklist for the live site. Nothing is "done" until she's verified it works.
- **Never commit secrets.** No service_role key, no database password, no personal data in the repo — ever, including in comments or commit messages.
- **Don't expand scope on your own.** Stick to what she asked; if you spot something else worth doing, suggest it and let her decide.

## Current business state (July 2026)
Founding-provider outreach in progress (Wilmington first: DMs from her personal Facebook + emails from the business gmail; founding offer = 6 months free, no card). First real provider signups expected via provider.html. Approval workflow: Supabase Table Editor → providers → set is_approved = true.

## Recently completed (most recent first)
- Fixed the "Preview" badge on browse-preview.html overlapping the type/approved badges — moved into the same flowing badge row instead of an absolute corner ribbon
- Added a green "✓ Personally Approved" badge to provider cards on both browse.html and browse-preview.html (chosen over "Verified"/"Reviewed" — accurate for every care type including nanny/babysitter listings that don't have formal credentials)
- Added coverage-expansion copy: Greensboro, High Point, and Durham mentioned as planned-next cities in index.html's meta description, waitlist section, and FAQ, plus browse-preview.html's meta description (Wilmington/Raleigh/Charlotte kept as the active hero pill)
- Added OG/Twitter social-preview meta tags + a branded 1200×630 og-image.png to index.html, provider.html, and browse-preview.html (image generated locally via macOS WebKit/Swift — no new service)
- Added a "See what it'll look like →" link from index.html's parent waitlist card to browse-preview.html
- Built browse-preview.html: 4 sample provider cards, responsive grid, working client-side filters, clearly labeled as sample data
- Confirmed provider.html's pending/live status system (statusRow + pendingNote card) was already solid — no changes needed there
