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
- `index.html` — landing page. Parent CTA → Tally waitlist (intentional until enough listings exist to make browsing worthwhile). Provider CTAs → provider.html. Footer links to legal pages.
- `provider.html` — provider portal: email/password auth (Supabase), profile create/edit, photo upload, approval-status badge, inbox of parent inquiries. Signup requires a T&C agreement checkbox.
- `browse.html` — public parent-facing search: filter by city + care type, provider cards (photo banner if photo_url set), inquiry modal that inserts into `inquiries`. Not yet linked from the homepage nav (deliberate soft launch).
- `terms.html`, `privacy.html` — legal pages, styled to match. Effective June 12, 2026.

## Database (Supabase)
**providers** — id (uuid, = auth.users id), business_name, contact_name, email, phone, city, care_type, ages_served (text), price_range (text, free-form), bio, photo_url, is_approved (boolean, default false), created_at.

**inquiries** — id, provider_id (FK), parent_name, parent_email, child_age_range, message, is_read, created_at.

**RLS (all enforced, do not weaken):**
- Public can SELECT providers only where is_approved = true; providers can SELECT/INSERT/UPDATE only their own row; nobody can set is_approved through the client — approval is done manually in the Supabase Table Editor (that's the admin workflow).
- Anyone can INSERT inquiries (parents have no accounts); providers can SELECT/UPDATE only inquiries where provider_id = their auth.uid().

**Storage:** public bucket `provider-photos`, 5 MB limit, image/jpeg|png|webp only. Policies restrict insert/update/delete to a folder named with the uploader's auth.uid(); path convention `{user_id}/listing.{ext}`, upsert + cache-busting `?v=` query param.

## Product principles (non-negotiable)
1. **Data minimization** — never collect children's names, birthdates, or photos. Inquiries use age ranges only. Photo-upload UI tells providers not to include children's faces.
2. **Honest claims** — listing review is a basic screening, not verification. Marketing copy must never promise more than terms.html disclaims (see FAQ wording on index.html). No fake stats or invented user counts.
3. **Approval gate** — every listing starts is_approved = false and is invisible until Nikieta approves it (her vetting includes checking NC DCDEE public records).
4. **Free tier first** — GitHub Pages + Supabase free tier + Tally free. Founding providers get 6 months free; no payment processing exists yet.

## Design system
Cream background #FDFAF5; ink #11302E; body text #4F5D58; teal (trust) #0E4D4A with soft #E3EFEA; coral (warmth/CTAs) #E76F51, hover #C9543A, soft #FBE6DE; lines #E8E0D3; success green #1D9E75. Rounded (10–26px radii), pill buttons, warm + personable small-business tone — never corporate. Headings Fraunces, everything else Nunito Sans.

## Parked checklist (the backlog)
- Replace hero illustration on index.html with a real photo (instructions are in a comment in the file)
- Graphics fine-tuning pass
- Re-enable Supabase email confirmation before parent-facing launch (Authentication → set Site URL to https://childcareconnectnc.com first; it was disabled deliberately to reduce founding-provider signup friction)
- Browse sorting: name A–Z/Z–A (easy); by price (requires adding a numeric starting_price column — price_range is free text); by rating (requires building a reviews system: table + RLS + moderation + display)
- Reviews system (prerequisite for rating sort)
- Eventually: link browse.html from the homepage and switch the parent CTA from the Tally waitlist to browse (when there are enough live listings)
- Eventually: paid provider subscriptions via Stripe (terms.html already covers this in "when paid subscriptions launch" language)

## Working with Nikieta (required workflow)
- **Explain before you act.** Before making changes, state in plain English what you're about to change and why. After changes, summarize exactly what was modified and how to test it on the live site.
- **She asks questions — answer them properly.** She's technical (QA engineer) but new to web development. When she asks how or why something works, explain it clearly without jargon-dumping and without condescension. Teaching her the system is part of the job.
- **Security changes require her explicit approval first.** Anything touching RLS policies, storage policies, auth settings, or Supabase configuration: propose it, explain the risk/benefit, and wait for her yes before applying or committing.
- **She is the QA gate.** After every deploy, give her a short test checklist for the live site. Nothing is "done" until she's verified it works.
- **Never commit secrets.** No service_role key, no database password, no personal data in the repo — ever, including in comments or commit messages.
- **Don't expand scope on your own.** Stick to what she asked; if you spot something else worth doing, suggest it and let her decide.

## Current business state (June 2026)
Founding-provider outreach in progress (Wilmington first: DMs from her personal Facebook + emails from the business gmail; founding offer = 6 months free, no card). First real provider signups expected via provider.html. Approval workflow: Supabase Table Editor → providers → set is_approved = true.
