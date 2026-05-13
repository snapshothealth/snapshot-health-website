# SnapShot Health Contract Documents — Deployment Guide

**Last updated:** May 13, 2026
**Status:** Production-ready. End-to-end signing pipeline live (page → HubSpot → Make.com → PDFShift → Gmail).
**Stack:** Netlify (static hosting), HubSpot Forms API, Make.com automation, PDFShift (PDF rendering), Gmail (delivery).

---

## What's Deployed

Six HTML documents at the root of the snapshothealth.io repo, plus a `_redirects` file for clean URLs.

| File | Live URL | Role |
|---|---|---|
| `review-and-sign.html` | `/review-and-sign` | Explainer/landing page. Friendly walkthrough of what's changed, with a CTA to the signing page. URL params forwarded. Optional warm intro path. |
| `sign_everything.html` | `/sign` and `/sign-everything` | The signing page — "One signature. Three agreements." Pre-fillable via URL params (`program`, `audience`, `practice`, `signer`, `email`, etc.). POSTs to HubSpot Forms API on submit. Also serves as the source for the executed PDF rendered by PDFShift (`?pdf_mode=1`). |
| `clinician-terms.html` | `/clinician-terms` | Master Service Agreement for **physician practices** (reference). |
| `hha-clinician-terms.html` | `/hha-clinician-terms` | Master Service Agreement for **Home Health Agencies** (reference). Used automatically when `audience=hha`. |
| `baa.html` | `/baa` | Business Associate Agreement (reference). |
| `order-form.html` | `/order` | Service Order Form preview (pricing, terms, device catalog). |
| `_redirects` | (config file) | Maps the clean URLs above to the underlying `.html` files. |

All six share the locked brand system: Outfit sans + JetBrains Mono + Dancing Script (typed signatures), navy/teal/cyan palette (`#0A1120` / `#00D4AA` / `#00B4D8`).

---

## Sales Flow — Two Paths to the Signing Page

### Path A — Warm intro (`/review-and-sign` explainer first)

For prospects who need context before signing. Jimmy sends:

```
https://snapshothealth.io/review-and-sign?practice=PRACTICE+NAME&signer=DR+NAME&title=TITLE&email=EMAIL
```

Example:

```
https://snapshothealth.io/review-and-sign?practice=Signet+Heart+Group&signer=Dr.+Nikhil+Joshi&title=Managing+Physician&email=dr.joshi@signet.com
```

Prospect reads the five-card explainer, optionally clicks through to the full MSA / BAA / Order Form, then clicks the **"Sign in under a minute"** CTA. URL params forward to `/sign-everything`.

### Path B — Direct signing link (skip the explainer)

For existing customers and warm prospects who don't need the walkthrough. Jimmy sends:

```
https://snapshothealth.io/sign?program=PROGRAM&audience=AUDIENCE&practice=...&signer=...&email=...
```

**Valid `program` values:** `turnkey` (Remote Concierge Care), `software-no-device` (Software 20), `software-with-device` (Software 45)

**Valid `audience` values:** `physician`, `hha`

**Compatibility rules** enforced by the page guard:
- `turnkey` + `physician` ✓
- `software-no-device` + `physician` ✓
- `software-with-device` + `physician` ✓
- `software-with-device` + `hha` ✓ (HHA is only offered this program)

Examples:

```
https://snapshothealth.io/sign?program=turnkey&audience=physician
https://snapshothealth.io/sign?program=software-no-device&audience=physician&practice=Cedar+Valley+Family+Medicine
https://snapshothealth.io/sign?program=software-with-device&audience=hha
```

### What happens after submit (both paths)

1. Prospect on `/sign-everything` types or draws signature, checks the agreement box, hits Submit.
2. Form POSTs directly to HubSpot Forms API. Submission lands in HubSpot as a Form Submission and Contact.
3. Make.com scenario polls HubSpot every 15 minutes for new submissions.
4. Make.com runs: Contact upsert → Deal search → Deal create/update → PDFShift renders executed PDF → Gmail sends branded welcome email with PDF attached, from billing@snapshothealth.io.
5. New Deal lands in HubSpot **Customer Acquisition** pipeline at **Signed** stage, associated to the Contact, with custom properties populated (Service Program, Audience, term, signed_at).
6. Practice receives "Welcome to SnapShot Health — your executed agreement for [Practice Name]" email within ~15 minutes of submitting.

**Encoding rules** for pre-filled URLs:
- Spaces → `+` or `%20`
- Ampersands in practice names ("Smith & Sons") → `%26`
- HubSpot personalization tokens can build these automatically once the proposal email template is set up.

---

## HubSpot Forms API Integration

Wired in `sign_everything.html`. Configuration lives near the top of the submission script.

- **Portal ID:** `47989991`
- **Form GUID:** `06eff455-1326-4fbc-b96d-88b4f4427c0a`
- **Endpoint:** `https://api.hsforms.com/submissions/v3/integration/submit/{portalId}/{formGuid}`

The payload includes practice name (Company name), signer first/last (split on the final space), email, jobtitle, signature mode (typed/drawn), signature data, agreement timestamp, page URI, the HubSpot tracking cookie (hutk) when present, AND — critically — the **`program` URL slug** as a dedicated Contact property. The slug is the single source of truth that flows from page → HubSpot → Make.com → PDFShift without any string transforms.

If you add a new program tier in the future, the change touches three places: (1) `sign_everything.html`'s `PROGRAM_CONFIG` object, (2) HubSpot's `Service Program` dropdown options (display value), and (3) ensure the page's POST payload still sends the slug to the `program` Contact property. Drift between any of these will produce the "Signing link incomplete" error page on the rendered PDF.

---

## Make.com Automation Pipeline

The post-submission pipeline runs in Make.com (scenario name: "SnapShot Health — Form to Executed PDF"). Polls HubSpot every 15 minutes for new form submissions.

### Modules

| # | Module | What it does |
|---|---|---|
| 1 | HubSpot › Watch Submissions for a Form | Trigger. Polls the signing form for new submissions. |
| 34 | HubSpot › Create or Update a Contact | Upserts by email. Maps email, firstname, lastname, jobtitle, company. |
| 15 | HubSpot › Search for Deals | Looks for an existing Deal by practice name. |
| 16 | Router | Splits into "Deal Exists" / "Deal Doesn't Exist" branches. |
| 17 | HubSpot › Update a Deal (Exists branch) | Updates the existing Deal with new agreement details. |
| 18 | HubSpot › Create a Deal (Doesn't Exist branch) | Creates new Deal in Customer Acquisition pipeline at Signed stage, associated to Contact (associationTypeId 3). |
| 22 / 32 | HTTP › POST to PDFShift `/v3/convert/pdf` | Renders the executed agreement PDF by fetching `https://snapshothealth.io/sign?pdf_mode=1&program=...&audience=...&practice=...&signer=...&...`. Returns binary PDF. Modules 22 and 32 are identical (one per Router branch). |
| 23 / 33 | Gmail › Send an email | Sends branded welcome email from billing@snapshothealth.io with the PDF attached. Personalized greeting, practice name in body, Outfit-styled HTML template. |

### Make.com Maintenance — Critical Notes

- **Modules 22 and 32 must be kept identical.** Any change to the PDFShift URL body content goes in both, or one Router branch will silently fail.
- **The current PDFShift URL body uses `{{1.fields.program}}` directly** (no `switch()`, no `lower(replace(...))`). This is intentional — the page POSTs the lowercase URL slug to HubSpot's `program` field, and Make.com just reads it back. Don't reintroduce string transforms; they were the source of multiple bugs during the May 13 build.
- **Polling cadence is 15 minutes.** Practices won't see their welcome email instantly. If a faster cadence becomes important, switch the trigger from "Watch Submissions" to an Instant Webhook configured against the HubSpot form.
- **Welcome email uses inline-styled Outfit font.** Only Gmail web reliably renders Google Fonts; other clients fall back to the system stack. Acceptable trade-off.

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
- [ ] `https://snapshothealth.io/clinician-terms` → Physician MSA
- [ ] `https://snapshothealth.io/hha-clinician-terms` → HHA MSA
- [ ] `https://snapshothealth.io/baa` → BAA
- [ ] `https://snapshothealth.io/order` → Order Form preview
- [ ] `https://snapshothealth.io/sign?program=turnkey&audience=physician` → signing page configures for Turnkey/Physician
- [ ] `https://snapshothealth.io/sign?program=software-no-device&audience=physician` → Software 20 / Physician
- [ ] `https://snapshothealth.io/sign?program=software-with-device&audience=hha` → Software 45 / HHA, with the HHA MSA linked
- [ ] `https://snapshothealth.io/review-and-sign?practice=Test+Practice&signer=Dr.+Test&title=Managing+Physician&email=test@test.com` → fields auto-populate, CTA forwards params to `/sign-everything`

If any URL 404s, the `_redirects` file isn't at the publish directory root or has been overwritten without the necessary entries. Verify it contains mappings for all of `/clinician-terms`, `/hha-clinician-terms`, `/baa`, `/order`, `/review-and-sign`, `/sign`, `/sign-everything` (with trailing-slash variants).

### Step 4 — End-to-end click-through (after any change to `sign_everything.html` or to a Make.com module)

This is the full pipeline test. Do it on every signing-page or Make.com change.

1. Open `/sign?program=turnkey&audience=physician` (or `software-with-device&audience=hha` to exercise the HHA branch).
2. Fill in test practice info (use `jimmy+TESTNAME@snapshothealth.io` for the email — plus-addressing routes back to your inbox).
3. Pick a term.
4. Click both the MSA and BAA links → confirm they open in new tabs.
5. Type or draw a signature.
6. Submit.
7. Confirm the success state appears on the page.
8. **Verify the submission landed in HubSpot** — check Contacts and the Customer Acquisition pipeline. Contact appears with all fields populated. Deal appears at Signed stage, associated to the Contact, with `Service Program`, `Audience`, and `program` (slug) properties set.
9. **Wait up to 15 minutes for Make.com to poll.** Check the Make.com scenario History — the run should show all modules green.
10. **Verify the executed PDF lands in the test inbox.** The attachment should be the actual signed Service Order Form (not the "Signing link incomplete" error page). Email subject: "Welcome to SnapShot Health — your executed agreement for [Practice]".

If submission fails on the page, open DevTools first and watch the network tab for the call to `api.hsforms.com`. The console logs `[SnapShot Health] HubSpot rejected submission:` with details on validation failures.

If the form submission lands in HubSpot but the email/PDF never arrives, open the Make.com History tab and find the failed run. Click into the failed module and read the Input → Body content to see the rendered URL or payload.

---

## Rollback Plan

### HTML changes — one-click in Netlify

1. Site dashboard → **Deploys** tab
2. Find the previous good deploy
3. Click the ⋯ menu → **Publish deploy**
4. Live in ~30 seconds.

This reverts the HTML files but does NOT undo HubSpot form submissions that already came in. Those are permanent records in HubSpot.

### Make.com changes — manual via scenario versioning

Make.com retains a revision history per scenario. To revert:

1. Open the scenario → click the clock icon (Scenario history) in the bottom toolbar.
2. Find the previous working version (timestamps and edit notes shown).
3. Click **Restore**.
4. Save the scenario.

This is independent of the Netlify rollback. If both a page change and a Make.com change shipped together and one broke, roll back both.

---

## Known Future Work

These don't block the live flow but are worth doing when the calendar allows. (Earlier items 1–3 from the May 8 guide — HubSpot Workflow, PDF generation, signer email — were all completed May 13 via the Make.com pipeline.)

1. **Cleanup items from the May 13 build** (see PROJECT_HANDOFF.md for details):
   - Deal `closedate` is off-by-one (timezone parse issue)
   - Deal `Amount` is blank — pipeline reports show $0 closed
   - Redundant HubSpot form fields (`Program` vs `Service Program` vs `Program Name`)
   - System fields visible on form (Signature Data, User Agent, etc.) should be hidden
   - Re-signer vs new-signer copy split (currently "Truly nothing" is global)

2. **Attorney review** — strongly recommended to have a Texas healthcare attorney eyeball §01 Scope and the Texas Medicaid clause in the MSA before too many practices have signed. ~$500. Catches edge cases an LLM-drafted document might miss.

3. **CPT code maintenance** — the 2026 fees in `sign_everything.html` and `order-form.html` are correct as of January 2026. When CMS publishes the 2027 PFS in late 2026, both files need to be updated together. Run a `grep "9945"` and similar before deploying to make sure no rate is stale.

4. **Document `Customer Acquisition` pipeline structure** — the custom Deal properties exist (Service Program, Audience, program slug) but should be enumerated in a single HubSpot Properties document so a successor knows what each is for.

5. **Instant trigger consideration** — current Make.com polling is 15 min. If/when signing volume grows beyond 5/week, evaluate whether an Instant Webhook trigger is worth the setup effort.

---

## File Inventory in This Bundle

All files live in `/mnt/user-data/outputs/` after generation:

- `_redirects`
- `PROJECT_HANDOFF.md`
- `DEPLOYMENT_GUIDE.md` (this document)

The six HTML files are already in the GitHub repo and aren't regenerated unless explicitly asked.

---

*End of deployment guide. Last updated May 13, 2026, after the Make.com post-submission automation pipeline went live and end-to-end signing was validated for all three program/audience combinations.*
