# Attio Data Model for CRM Integration

## Overview

This document explains how Attio structures CRM data and what that means for integrating Attio with Kalasar's CRM connector. The key insight: Attio has two layers — **Objects** (the data) and **Lists** (the workflows) — and both can serve as "pipelines" with their own stages.

## Objects vs Lists

### Objects = Your Data

Objects are entity types: **People**, **Companies**, **Deals**, and any custom objects. Each object has attributes (fields) and records (rows). The Deals object comes with a built-in `stage` attribute (a status field).

Example — Kalasar's Deals object stages:
- Sales Qualified Lead
- Opportunity
- PoV (Proof of Value)
- Won 🎉
- Lost

**API**: `GET /v2/objects` lists all objects, `POST /v2/objects/{slug}/records/query` queries records.

### Lists = Your Workflows

Lists organize records from one object type into a business process view. A list is essentially a filtered, enriched view of records with **its own status attribute and custom fields**. In Attio's own words: "a pipeline isn't a separate module; it's just a list switched into kanban view."

The same Person record can appear in multiple lists. Each list tracks its own status independently — a person might be "New Lead" in the Meta Inbound list while simultaneously being an "Opportunity" in a different outbound list.

Example — Kalasar's "Meta Inbound" list (parent: People):

| Stage | Description |
|---|---|
| New Lead | Just came in from Meta |
| Contacted | First outreach done |
| Call #1 Booked | Discovery call scheduled |
| Call #2+ Booked | Follow-up calls |
| Pending Decision | Waiting for buyer decision |
| Won | Closed deal |
| Lost | Did not convert |

**API**: `GET /v2/lists` lists all lists, `POST /v2/lists/{list_id}/entries/query` queries entries.

### Key Difference

| | Objects (e.g. Deals) | Lists (e.g. Meta Inbound) |
|---|---|---|
| What it tracks | Entity records | Records in a workflow context |
| Stages belong to | The object globally | The specific list |
| Parent data | Self-contained | Links to a parent record (Person, Company) |
| Custom fields | On the object | On the list (separate from object fields) |
| Typical use | Company-wide deal tracking | Channel-specific or team-specific pipelines |
| Contact info | On the deal record itself | On the parent Person/Company record (one API call away) |

## Which One Is "The Pipeline"?

Both. It depends on how the customer uses Attio:

1. **List-based pipeline** (more common, recommended by Attio): A list like "Meta Inbound" with its own stages. This is the modern, flexible approach. Lists can have custom fields like `meta_campaign`, `meta_ad_name`, `lead_quality`, `mrr_pipeline_value` that are specific to that workflow. Best for founder-led sales, simple processes, or channel-specific pipelines (e.g. Meta ad leads, recruiting, VC deal flow).

2. **Deals-based pipeline**: The Deals object with its global stage attribute. More traditional CRM approach. Attio describes this as the ["scalable, robust" approach](https://www.attio.community/c/workshops-library/lists-vs-deals-how-to-build-your-sales-pipelines) for dedicated sales teams. Supports multiple concurrent deals per customer (new business, upsell, renewal) — something lists cannot do (a person can only be in one stage per list).

3. **Both**: Some teams use a list for lead qualification (Meta Inbound → New Lead → Contacted → Call Booked) and then create a Deal when the lead becomes a real opportunity (Deal → Opportunity → PoV → Won). The list tracks the top of funnel, deals track the bottom.

Attio's own guidance: start with a list for simple use cases; move to the Deals object when you need multiple opportunities per customer or complex sales reporting.

For Kalasar's CRM connector, **both should be offered as pipeline options**. The user picks their pipeline, and we fetch the stages from wherever that pipeline lives.

## Data Structure: List Entries

A list entry contains:
- `entry_id` — unique ID for the entry
- `parent_record_id` — the Person/Company record this entry represents
- `parent_object` — which object type (e.g. "people")
- `created_at` — when the entry was added to the list
- `entry_values` — all list-specific field values, including:
  - `status` — the current stage, with `active_from` timestamp (tells you WHEN the stage changed)
  - Custom fields like `meta_campaign`, `meta_ad_name`, `company_name`, `mrr_pipeline_value`, etc.

To get contact info (email, phone, name), you need a second API call to fetch the parent record:
```
GET /v2/objects/people/records/{parent_record_id}
```

This returns `name` (first_name, last_name), `email_addresses`, `phone_numbers`.

## Data Structure: Deal Records

A deal record contains everything in one place:
- `name` — deal name
- `stage` — current stage with `active_from` timestamp
- `value` — deal value (currency field)
- `associated_people` — linked person records
- `associated_company` — linked company record
- Contact info requires fetching the associated person record separately

## OAuth Scopes Required

| Scope | What it unlocks |
|---|---|
| `record_permission:read` | Read records (People, Companies, Deals) |
| `object_configuration:read` | Read object schemas, attributes, stage definitions |
| `list_configuration:read` | Read list schemas, attributes, stage definitions |
| `list_entry:read` | Read entries within lists |

**Important**: Attio grants **no default scopes**. If scopes are empty, the token has zero access. All four scopes above must be explicitly requested in the OAuth flow.

## API Endpoints Reference

### Pipeline Discovery
- **Lists**: `GET /v2/lists` → returns all lists with their names and parent objects (requires `list_configuration:read`)
- **Deals**: `GET /v2/objects/deals` → always exists as a built-in object (requires `object_configuration:read`)

### Stage Discovery
- **List stages**: `GET /v2/lists/{list_id}/attributes/status/statuses` → returns all status options for a list
- **Deal stages**: `GET /v2/objects/deals/attributes/stage/statuses` → returns all stage options for deals

### Data Sync
- **List entries**: `POST /v2/lists/{list_id}/entries/query` → paginated, filterable by `created_at`
- **Deal records**: `POST /v2/objects/deals/records/query` → paginated, filterable by `created_at`
- **Person details**: `GET /v2/objects/people/records/{record_id}` → name, email, phone

### Timestamp Format

Attio requires timestamps in nanosecond-precision UTC format with Z suffix:
```
2026-02-16T12:36:32.000000000Z
```
Python's `datetime.isoformat()` outputs `+00:00` which Attio rejects with a 400 error. Use `strftime("%Y-%m-%dT%H:%M:%S.000000000Z")` instead.

## Implications for Kalasar's CRM Connector

1. **Pipeline selector** should show both Lists and "Deals" as options
2. **Stages** are fetched dynamically based on the selected pipeline type
3. **Sync source** differs:
   - List selected → sync from `/v2/lists/{id}/entries/query`, contact info from parent record
   - Deals selected → sync from `/v2/objects/deals/records/query`, contact info from associated people
4. **Stage change tracking**: Both entry statuses and deal stages have `active_from` timestamps — this tells us when a stage change happened, which is what we send to Meta CAPI
5. **List entries have richer context**: Fields like `meta_campaign` and `meta_ad_name` could improve attribution matching beyond just email/phone matching

## Live Data: Kalasar's Attio Workspace

As of April 2026:

**Lists:**
- **Meta Inbound** (People, 24 entries) — active, primary pipeline
- **Meta Pipeline** (Companies) — archived

**Deals:** 10+ records with stages Sales Qualified Lead, Opportunity, Lost. Previously used, now dormant.

**Meta Inbound stages**: New Lead → Contacted → Call #1 Booked → Call #2+ Booked → Pending Decision → Won / Lost

## Sources

### Attio Official Documentation
- [Objects and Lists](https://docs.attio.com/docs/objects-and-lists) — core data model explanation
- [Define your data model: objects, lists, and views](https://attio.com/help/reference/attio-101/attios-data-model/define-your-data-model-objects-lists-and-views) — Attio Help Center overview
- [Understanding lists](https://attio.com/help/reference/attio-101/attios-data-model/understanding-lists) — list concepts and usage patterns
- [Connect an app through OAuth](https://docs.attio.com/rest-api/tutorials/connect-an-app-through-oauth) — OAuth setup tutorial with multiple redirect URIs
- [Authentication guide](https://docs.attio.com/rest-api/guides/authentication) — scopes reference, token types
- [Filtering and sorting](https://docs.attio.com/rest-api/guides/filtering-and-sorting) — query filter syntax (`$gte`, `$lt`, etc.)

### Attio API Endpoint Reference
- [List all lists](https://docs.attio.com/rest-api/endpoint-reference/lists/list-all-lists) — `GET /v2/lists`
- [List entries (query)](https://docs.attio.com/rest-api/endpoint-reference/entries/list-entries) — `POST /v2/lists/{list}/entries/query`
- [List attribute values for entry](https://developers.attio.com/reference/get_v2-lists-list-entries-entry-id-attributes-attribute-values) — per-entry attribute access
- [List person record entries](https://docs.attio.com/rest-api/endpoint-reference/people/list-person-record-entries) — find which lists a person belongs to

### Attio Community & Guides
- [Lists vs Deals: How to build your sales pipelines](https://www.attio.community/c/workshops-library/lists-vs-deals-how-to-build-your-sales-pipelines) — Attio's own workshop on when to use which
- [How to Use Lists vs. Deals in Attio](https://supademo.com/blog/guides/how-to-use-lists-vs-deals-in-attio/) — step-by-step comparison
- [Mastering Lists and Views on Attio](https://novlini.io/blog/mastering-lists-and-views) — deep dive on list configuration

### Third-Party References
- [CRM.org: Attio Review 2026](https://crm.org/news/attio-review) — "a pipeline isn't a separate module; it's just a list switched into kanban view"
- [ClonePartner: The Ultimate Guide to Attio CRM](https://clonepartner.com/blog/ultimate-guide-attio-crm-2025) — comprehensive platform overview
- [Stacksync: Attio API Guide](https://www.stacksync.com/blog/how-to-get-an-attio-developer-account-and-start-working-with-attios-api) — developer account setup and authentication walkthrough
