# Finance Backend — Requirements & Pain Points Report (For Marcus Le)

*Prepared by Michael Beatie, Product Operations — May 18, 2026*

*Sources: ITS Jira tickets, Notion project documentation, RevOps and CS team input*

---

## Executive Summary

The Finance Backend (FBE) has been a persistent operational bottleneck for both Sales and Customer Success. Multiple rounds of feedback, Jira tickets, and internal documentation point to the same root problem: **FBE was built for Finance to operate, not for Sales and CS to use.** The result is a tool that forces manual workarounds, requires Finance team intervention for routine tasks, produces unreliable data in HubSpot, and blocks the reporting visibility that RevOps and Sales leadership need to manage the business.

The promise of FBE 2.0 solving everything has deferred action on individual issues for over a year. The requirements below represent the accumulated, unresolved backlog from the people who use or depend on the FBE daily — organized by category to give Marcus a structured view of what "done" looks like.

**Key stakeholders whose input is represented here:** Venus David (Dir. Revenue Operations), Jessica Cooper (Mgr. Enterprise CS), Aireal Orosco (CS), and broader Sales/CS/Finance input captured in Jira ITS tickets and Notion documentation.

---

## 1. Reporting & Analytics

**The core problem:** HubSpot does not store actual billed amounts, MRR, or discounts applied. The FBE is the system of record for all financial performance data, but it has no native reporting accessible to RevOps, and no integration to pipe data into HubSpot or any BI tool. Venus David cannot get a clean view of her own revenue metrics without submitting a ticket and waiting for a manual Finance export.

### What's missing

- **Rep-level sales transaction data** — closed/won deals tied to individual AEs with associated MRR
- **MRR breakdown by type** — new, expansion, contraction, and churned MRR per period per account
- **Discount tracking** — discount amount and % per deal, per rep, per period
- **Effective rate vs. list price** — what was actually billed vs. what was quoted in the deal
- **Churn attribution** — which accounts churned, when, and the MRR impact
- **Revenue reconciliation** — FBE 实收 data has shown discrepancies vs. billing system estimates with no reliable cross-check tool available
- **Self-service reporting** — no path from FBE data to HubSpot Data Studio or any dashboard without Finance team manual intervention

### Specific ask (Venus David, RPRT-115)

Venus submitted a formal report request for: sales transactions by rep, MRR breakdown, discounts applied, effective rate vs. list price, and churn attribution. None of this is accessible today without a custom data pull. A sample data schema has already been defined (RPRT-115 Notion page) for use if/when a Finance data feed becomes available.

### Longer-term vision

Route structured FBE data (or Sage Intacct data, which is already in motion via BTP Partners) into **HubSpot Data Studio** so RevOps can self-serve on revenue metrics without requiring Finance. This is the #1 RevOps reporting ask that has never been addressed.

---

## 2. User Access & Permissions

**The core problem:** FBE permission tiers are too blunt. Sales reps either have too much access (Super Admin, creating liability) or not enough (can't apply a discount without being promoted to Admin). There is no tiered, role-based model appropriate for a multi-team sales org — and no enforcement mechanism tied to the Universal Pricing Catalog (UPC) discount authority tiers.

### What's broken

- **Super Admin overexposure** — Sales reps under Sam's org had Super Admin access, giving them the ability to create, edit, and delete invoices, credit memos, and write-offs. Venus David flagged this as an urgent risk exposure in April 2026 (ITS-168)
- **No granular discount controls** — Applying a discount currently requires Admin-level access in FBE (ITS-214). The UPC has tiered discount authority (leadership-only, tier-gated), but FBE has no way to enforce this
- **No Finance-specific role** — Finance needs a dedicated role that other teams don't share. A separate "billing role" for Darlene's team has also been requested. Neither exists
- **Audit log access** — Who can view the audit log is unclear. This needs to be available to Finance and RevOps without requiring Super Admin
- **Read-only access for CS/Support** — CS teams need to see billing status and account details without any edit capability. No such role exists

### Role hierarchy needed

- **Sales Rep** — view only; can submit quote requests
- **Senior Rep / TL** — can apply standard discounts within UPC tier
- **Manager** — can approve non-standard discounts up to defined ceiling
- **Billing (Darlene's team)** — can manage payment events, dunning, refunds; no product catalog edit
- **Finance (Shawn/Carlos)** — full financial record access, audit logs, reporting
- **Admin** — configuration access
- **Super Admin** — restricted to designated system owners only

---

## 3. Visual Interface / UX

**The core problem:** Even the FBE 2.0 demo environment raised significant usability concerns during Venus David's October 2025 UAT review. The current FBE is worse. The system is built for engineers and Finance ops — not for sales reps who need to move quickly through a deal.

### Specific UX issues raised (Venus's FBE 2.0 UAT feedback, Oct 2025)

- **Quote builder is incomplete** — "Add Quote" does not show all products/plans. AEs cannot build an accurate quote from the FBE alone
- **Coupon vs. Discount terminology** — Using "Coupon" reads as consumer/grocery-store, not B2B SaaS. Teams are unsure which field to use for standard deal discounts
- **"Product Family" is undefined** — No explanation of what this grouping means or how it maps to Lofty's actual package structure
- **Custom Fields are unclear** — The "Custom Field" section is described as "can be added as features to charge plans" — teams don't know if this is for custom dev pricing or standard configuration
- **No inline field descriptions** — Every non-obvious field needs an info/tooltip explaining what it does and when to use it
- **Log viewer broken** — During the demo, the Logs section produced continuous black error codes. The Events / Operations / Transactions tabs are not explained
- **Pricing accuracy** — Venus flagged that the Enterprise onboarding fee shown ($1,499) may be incorrect. FBE must reflect the UPC, not hardcoded test/demo values
- **Main Invoice vs. Sub Invoice** — The distinction between these is not explained anywhere in the UI
- **Parent-child hierarchy** — No clear UI for multi-team or broker accounts where sub-teams exist under a parent account
- **Promotions / discount approval flow** — Who can add promotions? Do UPC tier limits apply? The flow is completely undefined

### Broader UX asks

- **Deal/order history** — A clean, chronological view of what was ordered, changed, invoiced, and paid per account
- **Human-readable order card** — Instead of raw data views, a formatted summary: package, seats, price, discount applied, effective date
- **Change order flow** — A structured process for modifying an existing order, not ad-hoc field edits
- **Account search that works** — Ability to look up an account by email, name, team ID, or HubSpot company ID without switching systems

---

## 4. Flexibility & Product Catalog

**The core problem:** The FBE billing engine is too rigid for how Lofty sells — especially Enterprise. Any deal requiring custom terms, revenue share, milestone billing, or bundle customization requires manual Finance intervention because the system cannot accommodate it natively. This is the single biggest reason deal cycles are slow for Enterprise.

### What's blocked today

- **Enterprise custom billing** — Every Enterprise deal with any billing customization requires manual Finance team involvement. This slows deal close and creates a single point of failure
- **Mass agent opt-in** — No native bulk opt-in for products or packages. Every change at scale requires manual execution
- **Revenue share** — Not supported in FBE at all. Currently handled manually. Sales needs a path for this; Finance can't sustain the manual load
- **Milestone / staged pricing** — Charging different amounts in different months (e.g., $X month 1, $Y month 2) is currently ad-hoc and messy. No standard process exists, and Finance wants to avoid these deals entirely because of the overhead
- **Package bundling flexibility** — Reps quote non-standard bundles and then hope Finance can accommodate in FBE. The mismatch between what sales promises and what FBE can create is a recurring escalation source
- **Unclear package and feature details** — Sales reps don't always know what is or isn't included in a package when quoting. FBE catalog and UPC should be one source of truth, but they're not aligned

### What's needed

- Product catalog that exactly matches the UPC, maintained in a single authoritative location
- Configurable bundle builder allowing custom packages within guardrails (discount limits, feature toggles, approval gates)
- Native revenue share structure with at minimum a defined workflow and approval chain
- Milestone billing option with structured setup (defined schedule, payment triggers, clear invoicing)

---

## 5. CRM Integration & Real-Time Data Sync

**The core problem:** FBE and HubSpot are supposed to be in sync, but they frequently aren't. Events fire against the wrong deals, statuses don't propagate, reps do double work in two systems, and sales leadership can't trust the data they see in HubSpot.

### Known sync failures

- **Deal owner not syncing HubSpot → FBE** — When a deal is created or closed in HubSpot, the AE is not automatically set as the deal owner in FBE. Reps must manually self-assign in FBE — duplicate work, error risk, comp tracking gaps (ITS-202, Venus David)
- **Payment Failed status inaccurate** — "Is Payment Failed" in HubSpot does not match the actual FBE payment status. Multiple CSMs have flagged examples where accounts show "Past Due" in FBE but "No" in HubSpot (ITS-80, Aireal Orosco)
- **Wrong deal auto-closed as Closed Won** — FBE has fired Closed Won events to HubSpot for the wrong deal when an oppId was reused across customers. This moved a deal to Closed Won that had no actual sale, requiring manual correction (ITS-173, ITS-220)
- **Lifecycle stage misalignment** — A cross-reference of HubSpot company records against the finance database identified 8,450 companies with lifecycle stages that don't match payment reality (ITS-235, ITS-248)
- **FBE linked to wrong account** — Team transfer scenarios leave some HubSpot records linked to the wrong FBE record, showing $0 MRR and "Unpaid" for accounts that are actually active (ITS-163, Aireal Orosco)
- **Lead source inconsistency** — DMA Upsell pipeline lead source options in HubSpot don't match what exists in FBE, creating data integrity problems (ITS-182, Venus David)

### What's needed

- **1:1 deal owner sync** — HubSpot deal owner must automatically propagate to FBE owner field on deal create and close
- **Real-time payment status push** — Payment status, past-due flag, and subscription status should push from FBE to HubSpot on every change event
- **OppId integrity** — Prevent reuse of oppIds; enforce 1:1 deal-to-order mapping at the FBE level
- **Closed Won guard rails** — Middleware must validate that the deal being closed in HubSpot matches the FBE order before firing the event
- **Periodic sync audit** — Tooling or an automated process to reconcile HubSpot company records against FBE payment status on a scheduled basis

---

## 6. Account & Order Visibility

**The core problem:** CSMs and Sales reps need to understand account status, billing, and seat structure without leaving HubSpot. Today they navigate 10+ separate systems. The FBE holds critical data that isn't surfaced where people actually work.

### What's missing in HubSpot today

- **Multi-team account structure** — No way to see how many Lofty teams exist under an account from the HubSpot company record without opening FBE separately (ITS-155, Aireal Orosco)
- **Cancelled accounts in CSM Book of Business** — BoB views include cancelled accounts. CSMs waste time on dead accounts with no way to auto-exclude them (ITS-156, Aireal Orosco)
- **FBE / Admin links not in Quick Views** — Jessica Cooper requested Account ID, FBE Link, and Admin Link be surfaced in the HubSpot Quick Views panel. Finding these manually slows every CS and sales interaction (ITS-186)
- **Account health requires 10+ systems** — CSMs navigate billing, seat counts, subscription details, renewal timelines, and adoption data across separate tools, all manually (ITS-216, Venus David)
- **Real-time subscription fields missing** — Active seat count, subscription tier, add-ons, contract dates, and payment status do not exist as live, auto-populated fields on the HubSpot Company record

### What's needed

- FBE-sourced fields on the HubSpot Company record: MRR, package, active seat count, subscription status, contract start/end, payment status
- Account hierarchy visualization for multi-team broker accounts
- BoB filter: auto-exclude cancelled/churned accounts from CSM views
- Quick-access links to FBE record and Admin backend on every HubSpot company record
- Adoption health score surfaced alongside billing fields for a unified account view

---

## 7. Workflow Automation & Process

**The core problem:** Key steps in the sales-to-billing workflow are either manual, poorly defined in ownership, or missing entirely. The result is client escalations, reps wasting time on administrative tasks, and recurring operational fires that burn CS and Finance team bandwidth.

### License Assignment

- No defined workflow for who assigns licenses after purchase: Finance, Sales, or CS?
- Licenses added via FBE and licenses via CRM self-assignment follow different paths with no consistent process
- Manual FBE additions are sometimes mislabeled as "Self Purchase," causing confusion for Support and Tech teams (Aireal's email thread)
- No validation checkpoint before Help Center instructions are sent to clients — clients receive setup instructions before licenses are actually assigned, causing failed steps and escalations
- No alert when a license remains unassigned after purchase, especially for bulk additions

### Payment Failure / Dunning

- Failed payment retry logic is not clearly automated or visible
- Customer communications for payment failures are inconsistent; the volume of failures relative to successful retries is high, causing excess processor fees
- CSMs have no automated alert when an account in their BoB goes past-due
- Clear dunning sequence (timing, channels, escalation path) needs to be defined and automated

### Quote-to-Close Flow

- The full flow (Quote → PandaDoc → Order in FBE → Closed Won in HubSpot) is fragmented across multiple systems with no clean handoff
- Agreement collection (PandaDoc/DocuSign) is not integrated with FBE order creation — reps manage this manually
- Change orders for upsells or mid-contract modifications have no structured process; reps improvise
- Sales reps must manually self-assign to FBE deals they already own in HubSpot — this step should not exist

---

## 8. Data Quality & Record Hygiene

**The core problem:** Bad data in FBE surfaces everywhere downstream — duplicate HubSpot records, wrong account associations, stale lifecycle stages, inaccurate purchase labels. Without clean foundational data, CS and Sales spend time on manual cleanup instead of customer work.

### Specific issues

- **Duplicate company records** — Multiple HubSpot company records for the same Lofty account are common. Root causes include imports without a Lofty Team ID, and team transfers creating new records instead of updating existing ones (ITS-73, ITS-115)
- **Lofty Team ID mismatches** — CSMs are sometimes given a Lofty Team ID that doesn't match the HubSpot company record ID, making account lookup and CSM assignment impossible (ITS-250)
- **FBE linked to wrong account** — Team transfers can leave FBE pointing at the wrong HubSpot company, showing $0 MRR and incorrect payment status (ITS-163)
- **Test accounts in live pipeline** — QA/test records enter the live HubSpot deal pipeline, inflating pipeline numbers and distorting forecasting (ITS-203, ITS-204)
- **Inaccurate purchase labels** — FBE assigns "Self Purchase" to accounts where Finance manually added a product. Labels must reflect how the purchase actually occurred, not a default classification

---

## Priority Summary

| Category | Pain Level | Blocking | Key Owner(s) |
| --- | --- | --- | --- |
| Reporting & Analytics | Critical | Revenue visibility, comp tracking | Venus David, Shawn Hagen |
| CRM Sync Accuracy | Critical | Closed Won integrity, BoB accuracy | Venus, Aireal Orosco |
| Access & Permissions | High | Security exposure, discount control | Venus David, Eric Thornley |
| Account Visibility | High | CSM efficiency, BoB management | Jessica Cooper, Aireal Orosco |
| Flexibility / Product Catalog | High | Enterprise deal velocity | Venus David, Sales Leadership |
| Workflow Automation | High | License escalations, dunning | CS, Finance, Sales |
| Visual Interface / UX | Medium-High | Adoption, AE usability | Venus David, Sales |
| Data Quality | Medium | BoB accuracy, pipeline integrity | Aireal Orosco, RevOps |

---

## Appendix: Source Tickets & Documents

| Source | ID / Link | Key Content |
| --- | --- | --- |
| FBE Super Admin Risk | ITS-168 | Sales reps had Super Admin access — liability exposure |
| FBE Audit Log / Discount Access | ITS-214 | Discount requires Admin; audit log access gap |
| Deal Owner Sync Gap | ITS-202 | HubSpot deal owner not propagating to FBE |
| Payment Failed Status | ITS-80 | "Is Payment Failed" wrong in HubSpot vs. FBE |
| Wrong Deal Closed Won (Giulia) | ITS-173 | Middleware closed wrong deal via mismatched oppId |
| Wrong Deal Closed Won (Rhiannon) | ITS-220 | Finance Backend oppId reuse caused errant Closed Won |
| CS Account Intelligence | ITS-216 | CSMs navigate 10+ systems; real-time field sync ask |
| FBE Links in Quick Views | ITS-186 | Account ID, FBE Link, Admin Link missing from HubSpot |
| Multi-Team Visibility | ITS-155 | Can't see team count in HubSpot without FBE |
| Cancelled Accounts in BoB | ITS-156 | Cancelled accounts not excluded from CSM BoB view |
| Lifecycle Stage Misalignment | ITS-235, ITS-248 | 8,450 companies wrong lifecycle stage vs. finance DB |
| FBE Linked to Wrong Account | ITS-163 | Team transfer scenarios break FBE-HubSpot association |
| Lead Source Mismatch | ITS-182 | DMA Upsell lead source options inconsistent with FBE |
| Revenue Reporting Request | [RPRT-115](https://www.notion.so/3454a6fce78a81b2a582f6303461b898?pvs=21) | Full data inquiry — MRR, discounts, churn, rep-level |
| FBE 2.0 UAT Feedback | [Notion](https://www.notion.so/Finance-Back-End-FBE-2-0-Optimization-Integration-28c4a6fce78a80878bb2d976e4f50e23?pvs=21) | Venus's Oct 2025 product review of FBE 2.0 demo |
| FBE Flexibility Issues | [Notion](https://www.notion.so/26f4a6fce78a80c1931ff309c1b4f2fa?pvs=21) | 5 core rigidity issues — Enterprise, mass opt-in, catalog |
| Aireal's License Requirements | [Notion](https://www.notion.so/Requirements-from-Aireal-s-Email-Thread-2174a6fce78a807a9cbff7e297fc1019?pvs=21) | License assignment, labeling, escalation prevention |
| Finance & FBE Meeting Notes | [Notion](https://www.notion.so/2794a6fce78a8059afb2e35b786b6491?pvs=21) | Dunning, PCI, revenue share, milestone pricing, reporting |
| Billing MVP Definition | [Notion](https://www.notion.so/Billing-MVP-Definition-2a14a6fce78a8039af3df5e1e5cb8b95?pvs=21) | MVP requirements for billing ticket and workflow automation |
| Finance & Billing Project | [Finance & Billing ](https://www.notion.so/Finance-Billing-1ae4a6fce78a809b8c12ee63aa1bf98e?pvs=21) | Parent project page — FBE overhaul initiative |

---

## What's Being Built: Lofty Subscription Object + New Sales Ordering Tool

Two major initiatives are underway that will resolve a meaningful share of the issues documented above. This section maps what each initiative fixes, and defines what remains open after both ship.

---

### A. Michael Wang's Lofty Subscription Object Plan (Engineering — Phased)

Michael Wang (Engineering) published a 3-phase implementation plan on Confluence ([CSM Hubspot process and workflow project](https://loftyinc.atlassian.net/wiki/spaces/CRM/pages/1738244098/CSM+Hubspot+process+and+workflow+project), last updated May 19, 2026). This defines how the Lofty Subscription Object will be populated and maintained by upstream FBE servers and middleware. It was discussed with Eric Thornley and Michael Beatie on May 18, 2026.

The **Lofty Subscription Object already exists in HubSpot** (API name: `lofty_subscriptions`, type ID: `2-53567820`). As of May 1, 2026, it has 22 properties including MRR, ARR, subscription status, cancellation date and reason, reactivation date, contract start/end/renewal, billing frequency, max seats, discount amount and type. Associations to Company, Contact, and Deal all exist. The object is ready for engineering to write to.

**Phase 1 — Main Subscription + Company + Lifecycle Events**

- One Subscription record per main CRM package subscription
- Subscription linked to payer Contact and to Company record
- For **multi-team accounts**: one shared Subscription linked to all sub-team companies AND the top-level special company
- **Company-to-company association labels** created via API: `Multi-team Parent / Multi-team Member` and `Vendor Parent / Vendor Member` — enabling native account hierarchy display in HubSpot
- All main package lifecycle events pushed to Subscription: payer update, MRR update, cancellation (with reason code), reactivation, contract renewal, Payment Failed
- New closed sales deals → new Subscription record created (deal-to-subscription link is live)
- **One-time Phase 1 data cleanup SOP**: export all active main subscriptions from FBE, create/update HubSpot records, rebuild all associations via `lofty_team_id` lookup; middleware stamps `_phase1_category` on each Company for audit (temporary field, deprecated ~6 months post-cleanup)
- New Company fields added: `multi_team_name`, `multi_team_id`, `vendor_id`, `vendor_name`, `_phase1_category`

**Three account structure cases handled in Phase 1:**

- **Case A (Multi-team):** Top-level company associated as "Multi-team Parent" to all sub-team companies; one shared Subscription linked to all of them
- **Case B (Vendor — Real Brokerage, Curaytor, etc.):** SPECIAL parent company linked to sub-teams via "Vendor Parent"; each sub-team has its **own independent** Subscription; SPECIAL parent has no Subscription
- **Case C (Single instance):** 90%+ of subscriptions; one Subscription → one Company, standard

**Phase 2 — Add-on Subscriptions**

- One Subscription record per add-on (Website Add-on, Office Add-on, private instances)
- Payer can be a team member or office owner — not necessarily the company owner
- Same lifecycle event set as Phase 1
- Self-purchase path: upstream signal → middleware → Subscription + Company + Contact creation and association
- Upsell-driven path: upstream signal → lead/deal creation → sales conversion → Subscription creation
- Note: special company record may be needed for Office add-on accounts (May 18 discussion)

**Phase 3 — Contact + Company Cleanup + Lifecycle Events** *(scope TBD)*

#### What Phase 1/2 directly resolves from the pain points above

| Pain Point | Status | Notes |
| --- | --- | --- |
| Multi-team accounts not visible in HubSpot (ITS-155) | ✅ Phase 1 | Multi-team Parent/Member labels + shared Subscription on all sub-teams |
| Payment Failed not syncing to HubSpot (ITS-80) | ✅ Phase 1 | Payment Failed event written to Subscription `payment_failed_` and `subscription_status` |
| FBE linked to wrong account — $0 MRR after team transfer (ITS-163) | ✅ Phase 1 cleanup | SOP rebuilds all associations from authoritative FBE data |
| Cancelled accounts remaining in CSM BoB (ITS-156) | ✅ Phase 1 | Subscription `subscription_status` = Cancelled enables reliable BoB filtering |
| Wrong deal auto-closed as Closed Won / oppId reuse (ITS-173, ITS-220) | ✅ Phase 1 (partial) | New Subscription created from closed sales deal; deal↔subscription link provides guard rails |
| Lifecycle stage misalignment — 8,450 records (ITS-235, ITS-248) | ✅ Phase 1 + Phase 3 | Phase 1 cleanup corrects associations; Phase 3 adds lifecycle event handling |
| MRR, ARR, subscription status not in HubSpot | ✅ Phase 1 | All written to Subscription by middleware in real time |
| Churn attribution — cancel reason and date | ✅ Phase 1 | `cancel_reason` and `cancel_date` per subscription episode; supports multiple cancel/reactivate cycles |
| Reactivation tracking | ✅ Phase 1 | `reactivation_date` on Subscription |
| Contract start/end/renewal dates missing | ✅ Phase 1 | On Subscription Object |
| Add-on subscription visibility | ✅ Phase 2 | One Subscription per add-on, linked to Company and payer Contact |
| Duplicate contacts representing two subscription relationships (ITS-199) | ✅ Phase 1/3 (partial) | Subscription context per person allows safe disambiguation |
| Lofty Team ID mismatches / wrong associations (ITS-250, ITS-163) | ✅ Phase 1 cleanup | SOP rebuilds all records using `lofty_team_id` as lookup key |

---

### B. Marcus's New Sales Ordering Tool

Marcus Le has built and deployed (production as of May 2026) a new sales ordering and quoting tool that replaces the fragmented quote-to-close process. It launched alongside the AI Agents Bundle Pool Model pricing.

**What it does (per Quotation & Billing System initiative, INI-7):**

- Sales creates quotes and generates compliant payment links — without direct access to the billing system
- Billing access is controlled: Chantal manages billing; sales operates through the tool's guardrails
- Integrated with HubSpot for deal creation, affiliate tracking, and reseller sales code attribution
- AI discount compliance layer: discounts are validated against approved tiers before a quote can go out
- SKU catalog aligned with Venus's finalized structure (AI Agents Bundle Pool Model, CRM packages)
- HubSpot order process connects to FBE to initiate the sale in the billing system
- Resellers use the same tool with their sales code for automatic commission attribution

**What this directly resolves from the pain points above:**

| Pain Point | Status | Notes |
| --- | --- | --- |
| Quote builder doesn't show all products (FBE 2.0 UAT) | ✅ Addressed | Tool built around the finalized SKU catalog |
| Discount compliance not enforced | ✅ Addressed | AI discount compliance layer validates before quote generation |
| Reps manually self-assigning in FBE (ITS-202) | ✅ Partial | Tool initiates the FBE order from HubSpot; reduces manual FBE entry |
| Quote → payment link fragmentation | ✅ Addressed | End-to-end: quote → compliant payment link, generated in the tool |
| Sales had no billing access controls | ✅ Addressed | Chantal controls billing; sales operates within guardrails |
| Affiliate/reseller commission attribution | ✅ Addressed | Sales code integration; commission tracking auto-calculated |
| No structured quote-to-close flow | ✅ Addressed | Quote → payment link → FBE order initiation in one tool |

---

## Remaining Gaps — What Neither Initiative Addresses

These are the pain points **not resolved** by the Subscription Object or the new ordering tool. They represent the true remaining ask from Sales and CS.

### Reporting & Analytics — LARGELY UNRESOLVED

- **Rep-level sales transaction data** — Subscription Object provides MRR and contract dates per account, but sales performance data (deals closed, discounts given, effective rate vs. list price by AE by period) still requires a Finance data feed. HubSpot Data Studio integration with FBE or Sage Intacct is not built.
- **Self-service revenue reporting** — Venus still cannot pull a rep-level revenue report without Finance team involvement. This gap is entirely standalone.
- **Discount tracking at deal level** — Subscription Object has `discount_amount` and `discount_type`, but these fields must be written by FBE middleware. The data path from the ordering tool's quote → discount → FBE → Subscription is not confirmed.

### User Access & Permissions — UNRESOLVED

- FBE role hierarchy (Rep / TL / Manager / Finance / Admin / Super Admin) is not part of either initiative
- Discount limits enforced in the ordering tool but **not in FBE directly** — a user who bypasses the tool has the same FBE access they always had
- Finance-specific role and dedicated billing role for Darlene's team: still open
- Audit log access for Finance and RevOps: still open

### FBE Visual Interface / UX — PARTIALLY RESOLVED

- The ordering tool replaces the FBE quote builder for sales — resolving "quote builder incomplete," "Coupon vs. Discount," and discount compliance UX issues within that surface
- **Still open in FBE directly:** field-level tooltips, log viewer errors, parent-child hierarchy display in FBE, change order UI, Main vs. Sub Invoice distinction
- CS and Finance still navigate FBE directly for non-quoting tasks with no UX improvement

### Flexibility & Product Catalog — PARTIALLY RESOLVED

- Ordering tool uses the finalized SKU catalog — a step forward on catalog alignment
- **Still open:** revenue share support, milestone/staged pricing, mass agent bulk opt-in, true bundle customization engine within FBE. Enterprise deals requiring non-standard terms still require Finance manual intervention.

### CRM Integration & Data Sync — LARGELY RESOLVED

- Most sync failures are addressed by Phase 1
- **Still open:** Deal owner sync HubSpot → FBE (ITS-202) — unclear if the ordering tool handles this at order creation, or if it still requires a separate middleware fix
- Lead source inconsistency between DMA pipeline and FBE (ITS-182)

### Account & Order Visibility — LARGELY RESOLVED

- Multi-team visibility, Subscription status filtering for BoB, lifecycle alignment all addressed by Phase 1
- **Still open:** FBE and Admin links in HubSpot Quick Views (ITS-186) — this is a HubSpot admin config task not covered by any initiative
- Adoption health score alongside billing fields — Subscription Object covers the billing side; the adoption data integration is a separate initiative

### Workflow Automation — PARTIALLY RESOLVED

- Quote-to-close: ordering tool covers quote → payment link; Subscription Object covers deal → subscription creation
- **Still open:** license assignment ownership and workflow (no clear owner, no automation), dunning and retry automation with customer-facing comms, structured change order process for mid-contract modifications, alerts for unassigned licenses after purchase

### Data Quality & Record Hygiene — LARGELY RESOLVED

- Phase 1 cleanup SOP will correct company associations, FBE links, and Team ID mismatches at the record level
- **Still open:** test accounts in live deal pipeline (ITS-203, ITS-204), FBE purchase labels that mislabel manual additions as "Self Purchase"