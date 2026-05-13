# SnapShot Health — Contract Documents & Customer Acquisition Architecture

**Project handoff document · Originally prepared April 17, 2026 · Last updated May 13, 2026**

This document captures the full context of work between Jimmy LaRose (Founder & CEO, SnapShot Health Inc.) and Claude. Future Claude instances working in this project should read this first before responding to questions. Everything below is decided and locked in unless Jimmy explicitly changes it.

---

## Changelog (Recent Updates)

**May 13, 2026** — End-to-end signing pipeline went production-ready. Make.com automation built and validated. Brand stack fully resolved to Outfit. Six HTML files audited and confirmed consistent. See "What's Live" and "Make.com Automation Pipeline" below.

**May 8, 2026** — Dual-program signing flow shipped. `sign_and_start.html` retired, `sign_everything.html` and `review-and-sign.html` deployed. HubSpot Forms API wired in. Font choice resolved as Outfit (matching the live website).

**April 17, 2026** — Initial project: Order Form, MSA, BAA documents drafted with brand system. Customer acquisition architecture mapped (8-layer plan).

---

## Who Jimmy Is

Jimmy LaRose runs SnapShot Health Inc., a Dallas-based Remote Patient Monitoring (RPM) and Chronic Care Management (CCM) company serving 30+ medical practices with ~1,429 billable patients as of March 2026. Largest account is Signet Heart Group (Dr. Nikhil Joshi, agency code `comprehensivecardio`). Internal clinical team is Lucy Brandt (BSN, RN) and Angie Wiser (MA). Adrienne Davis is the head biller for Signet — external, not an employee.

Jimmy is a solo technical founder who does nearly everything himself. He's built substantial automation already: Power Automate Desktop flows on SSH-Bot1 for VitalWatch alert processing, Playwright-based vitals entry agents (`PF_Vitals_Entry`), individual patient PDF report pipelines (`run_patient_reports_v3.py`), billing pipelines (`snapshot_unified_pipeline_new.py`), a JARVIS voice assistant, Transtek warehouse API integration with HMAC-SHA256 signing, HubSpot → Shopify → ShipStation → Zapier device fulfillment pipelines, and a patient portal at portal.snapshothealth.io built in React/TypeScript/Supabase.

Key personal context: coaches Wow Factor NTX White 11U baseball, livestreams games with Obsbot Tail 2. Former athlete. Operates SSH-Main (64GB RAM workstation, billing@snapshothealth.io) and SSH-Bot1 (Ryzen 7 7840HS automation workhorse). Uses Python via `py` launcher on Windows, not `python` or `pip` directly.

---

## Project Origin

This project started in April 2026 when Jimmy asked Claude to help create a SnapShot Health service agreement in modern SaaS-style format. What began as a single document grew into a complete customer acquisition architecture, then evolved over the following weeks into the dual-program signing flow that shipped May 8, 2026. On May 12–13, the post-submission automation pipeline (Make.com → PDFShift → Gmail) was built and validated end-to-end across all three program/audience combinations.

---

## What's Live (Current Architecture)

Six HTML files plus `_redirects`, deployed at the root of the snapshothealth.io Netlify repo. All six share the locked brand system: **Outfit sans + JetBrains Mono + Dancing Script** for typed signatures, navy/cyan palette (`#0A1120` navy, `#00D4AA` teal, `#00B4D8` cyan), § numbered sections, embedded base64 logos, responsive design with fluid `clamp()` sizing, and print-to-PDF styling.

### The Six Documents

| File | Live URL | Purpose |
|---|---|---|
| `review-and-sign.html` | `/review-and-sign` | Friendly explainer/landing page. "A quick update for [Practice]" with five-card walkthrough of what's changed and links to detailed docs. CTA routes prospects to `/sign-everything` with URL params forwarded. Optional warm intro path. |
| `sign_everything.html` | `/sign` and `/sign-everything` | The signing page. "One signature. Three agreements." Single signing session executes Order Form + MSA + BAA together. Reads `?program=` and `?audience=` URL params to configure for the practice type. POSTs to HubSpot Forms API on submit. Also serves as the source for the executed PDF rendered by PDFShift (`?pdf_mode=1`). |
| `clinician-terms.html` | `/clinician-terms` | Master Service Agreement for **physician practices** (§ 01–18). Public reference, linked from Sign Everything. No signature block — signature on `/sign` constitutes acceptance by reference. |
| `hha-clinician-terms.html` | `/hha-clinician-terms` | Master Service Agreement for **Home Health Agencies** (§ 01–18). HHA-specific terms (skilled nursing, plan of care coordination, etc.). Routed to automatically when `audience=hha`. |
| `baa.html` | `/baa` | Business Associate Agreement (§ 01–12). Public reference, linked from Sign Everything. HIPAA §164.504 citations throughout. Acceptance handled in the unified signing session. |
| `order-form.html` | `/order` | Service Order Form (§ 01–08). Public preview for prospects to see pricing, terms, and the device catalog before they sign. Includes the embedded Transtek device hero photo. |

### Production Customer Flow

Two paths to the signing page, both ending at the same Make.com automation:

```
PATH A (warm intro) — Jimmy sends pre-filled explainer link:
  https://snapshothealth.io/review-and-sign?practice=...&signer=...&title=...&email=...
  ↓
  Prospect lands on /review-and-sign, reads five-card walkthrough
  ↓
  Clicks "Sign in under a minute" → URL params forwarded to /sign-everything

PATH B (direct send) — Jimmy sends pre-configured signing link:
  https://snapshothealth.io/sign?program=turnkey&audience=physician
  (Valid program values: turnkey | software-no-device | software-with-device)
  (Valid audience values: physician | hha)
  ↓
  Prospect lands directly on /sign-everything, page auto-configures for
  the chosen program and audience

BOTH PATHS CONVERGE:
  Prospect on /sign-everything (sign_everything.html)
  → Fills practice info, signer info, picks term, types or draws signature
  → Hits Submit
  ↓
  Form POSTs to HubSpot Forms API (Portal 47989991, Form GUID 06eff455...)
  → HubSpot stores the submission and creates/updates the Contact
  ↓
  Make.com scenario polls HubSpot for new form submissions (every 15 min)
  → Contact upsert → Deal search → Deal create/update → PDFShift render → Gmail
  ↓
  Practice receives "Welcome to SnapShot Health" email with executed PDF attached
  Deal lands in HubSpot Customer Acquisition pipeline at "Signed" stage
```

### HubSpot Configuration

**Forms API integration (page-side):**
- Portal ID: `47989991`
- Form GUID: `06eff455-1326-4fbc-b96d-88b4f4427c0a`
- Endpoint: `https://api.hsforms.com/submissions/v3/integration/submit/{portalId}/{formGuid}`
- Payload: practice name (Company name), signer first/last, email, jobtitle, service_program (display value: `Turnkey` / `Software 20` / `Software 45`), audience (display value: `Physician` / `HHA`), **program (URL slug: `turnkey` / `software-no-device` / `software-with-device`)**, term, discount_applied, signature_mode, signature_data, signed_at, page URI, hutk

**The `program` slug field is the architectural keystone** added May 13, 2026. It carries the URL slug (lowercase, hyphenated) from page → HubSpot → Make.com → PDFShift unchanged, eliminating the previous fragile pattern of string-transforming `service_program` display values inside Make.com. Single source of truth.

**HubSpot Deal pipeline:**
- Pipeline: **Customer Acquisition**
- Stages: New Inquiry → Discovery → Proposal → **Signed** → Onboarding → Active → At Risk
- Deals created by the Make.com pipeline land at the **Signed** stage with deal name = Practice name + " - " + Service Program + " (" + Audience + ")"
- Custom Deal properties populated: Service Program, Audience

## Make.com Automation Pipeline

Built and validated May 13, 2026. Runs on a 15-minute polling schedule against HubSpot Form Submissions. Scenario lives in Jimmy's Make.com account (organization-level connection name: "SnapShot Health Signed A...").

### Module-by-Module

| # | Module | Purpose |
|---|---|---|
| 1 | HubSpot CRM › Watch Submissions for a Form | Trigger. Polls the SnapShot Health Signed Agreement form for new submissions. |
| 34 | HubSpot CRM › Create or Update a Contact | Upserts the contact by email. Maps: email, firstname, lastname, jobtitle, company. Returns contact ID for downstream association. |
| 15 | HubSpot CRM › Search for Deals | Searches existing deals by name (practice name). Determines whether this is a new acquisition or a re-signing. |
| 16 | Router | Splits the flow based on whether a Deal was found. |
| 17 | HubSpot CRM › Update a Deal (Deal Exists branch) | Updates the existing Deal with new agreement details. |
| 18 | HubSpot CRM › Create a Deal (Deal Doesn't Exist branch) | Creates a new Deal in Customer Acquisition pipeline at Signed stage, with associations to Contact (associationTypeId: 3). |
| 22 / 32 | HTTP › POST /v3/convert/pdf to PDFShift | Renders the executed agreement PDF by fetching `https://snapshothealth.io/sign?pdf_mode=1&practice=...&signer=...&program=...&audience=...` etc. Returns a binary PDF. **Modules 22 and 32 are functionally identical** — one per Router branch. Keep them in sync when editing. |
| 23 / 33 | Gmail › Send an email | Sends the branded "Welcome to SnapShot Health" email from billing@snapshothealth.io to the signer, with the PDF attached. Personalized greeting, practice name in body. |

### Critical Maintenance Notes

- **Both PDFShift modules (22 and 32) must be kept identical.** When editing the URL body content, update both. The architectural fix on May 13 collapsed all string transforms into the page-side `program` slug, so the current URL is clean: `&program={{1.fields.program}}` directly, no `switch()` or `lower(replace(...))` gymnastics.
- **The page guard at `/sign` validates `program` and `audience` against `PROGRAM_CONFIG` and `AUDIENCE_CONFIG` keys** (lowercase slugs). Any new program added in the future must be added to both the page config AND to the HubSpot Service Program dropdown (display value) AND should have its slug appear in the `program` field on the HubSpot form. Drift between these will produce the "Signing link incomplete" error on the rendered PDF.
- **Make.com polls every 15 minutes.** A signing submitted at 9:01 will not appear in HubSpot Make.com runs until ~9:15. Don't expect instant delivery; explain the lag to practices if asked.
- **The welcome email uses inline-styled Outfit font** (updated May 13 from the previous Inter + Fraunces stack). Only Gmail web reliably renders Google Fonts; other clients fall back to the system stack. That's acceptable.

### Validated End-to-End

All three program/audience combinations were validated May 13 with real form submissions:

| Combination | Test practices | Result |
|---|---|---|
| `turnkey` + `physician` | Highland Park Cardiology, Riverbend Medical Associates, Brookhaven Primary Care | ✓ Executed PDF delivered |
| `software-no-device` + `physician` | Cedar Valley Family Medicine, Summit Ridge Pediatrics, Greenwood Internal Medicine | ✓ Executed PDF delivered |
| `software-with-device` + `hha` | Magnolia Springs Home Health, Ironwood Hills Home Health, Willow Creek Home Health | ✓ Executed PDF delivered (routes to `hha-clinician-terms` MSA) |

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

1. **Architecture: explainer + signing + reference docs** — `review-and-sign` is the warm landing, `sign_everything` is the conversion endpoint with URL-param configuration, `clinician-terms` / `hha-clinician-terms` / `baa` / `order-form` are public references. This pattern beats DocuSign-style single-doc signing.

2. **One-session triple-document signing** — Order Form, MSA, and BAA are all executed by a single signature on `/sign-everything`. Earlier plan separated BAA into a follow-up; scrapped because forcing two signing sessions kills momentum.

3. **Three program tiers, two audience types** — `turnkey` (Remote Concierge Care, physician-only), `software-no-device` (Software 20, physician-only), `software-with-device` (Software 45, only program offered to HHA). Compatibility enforced by page guard.

4. **Brand-neutral device descriptions in customer-facing docs** — generic "FDA-cleared 4G cellular devices" language. Branded hardware in the hero photo is fine.

5. **No DocuSign** — Jimmy does 10–15 signings/year. ESIGN Act (2000) and UETA make typed/drawn signatures fully enforceable. Custom signature pad is legally sufficient.

6. **Font: Outfit (resolved 2026-05-13)** — matches the rest of snapshothealth.io. Earlier candidates Inter, Poppins, and Fraunces were rejected. All six HTML files audited and confirmed clean on the new stack.

7. **Brand palette: navy/teal/cyan** — `#0A1120` navy, `#00D4AA` teal, `#00B4D8` cyan. Locked across all six files.

8. **HubSpot as system of record** — HubSpot Professional with Workflows. Sign Everything POSTs directly to HubSpot Forms API. Post-submission automation runs in Make.com (not native HubSpot Workflow) — chosen for better logs, simpler iteration, and clearer scenario diagrams during diligence.

9. **Make.com over native HubSpot Workflow** — Decided May 13. Native HubSpot Workflows are harder to inspect and iterate. Make.com's visual scenario builder produces a diagram that's self-documenting for a future technical buyer.

10. **`program` URL slug as HubSpot Contact property** — added May 13 as the single source of truth for the program identifier flowing from page → HubSpot → Make.com → PDFShift. Eliminates string-transform fragility.

11. **No custom portal build yet** — defer until 50+ signed practices or a design partner explicitly asks. Stack when it's time: Next.js + Supabase + shadcn/ui on Vercel. Hire a contractor, don't founder-build.

12. **No custom database yet** — MioConnect is system of record for clinical data. Any future portal is a thin presentation layer pulling live from MioConnect's API.

---

## Long-Term Architecture (The 8-Layer Plan)

Jimmy asked: *"If you were starting this project from scratch, knowing I'm the only one doing all of this, what architecture would you create — from website to marketing to fulfillment to invoicing, with Claude Code automation, designed to be investor/PE-friendly?"*

**Layer 1 — Public Website:** snapshothealth.io on Netlify. ✅ Six contract pages live.

**Layer 2 — CRM / System of Record:** HubSpot Professional with Workflows. ✅ Customer Acquisition pipeline formalized with custom Deal properties (Service Program, Audience). ✅ Sign Everything form wired to HubSpot Forms API. ✅ Post-submission automation pipeline live in Make.com (Contact upsert, Deal create/update, PDFShift PDF render, Gmail welcome email).

**Layer 3 — Clinical Operations:** VitalWatch (at reseller.medius.us) + MioConnect Labs. Both already have BAAs, both are HIPAA-compliant. SnapShot Health never becomes the primary custodian of PHI — always consumes it via API.

**Layer 4 — Billing & Invoicing:** Migrate from Excel to QuickBooks Online + Stripe (or Melio for check-based clients). Invoices auto-generated from HubSpot billing data via Make.com. Major valuation unlock — transforms financials from "trust me, here's a spreadsheet" to "here's QuickBooks access." **This is Month 2 priority — the next major piece of acquisition-grade infrastructure.**

**Layer 5 — Operations Automation:** Keep Power Automate Desktop on SSH-Bot1 for legacy VitalWatch screen-scraping. Use Make.com for new modern-API workflows (HubSpot, Stripe, QuickBooks, PDFShift, Gmail). ✅ First production Make.com scenario (signing pipeline) live since May 13.

**Layer 6 — Communications:** Front (or shared Gmail at careteam@snapshothealth.io) so every customer interaction is visible to whoever takes over post-acquisition. No more customer emails in personal Gmail.

**Layer 7 — AI Assistant / Automation Layer:** Claude Code for technical scripting. Claude Projects for domain knowledge (this contract project is the template; future projects cover Clinical Ops, Billing, Device Ops, Sales & Marketing).

**Layer 8 — Portal (Future, Not Now):** Next.js + Supabase + shadcn/ui on Vercel + Resend for email. Built by a contractor over 3 months when demand is validated. NOT built by Jimmy on weekends.

### Why This Architecture Appeals to PE / Investors

Not because of sophistication — because of **transferability**. Every critical piece of data lives in a portable commercial system:
- Customer data in HubSpot (export or grant admin access)
- Financial data in QuickBooks (trailing 12-month P&L, AR aging, MRR) — pending Month 2 migration
- Clinical data in VitalWatch + MioConnect (audit logs, patient volume)
- Automation in Make.com + Power Automate (execution history, scenario diagrams)
- Knowledge in Claude Projects (domain docs, protocols)

A PE buyer can complete diligence with four logins and a 90-day transition. Dramatically more valuable than custom tech that requires Jimmy to train a successor for 6+ months.

### What Jimmy Should Build Next, In Order

1. ✅ **Done (April):** Order Form, MSA, BAA documents drafted with brand system.
2. ✅ **Done (May 8):** Dual-program signing flow with `review-and-sign` + `sign_everything`. HubSpot Forms API wired.
3. ✅ **Done (May 13):** Make.com post-submission pipeline live. Contact/Deal upsert, PDFShift, welcome email. End-to-end validated across all three program/audience combinations. Brand stack fully consolidated to Outfit.
4. ⏳ **Next (current cleanup, this week or next):** Address the small backlog from the May 13 build — see "Open Cleanup Items" below. None block production.
5. **Month 2:** Migrate billing from Excel to QuickBooks Online + Stripe. Run parallel for one cycle, then cut over. Biggest remaining acquisition-grade infrastructure piece.
6. **Month 3:** Centralize all customer communication in Front or shared Gmail.
7. **Months 4–6:** Build HubSpot reporting dashboards. Automate monthly investor-ready metrics. Trailing-12 financials out of QuickBooks.
8. **Month 6+:** Evaluate portal demand. If real, hire contractor for 3-month Next.js + Supabase build.

### Approximate Monthly Cost When Fully Deployed

Without portal: $800–1,200/mo (HubSpot + QuickBooks + Make.com + PDFShift + Front/Gmail + Claude Max + Power Automate + Stripe %).

With portal: Add $300–700/mo (Vercel Pro + Supabase Team for HIPAA BAA + Resend).

For a business doing seven figures ARR, this is a rounding error.

---

## Open Cleanup Items (Non-Blocking, From May 13 Build)

These don't block production. Customers can sign and receive executed agreements without any of these being fixed. They're polish for the PE diligence story.

1. **Deal `closedate` is off-by-one.** Make.com Create Deal module receives `signed_at` as an ISO timestamp; HubSpot stores it one day earlier than the actual signing date. Likely a timezone parse issue. Fix: investigate the Module 18 mapper, possibly add explicit UTC offset handling.

2. **Deal `Amount` is blank** on all created deals. Pipeline reports show $0 closed for every deal that comes through this flow. Two options: (a) populate with an estimated monthly MRR placeholder (e.g., $1,000), or (b) add a Forecast MRR custom property that gets updated as patients enroll. Without an Amount, your pipeline dashboards understate revenue.

3. **Redundant HubSpot form fields.** The signing form currently has three program-related fields: `Service Program` (dropdown, used for display value), `Program` (text, holds the URL slug — added May 13), and `Program Name` (text, legacy). The second is canonical; the third should be deleted. Same audit for any other lingering duplicates.

4. **System fields visible on the HubSpot form.** `Signature Mode`, `Signature Data`, `User Agent`, `Order Form Version`, `MSA Version`, `BAA Version` are all set as visible fields on the form, even though they're populated by the page's JavaScript and should never be edited by humans. Hide them in the form editor so anyone who lands on the raw HubSpot form (instead of `/sign`) doesn't see internal plumbing.

5. **Re-signer vs new-signer copy split.** Current welcome email copy and success page copy say "Truly nothing. Your patients, devices, billing, and care team continue exactly as they have…" — which is correct for existing customers re-signing under the 2026 framework. For genuinely new acquisitions, the "Angie Wiser will reach out within one business day" copy is more appropriate. The page already has `existingBannerText` and "Welcome back" detection logic; the right pattern is to drive both the success view AND the welcome email copy off that same signal. Pass `submission_type: 'new' | 'resign'` as a hidden field; Make.com branches the email template on it.

6. **Document `Customer Acquisition` HubSpot pipeline.** The pipeline exists and is in use, but the custom Deal properties (Service Program, Audience, Program slug, future MRR forecast) should be enumerated in a single HubSpot Properties document so a successor knows what each is for.

---

## Earlier Open Questions — All Resolved

- ~~**Font choice (Inter vs. Poppins vs. Fraunces)**~~ → Resolved May 13: Outfit, single canonical stack.
- ~~**HubSpot Workflow on form submission**~~ → Resolved May 13: built in Make.com instead of native HubSpot Workflow.
- ~~**Post-signing PDF generation**~~ → Resolved May 13: PDFShift renders the executed agreement, attached to welcome email.
- ~~**Email confirmation to signer**~~ → Resolved May 13: branded Gmail welcome email with PDF attachment, sent from billing@snapshothealth.io.
- ~~**BAA delivery workflow**~~ → Resolved May 8: single signing session executes Order Form, MSA, and BAA together.

---

## Still TBD (Architectural, Larger Scope)

These are real future work, not cleanup items:

1. **QuickBooks migration** — currently runs billing via `snapshot_unified_pipeline_new.py` generating Excel reports. Migration is Month 2 priority but not scoped yet. Biggest valuation unlock remaining.

2. **Attorney review** — the MSA and BAA language was drafted carefully but a Texas healthcare attorney should eyeball it once before too many practices have signed. Particularly §01 Scope and the Texas Medicaid clause. ~$500.

3. **CPT code maintenance** — the 2026 fees in `sign_everything.html` and `order-form.html` are correct as of January 2026. When CMS publishes the 2027 PFS in late 2026, both files need to be updated together. Run a `grep "9945"` and similar before deploying to ensure no rate is stale.

4. **Portal timing** — deferred indefinitely. Decision trigger is 50+ signed practices OR explicit client demand for branded portal access.

---

## How Future Claude Should Work in This Project

**When Jimmy starts a new conversation, assume:**
- He has the context above. Don't re-ask architectural questions that are already decided.
- He wants concrete deliverables, not theoretical options. Build the thing, don't describe it.
- He values honesty over flattery. Push back if he's making a mistake. Don't agree just to be agreeable.
- He's technically capable but time-constrained. Ship working code, not 500-line explanations of what the code should do.
- He prefers branded HTML for his 10–15/year signing volume over commercial alternatives (DocuSign, PandaDoc).
- The brand system (Outfit + JetBrains Mono + Dancing Script for signatures, navy/teal/cyan, § numbering) is locked across all docs.

**Before assuming you know the current state:**
- This handoff is a snapshot. The repo is the source of truth.
- If a question depends on what's deployed, check the live files (Jimmy can paste them or upload from GitHub).
- The earlier `INTER_*` / `POPPINS_*` / Fraunces font variants and `sign_and_start.html` are dead. Don't suggest reviving them.
- The earlier `switch()` and `lower(replace(...))` patterns in Make.com PDFShift modules are dead. The current pattern is `&program={{1.fields.program}}` directly, reading the URL slug HubSpot stored from the page POST. Don't suggest reviving the string-transform pattern.

**When iterating on the contract documents:**
- Preserve all brand elements (colors, type, § numbering, logo embeds)
- Maintain 2026 CPT code compliance
- Keep Signet-style pricing unless Jimmy explicitly changes rates
- Test that div/section balance is preserved after edits
- Output files to `/mnt/user-data/outputs/` and use `present_files` to share

**When iterating on Make.com:**
- The two PDFShift modules (22 and 32) must be kept identical. Update both for any change.
- Don't suggest moving back to native HubSpot Workflows. Make.com is the chosen tool.
- The 15-minute polling cadence is acceptable; don't push for instant-trigger webhooks unless Jimmy asks.

**When discussing architecture beyond the documents:**
- Default to commercial SaaS (HubSpot, QuickBooks, Stripe, Make.com, PDFShift) over custom builds
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
- `sign_everything.html` — unified signing page with HubSpot wiring (~155 KB)
- `clinician-terms.html` — Physician MSA reference (~244 KB)
- `hha-clinician-terms.html` — HHA MSA reference (~248 KB)
- `baa.html` — BAA reference (~243 KB)
- `order-form.html` — Service Order Form preview (~208 KB)
- `_redirects` — Netlify clean-URL routing
- `DEPLOYMENT_GUIDE.md` — operator-focused deployment doc

**Make.com scenario:**
- Account: SnapShot Health Signed Agreement (HubSpot connection name)
- Scenario name: SnapShot Health — Form to Executed PDF
- Modules: 1, 34, 15, 16 (router), 17, 18, 22, 23, 32, 33
- Schedule: every 15 minutes

**Deprecated, should be deleted from repo if still present:**
- `sign_and_start.html` — superseded by `sign_everything.html` (May 8, 2026)
- `INTER_*.html`, `POPPINS_*.html` — font A/B variants from April; Outfit was chosen May 13

---

## Key Contacts & Credentials (for Jimmy's reference, do not store elsewhere)

- SnapShot Health Inc., Dallas, Texas
- Founder/CEO: Jimmy LaRose
- Clinical: Lucy Brandt (BSN, RN) + Angie Wiser (MA)
- Email: careteam@snapshothealth.io (main) · billing@snapshothealth.io · privacy@snapshothealth.io (BAA)
- Phone: (469) 342-3753
- Website: snapshothealth.io (Netlify, Git-connected, auto-deploy on push to main)
- Patient portal: portal.snapshothealth.io (Supabase)
- VitalWatch: reseller.medius.us (Jason Wu at Transtek/MioConnect Labs)
- Transtek API: OrgID 00000062, ApiKey 6E75E2
- HubSpot Portal: 47989991
- HubSpot Sign-Everything Form GUID: 06eff455-1326-4fbc-b96d-88b4f4427c0a
- PDFShift: API key in Make.com HTTP module Authorization header

---

*End of handoff document. Last updated May 13, 2026, after the Make.com post-submission automation pipeline went live, the brand stack was fully consolidated to Outfit across all six deployed HTML files, and end-to-end signing was validated for all three program/audience combinations.*
