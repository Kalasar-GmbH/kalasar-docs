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

Switching from "Maximize leads" to "Maximize conversion leads" on an existing ad set counts as a **significant edit** that resets the learning phase. Meta needs ~50 conversion events within 7 days to exit learning. It's cleaner to create new campaigns.

### Datasets and Targeting

Datasets ARE used for targeting — they power custom audiences, lookalike audiences, and retargeting. This is why you use the EXISTING dataset:
- If you create a brand new dataset, all existing custom/lookalike audiences built from the old one become disconnected
- By adding CRM as a data source to the existing dataset, everything stays connected AND you add the quality signal

**The targeting change with Conversion Leads is not about the dataset — it's about the optimization goal.** When you set "Maximize conversion leads":
- Meta's algorithm learns which TYPES of people become qualified leads (not just which people submit forms)
- Cost per raw lead goes UP (Meta is more selective)
- Cost per QUALIFIED lead goes DOWN (~15-20% improvement per Meta's data)
- Lead volume decreases, quality increases significantly

### Prerequisites for Conversion Leads

- **Minimum volumes:** ~50 conversion events per week per ad set, ~200-250 leads per month total
- **Events must happen within 28 days** of lead generation (Meta's attribution window)
- **Conversion rate** for the target stage should be between 1% and 40%
- **Meta Lead ID** must flow from webhook to CRM (this is `leadgen_id`)
- **Events older than 7 days** are rejected by Meta (see [Event Time Windows](#event-time-windows--the-7-day-rule) below for the full story and mitigation)

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

## Sources

- [Meta: Set Up Your CRM for Conversion Leads](https://www.facebook.com/business/help/279369167153556)
- [Meta Developers: Conversion Leads Integration](https://developers.facebook.com/docs/marketing-api/conversions-api/conversion-leads-integration/)
- [Meta Developers: Conversion Leads Payload Specification](https://developers.facebook.com/docs/marketing-api/conversions-api/conversion-leads-integration/payload-specification/)
- [Meta Developers: Conversions API for CRM Platforms](https://developers.facebook.com/docs/marketing-api/conversions-api/guides/conversions-api-crm-for-platforms)
- [Meta: About the Learning Phase](https://www.facebook.com/business/help/112167992830700)
- [LeadsBridge: Conversion Leads Optimization on Facebook](https://leadsbridge.com/blog/conversion-leads-optimization-facebook/)
- [Driftrock: Meta Conversion Leads Optimization](https://www.driftrock.com/features/meta-conversion-leads-optimization)
- [HighLevel: Facebook Conversion Leads Walkthrough](https://help.gohighlevel.com/support/solutions/articles/48001233833-facebook-conversion-leads-walkthrough)
- [GLO: Optimising Meta Ads for HubSpot CRM Conversion Events](https://generateleads.online/optimising-meta-campaigns-for-hubspot-crm-conversion-events/)
- [Leadsie: What are Facebook Datasets?](https://www.leadsie.com/blog/all-you-need-to-know-about-facebook-metas-new-datasets)
- [Transcend Digital: Meta Datasets vs Facebook Pixel](https://transcenddigital.com/blog/pixel-vs-dataset/)
- [Privyr: Meta Conversions API for Lead Ads](https://help.privyr.com/knowledge-base/meta-conversions-api/)
- [LeadSync: The Facebook Pixel and Meta Lead Ads](https://leadsync.me/blog/facebook-pixel-meta-lead-ads/)
- [Meta Developers: Server Event Parameters (event_time constraints)](https://developers.facebook.com/docs/marketing-api/conversions-api/parameters/server-event/)
- [AdsFox: CRM CAPI — Optimize Ads for Qualified Leads, Not Form Fills](https://adsfox.com/crm-capi/)
- [Datahash: Meta CAPI for CRM (90-day lead retention)](https://www.datahash.com/docs/meta-capi-for-crm/)
- [LiveRamp: The Meta Conversions API Program for Offline Conversions](https://docs.liveramp.com/connect/en/the-meta-conversions-api-for-offline-conversions.html)
