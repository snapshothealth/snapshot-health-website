# SnapShot Health Contract Documents — Deployment Guide

**Last updated:** May 8, 2026
**Status:** Production-ready. HubSpot Forms API integration live.
**Stack:** Netlify (static hosting)

---

## What's Deployed

Five HTML documents at the root of the snapshothealth.io repo, plus a `_redirects` file for clean URLs.

| File | Live URL | Role |
|---|---|---|
| `review-and-sign.html` | `/review-and-sign` | Explainer/landing page. Friendly walkthrough of what's changed, with a CTA to the signing page. URL params forwarded. |
| `sign_everything.html` | `/sign` and `/sign-everything` | The signing page — "One signature. Three agreements." Pre-fillable. POSTs to HubSpot Forms API on submit. |
| `clinician-terms.html` | `/clinician-terms` | Master Service Agreement (reference). |
| `baa.html` | `/baa` | Business Associate Agreement (reference). |
| `order-form.html` | `/order` | Service Order Form preview (pricing, terms, device catalog). |
| `_redirects` | (config file) | Maps the clean URLs above to the underlying `.html` files. |

All five share the locked brand system: Outfit sans + JetBrains Mono + Dancing Script (typed signatures), navy/teal/cyan palette (`#0A1120` / `#00D4AA` / `#00B4D8`).

---

## Sales Flow (How a Practice Gets to Signing)

1. Discovery call with Jimmy goes well.
2. Jimmy advances the deal to **Proposal** stage in HubSpot.
3. Jimmy sends the prospect a pre-filled link to `/review-and-sign`:

   ```
   https://snapshothealth.io/review-and-sign?practice=PRACTICE+NAME&signer=DR+NAME&title=TITLE&email=EMAIL
   ```

   Example:

   ```
   https://snapshothealth.io/review-and-sign?practice=Signet+Heart+Group&signer=Dr.+Nikhil+Joshi&title=Managing+Physician&email=dr.joshi@signet.com
   ```

4. Prospect lands on the explainer page, reads the five-card walkthrough, optionally clicks through to the full MSA / BAA / Order Form.
5. Prospect clicks the **"Sign in under a minute"** CTA → routed to `/sign-everything` with URL params forwarded (and pre-filled fields visually marked with a subtle green tint).
6. Prospect types or draws a signature, checks the agreement box, hits Submit.
7. Form POSTs to HubSpot Forms API. Submission lands in HubSpot as a Form Submission tied to the contact.

**Encoding rules** for pre-filled URLs:
- Spaces → `+` or `%20`
- Ampersands in practice names ("Smith & Sons") → `%26`
- HubSpot personalization tokens can build these automatically once the proposal email template is set up.

---

## HubSpot Integration — Already Wired

`sign_everything.html` POSTs directly to the HubSpot Forms API. Configuration lives near the top of the submission script:

- **Portal ID:** `47989991`
- **Form GUID:** `06eff455-1326-4fbc-b96d-88b4f4427c0a`
- **Endpoint:** `https://api.hsforms.com/submissions/v3/integration/submit/{portalId}/{formGuid}`

The payload includes practice name, signer first/last (split on the final space), email, jobtitle, signature mode (typed/drawn), signature data, agreement timestamp, page URI, and the HubSpot tracking cookie (hutk) when present. This goes into HubSpot's `objectTypeId: 0-1` (Contact) and creates/updates the contact record.

**What's NOT yet built (HubSpot side):** the Workflow that fires on form submission to advance deal stage, generate a PDF, email the signer a copy, and create internal onboarding tasks. That's the highest-priority next task and is configured inside HubSpot, not in any HTML file. No site redeploy required when that's wired.

---

## Deploying Updates

### Step 1 — Locate the publish directory

Netlify dashboard → snapshothealth.io site → **Site configuration** → **Build & deploy** → look for **Publish directory**. It's the repo root (`.`) for this project. Files live at the root alongside `index.html`.

### Step 2 — Push changes via Git

The site is Git-connected. Standard flow:

```bash
git add .
git commit -m "Describe the change"
git push
```

Netlify auto-deploys on push to `main`. Build takes 30–60 seconds.

### Step 3 — Smoke test the URLs

After every deploy, hit each URL and verify it loads:

- [ ] `https://snapshothealth.io/review-and-sign` → explainer page
- [ ] `https://snapshothealth.io/sign` → signing page (sign_everything)
- [ ] `https://snapshothealth.io/sign-everything` → same signing page
- [ ] `https://snapshothealth.io/clinician-terms` → MSA
- [ ] `https://snapshothealth.io/baa` → BAA
- [ ] `https://snapshothealth.io/order` → Order Form preview
- [ ] `https://snapshothealth.io/review-and-sign?practice=Test+Practice&signer=Dr.+Test&title=Managing+Physician&email=test@test.com` → fields auto-populate, CTA forwards params to `/sign-everything`

If any URL 404s, the `_redirects` file isn't at the publish directory root or has been overwritten without the necessary entries. Verify it contains all six `/clinician-terms`, `/baa`, `/order`, `/review-and-sign`, `/sign`, `/sign-everything` mappings (with trailing-slash variants).

### Step 4 — Click through end-to-end on a real signing

After any change to `sign_everything.html`, do a full live test:

1. Open `/review-and-sign` with test URL params.
2. Click the CTA → land on `/sign-everything`.
3. Verify pre-filled fields display correctly with the green tint.
4. Click both the MSA and BAA links → confirm they open in new tabs.
5. Type or draw a signature.
6. Submit.
7. Confirm the success state appears.
8. **Verify the submission landed in HubSpot** — check Contacts and Form Submissions in the HubSpot dashboard. The contact should appear with all fields populated.

If submission fails, open DevTools first and watch the network tab for the call to `api.hsforms.com`. The console logs `[SnapShot Health] HubSpot rejected submission:` with details on validation failures.

---

## Rollback Plan

One-click in Netlify:

1. Site dashboard → **Deploys** tab
2. Find the previous good deploy
3. Click the ⋯ menu → **Publish deploy**
4. Live in ~30 seconds.

This reverts the HTML files but does NOT undo HubSpot form submissions that already came in. Those are permanent records in HubSpot.

---

## Known Future Work

These don't block the live flow but are worth doing when the calendar allows:

1. **HubSpot Workflow on form submission** — the highest-priority item. Should advance deal stage, generate a PDF of the executed agreement, email the signer a copy, create internal onboarding tasks for Lucy and Angie, and trigger Make.com scenarios for Transtek and (eventually) QuickBooks. Built entirely inside HubSpot — no code change to the HTML files.

2. **Post-signing PDF generation** — signers see a success message but don't yet receive a PDF copy. Three options: (a) HubSpot Workflow + a PDF service like DocRaptor or PDFShift, (b) Netlify Function + headless Chrome / Puppeteer, (c) client-side PDF generation (jsPDF or html2pdf) before submit, attached to the HubSpot payload.

3. **Email confirmation to signer** — pairs naturally with PDF generation. HubSpot Workflow can send a templated email with the PDF attached as soon as a Form Submission lands.

4. **Attorney review** — recommended before too many practices sign. Strongly recommended to have a Texas healthcare attorney eyeball §01 Scope and the Texas Medicaid clause in the MSA. ~$500. Catches edge cases an LLM-drafted document might miss.

5. **CPT code maintenance** — the 2026 fees in `sign_everything.html` and `order-form.html` are correct as of January 2026. When CMS publishes the 2027 PFS in late 2026, both files need to be updated together. Run a `grep "9945"` and similar before deploying to make sure no rate is stale.

---

## File Inventory in This Bundle

All files live in `/mnt/user-data/outputs/` after generation:

- `_redirects`
- `PROJECT_HANDOFF.md`
- `DEPLOYMENT_GUIDE.md` (this document)

The five HTML files are already in the GitHub repo and aren't regenerated unless explicitly asked.
