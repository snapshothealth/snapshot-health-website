# SnapShot Health — Customer Acquisition Pipeline Reference

**Last updated:** May 13, 2026
**Audience:** Future Claude sessions, technical successor, PE diligence team.
**Source of truth:** HubSpot Portal `47989991` — Deal properties under the **SnapShot Account** group.

This document enumerates the structure of SnapShot Health's sales pipeline in HubSpot: stages, custom Deal properties, and who/what populates each. Read this if you need to understand how a Deal flows from first contact to active customer, or if you're auditing what data is captured at each stage.

---

## Pipeline: Customer Acquisition

**Name in HubSpot:** Customer Acquisition

### Stages

Nine stages total, with HubSpot probability weights configured for weighted pipeline forecasting. **Onboarding, Active, and At Risk are all set to "Won (100%)"** — meaning HubSpot treats a Deal as closed-won the moment it signs. "At Risk" isn't a re-litigation of whether the sale closed; it's an operational flag for accounts that need attention.

| # | Stage | Probability | Trigger to enter | Owner |
|---|---|---|---|---|
| 1 | New Inquiry | 10% | Lead form submission or cold outreach reply | Jimmy |
| 2 | Discovery | 25% | First call scheduled or completed | Jimmy |
| 3 | Proposal | 50% | Signing link sent (Path A: `/review-and-sign`, or Path B: `/sign?program=X&audience=Y`) | Jimmy |
| 4 | **Signed** | 90% | Make.com creates/updates the Deal on form submission | Automated |
| 5 | Onboarding | Won (100%) | Lucy/Angie schedule onboarding call (manual today; could be automated later) | Lucy / Angie |
| 6 | Active | Won (100%) | First billable patient enrolled | Lucy / Angie |
| 7 | At Risk | Won (100%) | Account Health flags concern (manual) | Jimmy / Lucy |
| 8 | Churned | Lost (0%) | Customer ends service after being active | Jimmy |
| 9 | Closed Lost | Lost (0%) | Lead/prospect goes cold or declines before signing | Jimmy |

### Stage volume notes (as of May 13, 2026)

- **Active: 17 deals** — current customer base (Signet, etc.). These are the practices generating revenue.
- **Churned: 11 deals** — historical churn. Worth keeping as a clean diligence data point: shows the business has tracked attrition rather than hidden it.
- **All other stages: 0 deals at the moment**, since the new signing pipeline only started producing automated deals on May 13.

### Why 90% for Signed (not 100%)

The 90% on Signed (vs. 100% on Onboarding) is HubSpot's nudge that a signature isn't quite the same as a paying customer. Edge case: a practice signs but then ghosts onboarding. In practice this is rare — but the 10% gap captures the risk for forecasting purposes.

---

## Custom Deal Properties (SnapShot Account group)

12 custom properties beyond HubSpot's defaults. All in the **SnapShot Account** property group, created by Jimmy.

### Populated automatically by the Make.com signing pipeline

| Property | Type | What it stores | Notes |
|---|---|---|---|
| **Service Program** | Dropdown | `Turnkey` / `Software 20` / `Software 45` (display values) | Set by Module 18 (Create Deal) from the HubSpot form's `service_program` field. The page-side `PROGRAM_CONFIG.hubspotValue` controls these display values — keep in sync. |
| **Audience** | Dropdown | `Physician` / `HHA` (display values) | Set by Module 18 from the form's `audience` field. The page-side `AUDIENCE_CONFIG.hubspotValue` controls these display values. |
| **Group Code** | Dropdown | Internal grouping code (e.g. `comprehensivecardio` for Signet) | Populated for 9 deals at 100% fill — these are pre-existing accounts linked to a parent organization. Currently set manually; could be auto-derived from email domain in the future. |

### Populated manually (or to be auto-populated later)

| Property | Type | What it stores | Populated by |
|---|---|---|---|
| **Account Health** | Dropdown | `Healthy` / `Watch` / `At Risk` / `Critical` (verify exact options in HubSpot) | Jimmy, weekly review. Drives stage transition to "At Risk." |
| **Patient Count** | Number | Current billable patient count for the practice | Today: manual entry. Future: pulled live from VitalWatch / MioConnect API via Make.com. |
| **MRR** | Number | Current monthly recurring revenue from the account | Today: manual entry. Future: computed from Patient Count × CPT mix via the billing pipeline. |
| **EHR System** | Single-line text | Practice's EHR (Athena, Epic, eClinicalWorks, etc.) | Captured during discovery call, manually entered. |
| **Onboarding Start Date** | Date | When Lucy/Angie begin onboarding | Set manually when Stage transitions to Onboarding. |
| **First Bill Date** | Date | First invoice issued for the account | Manual today. Future: auto-set when QuickBooks creates first invoice (Month 2 priority). |
| **Last Invoice Paid Date** | Date | Most recent payment received | Manual today. Future: auto-updated from QuickBooks → Stripe webhook. |
| **Agreement Version Signed** | Single-line text | Order Form / MSA / BAA version executed (e.g. `v2026.1`) | Captured by the signing page's `order_form_version` / `msa_version` / `baa_version` hidden fields. *Could be wired to Make.com to auto-populate.* Currently 0% fill. |
| **Churn Reason** | Single-line text | Free-text reason if the account churns | Manual, set at the time of churn. |

### Not yet a property but referenced in handoff

| Concept | Status |
|---|---|
| **program** (URL slug) | Lives on the **Contact** object, not Deal — `turnkey` / `software-no-device` / `software-with-device`. Set by the signing page; read by Make.com to render the executed PDF. Don't duplicate to the Deal object; the Contact-level slug is sufficient because Make.com associates Contact → Deal. |
| **Forecast MRR / Revenue Dashboard** | Future. Deal `Amount` is intentionally left blank because every Deal's revenue depends on billable patient count, not a flat contract value. A separate dashboard (Looker Studio or HubSpot Custom Report) will surface trailing-12 MRR, ARR, churn, and per-program patient count. Defer scoping until QuickBooks migration is done. |

---

## Standard HubSpot Deal Properties (used)

For completeness, these are the out-of-the-box properties the pipeline relies on:

| Property | What it stores | Set by |
|---|---|---|
| Deal Name | `<Practice Name> - <Service Program> (<Audience>)` | Make.com Module 18 |
| Pipeline | `Customer Acquisition` | Make.com Module 18 |
| Deal Stage | `Signed` (default for new submissions) | Make.com Module 18 |
| Close Date | The `signed_at` timestamp (account timezone: Central) | Make.com Module 18 |
| Amount | **Intentionally blank** — see "Forecast MRR" note above | Not populated |
| Deal Owner | Jimmy LaRose | HubSpot default |
| Create Date | Set when Make.com creates the Deal | HubSpot automatic |
| Associated Contact | The signer | Make.com Module 18 via associationTypeId 3 |

---

## How a Deal Moves Through the Pipeline

```
Stage 1 — NEW INQUIRY (10%)
  Triggered by: cold outreach reply, demo form submission, referral
  Owner: Jimmy
  Data captured: Contact name, email, practice name (sometimes)
  Exits to: Discovery, or Closed Lost

Stage 2 — DISCOVERY (25%)
  Triggered by: first call scheduled
  Owner: Jimmy
  Data captured: EHR System, Patient Count (estimated), Audience type
  Exits to: Proposal, or Closed Lost

Stage 3 — PROPOSAL (50%)
  Triggered by: signing link sent to prospect
  Owner: Jimmy
  Data captured: Service Program decided (Turnkey / Software 20 / Software 45),
                 Term offered (Month-to-month / 12-month)
  Exits to: Signed, or Closed Lost

Stage 4 — SIGNED (90%)  ← Make.com pipeline lives here
  Triggered by: form submission at /sign or /sign-everything
  Owner: Automated (Make.com Module 18)
  Data captured: Service Program (final), Audience (final), Term, signed_at,
                 Agreement Version Signed (if wired), Deal Name auto-generated
  Side effects: executed PDF rendered by PDFShift, welcome email sent via Gmail
  Exits to: Onboarding (almost always); rarely Closed Lost (signed-but-ghosted)

Stage 5 — ONBOARDING (Won 100%)
  Triggered by: Lucy or Angie schedules first onboarding call
  Owner: Lucy / Angie
  Data captured: Onboarding Start Date
  Exits to: Active

Stage 6 — ACTIVE (Won 100%)
  Triggered by: first billable patient enrolled in VitalWatch
  Owner: Lucy / Angie
  Data captured: First Bill Date (when QuickBooks issues first invoice),
                 Patient Count (rolling), MRR (rolling), Last Invoice Paid Date
  Exits to: At Risk (if health degrades), or Churned (if customer leaves)

Stage 7 — AT RISK (Won 100%)
  Triggered by: Account Health drops to "Watch" or worse, OR
                churn signal (paused billing, missed onboarding milestones)
  Owner: Jimmy
  Data captured: Account Health rating, optional Churn Reason notes
  Exits to: Active (if recovered), or Churned (if lost)

Stage 8 — CHURNED (Lost 0%)
  End state. Customer was Active and ended service.
  Owner: Jimmy
  Data captured: Churn Reason
  Historical: 11 deals currently in this stage — kept as honest attrition record

Stage 9 — CLOSED LOST (Lost 0%)
  End state. Lead never signed.
  Owner: Jimmy
  Data captured: Optional reason in notes
```

---

## What's Auto-Populated Today vs. Manual vs. Aspirational

| Property | Status | If aspirational, where it'll come from |
|---|---|---|
| Service Program | ✅ Auto (Make.com) | — |
| Audience | ✅ Auto (Make.com) | — |
| Deal Name, Pipeline, Stage, Close Date, Contact association | ✅ Auto (Make.com) | — |
| Group Code | 📝 Manual | Could be auto-derived from email domain in a future iteration |
| Account Health | 📝 Manual | Weekly review by Jimmy — keep manual |
| Patient Count | 📝 Manual | ⏳ VitalWatch/MioConnect API via Make.com |
| MRR | 📝 Manual | ⏳ Computed from Patient Count × CPT mix, post-QuickBooks migration |
| EHR System | 📝 Manual | Captured during discovery — keep manual |
| Onboarding Start Date | 📝 Manual | Could auto-set on stage transition to Onboarding |
| First Bill Date, Last Invoice Paid Date | 📝 Manual | ⏳ QuickBooks/Stripe webhooks, post-migration |
| Agreement Version Signed | ❌ Empty | Could be wired to Make.com now — small task |
| Churn Reason | 📝 Manual | Set when stage moves to lost/churned — keep manual |

---

## Quick Wins Available Right Now

These are small, isolated improvements to the pipeline data hygiene. Not required for the pipeline to work, but they raise the fill rate and the diligence story.

1. **Wire `Agreement Version Signed`** — the signing page already collects `order_form_version`, `msa_version`, `baa_version` as hidden fields. Add a mapper in Make.com Module 18 to concatenate them as `OF:v2026.1 / MSA:v2026.1 / BAA:v2026.1` and write to this property. ~10 min.

2. **Auto-set `Onboarding Start Date`** when a Deal transitions to Onboarding stage — HubSpot Workflow trigger, no Make.com needed. ~5 min.

3. **Establish `Account Health` review cadence** — even a simple Friday-morning review where Jimmy sets all Active accounts to one of four buckets gives you a data series for diligence.

---

## Maintenance Rules

- **Don't add new Deal properties without adding them here.** This doc is the single reference. If the property isn't documented, a successor won't know what it's for.
- **Display values for dropdowns must match the page configuration.** `Service Program` options (`Turnkey`, `Software 20`, `Software 45`) and `Audience` options (`Physician`, `HHA`) are mirrored in `sign_everything.html`'s `PROGRAM_CONFIG.hubspotValue` and `AUDIENCE_CONFIG.hubspotValue`. Changing one without the other will produce mismatched Deal data.
- **The pipeline name `Customer Acquisition` is referenced by ID in Make.com Module 18.** If you rename the pipeline in HubSpot, also update the module mapper (it stores the pipeline ID, not the name, so it usually survives renames — but verify after any rename).
- **The `Signed` stage is the entry point for the Make.com pipeline.** Don't rename it without updating Module 18's `dealstage` mapper.

---

*End of pipeline reference. Update this doc when properties are added, removed, or repurposed. Last updated May 13, 2026.*
