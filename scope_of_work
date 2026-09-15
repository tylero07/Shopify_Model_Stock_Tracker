# Shopify → Google Sheets Inventory Reporting

Implementation brief · September 15, 2026

Based on **Shopify Spreadsheet Sync Brainstorm**. Business requirements below come from that conversation; technical defaults are proposed implementation choices. Confirm the short configuration checklist before a live rollout. Build deterministic Python; no AI integration is needed.

## 1. Goal and scope

Give two or three internal users a shared, daily report of high-stock items so the team dressing models can prioritize products that need exposure. Show product names, variant/SKU details, quantities, storefront links, and product photos. Preserve a ledger showing items entering the report, growing, declining, and dropping below the threshold.

**MVP:** one Shopify store, one Google spreadsheet, one-way reporting, configurable threshold, product images, daily comparisons, local execution, and durable local history. Shopify remains the source of truth. A nontechnical operator should be able to run a documented launcher and open the sheet.

**Deferred:** Shopify inventory writes, bidirectional sync, orders/customer data, sales forecasting, AI summaries, category-specific thresholds, real-time webhooks, and a custom web interface. Google Sheets is the shared output; Excel export is optional later.

## 2. Reporting rules and data model

- One report row per variant. Use `(shop, variant_id)` as identity; SKUs can be blank, duplicated, or changed.
- Proposed threshold rule: `available_quantity >= threshold`, with an illustrative default of 15. Make the operator configurable if the business intends strictly “above.”
- Sum **available** units across explicitly selected locations. Confirm whether the team instead wants on-hand units, a single warehouse, or product totals across sizes. Never combine different inventory quantity types.
- Include active, online-store products by default; log exclusions and make eligibility configurable. Exclude untracked inventory from threshold decisions and report it as a data-quality warning.
- Compare against the previous successfully published reporting day, not an assumed yesterday. Show the comparison date; quantity changes are net inventory movement, not proof of sales.
- First run labels eligible rows `BASELINE`. Later crossings from below threshold are `NEW`; re-entry after a prior qualifying period is `RETURNED`. Other qualifying rows are `UP`, `DOWN`, or `UNCHANGED`, with a numeric delta.
- A previously qualifying item now below threshold receives `DROPPED`, moves to **Recently Dropped**, and stays visible until the next successful reporting day. Same-day retries do not age it out. History retains the event permanently.
- Missing, deleted, archived, unpublished, or untracked items receive an explicit reason such as `NO_LONGER_ELIGIBLE` or `MISSING`; do not silently interpret missing inventory as zero.
- Record the threshold, location selection, and rules version with each snapshot. A rule change establishes a labeled new baseline rather than generating misleading stock-movement events.

Normalized record: shop, product/variant/inventory-item IDs, SKU, product and variant titles, product status, storefront URL, location quantities, aggregate quantity, ordered image URLs, preferred image, and extraction timestamps. Store UTC timestamps and display business-local dates.

## 3. Architecture and data flow

```text
Manual launcher / daily scheduler
  → validate configuration and acquire run lock
  → authenticate Shopify and start or resume export
  → download JSONL and normalize products, variants, inventory, media
  → validate completeness and calculate report against saved baseline
  → save pending snapshot and deterministic publication payload
  → publish Google Sheets and verify run marker
  → commit successful baseline/history and release lock
```

Use a small Python package with `httpx`, `google-auth`, `google-api-python-client`, standard-library SQLite, and `pytest`; pin dependencies. Keep API adapters separate from pure reporting logic. SQLite holds snapshots, events, run status, bulk-operation IDs, and pending publication data. No Firestore is needed locally.

Suggested modules:

| Module | Responsibility |
| --- | --- |
| `config.py` | Validate store, API version, sheet ID, threshold, locations, timezone, paths |
| `shopify_auth.py` | Obtain/cache/renew scoped app tokens |
| `shopify_client.py` + `queries/` | Versioned GraphQL, export submission, polling, downloads |
| `normalize.py` | Join JSONL objects by IDs and parent IDs; normalize quantities/media |
| `rules.py` | Eligibility, threshold, deltas, transitions; pure functions |
| `state.py` | SQLite migrations, snapshots, events, locks, recovery records |
| `sheets_writer.py` | Typed cells, formatting, staging, publication, verification |
| `runner.py` + `__main__.py` | Orchestration, CLI, structured logs, exit codes |
| `tests/fixtures/` | Synthetic exports and expected multi-day reports |

## 4. API and authentication strategy

### Shopify

Use the official Admin GraphQL API at `https://{shop}.myshopify.com/admin/api/{version}/graphql.json`; pin a supported stable version and validate queries against it. Request only required read scopes: initially `read_products`, `read_inventory`, and `read_locations`, subject to field-level validation. The bulk-query submission is a GraphQL mutation but does not modify inventory.

Install an app owned by the business. If app and store belong to the same Shopify organization, use the client-credentials grant and renew its approximately 24-hour access token automatically. Otherwise use the supported app installation/OAuth flow with offline access and implement the selected token type's refresh behavior. Do not assume a permanent token is available from the dashboard. No staff passwords, browser sessions, or scraping. See [Shopify authentication](https://shopify.dev/docs/apps/build/authentication-authorization/access-tokens) and [client credentials](https://shopify.dev/docs/apps/build/authentication-authorization/access-tokens/client-credentials-grant).

### Google

Enable the Sheets API in a business-owned Google Cloud project. Have the business create the spreadsheet and share only that file with a dedicated service-account email as editor. Use the Sheets OAuth scope; avoid Drive permissions unless a later feature needs them. Service-account access follows granted resource permissions; domain-wide delegation is unnecessary. See [Google credentials](https://developers.google.com/workspace/guides/create-credentials).

For local development, prefer service-account impersonation where available; otherwise keep a restricted service-account credential file outside the repository and reference its path. In Cloud Run, use the attached service account with Application Default Credentials and no downloaded key.

## 5. Shopify GraphQL and bulk-operation notes

Target one daily bulk export, **not literally one HTTP request**. Submission, status checks, authentication, and downloading add requests. Bulk query execution avoids normal query-cost accounting, while submission and polling remain rate-limited. GraphQL cost points are a throttle budget, not dollar amounts.

Bulk queries permit one top-level field, at most five connections, and at most two nested connection levels. API versions from `2026-01` allow five concurrent bulk queries per app/shop; this tool should run one at a time. Download JSONL and reconstruct relationships using IDs and `__parentId`. Never publish partial results. See [bulk-operation documentation](https://shopify.dev/docs/api/usage/bulk-operations/queries).

Prototype a `products` export containing variants, inventory-item location levels/available quantities, and product media. Validate the exact nesting and supported fields before committing to one export; split inventory and product/media exports and join by IDs if necessary. Use cursor pagination for a small diagnostic sample; ordinary connections generally page up to 250 items and query cost grows with requested structure. Inspect throttle metadata rather than assuming a fixed daily quota. See [API limits](https://shopify.dev/docs/api/usage/limits).

Fetch the full product media connection, keep image media in source order, and prefer a variant-associated image for its thumbnail with a product-image fallback. Use Shopify's online-store URL when present. Admin media may include assets the storefront theme does not display; verify a few real products before claiming exact storefront parity. See [Product fields](https://shopify.dev/docs/api/admin-graphql/latest/objects/Product).

## 6. Google Sheets output

| Tab | Contents |
| --- | --- |
| **Current Overstock** | Status, thumbnail, product, variant, SKU, quantity, previous quantity, delta, threshold, product link, image count, first qualifying date, last observed timestamp |
| **Recently Dropped** | Previous/current quantities, delta, reason, links/photos, transition date; one successful reporting-day visibility |
| **History** | One daily record per currently/prior qualifying variant, including drops; date, stable IDs, quantities, event, threshold, rule version, run ID |
| **Photos** | One row per product image: product ID, title, source order, image URL, optional preview; avoids fixed image-column limits |
| **Run Status** | Last success, last attempt, comparison date, timezone, selected locations, counts, warnings, rules version, run ID |

Freeze headers, enable filters, wrap names, use fixed thumbnail sizes, and sort high quantities first. Highlight new/returned rows and drops; retain readable labels so color is not essential. Use one thumbnail per report row plus all image links in Photos. Missing images must not block publication.

Write product text/SKUs as literal strings, preserving leading zeros and preventing formula injection. Generate only controlled image/link formulas. Keep human notes in a separate manually maintained tab keyed by variant ID.

Build output before touching live tabs. For a small report, use a single atomic `spreadsheets.batchUpdate` with typed cell updates, old-row clearing, formatting, and the success marker. If the payload grows, upload hidden staging tabs in chunks and publish via a final atomic batch that copies staged data into stable live tabs. Never clear the live report first. Batch writes and use bounded backoff; Google recommends roughly 2 MB request payloads. Atomicity applies per request, not across multiple calls. See [Sheets limits](https://developers.google.com/workspace/sheets/api/limits).

## 7. Scheduling, reliability, and security

- Start with manual execution; add the operating system's daily scheduler after validation. Proposed schedule: 06:00 in the business's confirmed timezone. A local computer must be awake and online; document missed-run recovery.
- Enforce one active run. Reuse a persisted bulk-operation ID after interruption; reconcile ambiguous submission responses before submitting another export.
- Retry transient network errors, throttling, and server failures with exponential backoff/jitter and a fixed deadline. Honor retry hints. Inspect HTTP errors, GraphQL `errors`, and mutation `userErrors`; stop on invalid scopes/query/configuration.
- Validate completed exports, required IDs, quantity types, joins, and plausible counts. Distinguish a valid empty report from an empty or truncated source export; require review for unexplained large source-count loss.
- Use a deterministic `(shop, report_date, rules_version)` daily key. Default same-day reruns resume or skip; explicit refresh replaces that day's data and compares to the prior successful day. History must not duplicate on retries.
- Save a pending publication before writing Sheets. If a response is lost, read its run marker to reconcile; commit the local baseline only after publication is confirmed. Recover a crash between Sheets success and local commit from the pending record.
- On failure preserve the last good report and baseline, exit nonzero, and show an actionable local message. Record the failed attempt in Run Status if reachable. Add owner failure alerts during cloud hardening; leave last-success timestamps unchanged.
- Ignore credentials, `.env`, databases, exports, and logs in Git. Supply only placeholder configuration. Redact tokens and signed download URLs; keep production inventory out of fixtures and public repositories. Restrict spreadsheet sharing and set snapshot/log retention with the business.

## 8. Local-first phases and cloud migration

1. **Fixture-based skeleton:** implement configuration, normalized models, SQLite state, rules, and a dry-run report without credentials.
2. **Live vertical slice:** authenticate, validate a sample query, complete an export, and publish a test spreadsheet.
3. **Usable local MVP:** all photos, multi-day transitions/history, formatting, recovery, launcher, and a short operator README. The brainstorm's five-hour target is a prototype aspiration; access setup and reliability work may require more time.
4. **Short business trial:** reconcile sample SKUs with Shopify, confirm inventory meaning and threshold behavior, and gather feedback from the two or three users.
5. **Cloud hardening:** containerize the same runner; use Cloud Run Jobs, Cloud Scheduler, Secret Manager, and Cloud Logging. Use an authenticated scheduler identity authorized to execute the job. See [scheduled Cloud Run jobs](https://docs.cloud.google.com/run/docs/execute/jobs-on-schedule).

Cloud Run's local filesystem is not durable state. Before migrating, replace the SQLite adapter with Firestore (snapshots, events, and transactional run lease) or another durable store; migrate the successful baseline and history, then disable the local schedule. Add failure/staleness alerts and billing budgets. Long exports can later use completion webhooks and a separate continuation job. No always-running server is required for the polling MVP.

Local execution avoids hosting charges, but still needs internet and existing services. Do not promise zero total operating cost; review current provider pricing and quotas when deploying.

## 9. Testing and acceptance

- Unit tests: threshold equality, negative/zero stock, multiple locations, untracked inventory, duplicate/missing SKUs, missing photos, rule changes, and every status transition.
- Multi-day fixture: baseline → new/up/down → dropped → removed from Recently Dropped → returned. Repeat a date and inject a failed day; confirm history uniqueness and the correct comparison date.
- Adapter tests: JSONL parent joins, malformed/partial exports, GraphQL errors, throttling, token renewal, expired download links, and a legitimate empty report.
- Recovery tests: interrupted bulk export, ambiguous Sheets success, crash before local commit, concurrent launch, and staging failure leave a recoverable state and last good output.
- Live acceptance: use a test sheet; manually reconcile representative variants and all their images, confirm no Shopify write scopes, verify rerun behavior, and have a nontechnical user run the launcher and identify an overstock item from the sheet.

## 10. Concrete Codex starter tasks

- [ ] Scaffold `src/inventory_report/`, `tests/`, `pyproject.toml`, `.gitignore`, placeholder `.env.example`, and operator README.
- [ ] Add validated settings for store, pinned API version, spreadsheet ID, threshold/operator, location IDs, timezone, auth mode, state path, and timeouts.
- [ ] Create synthetic multi-location/product/media fixtures and normalized dataclasses.
- [ ] Implement pure threshold/transition logic and multi-day tests first.
- [ ] Implement SQLite migrations, daily keys, pending-publication recovery, and run lock.
- [ ] Add `python -m inventory_report run --fixture tests/fixtures/catalog.jsonl --dry-run`; produce inspectable rows without API writes.
- [ ] Implement Shopify authentication and validate the intended export query against the chosen version/store.
- [ ] Implement resumable bulk export, streaming JSONL normalization, and completeness checks.
- [ ] Implement Google authentication, tab setup, typed batch output, formatting, and publication verification against a test sheet.
- [ ] Wire the full runner; add fault/retry tests and live sample reconciliation.
- [ ] Create the appropriate local launcher and scheduler instructions; document configuration changes and recovery.
- [ ] After business acceptance, add container/deployment configuration, durable cloud state, scheduler, secrets, and alerts.

**Configuration to obtain before live setup:** store/app ownership and installation access; target spreadsheet and Google project; available versus on-hand quantity; included locations; variant versus product aggregation; threshold and comparison operator; product eligibility; business timezone; operator OS; and retention period. Proceed with fixtures and the stated defaults while these are unresolved.
