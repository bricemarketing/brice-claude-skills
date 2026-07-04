---
name: meta-catalog-audit
description: >
  Runs a Meta product catalog diagnostic audit for any ad account and catalog ID,
  and outputs a prioritised action table covering feed health, item-level issues,
  DA readiness, product set integrity, and pixel matching. Use this skill whenever
  the user asks to audit a Meta catalog, check feed health, diagnose DPA issues,
  or asks why Dynamic Ads are underperforming. Trigger on any mention of catalog
  audit, feed diagnostics, DPA health, product set issues, or pixel content_id
  mismatches — even if the user doesn't name the skill explicitly. Ask the user
  for the ad account ID and catalog ID if not already provided.
---
# Meta Catalog Audit Skill

## Purpose

Run a structured diagnostic against any Meta product catalog and produce a
**priority action table** as the sole output. No narrative report — actions only,
ranked by urgency, with issue description and owner.

---

## Step 0: Collect Inputs

Before running any tools, confirm:

1. **Ad account ID** — numeric string (e.g. `967833037474324`)
2. **Catalog ID** — numeric string (e.g. `258156000114611`)
3. **Feed ID** — if unknown, retrieve it via `ads_catalog_get_details` with
   `feed_limit=5` and use the first feed returned

If the user provides an ad account but no catalog ID, use `ads_catalog_get_catalogs`
to list available catalogs and ask the user to confirm which one to audit.

---

## Step 1: Run All Checks in Parallel

Fire all of the following tool calls simultaneously — do not sequence them:

### 1a. Catalog overview + feed list
`ads_catalog_get_details` → `catalog_id`, `feed_limit=5`

Captures: product_count (parent groups), product_set_count, feed IDs.

### 1b. Item-level diagnostics
`ads_catalog_get_diagnostics` → `catalog_id`, `limit=100`

Captures: all severity levels (must_fix, should_fix, opportunity), affected item
counts, affected_channels per issue.

### 1c. Dynamic Ads health
`ads_catalog_get_dynamic_ads_health` → `catalog_id`, `with_issue_only=false`

Captures: all 13 DA checks, pass/fail per check, severity.

### 1d. Feed details + latest upload
`ads_catalog_get_product_feed_details` → `feed_id`

Captures: schedule, source URL, deletion_enabled, latest upload result,
detected/persisted/invalid/deleted counts, warning_count.

### 1e. Upload session history (last 5)
`ads_catalog_get_product_feed_upload_sessions` → `product_feed_id`, `limit=5`

Captures: trend in item counts, deletion pattern, warning consistency.

### 1f. Feed transformation rules
`ads_catalog_get_feed_rules` → `feed_id`, `limit=100`

Captures: any server-side mapping, value, fallback, or regex rules applied at
ingestion. Zero rules = Adsmurai or similar partner handles transforms upstream.

### 1g. Product sample + total count
`ads_catalog_search_product` → `catalog_id`, `filter={}`, `limit=5`

Captures: total_count (variant-level), sample product fields (category, availability,
image_fetch_status, visibility), confirms parent/variant ratio.

### 1h. Out-of-stock count
`ads_catalog_search_product` → `catalog_id`,
`filter={"availability":{"eq":"out of stock"}}`, `limit=1`

Captures: total_count of out-of-stock variants.

### 1i. Broken image count
`ads_catalog_search_product` → `catalog_id`,
`filter={"images_fetch_status":{"eq":"fetch_failed"}}`, `limit=1`

Captures: total_count of items with failed image fetches.

### 1j. Product sets
`ads_catalog_get_product_sets` → `catalog_id`, `limit=20`

Captures: set names, product_count per set, filter_rule type (dynamic vs static).
Flag sets with product_count = 0.

---

## Step 2: Derive Findings

Apply this logic to the raw tool outputs before building the action table.

### Feed item count interpretation
- `catalog product_count` = parent product groups
- `search total_count` = active variants
- Ratio check: variants ÷ parents = avg variants/group. For fashion, 5–10 is normal.
  Flag if ratio < 2 (may indicate variant collapse) or > 20 (may indicate duplicate
  ingestion).

### Deletion pattern
- Compare `num_deleted_items` across the 5 upload sessions.
- If consistent non-zero deletions (e.g. 39/session), flag as active SKU churn —
  likely intentional promotional rotation but needs confirmation.
- If deletions are sporadic/escalating, flag as feed instability.

### `invalid_facebook_product_category`
- If affected item count ≈ total variant count → 100% of catalog affected. Likely
  a format issue (string label sent instead of numeric GPC ID), not a missing field.
  Do not claim the field is absent without verifying.
- `affected_channels: []` = no ad surfaces blocked. Severity is data quality, not
  delivery blocker.

### DA health failure
- `catalog_has_feed_upload_errors` failing = mirrors the feed warning above. It is
  a catalog-level check, not tied to a specific ad or creative. Do not report as
  "an ad failed."
- Any `must_fix` DA check failing = escalate to High urgency immediately.

### Product sets
- Count sets with product_count = 0.
- If > 20% of sets are empty, flag for cross-reference against live DPA ad sets.
- Static sets (filter on `retailer_product_group_id` lists) are more vulnerable to
  emptying from feed rotation than dynamic sets (filter on attributes like gender).

### Pixel issues
- `NO_CONTENT_ID`: pixel implementation gap — events firing without content_id.
  Affects retargeting pool and attribution. Owner = client tech team.
- `UNMATCHED_EVENTS`: content_id exists in pixel event but not in catalog. May be
  feed rotation lag or retailer_id format mismatch (parent ID vs variant ID).
- Neither is retrievable as a list via MCP. Do not claim you can provide the IDs.

---

## Step 3: Output — Priority Action Table Only

No preamble. No catalog overview section. No narrative. Output this table and nothing
else (unless the user asks for elaboration):

| # | Issue | Action | Owner | Urgency |
|---|---|---|---|---|

**Column definitions:**

- **Issue**: What is wrong and why it matters — specific, not generic. Include
  affected counts and impact on delivery where relevant.
- **Action**: Concrete next step. Verb-first. Specific enough that the owner can
  execute without asking follow-up questions.
- **Owner**: Who executes — one of: `[Agency]`, `[Feed partner]`, `[Client tech]`,
  `[Client + Feed partner]`, or a named tool/platform if obvious (e.g. `Adsmurai`).
- **Urgency**: `Critical` (delivery blocked), `High` (active signal/revenue loss),
  `Medium` (chronic quality issue, no current blocking), `Low` (informational).

**Ranking**: Sort by Urgency descending (Critical → High → Medium → Low). Within
same urgency, sort by impact on delivery first, then on measurement.

**Only include findings with evidence from the tool outputs.** Do not infer issues
not observed. If all checks pass cleanly, say so in one line and skip the table.

---

## Known MCP Limitations

State these only if the finding is relevant to a flagged issue:

- `fb_product_category` field value not returned by `ads_catalog_search_product` —
  cannot verify string vs numeric GPC ID format directly via MCP.
- Unmatched content_ids and missing content_ids are counts only — specific IDs
  require Events Manager UI or pixel event log export.
- `ads_catalog_get_diagnostics` returns affected item counts but not specific SKUs.
  Use `ads_catalog_search_product` with `error_type` parameter as best-effort
  (not all diagnostic types are filterable this way).
- Feed transformation rule reads are limited to Meta-side rules — upstream partner
  transforms (Adsmurai, etc.) are not visible.

---
