# Meta Conversions API (CAPI) & Conversion Leads

## Overview

This document covers the complete flow of integrating CRM data with Meta Ads via the Conversions API (CAPI), specifically for the "Conversion Leads" optimization objective. It explains datasets vs pixels, the migration path from basic Lead Ads to Conversion Leads, and practical setup considerations.

## Dataset vs. Pixel — Same Thing, Different Hat

A **dataset** is a container; a **pixel** is a tracking tool inside it. Meta restructured Events Manager so that a "dataset" is the top-level organizational unit that collects ALL event data for a business — from the website pixel, from CAPI server events, from mobile apps, from offline activity, and from CRM/messaging channels. The pixel still exists as a browser-side tracking integration, but it now lives *inside* a dataset.

**The critical practical fact: the dataset ID is the same numeric ID as the pixel ID.** If your existing pixel was `1234567890`, that same number is now also your dataset ID. They are interchangeable in the API endpoint:

```
POST https://graph.facebook.com/{API_VERSION}/{PIXEL_ID}/events
POST https://graph.facebook.com/{API_VERSION}/{DATASET_ID}/events
```

These are the same endpoint. Whether you call it `pixel_id` or `dataset_id`, it's the same number.

## Starting Point: Lead Ads Without a Pixel

Lead Ads with instant forms do **not** need a pixel. Meta tracks form submissions natively. There IS an auto-created dataset behind the scenes (named after the Facebook Page, e.g. "ALVA Energie Event Data"), but it's passive. The customer just collects leads.

The pixel only becomes relevant when you want to:
- Track website behavior
- Build website-based custom audiences
- Use Conversion Leads optimization (sending CRM data back to Meta)

## CRM Dataset Type — The Missing Piece

For **Conversion Leads optimization** to work (Meta optimizing for qualified leads, not just form submissions), the dataset needs CRM data sourcing enabled. This is configured in Events Manager:

1. Go to Events Manager → Connect Data Sources → choose **"CRM"**
2. This creates a dataset (or enables CRM data sourcing on an existing one) that unlocks the "Conversion Leads" optimization objective in Ads Manager

**You can add CRM as a data source to an EXISTING pixel/dataset.** You don't need to create a new one. In fact, Meta recommends using the existing dataset because:
- A new empty dataset has zero historical data (takes up to 180 days to mature)
- The existing dataset already has lead form submission data
- Custom audiences built from the existing dataset remain intact

## How Meta Connects CAPI Events to Campaigns

The bridge is the **`lead_id`** (leadgen_id):

1. User submits a Meta Lead Form → Meta generates a `leadgen_id` (15-17 digit number)
2. Your app receives this `leadgen_id` via the leadgen webhook
3. You store the `leadgen_id` in your database
4. When a CRM stage changes, you send a CAPI event with `user_data.lead_id` set to that same `leadgen_id`
5. Meta matches it back to the original ad, ad set, and campaign

**The `lead_id` is NOT hashed** — it's sent as a plain integer. This is different from email/phone which must be SHA-256 hashed.

**Fallback matching:** If you don't have the `lead_id`, Meta can attempt matching using hashed email (`em`), hashed phone (`ph`), or click ID. But `lead_id` is the highest-priority and most reliable match signal.

**The pixel/dataset association matters too:** Events must go to a pixel/dataset associated with the same ad account that runs the Lead Ads. Meta uses BOTH the pixel association AND the lead_id to connect events to campaigns.

### Match keys for Meta Lead Ads specifically

Generic CAPI documentation recommends sending a "full match key set" of roughly 10 identifiers: `lead_id`, `em`, `ph`, `fn`, `ln`, `fbp`, `fbc`, `client_ip_address`, `client_user_agent`, `external_id`. **For Meta Lead Ads, half of that set is not available at all** — and it's a design choice, not a gap.

The browser-side tracking fields — `fbp` (Facebook browser ID), `fbc` (Facebook click ID), `client_ip_address`, `client_user_agent` — are captured by the Meta Pixel at form-view time on a *website*. Meta Lead Ads run Instant Forms *inside Meta's own apps*, so there is no browser session, no pixel fire, and no click-through to track. Meta's leadgen webhook and Graph API lead endpoint both reflect this — they deliver `leadgen_id`, `form_id`, `ad_id`, `page_id`, `created_time`, and `field_data` (form answers). No browser tracking fields.

**`lead_id` is Meta's deliberate replacement for those browser-side IDs.** The Conversion Leads integration was designed around the premise that Meta already knows which ad, ad set, and campaign produced each lead — `lead_id` is the server-side pointer into that. That's why Meta documents `lead_id` as the highest-priority match signal specifically for CL events.

For Meta Lead Ads, the realistic full match-key set is: `lead_id` (primary) + `em` + `ph` + `fn` + `ln` + `external_id`. That's the ceiling — there is no extra EMQ lift available from adding browser fields that don't exist. If you also run website lead forms captured by your own pixel, those events *do* have browser fields and should include them; for Meta Lead Ads specifically, 6 keys is full.

`external_id` is worth sending: a stable advertiser-side identifier (hashed) gives Meta a way to correlate multiple CAPI events for the same person over time, independent of `lead_id` — useful if the Lead record is eventually deleted but later events still reference the same contact.

## CAPI Payload for Conversion Leads

For Conversion Leads, the CAPI payload has specific requirements:

```json
{
  "data": [{
    "event_name": "qualified_lead",
    "event_time": 1718371200,
    "action_source": "system_generated",
    "event_id": "deal_001_qualified_1718371200",
    "user_data": {
      "lead_id": 99887766,
      "external_id": ["sha256_hashed_internal_id"],
      "em": ["sha256_hashed_email"],
      "ph": ["sha256_hashed_phone"],
      "fn": ["sha256_hashed_first_name"],
      "ln": ["sha256_hashed_last_name"]
    },
    "custom_data": {
      "lead_event_source": "Kalasar",
      "event_source": "crm",
      "value": 5000.0,
      "currency": "EUR"
    }
  }]
}
```

Key fields:
- `action_source`: Must be `"system_generated"` (not `"website"`)
- `event_name`: Free-form string — your CRM stage name, NOT a standard Meta event
- `custom_data.event_source`: Must be `"crm"`
- `custom_data.lead_event_source`: Name of your CRM tool (e.g., `"Kalasar"`)
- `custom_data.value` + `currency`: Only for won/purchase events with revenue

**Event naming:** Meta's Conversion Leads integration uses **free-text stage names**, not standard events. The lead form submission already counts as a "Lead" event, so don't send "Lead" again for qualified deals. Use your own CRM stage names (e.g., `"qualified_lead"`, `"meeting_booked"`, `"closed_won"`).

**Send ALL stages:** Meta recommends sending all pipeline stages (not just qualified + won) for better optimization. The more funnel visibility, the better.

## The Migration Flow: From "Collect Leads" to "Conversion Leads"

### Don't Touch Existing Campaigns

This is the key insight from every agency guide and Meta's own recommendations:

- **Create NEW campaigns** with "Conversion Leads" optimization
- **Keep old "Maximize Leads" campaigns running** in parallel
- The two can coexist — it's a per-campaign/ad-set level setting, NOT account-level
- Gradually shift budget as the new campaigns prove themselves

There is **no dataset switching on existing ad sets** needed. The migration is:
1. Send CAPI data to the same dataset the Lead Ads already use
2. Create new campaigns pointing at that same dataset, but with "Conversion Leads" optimization
3. Run both in parallel

### Why Not Modify Existing Campaigns?

Switching from "Maximize leads" to "Maximize conversion leads" on an existing ad set counts as a **significant edit** that resets the learning phase. Meta typically needs ~50 conversion events within 7 days for an ad set to exit learning — though per Meta's own hedge, this "can fluctuate based on campaign objective, budget, audience size, and competition." It's cleaner to create new campaigns.

### Datasets and Targeting

Datasets ARE used for targeting — they power custom audiences, lookalike audiences, and retargeting. This is why you use the EXISTING dataset:
- If you create a brand new dataset, all existing custom/lookalike audiences built from the old one become disconnected
- By adding CRM as a data source to the existing dataset, everything stays connected AND you add the quality signal

**The targeting change with Conversion Leads is not about the dataset — it's about the optimization goal.** When you set "Maximize conversion leads":
- Meta's algorithm learns which TYPES of people become qualified leads (not just which people submit forms)
- Cost per raw lead goes UP (Meta is more selective)
- Cost per QUALIFIED lead goes DOWN (~15-20% improvement per Meta's marketing claims)
- Lead volume decreases, quality increases significantly

### Prerequisites for Conversion Leads

Two different gates apply here — third-party write-ups routinely conflate them. Separate them in your head before evaluating any customer.

**Conversion Leads eligibility (CL-specific):**
- **≥200–250 leads per month at the account / CRM-connection level.** Sources disagree: LeadsBridge cites 200 with a direct link to Meta's dev doc; Datahash and HighLevel cite 250 (likely a newer Meta revision). No public source scopes it verbatim (ad account vs. business), but the phrasing "Facebook must be generating…" implies account-level, not per ad set.
- **Chosen stage has a 1–40% conversion rate.** Likely lead-to-stage; Meta doesn't publish the denominator verbatim. Treat as inferred.
- **Chosen stage happens within 28 days of lead generation.** This is a stage-latency criterion for the CL integration. It is NOT Meta's ad attribution window (7-day click / 1-day view) and NOT the 7-day `event_time` ingestion gate below.
- **Meta Lead ID must flow from webhook to CRM** (`leadgen_id`) and back out via CAPI as `user_data.lead_id`.
- **Conversion Leads is only available for Instant Forms.** Not website lead forms, Messenger ads, or Calls.

**Learning-phase rule (generic Meta, applies to Conversion Leads too):**
- **~50 conversion events per ad set per 7 days** at the chosen stage to exit the learning phase. This is Meta's universal learning-phase floor — same rule as Purchase or any other optimization. Meta's own hedge: the number "can fluctuate based on campaign objective, budget, audience size, and competition." Below this, the ad set stays in "Learning Limited" — delivery continues but performance is more volatile.
- **~30-day model-training window** precedes the standard 7-day learning phase for the CL ranking model specifically (per CustomerLabs, citing Meta). Total warm-up before results stabilize: roughly 37 days, not 7.

**Event ingestion (applies to every CAPI event regardless of objective):**
- **Events older than 7 days are rejected** by CAPI on the `event_time` field. See [Event Time Windows](#event-time-windows--the-7-day-rule) for the 7-day rule, the 90-day lead retention cliff, and the clamp mitigation.

### When Conversion Leads is worth enabling

The readiness question is not "are we eligible?" — it is "can we generate enough events at the chosen stage to exit learning phase on a single ad set?"

**The readiness formula:**

```
(leads/month × lead-to-stage rate) / 4.3 weeks ≥ 50 events/week/ad set
```

Plus the Meta gates above (≥200–250 leads/month at the account level, chosen stage 1–40% rate, chosen stage ≤28 days from lead gen, Instant Forms only).

The binding constraint in every realistic scenario is the **50 events/week/ad set learning-phase rule (~215/month/ad set)**. The 200–250 leads/month eligibility gate is rarely the bottleneck.

#### Scenario A — Optimize on "Qualified"

| Input | Value |
|---|---|
| Performance goal | "Maximize number of conversion leads" |
| Selected stage | Qualified (e.g. completed 1st call, showed interest) |
| Lead-to-Qualified rate | 25% (middle of Meta's 1–40% band) |
| **Qualified events needed** | **≥50/week = ~215/month per ad set** |
| **Minimum leads/month** | **~860** |
| Ad sets | 1 (multi-ad-set multiplies the volume requirement linearly) |
| Fits 28-day stage-latency? | Yes — 1st call typically happens within days |
| Warm-up time | ~30 days model training + 7 days learning |

**Verdict:** viable for businesses doing ~200 leads/week. This is the realistic CL target for most customers.

#### Scenario B — Optimize on "Closed-Won"

| Input | Value |
|---|---|
| Performance goal | "Maximize number of conversion leads" |
| Selected stage | Closed-Won |
| Lead-to-Won rate | 10% (realistic for short-cycle SMB services / online memberships) |
| **Won events needed** | **≥50/week = ~215/month per ad set** |
| **Minimum leads/month** | **~2,150** |
| Ad sets | 1 |
| Fits 28-day stage-latency? | **Only if sales cycle ≤28 days** — rules out most B2B |
| Warm-up time | ~30 + 7 days |

At a 2% Lead-to-Won rate (typical for higher-ticket B2B), the minimum leads/month balloons to ~10,750. Scenario B is feasible only for high-velocity, short-cycle funnels — online courses, memberships, low-ticket services.

#### Scenario C — "Optimize for total revenue" (not a configuration Meta offers)

**Meta does NOT offer a "Maximize value of conversion leads" performance goal.** Every credible source confirms only two performance goals exist for Lead Ads with Instant Forms: "Maximize number of leads" and "Maximize number of conversion leads." Both optimize for count of a chosen stage, not for revenue.

The CAPI payload accepts `custom_data.value` and `currency`, but for Conversion Leads those drive **reporting only**, not bidding. Revenue-optimized bidding exists only under the Purchase objective (website checkout events, not CRM lead stages).

Four closest substitutes, in order of realism:

**C1. Optimize on Closed-Won, accept flat-count optimization.**
Meta maximizes the *count* of wins. Revenue grows proportionally with win-count, but Meta does not favor high-value leads. Minimums = Scenario B (~2,150 leads/month at 10% Lead-to-Won).

**C2. Optimize on Qualified, attach revenue for reporting only. (recommended for most customers)**
Send `custom_data.value` + `currency` on downstream Closed-Won CAPI events. Meta still optimizes on Qualified count; the revenue attaches to Ads Manager reporting so you can see ROAS per creative/audience. Minimums = Scenario A (~860 leads/month at 25% Lead-to-Qualified).

**C3. Layer Value Rules on top of "Maximize conversion leads" (advanced, unverified).**
Value Rules are bid-adjustment modifiers for audience segments you define as high-value. Meta has extended Value Rules beyond Purchase, but **no public source verifies they work on the CL performance goal specifically.** Treat as unproven. If supported, requires Scenario A or B minimums plus enough per-segment volume to make bid adjustments meaningful.

**C4. Switch objective entirely — Sales + Purchase event.**
Not applicable for CRM-based lead-to-sale pipelines (phone sales, field sales, anything offline). Only fits e-commerce and self-serve SaaS where the sale happens on a web page.

**Honest conclusion:** "Optimize Meta for revenue" on a CRM-connected Lead Ads setup is effectively not a setting you turn on. The practical recommendation for most customers is **C2** — optimize on Qualified, send revenue on downstream events for reporting, accept that Meta optimizes on count.

#### Calibration against a real customer pipeline

Example: 200 leads/month, 50 Qualified/month, 25 Sales/month, 1 ad set. Weekly equivalents: ~46 leads/week, ~12 qualified/week, ~6 sales/week.

| Stage as optimization target | Meets 50/week/ad set? | Meets 1–40% rate? | Meets 28-day window? | Verdict |
|---|---|---|---|---|
| Qualified (12/week) | ❌ 24% of threshold | ✅ 25% | ✅ | **Eligible, but "Learning Limited"** |
| Closed-Won (6/week) | ❌ 12% of threshold | ✅ 12.5% | ⚠️ 2nd call often >28d | **Do not optimize here** |
| Revenue | **Not a Meta option** | — | — | **See C2** |

**Recommendation for this customer:**
- Optimize on **Qualified** — best of the available choices. Expect "Learning Limited" status.
- **Keep the parallel Maximize Leads campaign running** — hedges against learning-phase volatility.
- **Send all stages to Meta via CAPI**, including Closed-Won with `value` attached. Even when not the optimization target, downstream events feed Meta's ML and enable ROAS reporting.
- **Do not expect the headline 15–20% CPQL / 44% quality improvement numbers.** Those are Meta's marketing claims for advertisers clearing the learning-phase threshold cleanly.

### What the Ad Set Looks Like

When creating a Conversion Leads campaign:

1. **Campaign level:** Select "Leads" objective (same as before)
2. **Ad set level — Conversion Location:** Select "Instant forms" (same as before)
3. **Ad set level — Performance Goal:** **"Maximize number of conversion leads"** (the new option)
4. **Ad set level — Dataset:** Select your CRM-connected dataset
5. **Ad set level — Conversion Event:** Pick which CRM stage to optimize for (e.g., "Qualified Lead")

### Expected Results

| Metric | Change |
|---|---|
| Cost per raw lead | Goes UP (~10-30%) |
| Cost per qualified lead | Goes DOWN (~15-20%) |
| Lead volume | Decreases |
| Lead quality | Significantly increases |
| Lead-to-qualified conversion rate | ~21-44% increase |

Meta is being more selective about who sees the ad. Fewer junk leads, more real prospects.

**Caveat:** These figures come from Meta's own marketing materials (not independently audited research) and assume the ad set clears the 50/week/ad-set learning-phase threshold cleanly. "Learning Limited" ad sets below the threshold should not expect these gains.

## Event Time Windows — The 7-Day Rule

Meta's CAPI has strict timing rules on the `event_time` field. Getting this wrong is the #1 cause of CRM events silently failing to optimize campaigns, and it hits hardest on first-time CRM sync and on long B2B sales cycles.

### The Rules

- **Past limit:** `event_time` must be within **7 days** of the current time. Meta's own docs: *"The `event_time` can be up to 7 days before the time at which you send an event to Facebook."*
- **Future limit:** `event_time` can be at most **1 hour** in the future.
- **Batch-level rejection:** If a single event in a batch exceeds the 7-day window, Meta rejects the **entire batch**. No partial success.
- **After lead generation only:** `event_time` must be *after* the lead's generation timestamp. You cannot report a qualification that happened before the lead existed.
- **`lead_id` does not extend the window.** Having the Meta lead ID improves matching quality but does not bypass the 7-day rule.
- **`action_source` makes no difference.** Whether you use `website`, `system_generated`, `offline`, or any other source, the 7-day limit applies uniformly.

### The 90-Day Lead Retention Cliff

Separate from the 7-day ingestion window: **Meta keeps lead records for 90 days**. After that, the lead is flushed from their system. Updates referencing leads older than 90 days are silently ignored for optimization, even if the API accepts the payload.

Two cliffs in sequence:
- **Day 7:** API rejects the event. Nothing lands.
- **Day 90:** Even if you get the event through the gate (via clamping, below), Meta has no lead record to attribute it to.

### Why Meta Has This Rule

Three real reasons behind the 7-day window:

1. **Click log retention.** Meta's hot attribution pipeline keeps ad click and impression data for roughly 7 days. After that, the click that drove the lead is no longer available for real-time matching in the normal attribution flow. (Match via `lead_id` still works, but it is a separate code path.)
2. **Model freshness.** The ad optimizer retrains on recent conversions. Signals from 45+ days ago reflect stale creative, audience, and seasonality conditions, which would drift the model.
3. **Deduplication window.** Meta dedupes events across a 7-day window. Beyond that, dedup state is flushed and duplicate risk rises.

The rule makes sense for consumer e-commerce — click an ad, buy in hours or days. **It fits B2B reality badly.** Enterprise sales cycles routinely run 30–180 days, and the moment a customer qualifies or closes is often well outside Meta's window.

### The First-Sync Problem

This hits hardest on **initial CRM integration.** The customer has months of historical deals sitting in their CRM. We want to push that history to Meta so the optimizer learns quickly. But the 7-day rule means most of it bounces on arrival.

Meta's official answer is to use Events Manager's **offline events CSV upload**, which has a 62-day window. That is a separate manual tool in the Meta UI, not something a CRM-CAPI integration can call programmatically.

### Mitigation: The Clamp Approach

A pragmatic workaround for events older than 7 days: **set `event_time` to roughly 6 days ago** (a safety margin inside the window), while keeping the correct lead match via `lead_id`.

What happens in practice:

- ✅ Meta accepts the event (passes the 7-day gate)
- ✅ `lead_id` matching still attaches the conversion to the correct lead, ad, ad set, and campaign — for leads ≤90 days old
- ✅ Conversion Leads optimization still learns which creative/audience produced qualified leads
- ⚠️ Attribution date in Ads Manager reports will show ~6 days ago, not the real conversion date
- ⚠️ Technically against Meta's stated rule (`event_time` should reflect reality). Low practical risk — Meta does not appear to penalize this, and it is common across CRM-CAPI integrations — but it is a rule bend

**Clamp math must respect the "after lead generation" rule:**

```
event_time = max(lead.created_at + 1 second, now() - 6 days)
```

Otherwise you risk sending a timestamp that predates the lead itself, which Meta also rejects.

**Implemented in Kalasar's CAPI sender** (`backend/app/features/crm/capi.py`). Events with `event_timestamp` older than 7 days are clamped to 6 days ago (respecting the lead-gen floor); events whose underlying Meta Lead is older than 90 days are skipped entirely (past the retention cliff — Meta has no record to attribute to). See `_compute_event_time`.

### What NOT to Do

- **Do not rewrite `event_time` to `now()`** for old events. That compresses months of CRM history into today, seriously misleading the optimizer about which creative produced which outcomes. Clamping to 6 days ago is a minor inaccuracy; rewriting to today is a large one.
- **Do not submit events for leads older than 90 days via CAPI.** Meta no longer has the lead record, so the match fails silently downstream. These should be classified in the product UI as "truly unrecoverable," distinct from "expired but clampable."

### The 7-Day vs. 90-Day Backfill Contradiction

Meta's own product guidance says *"backfilling up to 90 days of CRM history helps the algorithm learn faster."* Meta's API docs say `event_time` is capped at 7 days. These two official statements contradict each other.

The resolution:

- **"Backfill 90 days"** is achievable only via the Events Manager offline events CSV import, not via CAPI.
- **Via CAPI,** you either send events as they happen (ideal), or clamp older events into the 7-day window (pragmatic).

Any integration that claims "automatic 90-day CAPI backfill" is either clamping behind the scenes or silently dropping most events. Be honest with customers about which of these is happening.

## Practical Considerations for Kalasar

1. **Pixel/dataset selector is needed** — list existing datasets (including the auto-created Page one) and recommend picking the one already associated with their Lead Ads
2. **No migration/switching logic needed** — customers create new campaigns for Conversion Leads, they don't modify existing ones
3. **We should send MORE stages** — Meta recommends ALL pipeline stages (not just qualified + won) for better optimization
4. **Onboarding should explain** that after connecting CRM + selecting the pixel, the customer needs to create new campaigns with the "Conversion Leads" objective in Ads Manager — we can't automate that part
5. **Backfill is valuable but constrained** — Meta's algorithm learns faster with historical data, but the CAPI enforces a 7-day `event_time` window. Full 90-day backfill is only possible via Events Manager CSV upload (manual). Via CAPI we either send events live or clamp older events into the window (see [Event Time Windows](#event-time-windows--the-7-day-rule))

## What Kalasar's CAPI sender does today

Implemented in `backend/app/features/crm/capi.py` and tested in `backend/tests/features/crm/test_capi.py`:

- **Match keys sent**: `lead_id` (unhashed, plain integer), `external_id` (hashed `Lead.id` UUID), `em` (hashed email), `ph` (hashed phone), `fn` + `ln` (hashed first/last name). That's the realistic full set for Meta Lead Ads — see [Match keys for Meta Lead Ads specifically](#match-keys-for-meta-lead-ads-specifically) for why the browser fields aren't available.
- **Event names**: `qualified_lead` for QUALIFIED stage, `closed_won` for WON stage — CRM-native names throughout, per Meta's recommendation to avoid colliding with the native `Lead` event.
- **action_source**: `"system_generated"` on all events.
- **custom_data**: `"lead_event_source": "Kalasar"` + `"event_source": "crm"` on every event; `value` + `currency` added on WON events (for ROAS reporting — Meta does not bid on value for Conversion Leads, see [Scenario C](#scenario-c--optimize-for-total-revenue-not-a-configuration-meta-offers)).
- **event_id**: deterministic `{deal_source_id}_{stage_raw}_{original_event_timestamp}` — stable across retries even when `event_time` is clamped.
- **event_time handling**: events ≤7 days old are sent as-is; events 7–90 days old are clamped to 6 days ago (respecting the lead-gen floor); events for Meta Leads older than 90 days are skipped entirely.
- **Eligible events**: only HIGH + MEDIUM attribution confidence (set by `attribution.py`) — LOW confidence (name-match only) is tracked for reporting but not forwarded to Meta.
- **Idempotency**: `CrmEvent.capi_sent` flag prevents resend; errors are recorded to `capi_error` without flipping the flag so retries happen naturally on the next sync cycle.

## Sources

**Note on verifiability:** Meta's primary docs (`facebook.com/business/help/*` and `developers.facebook.com/docs/*`) are JavaScript-rendered and cannot be fetched programmatically — only page titles confirm existence. Specific claims attributed to Meta below are sourced via third-party Meta Business Partner docs (LeadsBridge, Datahash, HighLevel, CustomerLabs, Privyr) that quote Meta's wording. Where sources disagree (e.g. 200 vs. 250 leads/month), both are flagged in-text. For bulletproof verification, open the Meta pages in an authenticated Ads Manager session.

- [Meta: Set Up Your CRM for Conversion Leads](https://www.facebook.com/business/help/279369167153556)
- [Meta Developers: Conversion Leads Integration](https://developers.facebook.com/docs/marketing-api/conversions-api/conversion-leads-integration/)
- [Meta Developers: Conversion Leads Payload Specification](https://developers.facebook.com/docs/marketing-api/conversions-api/conversion-leads-integration/payload-specification/)
- [Meta Developers: Conversions API for CRM Platforms](https://developers.facebook.com/docs/marketing-api/conversions-api/guides/conversions-api-crm-for-platforms)
- [Meta: About the Learning Phase](https://www.facebook.com/business/help/112167992830700)
- [Meta: Conversion Lead Rate (metric definition)](https://www.facebook.com/business/help/1369576410311305)
- [LeadsBridge: Conversion Leads Optimization on Facebook](https://leadsbridge.com/blog/conversion-leads-optimization-facebook/)
- [Driftrock: Meta Conversion Leads Optimization](https://www.driftrock.com/features/meta-conversion-leads-optimization)
- [HighLevel: Facebook Conversion Leads Walkthrough](https://help.gohighlevel.com/support/solutions/articles/48001233833-facebook-conversion-leads-walkthrough)
- [GLO: Optimising Meta Ads for HubSpot CRM Conversion Events](https://generateleads.online/optimising-meta-campaigns-for-hubspot-crm-conversion-events/)
- [One PPC: Meta Lead Ads quantity vs. quality (performance-goal options)](https://oneppcagency.co.uk/facebook-ads/facebook-metas-leads-ads-optimisation-lead-quantity-vs-quality/)
- [CustomerLabs: Conversion Lead Optimization Goal (30-day model-training window)](https://www.customerlabs.com/blog/conversion-lead-optimization-for-facebook-lead-ads/)
- [CustomerLabs: Meta Ads Value Rules Optimization](https://www.customerlabs.com/blog/meta-ads-value-rules-optimization/)
- [Leadsie: What are Facebook Datasets?](https://www.leadsie.com/blog/all-you-need-to-know-about-facebook-metas-new-datasets) (dataset ID = pixel ID)
- [Transcend Digital: Meta Datasets vs Facebook Pixel](https://transcenddigital.com/blog/pixel-vs-dataset/)
- [Privyr: Meta Conversions API for Lead Ads](https://help.privyr.com/knowledge-base/meta-conversions-api/)
- [LeadSync: The Facebook Pixel and Meta Lead Ads](https://leadsync.me/blog/facebook-pixel-meta-lead-ads/)
- [Meta Developers: Server Event Parameters (event_time constraints)](https://developers.facebook.com/docs/marketing-api/conversions-api/parameters/server-event/)
- [Datahash: Meta Conversion Leads Implementations (90-day lead retention)](https://www.datahash.com/docs/meta-conversion-leads-implementations/)
- [LiveRamp: The Meta Conversions API Program for Offline Conversions](https://docs.liveramp.com/connect/en/the-meta-conversions-api-for-offline-conversions.html)
