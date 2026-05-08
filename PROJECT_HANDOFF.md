# SnapShot Health — Contract Documents & Customer Acquisition Architecture

**Project handoff document · Originally prepared April 17, 2026 · Last updated May 8, 2026**

This document captures the full context of work between Jimmy LaRose (Founder & CEO, SnapShot Health Inc.) and Claude. Future Claude instances working in this project should read this first before responding to questions. Everything below is decided and locked in unless Jimmy explicitly changes it.

---

## Who Jimmy Is

Jimmy LaRose runs SnapShot Health Inc., a Dallas-based Remote Patient Monitoring (RPM) and Chronic Care Management (CCM) company serving 30+ medical practices with ~1,429 billable patients as of March 2026. Largest account is Signet Heart Group (Dr. Nikhil Joshi, agency code `comprehensivecardio`). Internal clinical team is Lucy Brandt (BSN, RN) and Angie Wiser (MA). Adrienne Davis is the head biller for Signet — external, not an employee.

Jimmy is a solo technical founder who does nearly everything himself. He's built substantial automation already: Power Automate Desktop flows on SSH-Bot1 for VitalWatch alert processing, Playwright-based vitals entry agents (`PF_Vitals_Entry`), individual patient PDF report pipelines (`run_patient_reports_v3.py`), billing pipelines (`snapshot_unified_pipeline_new.py`), a JARVIS voice assistant, Transtek warehouse API integration with HMAC-SHA256 signing, HubSpot → Shopify → ShipStation → Zapier device fulfillment pipelines, and a patient portal at portal.snapshothealth.io built in React/TypeScript/Supabase.

Key personal context: coaches Wow Factor NTX White 11U baseball, livestreams games with Obsbot Tail 2. Former athlete. Operates SSH-Main (64GB RAM workstation, billing@snapshothealth.io) and SSH-Bot1 (Ryzen 7 7840HS automation workhorse). Uses Python via `py` launcher on Windows, not `python` or `pip` directly.

---

## Project Origin

This conversation started in April 2026 when Jimmy asked Claude to help create a SnapShot Health service agreement in modern SaaS-style format. What began as a single document grew into a complete customer acquisition architecture, then evolved over the following weeks into the dual-program signing flow now live in production.

---

## What's Live (Current Architecture)

Five HTML files plus `_redirects`, deployed at the root of the snapshothealth.io Netlify repo. All five share the locked brand system: Outfit sans + JetBrains Mono mono + Dancing Script for typed signatures, navy/cyan palette (`#0A1120` navy, `#00D4AA` teal, `#00B4D8` cyan), § numbered sections, embedded base64 logos, responsive design with fluid `clamp()` sizing, and print-to-PDF styling.

### The Five Documents

| File | Live URL | Purpose |
|---|---|---|
| `review-and-sign.html` | `/review-and-sign` | Friendly explainer/landing page. "A quick update for [Practice]" with five-card walkthrough of what's changed and links to detailed docs. CTA routes prospects to `/sign-everything` with URL params forwarded. |
| `sign_everything.html` | `/sign` and `/sign-everything` | The signing page. "One signature. Three agreements." Single signing session executes Order Form + MSA + BAA together. Pre-fillable via URL params. POSTs to HubSpot Forms API on submit. |
| `clinician-terms.html` | `/clinician-terms` | Master Service Agreement (§ 01–18). Public reference, linked from Sign Everything. No signature block — signature on `/sign` constitutes acceptance by reference. |
| `baa.html` | `/baa` | Business Associate Agreement (§ 01–12). Public reference, linked from Sign Everything. HIPAA §164.504 citations throughout. Acceptance handled in the unified signing session, not on this page. |
| `order-form.html` | `/order` | Service Order Form (§ 01–08). Public preview for prospects to see pricing, terms, and the device catalog before they sign. Includes the embedded Transtek device hero photo. |

### The Customer Flow

```
Prospect receives pre-filled link from Jimmy:
  https://snapshothealth.io/review-and-sign?practice=...&signer=...&title=...&email=...
  ↓
Lands on /review-and-sign
  → Reads the friendly explainer, sees what's changed, links to MSA/BAA/Order Form for detail
  ↓
Clicks "Sign in under a minute" CTA
  → Routes to /sign-everything with URL params forwarded
  ↓
Lands on /sign-everything (sign_everything.html)
  → Pre-filled fields are visually marked (subtle green tint)
  → Reads scope, fees, term commitment
  → Checks agreement box, types or draws signature
  → Clicks Sign & Submit
  ↓
Form POSTs to HubSpot Forms API
  → HubSpot Workflow fires (when configured): stage advance, PDF generation,
    Lucy/Angie task creation, etc.
```

### HubSpot Integration — LIVE

Wired in `sign_everything.html`:
- **Portal ID:** `47989991`
- **Form GUID:** `06eff455-1326-4fbc-b96d-88b4f4427c0a`
- **Endpoint:** `https://api.hsforms.com/submissions/v3/integration/submit/{portalId}/{formGuid}`
- Payload includes: practice name, signer first/last (split on final space), email, jobtitle, signature mode (typed/drawn), signature data, agreement timestamp, page URI

The HubSpot **Workflow** that fires on form submission (stage advance, PDF generation, BAA/Lucy/Angie automation) is the next thing to build inside HubSpot itself. The form-side wiring is done.

### Pricing (Signet-Style Per-CPT Stack)

Locked in `order-form.html` and `sign_everything.html`. Update both files if rates change.

| Code | Service | Fee |
|------|---------|-----|
| 99453 | RPM Setup | $30 |
| 99454 | RPM Device Supply 16+ days | $23 |
| 99445 | RPM Device Supply 2–15 days (NEW 2026) | $23 |
| 99457 | RPM Treatment Mgmt 20 min | $23 |
| 99458 | RPM Treatment Mgmt each addtl 20 | $20 |
| 99470 | RPM Treatment Mgmt 10–19 min (NEW 2026) | $15 |
| 99490 | CCM Initial 20 min | $23 |
| 99439 | CCM each addtl 20 min | $20 |
| 99487 | Complex CCM Initial 60 min | $60 |
| 99489 | Complex CCM each addtl 30 min | $33 |
| 99484 | BHI | $23 |
| G0556 | APCM Level 1 | $15 |
| G0557 | APCM Level 2 | $25 |
| G0558 | APCM Level 3 (QMB) | $50 |
| G0568 | APCM + CoCM Initial (NEW 2026) | $50 |
| G0569 | APCM + CoCM Subsequent (NEW 2026) | $40 |
| G0570 | APCM + BHI (NEW 2026) | $23 |

Onboarding fee: $0.

---

## Decisions Locked In

These are settled. Don't re-litigate unless Jimmy raises them.

1. **Architecture: explainer + signing + three reference docs** — `review-and-sign` is the warm landing, `sign_everything` is the conversion endpoint, `clinician-terms` / `baa` / `order-form` are reference. This pattern beats DocuSign-style single-doc signing.

2. **One-session triple-document signing** — Order Form, MSA, and BAA are all executed by a single signature on `/sign-everything`. (Earlier plan separated BAA into a follow-up; that was scrapped because forcing two signing sessions kills momentum and the unified flow has higher completion.)

3. **Remote Concierge Care as the default package** — no à la carte checkboxes. SnapShot Health is fully turnkey, the form reflects that.

4. **Brand-neutral device descriptions** — don't name Transtek/MioConnect Labs in customer-facing docs. Generic "FDA-cleared 4G cellular devices" language. Branded hardware in the hero photo is fine — it looks more credible than stock images.

5. **No DocuSign** — Jimmy does 10–15 signings/year. ESIGN Act (2000) and UETA make typed/drawn signatures in the form fully enforceable. Custom signature pad is legally sufficient and 100% under his control.

6. **Font: Outfit** — matches the rest of snapshothealth.io. (Earlier candidates Inter and Poppins were rejected.)

7. **Brand palette: navy/teal/cyan** — `#0A1120` navy, `#00D4AA` teal, `#00B4D8` cyan. Locked across all five files.

8. **HubSpot as system of record** — HubSpot Professional with Workflows. Sign Everything POSTs directly to HubSpot Forms API. No Formspree, no Zapier bridge.

9. **No custom portal build yet** — Defer until 50+ signed practices or a design partner explicitly asks. Stack when it's time: Next.js + Supabase + shadcn/ui on Vercel. Hire a contractor, don't founder-build.

10. **No custom database yet** — MioConnect is system of record for clinical data. Any future portal is a thin presentation layer pulling live from MioConnect's API, not a duplicate data store. Minimizes HIPAA compliance scope.

---

## Long-Term Architecture (The 8-Layer Plan)

Jimmy asked: *"If you were starting this project from scratch, knowing I'm the only one doing all of this, what architecture would you create — from website to marketing to fulfillment to invoicing, with Claude Code automation, designed to be investor/PE-friendly?"*

**Layer 1 — Public Website:** snapshothealth.io on Netlify. ✅ Five contract pages live at /clinician-terms, /baa, /order, /review-and-sign, /sign(-everything).

**Layer 2 — CRM / System of Record:** HubSpot Professional with Workflows. Custom Deal properties for MRR, patient count, adherence, EHR, last-invoice-paid. Pipeline stages: New Inquiry → Discovery → Proposal → Signed → Onboarding → Active → At Risk. ✅ Sign Everything form is wired to HubSpot Forms API. ⏳ Workflow on submission still to build.

**Layer 3 — Clinical Operations:** VitalWatch (at reseller.medius.us) + MioConnect Labs. Both already have BAAs, both are HIPAA-compliant. SnapShot Health never becomes the primary custodian of PHI — always consumes it via API.

**Layer 4 — Billing & Invoicing:** Migrate from Excel to QuickBooks Online + Stripe (or Melio for check-based clients). Invoices auto-generated from HubSpot billing data via Make.com. Major valuation unlock — transforms financials from "trust me, here's a spreadsheet" to "here's QuickBooks access."

**Layer 5 — Operations Automation:** Keep Power Automate Desktop on SSH-Bot1 for legacy VitalWatch screen-scraping. Use Make.com for new modern-API workflows (HubSpot, Stripe, QuickBooks). Cloud-based, better logs for diligence.

**Layer 6 — Communications:** Front (or shared Gmail at careteam@snapshothealth.io) so every customer interaction is visible to whoever takes over post-acquisition. No customer email in personal Gmail.

**Layer 7 — AI Assistant / Automation Layer:** Claude Code for technical scripting. Claude Projects for domain knowledge (this contract project is the template; future projects cover Clinical Ops, Billing, Device Ops, Sales & Marketing).

**Layer 8 — Portal (Future, Not Now):** Next.js + Supabase + shadcn/ui on Vercel + Resend for email. Built by a contractor over 3 months when demand is validated. NOT built by Jimmy on weekends.

### Why This Architecture Appeals to PE / Investors

Not because of sophistication — because of **transferability**. Every critical piece of data lives in a portable commercial system:
- Customer data in HubSpot (export or grant admin access)
- Financial data in QuickBooks (trailing 12-month P&L, AR aging, MRR)
- Clinical data in VitalWatch + MioConnect (audit logs, patient volume)
- Automation in Make.com + Power Automate (execution history, documented)
- Knowledge in Claude Projects (domain docs, protocols)

A PE buyer can complete diligence with four logins and a 90-day transition. Dramatically more valuable than custom tech that requires Jimmy to train a successor for 6+ months.

### What Jimmy Should Build Next, In Order

1. ✅ **Done (April):** Order Form, MSA, BAA documents drafted with brand system.
2. ✅ **Done (late April):** Evolved to dual-program signing flow with `review-and-sign` + `sign_everything`. HubSpot Forms API wired.
3. ⏳ **Now:** Build the HubSpot Workflow that fires on Sign Everything submission — stage advance, PDF generation/email, BAA/Lucy/Angie task creation, Make.com triggers to Transtek and QuickBooks.
4. ⏳ **Next:** Formalize HubSpot Deal pipeline and custom properties. Document existing Power Automate flows in a Claude Project.
5. **Month 2:** Migrate billing from Excel to QuickBooks Online + Stripe. Run parallel for one cycle, then cut over.
6. **Month 3:** Centralize all customer communication in Front or shared Gmail.
7. **Months 4–6:** Build HubSpot reporting dashboards. Automate monthly investor-ready metrics. Trailing-12 financials out of QuickBooks.
8. **Month 6+:** Evaluate portal demand. If real, hire contractor for 3-month Next.js + Supabase build.

### Approximate Monthly Cost When Fully Deployed

Without portal: $800–1,200/mo (HubSpot + QuickBooks + Make.com + Front/Gmail + Claude Max + Power Automate + Stripe %).

With portal: Add $300–700/mo (Vercel Pro + Supabase Team for HIPAA BAA + Resend).

For a business doing seven figures ARR, this is a rounding error.

---

## Open Questions / Things Still TBD

1. **HubSpot Workflow on form submission** — the Forms API submission lands data in HubSpot, but the post-submission automation (PDF generation, email confirmation, BAA delivery, internal task creation) isn't built yet. Highest-priority next task.

2. **Post-signing PDF generation** — signers currently see a success message but don't get a PDF copy of what they signed. Worth solving for trust and recordkeeping. Options: HubSpot Workflow + a PDF service, Netlify Function + Puppeteer, or client-side PDF generation before submit.

3. **Email confirmation to signer** — signer should automatically receive a copy of the executed agreement. Can ride on top of the PDF generation solution.

4. **Attorney review** — the MSA and BAA language was drafted carefully but a Texas healthcare attorney should eyeball it once before too many practices have signed. Particularly §01 Scope and the Texas Medicaid clause. ~$500.

5. **QuickBooks migration** — currently runs billing via `snapshot_unified_pipeline_new.py` generating Excel reports. Migration is Month 2 priority but not scoped yet.

6. **Portal timing** — deferred indefinitely. Decision trigger is 50+ signed practices OR explicit client demand for branded portal access.

---

## How Future Claude Should Work in This Project

**When Jimmy starts a new conversation, assume:**
- He has the context above. Don't re-ask architectural questions that are already decided.
- He wants concrete deliverables, not theoretical options. Build the thing, don't describe it.
- He values honesty over flattery. Push back if he's making a mistake. Don't agree just to be agreeable.
- He's technically capable but time-constrained. Ship working code, not 500-line explanations of what the code should do.
- He prefers branded HTML for his 10–15/year signing volume over commercial alternatives (DocuSign, PandaDoc).
- The brand system (Outfit + JetBrains Mono, navy/teal/cyan, § numbering) is locked across all docs.

**Before assuming you know the current state:**
- This handoff is a snapshot. The repo is the source of truth.
- If a question depends on what's deployed, check the live files (Jimmy can paste them or upload from GitHub).
- The two earlier "INTER" / "POPPINS" font variants and `sign_and_start.html` are dead. Don't suggest reviving them.

**When iterating on the contract documents:**
- Preserve all brand elements (colors, type, § numbering, logo embeds)
- Maintain 2026 CPT code compliance
- Keep Signet-style pricing unless Jimmy explicitly changes rates
- Test that div/section balance is preserved after edits
- Output files to `/mnt/user-data/outputs/` and use `present_files` to share

**When discussing architecture beyond the documents:**
- Default to commercial SaaS (HubSpot, QuickBooks, Stripe, Make.com) over custom builds
- Respect the "no portal yet" decision unless Jimmy raises it
- Frame recommendations in terms of diligence/transferability for eventual acquisition
- Remember Jimmy is solo — every new system is maintenance burden, weigh accordingly

**Style notes from how Jimmy works:**
- He uses voice dictation sometimes, so autocorrect mistakes happen ("foldable code" meant "solid code"). Confirm interpretation when ambiguous.
- He responds better to structured options (multiple choice) than open-ended questions when making decisions.
- He's used Bolt and Lovable before and knows their limitations. Don't recommend them for production.
- He has four Google Workspace accounts mapped to drives G:/H:/I:/J: on SSH-Main.
- He uses `py` launcher on Windows, not `python` or `pip` directly.

---

## Reference Files

**Currently deployed at the root of the snapshothealth.io GitHub repo:**
- `review-and-sign.html` — explainer/landing page (~127 KB)
- `sign_everything.html` — unified signing page with HubSpot wiring (~149 KB)
- `clinician-terms.html` — MSA reference (~231 KB)
- `baa.html` — BAA reference (~242 KB)
- `order-form.html` — Service Order Form preview (~187 KB)
- `_redirects` — Netlify clean-URL routing
- `DEPLOYMENT_GUIDE.md` — operator-focused deployment doc

**Deprecated, should be deleted from repo if still present:**
- `sign_and_start.html` — superseded by `sign_everything.html` (May 8, 2026)
- `INTER_*.html`, `POPPINS_*.html` — font A/B variants from April; Outfit was chosen

---

## Key Contacts & Credentials (for Jimmy's reference, do not store elsewhere)

- SnapShot Health Inc., Dallas, Texas
- Founder/CEO: Jimmy LaRose
- Clinical: Lucy Brandt (BSN, RN) + Angie Wiser (MA)
- Email: careteam@snapshothealth.io (main) · billing@snapshothealth.io · privacy@snapshothealth.io (BAA)
- Phone: (469) 342-3753
- Website: snapshothealth.io (Netlify)
- Patient portal: portal.snapshothealth.io (Supabase)
- VitalWatch: reseller.medius.us (Jason Wu at Transtek/MioConnect Labs)
- Transtek API: OrgID 00000062, ApiKey 6E75E2
- HubSpot Portal: 47989991
- HubSpot Sign-Everything Form GUID: 06eff455-1326-4fbc-b96d-88b4f4427c0a

---

*End of handoff document. Last updated May 8, 2026, after the dual-program signing flow shipped, the font choice was resolved (Outfit), HubSpot Forms API integration was completed, and `sign_and_start.html` was retired in favor of `sign_everything.html`.*
