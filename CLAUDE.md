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
- `provider.html` — provider portal: email/password auth (Supabase), profile create/edit with photo upload (wired to the `provider-photos` bucket), approval-status badge (pending/live — always shows "pending" even if rejected, by design, see Product principles). Profile fields load locked/read-only by default with an "Edit profile" button to unlock them (Save/Cancel); a brand-new signup with no listing yet starts directly in edit mode. Inbox of parent inquiries. Signup requires a T&C agreement checkbox. Has OG/Twitter meta tags (robots: noindex, so not search-indexed, but still gets a social preview card when shared).
- `browse.html` — public parent-facing search: filter by city + care type, sort by name (A–Z/Z–A), price (low to high, via `starting_price`), or rating (high to low, via approved `reviews`); provider cards show a care-type pill, a green "✓ Personally Approved" badge, a photo banner when `photo_url` is set, a star rating summary (or "No reviews yet"), and a "Leave a review" link opening a review modal (star picker + optional name/comment, inserts into `reviews` with is_approved=false pending moderation). Inquiry modal inserts into `inquiries`. Not yet linked from the homepage nav (deliberate soft launch).
- `browse-preview.html` — preview page for waitlisted parents: 4 static sample provider cards (initials avatars, star ratings, "Personally Approved" + "Preview" badges), working client-side city/type filters over the sample data only. No Supabase calls. Linked from index.html's waitlist section.
- `admin.html` — admin-only portal (robots: noindex, nofollow), gated to one designated login (`childcareconnectnc@gmail.com`, checked client-side; real enforcement is the RLS policies below). Lists providers in three sections — Pending review / Approved & live / Rejected — with Approve, Reject, Revoke, and Move-to-pending actions. Rejecting or revoking prompts for an optional note, saved to `admin_notes` (admin-only table). Providers never see rejection status or notes — provider.html always shows "Pending review" for both never-reviewed and rejected listings, by deliberate product choice (Nikieta follows up on rejections personally, off-platform). Also has a "Pending reviews" section for moderating parent-submitted reviews (Approve or permanently Delete — no reject-with-reason flow like providers, since reviews aren't recurring relationships to manage).
- `terms.html`, `privacy.html` — legal pages, styled to match. Effective June 12, 2026.

**Note on Tally provider link:** both this file and prior notes reference a Tally provider waitlist form (`7RPNY2`), but neither index.html nor provider.html currently links to it — the provider CTA goes straight to the real provider.html signup portal. Confirm with Nikieta whether that Tally form is still meant to be live anywhere before assuming it's part of the funnel.

## Database (Supabase)
**providers** — id (uuid, = auth.users id), business_name, contact_name, email, phone, city, care_type, ages_served (text), price_range (text, free-form display string), starting_price (numeric, nullable — plain number used only for sorting, entered alongside price_range), bio, photo_url, is_approved (boolean, default false — legacy column, no longer read anywhere; kept only because admin.html still writes it alongside review_status for backward compat), review_status (text: 'pending'|'approved'|'rejected', default 'pending' — **the actual source of truth** for both public visibility and the provider's own status badge), created_at.

**Known failure mode this fixed:** is_approved and review_status can drift apart if is_approved is ever edited directly in Table Editor without also touching review_status (this happened once — a provider showed "Pending" in admin.html but was still publicly visible, because the old public RLS policy checked is_approved while admin.html's UI checked review_status). Fixed by moving the public SELECT policy and provider.html's own status display to both read review_status exclusively. If you ever need to manually fix a provider's status in Table Editor, set review_status — is_approved no longer matters for anything user-facing.

**inquiries** — id, provider_id (FK), parent_name, parent_email, child_age_range, message, is_read, created_at.

**admin_notes** — id (uuid), provider_id (FK → providers), note (text), created_at. Admin-only rejection/revocation notes, deliberately kept in a separate table (not a column on `providers`) so a provider's own "select my row" RLS policy can never surface it, even accidentally.

**reviews** — id (uuid), provider_id (FK → providers), parent_name (nullable, no parent accounts exist), rating (int 1–5), comment (nullable), is_approved (boolean, default false), created_at. Parents submit freely; nothing shows publicly until admin approves it in admin.html's "Pending reviews" section.

**RLS (all enforced, do not weaken):**
- Public can SELECT providers only where review_status = 'approved' (changed from is_approved — see Known failure mode above); providers can SELECT/INSERT/UPDATE only their own row; nobody but the admin account can set is_approved/review_status through the client.
- Admin account (`childcareconnectnc@gmail.com`, matched via `auth.jwt() ->> 'email'`) can SELECT all providers and UPDATE any provider row — this is row-scoped, not column-scoped, so technically the admin session could edit any column via a direct API call, not just approval fields. Accepted risk; admin.html's own code only ever writes is_approved/review_status.
- admin_notes: only the admin account can SELECT/INSERT/UPDATE/DELETE (FOR ALL policy) — no policy grants providers or the public any access, so RLS default-denies everyone else.
- reviews: anyone can INSERT (no auth required, same as inquiries); public can SELECT only where is_approved = true; admin account can SELECT all (for moderation), UPDATE (to approve), and DELETE (to remove spam/inappropriate reviews).
- Anyone can INSERT inquiries (parents have no accounts); providers can SELECT/UPDATE only inquiries where provider_id = their auth.uid().

**Storage:** public bucket `provider-photos`, 5 MB limit, image/jpeg|png|webp only. Policies restrict insert/update/delete to a folder named with the uploader's auth.uid(); path convention `{user_id}/listing.{ext}`, upsert + cache-busting `?v=` query param. Wired up end-to-end: provider.html uploads and saves `photo_url`, browse.html renders it as a card photo banner.

## Product principles (non-negotiable)
1. **Data minimization** — never collect children's names, birthdates, or photos. Inquiries use age ranges only. Photo-upload UI tells providers not to include children's faces.
2. **Honest claims** — listing review is a basic screening, not verification. Marketing copy must never promise more than terms.html disclaims (see FAQ wording on index.html). No fake stats or invented user counts.
3. **Approval gate** — every listing starts is_approved = false and is invisible until Nikieta approves it (her vetting includes checking NC DCDEE public records).
4. **Free tier first** — GitHub Pages + Supabase free tier + Tally free. Founding providers get 6 months free; no payment processing exists yet.

## Design system
Cream background #FDFAF5; ink #11302E; body text #4F5D58; teal (trust) #0E4D4A with soft #E3EFEA; coral (warmth/CTAs) #E76F51, hover #C9543A, soft #FBE6DE; lines #E8E0D3; success green #1D9E75. Rounded (10–26px radii), pill buttons, warm + personable small-business tone — never corporate. Headings Fraunces, everything else Nunito Sans.

## Parked checklist (the backlog)
- Replace hero illustration on index.html with a real photo (instructions are in a comment in the file) — needs Nikieta to actually source/upload a photo; not something that can be done sight-unseen
- Graphics fine-tuning pass — no concrete spec, needs Nikieta's creative direction
- Re-enable Supabase email confirmation before parent-facing launch (Authentication → set Site URL to https://childcareconnectnc.com first; it was disabled deliberately to reduce founding-provider signup friction) — a dashboard/Auth-settings change only Nikieta can make
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
Founding-provider outreach in progress (Wilmington first: DMs from her personal Facebook + emails from the business gmail; founding offer = 6 months free, no card). First real provider signups expected via provider.html. Approval workflow: admin.html (preferred) or Supabase Table Editor → providers → set is_approved/review_status directly.

## Recently completed (most recent first)
- Fixed photo upload: (1) storage.objects on `provider-photos` was missing a SELECT policy, which the upsert-based upload needs internally to check whether a file already exists at that path — added one scoped to the uploader's own folder; (2) uploading a photo before ever saving profile details tried to insert a providers row with only id/email/photo_url set, violating the NOT NULL constraint on business_name — provider.html now checks a profile row already exists before allowing a photo upload, and tells the provider to save their details first if not
- Grouped admin.html's "Pending reviews" by provider (one card with a count badge, reviews listed underneath) instead of a flat list repeating the provider name per review
- Collapsed provider.html's profile card into a compact one-line summary by default (photo, name, type, city, price) with an Edit profile button, instead of always showing every field
- Fixed a real bug: is_approved and review_status could drift apart (e.g. if is_approved was ever edited directly in Table Editor), and the public RLS policy + provider.html's own status badge were reading different fields — both now read review_status exclusively, which is the single source of truth; is_approved is a legacy column no longer read by any client code
- Built the full reviews system: `reviews` table + RLS (public insert, public read only if approved, admin moderates), a review submission modal on browse.html (star picker + optional name/comment), star rating summaries on provider cards, and a "Pending reviews" moderation section in admin.html (Approve / Delete)
- Added `starting_price` numeric column to providers (alongside the existing free-text `price_range` display field) so price sorting is possible
- Added a sort dropdown to browse.html: Name (A–Z/Z–A), Price (low to high), Rating (high to low) — all client-side over the already-fetched provider list
- Fixed a bug in admin.html where clicking Cancel on the reject/revoke note prompt still rejected the listing anyway (prompt() returns null on Cancel, which the code didn't check for)
- Added a Rejected status to admin.html, distinct from Pending: providers table now has a `review_status` column (pending/approved/rejected) instead of relying on the is_approved boolean alone; revoking an approved listing now moves it to Rejected rather than back into the pending queue. Rejection/revoke reasons are optional and stored in a new admin-only `admin_notes` table. provider.html is unchanged by this — providers still see "Pending review" whether never-reviewed or rejected, a deliberate choice so Nikieta can handle rejections personally rather than the site delivering a cold automated one
- Added admin.html: a gated admin portal (RLS-restricted to childcareconnectnc@gmail.com) for approving/rejecting provider listings, replacing manual Supabase Table Editor edits
- Provider profile fields in provider.html now load locked/read-only with an "Edit profile" button, instead of always being editable; brand-new signups with no listing yet still start directly in edit mode
- Wired up photo upload end-to-end: provider.html has an upload UI hitting the existing `provider-photos` bucket, browse.html now renders the photo as a card banner when set
- Fixed the "Preview" badge on browse-preview.html overlapping the type/approved badges — moved into the same flowing badge row instead of an absolute corner ribbon
- Added a green "✓ Personally Approved" badge to provider cards on both browse.html and browse-preview.html (chosen over "Verified"/"Reviewed" — accurate for every care type including nanny/babysitter listings that don't have formal credentials)
- Added coverage-expansion copy: Greensboro, High Point, and Durham mentioned as planned-next cities in index.html's meta description, waitlist section, and FAQ, plus browse-preview.html's meta description (Wilmington/Raleigh/Charlotte kept as the active hero pill)
- Added OG/Twitter social-preview meta tags + a branded 1200×630 og-image.png to index.html, provider.html, and browse-preview.html (image generated locally via macOS WebKit/Swift — no new service)
- Added a "See what it'll look like →" link from index.html's parent waitlist card to browse-preview.html
- Built browse-preview.html: 4 sample provider cards, responsive grid, working client-side filters, clearly labeled as sample data
- Confirmed provider.html's pending/live status system (statusRow + pendingNote card) was already solid — no changes needed there
