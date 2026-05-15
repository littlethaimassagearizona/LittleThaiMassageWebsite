# Little Thai Massage — Website

A static website for the North Scottsdale studio at **7124 E Becker Lane, Unit A**. Replaces the existing Wix site at `littlethaimassagearizona.com`.

The main page is `index.html`. Editing copy means editing one file; temporary migrated media lives in `assets/`. Deploys to **GitHub Pages now** and can move unchanged to **Cloudflare Pages later** with zero build step.

---

## What it does

- **Booking** via embedded self-hosted Cal.diy — service-specific deep links (Thai 60, Thai 90, Swedish 60, etc.) so each "Book →" lands the customer on the right event type, not a generic picker.
- **Payment at booking** via Cal.diy's Stripe paid-events app — configured in the Cal.diy instance and Stripe Dashboard, no Stripe secrets in this website.
- **Gift cards** via Stripe Payment Links — $50 / $95 / $165 / $190 buttons that open Stripe Checkout with required recipient fields. Net-new revenue line vs. the current Wix site.
- **Local SEO** via `MassageBusiness` + `FAQPage` JSON-LD, geo meta, NAP-consistent address/phone, opening hours spec, offer catalog with all 10 services and prices.
- **Mobile sticky CTA bar** — persistent Call + Book buttons on phones for conversion.
- **First-time FAQ** — what to wear, parking, tipping, cancellation. Reduces booking friction and earns rich snippets in Google.

---

## Setup checklist (in order)

### 1. Deploy Cal.diy and set the booking domain

Recommended booking domain: `https://cal.littlethaimassagearizona.com`.

Deploy Cal.diy separately from this static website. GitHub Pages can only serve static files; it cannot run the Cal.diy Docker container, hold API secrets, terminate webhooks, or run a database. The website assumes:

| Value | Current website setting |
|---|---|
| Cal.diy base URL | `https://cal.littlethaimassagearizona.com` |
| Cal.diy username | `littlethaimassagearizona` |
| Default event | `thai-60` |

If you choose a different Cal.diy base URL, update `CAL_DIY_BASE_URL` in `index.html`. If you choose a different username, update `CAL_DIY_USERNAME` and replace all `littlethaimassagearizona/` path references in `index.html`.

Cal.diy is self-hosted infrastructure. Keep the Cal.diy database, app secrets, SMTP, OAuth, Stripe keys, API keys, and backups outside this website repo.

The static site does not need a Cal.diy API key for the current booking flow. It only loads Cal.diy's public embed script and links customers to event-type URLs. If a later version needs server-side Cal.diy API calls, put the API key in a Cloudflare Worker secret or in the Cal.diy host environment, never in `index.html`, GitHub Pages settings, or committed files.

### 2. Configure Stripe inside Cal.diy for paid bookings

In the Cal.diy deployment environment, set:

| Variable | Value source |
|---|---|
| `NEXT_PUBLIC_STRIPE_PUBLIC_KEY` | Stripe publishable key, `pk_live_...` |
| `STRIPE_PRIVATE_KEY` | Stripe secret key, `sk_live_...` |
| `STRIPE_CLIENT_ID` | Stripe Connect client ID, `ca_...` |
| `STRIPE_WEBHOOK_SECRET` | Stripe webhook signing secret, `whsec_...` |

In Stripe Dashboard:

1. Enable Stripe Connect OAuth for Standard accounts.
2. Add redirect URL: `https://cal.littlethaimassagearizona.com/api/integrations/stripepayment/callback`.
3. Add connected-application webhook endpoint: `https://cal.littlethaimassagearizona.com/api/integrations/stripepayment/webhook`.
4. Subscribe the webhook to all `payment_intent.*` and `setup_intent.*` events.
5. Restart/redeploy Cal.diy after setting the env vars, then install/activate Stripe in the Cal.diy app store.

Stripe publishable keys start with `pk_`. A value with another prefix is not the publishable key Cal.diy expects for `NEXT_PUBLIC_STRIPE_PUBLIC_KEY`. Gift-card Payment Links do not need a publishable key in this repo.

### 3. Create the 10 event types in Cal.diy

In Cal.diy → **Event Types** → **New**. Create each one exactly as below — the URL slug must match what the website references:

| Slug | Title | Duration | Price |
|---|---|---|---|
| `thai-60` | 60 min Traditional Thai Massage | 60 min | $95 |
| `thai-90` | 90 min Traditional Thai Massage | 90 min | $135 |
| `swedish-60` | 60 min Swedish Massage | 60 min | $95 |
| `swedish-90` | 90 min Swedish Massage | 90 min | $135 |
| `combination-60` | 60 min Thai & Swedish Combination | 60 min | $95 |
| `combination-90` | 90 min Thai & Swedish Combination | 90 min | $135 |
| `deep-tissue-60` | 60 min Deep Tissue Massage | 60 min | $95 |
| `deep-tissue-90` | 90 min Deep Tissue Massage | 90 min | $135 |
| `sauna-90` | 90 min Sauna | 90 min | $165 |
| `facial-90` | 90 min Facial Massage | 90 min | $165 |

For each event type, set:
- **Buffer time**: 15 min before, 15 min after (gives the therapist room to reset).
- **Minimum notice**: 1 hour to match current Wix; raise to 2 hours if the studio wants more buffer.
- **Limit future bookings**: 60 days (prevents people booking 6 months out and forgetting).
- **Working hours**: 10 am – 9 pm, all 7 days (match your Schedule).
- **Location**: "In Person" → `7124 E Becker Lane, Unit A, Scottsdale, AZ 85254`.
- **Payment**: enable Stripe paid event and set the exact price from the table.

### 4. Add required Cal.diy booking fields

Add these booking questions to every event type. Keep the slugs exact; they are the field keys you will see in booking exports/API payloads.

| Slug | Label | Type | Required | Options / helper text |
|---|---|---|---|---|
| `first_visit` | Is this your first visit with Little Thai Massage? | Dropdown or radio | Yes | `Yes`, `No` |
| `pressure_preference` | Pressure preference | Dropdown or radio | Yes | `Light`, `Medium`, `Firm`, `Therapist recommendation` |
| `focus_areas` | Areas to focus on today | Long text | Yes | Example: shoulders, back, hips, feet, relaxation, mobility |
| `policy_ack` | I understand the cancellation and payment policy | Checkbox | Yes | Required single checkbox |

Do not collect card numbers, Social Security numbers, full medical history, or other sensitive regulated information in booking fields. If the studio needs medical intake, handle that in person or through a purpose-built intake form.

### 5. Create Stripe Payment Links for gift cards

In Stripe Dashboard → **Payment Links** → **New**. Create four products:

| Product name | Price | Use |
|---|---|---|
| Little Thai Gift Card — $50 | $50 | Single payment |
| Little Thai Gift Card — $95 | $95 | Single payment |
| Little Thai Gift Card — $165 | $165 | Single payment |
| Little Thai Gift Card — $190 | $190 | Single payment |

For each Payment Link, configure:

| Stripe setting | Value |
|---|---|
| Collect customer email | Required |
| Collect customer phone number | Required |
| Payment methods | Dynamic payment methods, or at minimum card + Link |
| Confirmation | Stripe-hosted confirmation page is fine for v1 |

Add these required custom fields to every gift-card Payment Link:

| Key | Label | Type | Required |
|---|---|---|---|
| `recipient_name` | Recipient name | Text | Yes |
| `recipient_email` | Recipient email | Text | Yes |
| `sender_name` | Sender name | Text | Yes |

Stripe Payment Links allow only a small number of custom fields, so the required fields above intentionally prioritize fulfillment. Do not collect protected or sensitive data in Stripe custom fields.

For each Payment Link, copy the generated URL (looks like `https://buy.stripe.com/xxxxx`). In `index.html`, find the four `https://buy.stripe.com/REPLACE_*` placeholders and replace with your real URLs.

**Gift card fulfillment workflow** (until automated): Stripe emails you on each gift card purchase. Email the recipient a code (e.g. `LTM-50-A4B2`) and write it in a notebook with the redemption status. When redeemed, apply as cash discount at checkout. For volume, look at Square Gift Cards or a dedicated gift card platform later.

### 6. Verify Google Business Profile NAP consistency

Open your **Google Business Profile** (the listing tied to littlethaimassagearizona@gmail.com). Confirm these exactly match what's on the website:

- **Name**: Little Thai Massage
- **Address**: 7124 E Becker Lane, Unit A, Scottsdale, AZ 85254
- **Phone (primary)**: (602) 999-9023 — this is the local Phoenix number; use as primary for SEO
- **Hours**: Mon–Sun, 10:00 AM – 9:00 PM
- **Website**: https://www.littlethaimassagearizona.com
- **Category**: Massage therapist (primary), Massage spa (secondary), Thai massage therapist (if available)

Inconsistencies between the site and Google Business hurt local rank. Fix any mismatch on Google's side, not the site's.

### 7. Deploy without Vercel

Use three independent surfaces:

| Surface | Temporary / pre-transfer host | Long-term Cloudflare host | Secrets? |
|---|---|---|---|
| Marketing site | GitHub Pages | Cloudflare Pages or GitHub Pages behind Cloudflare DNS | No |
| Booking app | Cal.diy Docker host on Railway, Render, Northflank, VPS, etc. | Same Cal.diy host, DNS managed in Cloudflare | Yes |
| Gift cards | Stripe Payment Links | Stripe Payment Links | No website secrets |

**Phase A — publish the static site on GitHub Pages**

1. Push this repo to GitHub.
2. Repo → **Settings** → **Pages** → **Source**: deploy from branch `main`, folder `/`.
3. GitHub Pages will serve the staging URL at `https://littlethaimassagearizona.github.io/LittleThaiMassageWebsite/`.
4. Keep the included `.nojekyll` file. This disables Jekyll processing and lets GitHub Pages serve the repo as plain static files.
5. Do not add a `CNAME` file until you are ready for the custom-domain cutover. Adding the custom domain early makes the `github.io` staging URL redirect to the production domain.

**Phase B — deploy Cal.diy before moving the domain**

1. Deploy Cal.diy from Docker on a host that can run a persistent web app and Postgres.
2. Set `NEXT_PUBLIC_WEBAPP_URL` to the final booking origin, preferably `https://cal.littlethaimassagearizona.com`.
3. Configure the Stripe env vars from Step 2 on the Cal.diy host.
4. If the domain is still registered at Wix, add a DNS record at Wix for `cal.littlethaimassagearizona.com` pointing to the Cal.diy host. Domain registration transfer is separate from DNS cutover.
5. If you cannot point the `cal` subdomain yet, use the platform URL temporarily and update `CAL_DIY_BASE_URL` in `index.html`.

**Phase C — point the production domain to GitHub Pages while still registered at Wix**

1. In GitHub Pages settings, add `www.littlethaimassagearizona.com` as the custom domain.
2. In Wix DNS, set `www` as a CNAME pointing to `littlethaimassagearizona.github.io`.
3. For the apex domain, set GitHub Pages `A` records:
   - `185.199.108.153`
   - `185.199.109.153`
   - `185.199.110.153`
   - `185.199.111.153`
4. Enable **Enforce HTTPS** in GitHub Pages after GitHub finishes certificate provisioning.
5. Keep the existing Wix subscription paid for about 7 days after cutover. Cancel only after bookings, Stripe payments, email, and Google indexing look normal.

**Phase D — transfer DNS/registration to Cloudflare**

1. Recreate the same DNS records in Cloudflare before or during the registrar transfer.
2. Conservative first setting: keep GitHub Pages as the origin and use Cloudflare DNS records with GitHub Pages HTTPS working.
3. Cleaner long-term setting: create a Cloudflare Pages project connected to this GitHub repo, set build command to blank, output directory to `/`, then attach `www.littlethaimassagearizona.com`.
4. Keep `cal.littlethaimassagearizona.com` pointed at the Cal.diy Docker host. Cloudflare Pages will host the static website, not the Cal.diy container.

---

## File-level replacement checklist

Find-and-replace these placeholders in `index.html` before deploying:

| Placeholder | What to put there |
|---|---|
| `CAL_DIY_BASE_URL` | Your self-hosted Cal.diy origin if not `https://cal.littlethaimassagearizona.com` |
| `CAL_DIY_USERNAME` | Your Cal.diy username if not `littlethaimassagearizona` |
| `littlethaimassagearizona/` path references | Your Cal.diy `username/` path if different |
| `https://buy.stripe.com/REPLACE_50` | Stripe Payment Link for $50 gift card |
| `https://buy.stripe.com/REPLACE_95` | Stripe Payment Link for $95 |
| `https://buy.stripe.com/REPLACE_165` | Stripe Payment Link for $165 |
| `https://buy.stripe.com/REPLACE_190` | Stripe Payment Link for $190 |
Optional but recommended:
- Add a `favicon.ico` and `apple-touch-icon.png` to the repo root.
- Replace `assets/spa-detail.jpg` with a real photo of the studio interior or treatment room.
- Link previews currently use `assets/spa-detail.jpg`. If you later want a dedicated 1200×630 social preview image, add one and update the two `og:image` / JSON-LD image references in `index.html`.

---

## Editing copy

Everything is in `index.html`. The most-edited zones:

- **Hero headline**: search for `Traditional Thai massage,`.
- **Service descriptions** in `<small>` tags inside `.services-list`.
- **About copy** in `.about-copy`.
- **FAQ answers** in `.faq-list` and also mirrored in the JSON-LD schema at the top of `<head>` (keep them in sync for SEO).

---

## Decisions I made that you should weigh in on

1. **Primary phone is (602) 999-9023, not (619)**. The 619 is San Diego; using a local Phoenix number for the primary listing helps Scottsdale local SEO. The 619 is shown as a secondary number. Push back if (619) is the one customers actually call.

2. **Sauna service description.** Current Wix site says only "90 min Sauna," so the site keeps the copy generic. Add specifics only if the owner confirms whether it is sauna-only, far-infrared, or sauna + massage.

3. **Facial Massage duration.** The visible Wix card omits a duration, but the live Wix booking data configures it as a 90-minute appointment. I used `facial-90`; confirm with the owner before creating the Cal.diy event.

4. **Combination pricing.** Combination is priced identically to solo Thai or Swedish ($95 / $135). That's fine, but unusual — most studios charge a premium for combination. Confirm this is intentional.

5. **Therapist credential claims.** The site stays low-specificity with "trained in the Thai tradition" and "*nuad boran*." If your therapists have specific certifications (Wat Pho, ITM Chiang Mai, AZ state license #, etc.), add them to the About section — concrete credentials build trust.

6. **Hero photo / studio photo / OG image.** Temporary migrated Wix media is included so the site has no broken image slots. Replace it with 5–10 real phone photos of the studio (treatment room, sauna, lobby, exterior signage) in natural light.

7. **Gift card fulfillment is manual.** I gave you Stripe Payment Links, which means you'll get an email per sale and need to email a code to the recipient. Acceptable for volume <20/month. Beyond that, look at Square Gift Cards (~$15/mo) or a dedicated platform.

8. **Cancellation policy**. I matched the current Wix booking policy: free cancellation at least 2 hours before the appointment, 50% within 2 hours, 100% no-show. Confirm before launch — it's also encoded in the FAQ schema.

9. **Memberships / packages.** Not built. Thai studios commonly offer "buy 5 get 1 free." Worth considering as a future addition — drives repeat visits and prepayment.

10. **Reviews.** The site has no review widget yet. Once Google Business has 30+ reviews, consider adding a small "★ 4.9 · 200+ Google reviews" trust line in the hero. Don't fake it — wait until you have real reviews to display.

---

## What's not here

- **Multi-page structure** (separate About / Services / Book pages). Not needed at this scale — single-page sites convert better for local service businesses and Google ranks them fine if the schema is clean.
- **Build step / framework**. This is intentionally static. If you later want a CMS or server-side payments outside of Cal.diy, add a small Cloudflare Worker or port the site to a framework hosted on Cloudflare Pages.
- **Analytics**. Add Plausible, Cloudflare Web Analytics, or another privacy-preserving analytics script in `<head>` once deployed.
- **Newsletter / email capture**. Not included; the gift card and booking flows already capture emails through Stripe and Cal.diy.

---

## Domain & email

Domain `littlethaimassagearizona.com` is currently registered through Wix. You do not need to wait for the registrar transfer to launch the new site. Change DNS at Wix first, verify the new site and booking/payment flows, then transfer registration to Cloudflare when the production path is stable.

Email at `littlethaimassagearizona@gmail.com` is on Google (Gmail) and is unaffected by the website migration.
