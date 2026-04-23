# SnapShot Health Contract Documents — Deployment Guide

**Last updated:** April 23, 2026
**Status:** Production-ready, HubSpot wired
**Stack:** Netlify (static hosting) + HubSpot Forms API v3

---

## What You're Deploying

Eight files total: seven HTML documents + one Netlify routing config.

| File | Final URL | Purpose |
|---|---|---|
| `sign_everything.html` | `/sign-everything` | **Unified signing page.** Executes Order Form + MSA + BAA in one signature. Program-aware via `?program=` param. Submits to HubSpot. |
| `order-form.html` | `/order` | Service Order Form. Program-aware — shows turnkey CPT table OR software flat-fee card based on `?program=` param. |
| `clinician-terms.html` | `/clinician-terms` | Master Service Agreement. Dual-program language in §01, §03, §05. |
| `baa.html` | `/baa` | Business Associate Agreement (HIPAA §164.504(e)). |
| `review-and-sign.html` | `/review-and-sign` | Dual-audience (new + existing customer) prep page. CTA routes to `/sign-everything`. |
| `why-snapshot.html` | `/why-snapshot` | Trust-builder page. |
| `sign_and_start.html` | `/sign-and-start` | Legacy MSA+BAA-only signing page. Kept as fallback, not publicly linked. |
| `_redirects` | (config) | Maps clean URLs → HTML files. |

---

## The Three Programs

The signing flow and Order Form both support three program variants, driven by `?program=` URL parameter:

| URL param | Program | Pricing | Performance Guarantee |
|---|---|---|---|
| `turnkey` (default) | Remote Concierge Care | Per-CPT (17 codes) | Yes |
| `software-no-device` | Software Platform — No Device | $20/billable patient/month | No |
| `software-with-device` | Software Platform — With Device | $45/billable patient/month | No |

The `?program=` param flows through the customer journey end-to-end:
- HubSpot quote email links to `/review-and-sign?program=X&practice=Y&signer=Z&title=W&email=V`
- Review & Sign page forwards the param to `/sign-everything?program=X&...`
- Sign Everything page shows the right program badge, description, and PG indicator
- Customer clicks "view Service Order Form" → lands on `/order?program=X` with matching fee schedule

---

## Step-by-Step Deployment

### Step 1 — Locate your Netlify publish directory

Netlify dashboard → snapshothealth.io site → **Site settings** → **Build & deploy** → **Build settings** → **Publish directory**. Common values: `/`, `public`, `dist`, `build`, or `www`.

If you use drag-and-drop deploys, you'll upload the whole folder without worrying about a publish directory.

### Step 2 — Drop all eight files into the publish directory

**If you drag-and-drop deploy:** Put all eight files in one folder on your desktop, then drag that folder onto the Netlify Deploys tab. Wait for the green "Published" checkmark.

**If you git-deploy:** Copy the files into the correct location in your repo, commit, and push. Netlify deploys on push.

Critical gotcha: the `_redirects` file must be at the **root** of the publish directory and must have **no file extension** (just `_redirects`, not `_redirects.txt`). Some operating systems and editors add `.txt` silently — double-check after download.

### Step 3 — Delete deprecated file from Netlify

If you previously had `provider-terms.html` uploaded (it was split into MSA + BAA a while back), delete it. The `_redirects` file already 301-redirects old `/provider-terms` URLs to `/clinician-terms`, so anyone with a bookmark still lands on something useful.

### Step 4 — Verify the routes

Open each of these URLs in an incognito window and confirm they render:

```
https://snapshothealth.io/sign-everything
https://snapshothealth.io/sign-everything?program=turnkey
https://snapshothealth.io/sign-everything?program=software-no-device
https://snapshothealth.io/sign-everything?program=software-with-device
https://snapshothealth.io/order
https://snapshothealth.io/order?program=software-no-device
https://snapshothealth.io/order?program=software-with-device
https://snapshothealth.io/clinician-terms
https://snapshothealth.io/baa
https://snapshothealth.io/review-and-sign
https://snapshothealth.io/why-snapshot
https://snapshothealth.io/sign          (should redirect to /sign-everything?program=turnkey)
https://snapshothealth.io/provider-terms (should redirect to /clinician-terms)
```

If any return a 404, the `_redirects` file isn't being read — most common cause is it was renamed to `_redirects.txt` or placed in a subfolder.

---

## HubSpot Integration

### Wired endpoint

The Sign Everything page submits signed agreements to:

```
POST https://api.hsforms.com/submissions/v3/integration/submit/47989991/06eff455-1326-4fbc-b96d-88b4f4427c0a
```

**Portal ID:** `47989991`
**Form GUID:** `06eff455-1326-4fbc-b96d-88b4f4427c0a`
**Form name:** SnapShot Health Signed Agreement

Both are hardcoded as constants at the top of `submitSignature()` in `sign_everything.html` (around line 1240). If you ever need to swap to a different form (e.g. separate forms per program), edit these two constants.

### Fields submitted

Every signature creates a Contact and (automatically) an associated Company in HubSpot with these fields populated:

| Field | Object | Source |
|---|---|---|
| `email` | Contact | Email field on form |
| `firstname`, `lastname` | Contact | Split from Full Name field on last space |
| `jobtitle` | Contact | Title field on form |
| `name` | Company | Practice Name field on form |
| `program` | Contact | `turnkey` / `software-no-device` / `software-with-device` |
| `program_name` | Contact | Human-readable ("Remote Concierge Care" etc.) |
| `term` | Contact | `month-to-month` / `12-month` |
| `discount_applied` | Contact | `"5%"` if 12-month, blank otherwise |
| `signature_mode` | Contact | `type` or `draw` |
| `signature_data` | Contact | Typed name string OR base64 PNG of drawn signature |
| `signed_at` | Contact | ISO 8601 timestamp |
| `order_form_version` | Contact | `v2026.1` |
| `msa_version` | Contact | `v2026.1` |
| `baa_version` | Contact | `v2026.1` |
| `user_agent` | Contact | Browser user-agent string (audit trail) |

### Error handling

If HubSpot rejects the submission (400/500) or the network fails, the customer sees a red error banner above the submit button directing them to `careteam@snapshothealth.io`. The submit button re-enables so they can retry. The full payload is logged to `console` in devtools so submissions can be recovered manually if needed.

---

## Self-Test Protocol

Before you send anyone real to this flow, run a test submit yourself:

1. Open `https://snapshothealth.io/sign-everything?program=turnkey` in an **incognito** browser window
2. Open devtools → Console tab (F12) — leave it open
3. Fill out with identifiable test data:
   - Practice: `ZZZ_TEST_PRACTICE`
   - Name: `Test McTestington`
   - Title: `Test Signer`
   - Email: `jimmy+test@snapshothealth.io`
4. Select term, check the acknowledgment, draw or type a signature
5. Click Sign & Submit. You should see:
   - Button disables, shows "Submitting…"
   - Console logs `[SnapShot Health] Submitting signature payload: {...}`
   - Within 1–2 seconds: success screen appears
6. Go to HubSpot → Contacts → sort by Create date descending. `Test McTestington` should be at the top with all fields populated.
7. Once verified, delete the test contact and test company in HubSpot.

Run this same test for `?program=software-no-device` and `?program=software-with-device` to confirm all three program paths work end-to-end.

---

## URL Examples for Sales Emails

Templates to plug into HubSpot quote emails:

**New Remote Concierge Care customer:**
```
https://snapshothealth.io/review-and-sign?program=turnkey&practice=Signet+Heart+Group&signer=Dr.+Nikhil+Joshi&title=Managing+Physician&email=dr.joshi@signet.com
```

**New Software Platform (no device) customer:**
```
https://snapshothealth.io/review-and-sign?program=software-no-device&practice=Example+Medical+Group&signer=Dr.+Jane+Smith&title=Medical+Director&email=jane@example.com
```

**New Software Platform (with device) customer:**
```
https://snapshothealth.io/review-and-sign?program=software-with-device&practice=Example+Medical+Group&signer=Dr.+Jane+Smith&title=Medical+Director&email=jane@example.com
```

**Existing customer re-signing:**
Append `&existing=true` to any of the above. The Review & Sign page will show the "Welcome back" banner and emphasize what changed, not what's new.

---

## Pending Items

1. **Attorney review** — healthcare-focused attorney should review MSA (esp. §01 dual-program scope, §05 PG scoping, AI language), BAA (HIPAA §164.504(e) compliance, AI/ML vendor language), Order Form (billable-patient definition), and the ESIGN/UETA compliance of executing 3 docs via 1 signature. Budget: $2,500–$5,000.

2. **HubSpot confirmation email workflow** — Once signatures are flowing, build a HubSpot Workflow: trigger on Contact property `program` being known; action send email with copy of agreement + onboarding call calendar link to Lucy or Angie.

3. **First friendly customer test** — Walk Dr. Joshi or another trusted account through the flow while on the phone. Log every friction point. Fix.

4. **Post-signing PDF generation** — Not yet built. For now, when customers ask for a copy, print the `/sign-everything` page to PDF and email it manually. Automate after 10+ signings/month.

5. **Year-end cleanup (January 2, 2027)** — Remove "New 2026" labels from 99445, 99470, G0568, G0569, G0570 across `order-form.html` and `sign_everything.html`.

6. **Shared pricing config (eventual)** — Rates are currently hardcoded in `order-form.html` and `sign_everything.html` (PROGRAM_CONFIG.flatRate). When rates change, update both. Not urgent — small surface area.

---

## Rollback

If anything breaks after deploy, Netlify keeps every previous deploy. Netlify dashboard → **Deploys** tab → find the last known-good deploy → three-dot menu → **Publish deploy**. Reverts in about 10 seconds, no data loss, no customer-visible downtime.

---

## Quick Reference — File Versions

All current files are marked `v2026.1` in their version metadata. Signatures submitted after deploy will record this version string in HubSpot, so you can always tell which contract version a customer signed.

When you make a material change to any document, bump the version (e.g. `v2026.2`) in the `submitSignature()` payload fields and in any visible version markers in the document. This gives you per-version signing history for audit purposes.
