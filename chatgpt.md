# Boardwalk Land Use Intelligence API

Boardwalk turns public city and county records into two datasets: planned projects (from
planning filings: what is proposed, where, by whom, and what was decided) and building
permits (from building departments; Utah first, more areas through the end of 2026).

**Base URL:** `https://api.boardwalkai.com/api/v1`
**Documentation:** https://boardwalkai.com/docs/api/

## Read this before you spend anything

- **County, State, and National: every plan includes 100,000 export records a month, the same number on a county plan, a state plan, and National. 1 export credit per new or changed record delivered through the API, MCP, CSV, or the app. An unchanged record already delivered is free. Counts are free. Find Contact Info is separate: 1 credit per project the first time this account receives email, phone, or a personal LinkedIn profile. Standalone API Only: 1 credit per delivered record; Find Contact Info is included on records already purchased. Free trial: 50 record-export credits (search, CSV, MCP) and 5 Find Contact Info lookups, counted separately. Empty contact lookups cost nothing.**
- **`GET /projects/count` is free and takes the same filters as the search route.** Call it
  first (for building permits, `GET /permits/count` does the same). It does not deliver a record, so it spends neither counter. On County, State, and
  National, each new or changed record then counts toward the shared monthly export cap;
  Find Contact Info is a separate credit. On API Only and the free trial, new or changed
  record deliveries use record credits. A one-state plan is that state; a county plan those
  counties; National and API Only are nationwide. `GET /locations/states` marks each state
  `entitled`. Empty contact lookups cost nothing. Hitting the export cap returns
  `429 export_limit_reached` and charges nothing.
- **On API Only, a paid page is delivered whole or refused whole, never trimmed.**
  Each page is billed whole while it fits the balance; the first page that does not fit
  is refused with `402 insufficient_credits`, a quote, and no charge. A free-trial page
  can be shortened by the trial balance; check `meta.trialTruncation`.
- **A 402 tells you how to recover.** `quote.affordableRecordCount` is the `limit` to retry
  with (it buys what the balance still affords), and `quote.countUrl` re-prices the set
  for free so you can narrow. A `credits_exhausted` refusal carries no quote: the balance
  cannot pay for even one record.
- **Errors and zero-result queries cost nothing.** Neither does re-receiving a record the
  account already holds unchanged.

## Authentication

```
Authorization: Bearer bwk_live_YOUR_KEY
```

Keys beginning `bwk_live_` read the production corpus and can spend credits.
`bwk_test_` is reserved for a separate sandbox environment that is **not serving
data yet**. A test-prefix key is refused with `503 sandbox_unavailable` on
every route today, including the free ones, so use a `bwk_live_` key for
evaluation. Keys issued before the Boardwalk rename
begin `csk_` and are still accepted. A web session or login token is
**not** a credential for this API.

## Writes need an Idempotency-Key

`POST /enrichments` and `POST /alerts/{alertId}/run` **require** an
`Idempotency-Key` header. Without one they return `400 invalid_request` with
`error.param: "Idempotency-Key"` and nothing happens. Use any opaque string up
to 255 characters of `A-Z a-z 0-9 _ . : -`, one per distinct request. Re-sending
the same key replays the original result and charges nothing extra. That is the
safe way to retry. Reusing one key for a *different* request is `409
idempotency_key_reuse`.

## Connect Boardwalk to ChatGPT

Add Boardwalk once, install it, then select it in Chat or Work.

Server URL: `https://mcp.boardwalkai.com/mcp`. Use OAuth. No API key.

Provider instructions checked October 4, 2026.

### Personal account

Developer mode is documented for Plus, Pro, Business, Enterprise, and Education on the web. Your workspace may restrict it.

1. On the web, open Settings → Security and login and enable Developer mode.

2. Open ChatGPT Plugins → +. Name it Boardwalk and paste the server URL. Choose OAuth and automatic/dynamic client registration if offered, then create it. Leave optional client ID and secret blank.

3. Open Boardwalk in your personal Plugins directory, install it with the + button, and sign in to Boardwalk when prompted.

4. Start a new Chat or Work conversation. Select Boardwalk from the tools menu, or type @Boardwalk where available.

### Company account

Workspace policy controls who can add, publish, install, or use plugins. Ask your admin if Boardwalk is missing.

1. A workspace admin adds Boardwalk using the Personal account steps, with the server URL and OAuth.

2. In ChatGPT Plugins → Personal, the admin opens Boardwalk’s three-dot menu → Publish and chooses which workspace roles can use it.

3. Members open the workspace Plugins directory, install Boardwalk, and sign in with their own Boardwalk account. Select it in a new Chat or Work conversation.

[ChatGPT setup guide](https://developers.openai.com/plugins/quickstart)

[Workspace setup](https://developers.openai.com/plugins/build/plugins)

[Developer mode availability](https://developers.openai.com/api/docs/guides/developer-mode)

[Workspace permissions](https://learn.chatgpt.com/docs/enterprise/apps-and-connectors)

## Using Boardwalk from an agent

- Boardwalk records what the public filing stated, the same facts someone would have
  after reading that agenda. Absent fields were not in the filing. That is expected for
  early pre-construction intel, not an error, not “missing data,” and not a failed
  extraction. Tell the user what is known. Do not open with “this looks wrong.” If two
  fields disagree, say what each one says.
- Search like the map: `q` on `count_projects` / `get_projects` is the map search box
  (project name, developer, words from the filing). Combine it with Region, Type, Date
  Range, Acres, Project Size, Entitlement Status, Application Type, Govt. Decisions,
  and overlays. Use `ai_search_projects` only when they need evidence from the minutes.
- List tools return **project details** (name, headline, summary, status, place, party names
  without emails, size). Do not request `representation=full` or `get_document`
  in a loop.
- Always `count_projects` (or the named preflight) before hydrating. Chat pages stay
  small (about 25, never more than 50 in one result). If they want to review or rank
  in chat and `matchCount` is over 250, tell them the number and either tighten
  filters or work in batches of 250. If they said CRM, CSV, or email (including
  after a pull they further filtered), use `create_export` with that `alertId` /
  `listId` (same extra filters) or the `projectIds` you kept. Never paste CSV rows
  into chat.
- First session: `get_preferences`, then ask types + geography. Say once: counts are
  free; on County, State, and National each new or changed record (planned project or
  permit) uses 1 of the 100,000 export records a month, and an unchanged one already
  delivered does not; API Only uses 1 credit per new or changed record; the free trial has
  50 records in total, planned projects or permits in any mix. Find Contact Info uses 1
  credit per project on geographic plans, is included on purchased API Only records, and
  has a separate 5-lookup trial grant. Empty fields mean the filing or permit record Boardwalk could read did not say it.
- When they name something they already saved, `list_alerts` / `list_user_lists` then
  `get_alert_matches` (no deliveryId) or `get_list_results`. Pull every match except
  dismissed. Pending review stays in; helping manage those is a main use of this MCP.
  Say the name, how many are already on it, and how many are still pending review. If
  the name is unclear, or “leads” could mean a saved set or a new search, ask once.
  Do not search the whole database.
- “Find / search / like X in Y” with no saved name is `count_projects` then `get_projects`.
- “Filter these / of those / among them / 2+ acres / just the offices” after a pull
  stays inside those ids: extra filters on the same pull tool (`get_alert_matches` or
  `get_list_results`). If the set is too large for chat, tighten that way first, then
  pull the subset. Never drop the set and search again.
- Do not call `run_alert` unless they asked to run it now. Do not mention deliveries
  or delivery ids to the user.
- Confirm `create_enrichment` in the transcript before starting a job. Confirm
  `delete_alert`: it deletes for every member of a shared alert, and only the
  alert's owner or the team admin may call it (other members get a 403).
- On County, State, and National plans, searching, viewing, and exporting stay inside the
  plan geography, and each new or changed record they deliver uses 1 export record;
  re-reading a record already delivered unchanged does not. Find Contact Info uses **1
  credit per project** when it first delivers email or phone. Trial and API Only record
  pulls use 1 credit per new or changed record. Empty contact lookups cost nothing.
  Counting, project types, locations, and alert configuration are free. On County, State,
  and National, `get_credit_balance` reports the record allowance as `exportCredits`;
  its `creditsRemaining` is Find Contact Info.
- Building permits have their own tools, each the twin of a planned-project one
  (`count_permits`, `search_permits`, `get_permit`, `analyze_permits`,
  `search_permit_contractors`, permit alerts, `add_permits_to_list`,
  `create_permit_export`). Planned work (rezones, plats, site plans, approvals) is planned
  projects; building work being done (roofs, pools, new homes, who pulled the permit) is
  permits. When either could answer, count both and give both numbers. Never send a
  permit id to a project tool or the reverse.
- Permit alerts are separate from planned-project alerts (`palr_` ids, not `alr_`). For
  "pull my <name> alert", check `list_alerts` and `list_permit_alerts` and pull from
  whichever holds it. A permit alert pull is priced like `search_permits`.
- `describe_data` lists questions this corpus cannot answer (denial rates, time-to-
  entitlement, project value). If the question is there, say so and stop.
- You can connect without a paid Boardwalk plan. Upgrading does not require reconnecting.
  If a tool result says an active Boardwalk trial or plan is needed, relay that
  sentence verbatim. Do not tell them to reconnect. Do not invent a `bwk_live_` key as
  the MCP fix.

## Endpoints

| Endpoint | Cost | What it does |
| --- | --- | --- |
| `GET /projects/search` | Plan dependent | County, State, and National: 1 export record per project that is new to this account or changed since the last delivery, against the shared 100,000 a month. API, MCP, CSV, and the app share that counter. An unchanged record already delivered is free. `format=csv` queues the full filter match and returns HTTP 202. Find Contact Info is a separate allowance. Standalone API Only and the free trial: 1 credit per new or changed record; unchanged owned records are free. API Only includes Find Contact Info on records already purchased. |
| `GET /projects/{id}` | Plan dependent | County, State, and National: 1 export record the first time this account receives the project, against the shared 100,000 a month. A later read of the same unchanged project is free. Email, phone, and LinkedIn we already hold ship with the record. Find Contact Info looks for the ones we do not, which spends the contact allowance, not export records. Standalone API Only and the free trial: 1 credit the first time, which also delivers the contacts we hold. API Only includes Find Contact Info after the record is purchased. |
| `GET /projects/sync` | Plan dependent | County, State, and National: 1 export record per new or changed project, against the shared 100,000 a month. Unchanged records bill 0. Standalone API Only and the free trial: 1 credit per new or changed record. |
| `POST /projects/ai-search` | Free | **Not part of the public API. You cannot call this with an API key.** Any `bwk_live_`/`csk_` key is refused with `400 invalid_request` (`error.param: "endpoint"`, `details.asyncEndpoint`) before anything is parsed; use `POST /projects/ai-search/jobs` instead. On the session-authenticated web product this path is free, like structured search. |
| `POST /projects/ai-search/jobs` | Plan dependent | County, State, and National: confirmed new or changed matches count toward the shared 100,000 export records a month. Rejected candidates do not. Find Contact Info on those records is a separate job. Standalone API Only and the free trial: credits are reserved at 1 per requested record and settled against confirmed matches. API Only includes Find Contact Info after the record is purchased. |
| `GET /documents/{id}` | Plan dependent | County, State, and National: 1 export record the first time this account receives the document, against the shared 100,000 a month. A later read of the same document is free. Standalone API Only and the free trial: 1 credit the first time. |
| `GET /alerts/{alertId}/matches` | Free | Free, every time. Omit deliveryId to read every match except dismissed (pending review included), the same set the website table shows. Passing a deliveryId re-reads a past API run. Neither path charges, and neither does include=contacts here: it reads what the alert's own lookups already found. |
| `GET /projects/count` | Free | Free, always. A count does not deliver a project record, so it does not use export records or Find Contact Info credits. On API Only and the trial it also prices the set. |
| `GET /projects/ai-search/jobs/{id}` | Free | Polling is free. The records the job delivers were charged when the job settled, not when you read them. |
| `DELETE /projects/ai-search/jobs/{id}` | Free | Cancelling is free and gives credits back, never takes them: a queued job refunds its full reservation immediately; a running job settles at the next batch boundary, charging only confirmed records and refunding the rest. |
| `GET /account/credits` | Free | Free. Your plan, your Find Contact Info allowance, and exportCredits (limit, used, remaining, resetsAt) for the shared 100,000 export records a month. |
| `POST /enrichments` | Plan dependent | County, State, and National: 1 credit per project the first time this account receives email, phone, or a personal LinkedIn profile for it. Standalone API Only: included on records this pool has already purchased. Unpurchased ids are skipped as `notPurchased`. A live free trial spends the separate 5-lookup grant (not the 50 record-export credits). Empty lookups cost nothing, and so does a lookup that only returns contacts this account could already see on the project. The maximum is held when queued, excluding already-unlocked projects; completion charges qualifying deliveries and releases the rest. Company links and names alone do not qualify. |
| `GET /enrichments/{id}` | Free | Free to poll: meta.creditsUsed stays 0. contactBilling reports the cumulative reserved, charged, released, refunded, and qualifyingProjects counts; repeated polls do not charge again. |
| `GET /health` | Free | Free and unauthenticated. A **liveness probe**, not part of the record contract: its response shape carries no schema guarantee and can change without a schema-version bump, which is why it is deliberately absent from the OpenAPI document. Do not generate a client against it or parse its body. |
| `GET /taxonomy` | Free | Free. The complete building type tree. |
| `GET /taxonomy/project-types` | Free | Free. |
| `GET /taxonomy/subtypes` | Free | Free. |
| `GET /taxonomy/asset-classes` | Free | Free. |
| `GET /taxonomy/action-types` | Free | Free. |
| `GET /taxonomy/action-categories` | Free | Free. |
| `GET /locations/states` | Free | Free. Resolve the ids you filter with. |
| `GET /locations/states/{id}` | Free | Free. |
| `GET /locations/states/{id}/counties` | Free | Free. |
| `GET /locations/states/{stateId}/counties/{countyId}/cities` | Free | Free. |
| `GET /locations/counties/{id}` | Free | Free. |
| `GET /locations/cities/{id}` | Free | Free. |
| `GET /analytics/datasets` | Free | Free. GET only. What the analytics plane can and cannot answer, self-described. |
| `POST /analytics/aggregate` | Free | Free. POST only (GET returns 404/405). Returns counts and group keys, never a project id, name, address, contact, or document. |
| `POST /analytics/rankings` | Free | Free. POST only (GET returns 405). Same small-group suppression floor as /aggregate. |
| `GET /alerts` | Free | Free. Configuring alerts never costs credits; only delivered records do. |
| `POST /alerts` | Free | Free. |
| `GET /alerts/{alertId}` | Free | Free. |
| `PATCH /alerts/{alertId}` | Free | Free. |
| `DELETE /alerts/{alertId}` | Free | Free. |
| `POST/GET /alerts/{alertId}/preview` | Free | Free. Preview returns how many records a rule would deliver and what that would cost, not the records themselves. |
| `POST /alerts/{alertId}/run` | Plan dependent | A run, manual or scheduled, bills the records it delivers. County, State, and National: 1 export record per project that is new to this account or changed since, against the shared 100,000 a month. Unchanged projects already delivered are free. Standalone API Only and the free trial: 1 credit per new or changed project. A run the balance cannot cover delivers nothing and charges nothing. Finding contact info on those records is a separate enrichment. |
| `GET /alerts/{alertId}/deliveries` | Free | Free. The delivery history and what each run charged. |
| `GET /alerts/{alertId}/deliveries/{deliveryId}` | Free | Free. |
| `POST /alerts/{alertId}/dismissals/{projectId}` | Free | Free. Dismissing a match stops it being re-offered and takes a kept project off the alert's list. |
| `DELETE /alerts/{alertId}/dismissals/{projectId}` | Free | Free. |
| `GET /lists` | Free | Free. Every list your membership resolves: your own and the ones shared with you. |
| `GET /lists/{listId}` | Free | Free. One list's configuration and share state, plus `results.billable.credits`, what a full `/results` pull would charge right now. This is the free preflight for `/results`. |
| `GET /lists/{listId}/results` | Plan dependent | County, State, and National: 1 export record per project that is new or changed, against the shared 100,000 a month. Unchanged records already delivered are free. Standalone API Only and the free trial: 1 credit per new or changed record. Alert companions return the same match set as the alert (every match except dismissed, pending review included). Extra search filters apply only inside those existing ids. Filter and polygon lists return stored criteria plus a search link. |
| `POST /lists` | Free | Free. Creates a named membership list (the "new list" option on save). |
| `POST /lists/{listId}/projects` | Free | Free. Adds project ids to a membership-backed list. Idempotent per id. |
| `DELETE /lists/{listId}/projects/{projectId}` | Free | Free. Removes a project from a membership-backed list. |
| `GET /account/preferences` | Free | Free. The account's saved search preferences (types, geography, website). |
| `PATCH /account/preferences` | Free | Free. Update the account's saved search preferences. |
| `GET /reference/statuses` | Free | Free. Canonical status filter tokens plus display aliases (e.g. scheduled). |
| `GET /reference/owner-types` | Free | Free. The owner-type vocabulary used by alert criteria and CSV output. **Not** accepted as a `/projects/search` filter. The `ownerTypes` filter was withdrawn on 2026-08-27. |
| `GET /reference/decision-bodies` | Free | Free. Canonical values accepted by the decisionBodies filter. |
| `GET /reference/request-types` | Free | Free. Canonical values accepted by the requestTypes filter. |
| `POST /exports` | Plan dependent | County, State, and National: 1 export record per project in the file that is new to this account or changed, against the shared 100,000 a month, settled when the file is ready. Unchanged projects already delivered are free. Standalone API Only and the free trial: 1 credit per new or changed project. Pass alertId, listId, or projectIds to export a saved set (optional extra filters stay inside those ids). Omit all three and the body is a new search. Find Contact Info on those rows is 1 credit per project the first time this account receives email, phone, or a personal LinkedIn profile for it. |
| `GET /exports` | Free | Free. Lists export jobs for your team (including jobs queued with POST /exports or format=csv on search). |
| `GET /exports/{id}` | Free | Free. Poll job status until ready. |
| `GET /exports/{id}/download` | Free | Free re-download of a ready product-core CSV. The rows were counted when the file was built. Find Contact Info on those rows, if you asked for it, was 1 credit per project unlocked. |
| `GET /permits/search` | Plan dependent | County, State, and National: 1 export record per permit that is new to this account or changed since the last delivery, against the shared 100,000 a month. Permits and planned projects share that counter, across the API, MCP, CSV, and the app. An unchanged permit already delivered is free. Standalone API Only and the free trial: 1 credit per new or changed permit; unchanged owned permits are free. Contacts printed on the permit come with the record and use no Find Contact Info allowance. |
| `GET /permits/{id}` | Plan dependent | County, State, and National: 1 export record the first time this account receives the permit, or when it has changed since, against the shared 100,000 a month. A later read of the same unchanged permit is free. Standalone API Only and the free trial: 1 credit the first time. |
| `GET /permits/sync` | Plan dependent | County, State, and National: 1 export record per new or changed permit, against the shared 100,000 a month. Unchanged permits bill 0, so walking the same filters again costs only what changed. Standalone API Only and the free trial: 1 credit per new or changed permit. |
| `GET /permits/count` | Free | Free, and it does not use an export record. Returns the match count and what GET /permits/search would bill for it: new or changed permits against permits you already hold unchanged. |
| `GET /permits/reference` | Free | Free. Every permit filter value and its label. |
| `GET /permits/coverage` | Free | Free. The places with permits and whether your plan covers each one. |
| `POST /permits/aggregate` | Free | Free. Counts and declared job values grouped by place, work type, status, trade, or month. Never returns a permit, contractor, address, or party. |
| `POST /lists/{listId}/permits` | Free | Free. Saves up to 100 permits onto Saved or one of your named lists. Idempotent. |
| `DELETE /lists/{listId}/permits/{permitId}` | Free | Free. Removing a permit that is not on the list is not an error. |
| `GET /lists/{listId}/permits` | Plan dependent | County, State, and National: 1 export record per permit that is new to this account or changed since the last delivery, against the shared 100,000 a month. Permits and planned projects share that counter. An unchanged permit already delivered is free. Standalone API Only and the free trial: 1 credit per new or changed permit. |
| `POST /permits/exports` | Plan dependent | County, State, and National: 1 export record per permit in the file that is new to this account or changed, against the shared 100,000 a month, charged when the file is ready. Unchanged permits already delivered are free. Standalone API Only and the free trial: 1 credit per new or changed permit. Never more than maxCredits: a job that would bill more fails and charges nothing. Idempotency-Key required. Poll and download with the export routes. |
| `GET /permit-alerts` | Free | Free. Configuring permit alerts never uses export records or credits; only the permits a run delivers do. |
| `POST /permit-alerts` | Free | Free. |
| `GET /permit-alerts/{alertId}` | Free | Free. |
| `PATCH /permit-alerts/{alertId}` | Free | Free. |
| `DELETE /permit-alerts/{alertId}` | Free | Free. |
| `GET/POST /permit-alerts/{alertId}/preview` | Free | Free. How many permits match now, and how many the next run would add. |
| `POST /permit-alerts/{alertId}/run` | Plan dependent | A run, manual or scheduled, bills the permits it delivers like a planned-project API alert run. County, State, and National: 1 export record per permit that is new to this account or changed, against the shared 100,000 a month. Unchanged permits already delivered are free. Standalone API Only and the free trial: 1 credit per new or changed permit. A run the allowance cannot cover delivers nothing, charges nothing, and answers status insufficient_credits with a quote. Reading the permits a run delivered from the matches afterwards is free while they are unchanged. Every run started over the API or the MCP is billed, whatever created the alert. Scheduled runs of an alert made on the website, and their emails, are free until its filters are changed over the API. |
| `GET /permit-alerts/{alertId}/matches` | Plan dependent | Billed like permit search. County, State, and National: 1 export record per permit that is new to this account or changed since the last delivery, against the shared 100,000 a month. An unchanged permit already delivered is free, including one an alert run already billed. Standalone API Only and the free trial: 1 credit per new or changed permit. The permits already on this alert, with your saved, dismissed, or pending decision on each. |
| `GET /permit-alerts/{alertId}/deliveries` | Free | Free. Each run that found new permits, newest first. |
| `GET /permit-alerts/{alertId}/deliveries/{deliveryId}` | Free | Free. |
| `PUT /permit-alerts/{alertId}/decisions/{permitId}` | Free | Free. Your own saved, dismissed, or pending decision on one permit. |
| `POST /permit-alerts/{alertId}/dismissals/{permitId}` | Free | Free. |
| `DELETE /permit-alerts/{alertId}/dismissals/{permitId}` | Free | Free. |
| `GET /permits/contractors` | Plan dependent | County, State, and National: 1 export record per contractor new to this account, against the shared 100,000 a month. A contractor already delivered is free, whatever the filters. Standalone API Only and the free trial: 1 credit per new contractor. The contractor count is free through the permit aggregate. |
| `GET /permits/{permitId}/source-text` | Plan dependent | Counts with the permit record, not as a record of its own: 1 export record (or 1 credit on API Only and the free trial) when this account has not yet received the permit's current version, through this route or the permit record. Free when it has, and free when the permit has no source text on file. |

## Filtering

- **Geography accepts ids, USPS abbreviations, or names.** `states=UT`, `states=Utah`, and
  `states=1` are all valid. Naming a county or city **replaces** the state scope rather than
  narrowing within it: the states you also sent are not searched in full, and the response
  says so with a `geography_narrowed_to_smallest_scope` warning. An
  ambiguous county or city name is an error rather than a silently wider search. Send the
  parent state separately, for example `states=UT&counties=Davis`, even on a Utah-only plan.
  Do not combine a county and state into one value such as `Maricopa County, AZ`.
  Location URL path parameters are numeric IDs: resolve Texas through `GET /locations/states`,
  then use its ID in `/locations/states/{id}/counties`; `/locations/states/TX/counties` is not valid.
- Values inside one parameter are an **OR**; different parameters are **AND**ed.
- **Check plan geography before a batch.** Location reference responses identify entitled
  areas. A `403 geographic_access_denied` is not transient: use its allowed/denied geography
  and recovery guidance. Stop repeating that scope with different company names or filters.
  Ask the user about coverage or explicitly disclose an agreed narrower search.
- **An unknown parameter is a `400`, not a silent drop.** If you are unsure of a name, read
  the filter reference at https://boardwalkai.com/docs/api/filters/ rather than guessing.
- **A request with no date filter searches the last 12 months.** Pass `dateRange=all` for the
  whole corpus. The response discloses the applied default in `meta.appliedDefaults`.
- **Search returns private development leads by default** (`leadTypes=private`, disclosed in
  `meta.appliedDefaults`), the same view as the map. Add `leadTypes=public` for public
  projects (roads, schools, utilities, public buildings), `private_minor` for small private
  jobs, or `public_minor`; `private` is always included, so `leadTypes` only ever widens.
  Ground-up vs renovation is `constructionTypes`. Standard and full records carry
  `leadType` and `stage`; compact omits them. `developerName` (aliases `developer`,
  `company`) is an exact-phrase search over parties named on the filing.
- **Re-read records the account already bought with `projectIds`** (alias `ids`, at most 100
  per request; accepted on search, count, and sync). Owned unchanged records come back at 0
  credits, and no default date window applies to an id selection. On the free
  `GET /projects/count` it must be the **only** filter.
- **On a repeat search (`GET /projects/search`) add `deliveryMode=new`** so records the pool already owns are withheld
  instead of filling the page. `new` withholds an owned record whether or not it changed.
  Use `GET /projects/sync` when you want the updates. Sync does not accept `deliveryMode=new`;
  omit that parameter and continue with the returned cursor.
- `meta.appliedFilters` is a receipt: it shows what actually ran after name resolution.
  `meta.warnings[]` is the other one: branch on its `code`, never its prose.
- **A lost HTTP response has an unknown outcome.** It does not prove a request was free or
  did not run. Check job/delivery status; where the endpoint supports idempotency, retry the
  identical request with the original key. Do not create a new key for an uncertain write.
- **Queued jobs expose delay information.** Enrichment responses include `queue`; export
  descriptions include `job.queue`. When `queue.delayed` is true, honor `pollAfterSeconds`
  and keep polling the same job. Do not create duplicate jobs to get past a delay. The
  existing status remains `queued`, `running`, or its terminal state; delay is additional context.

- **Permit source text has separate availability.** Delivered permit records include
  `sourceText.status`: `available` means a rendered-text metadata row exists, `not_available`
  means none was found, and `unknown` means availability could not be checked. `available`
  does not guarantee the storage read will succeed. Skip a source-text request when it is
  `not_available`; the permit itself is still usable. Only an entitled, served permit can
  return the specific `source_text_unavailable` error reason.

## Representations

`representation=compact|standard|full`. All three cost the same. Use `compact` (21 fields) for anything you are holding in
context; use `full` (93 fields) only when you need decisions with named votes, document
metadata, or the source text.

## Building permits

Boardwalk also serves building permits from city and county building departments. Utah is
first; more areas are being added through the end of 2026. Permits have their own MCP tools
and REST routes under `/permits`, each the twin of a planned-project one. The two datasets
never share ids.

### Which dataset answers the question

- Planned work (what is proposed, rezones, plats, site plans, approvals): planned projects,
  `count_projects` then `get_projects`.
- Building work being done or finished (roofs, pools, new homes, remodels, who pulled the
  permit, inspections, contractors): building permits, `count_permits` then `search_permits`.
- "What is being built in Lehi" can mean both. Count both and give both numbers.
- To follow a planned project into building: `get_project` for its address and coordinates,
  then `search_permits` with that address as `q` (or a `bbox` around the point) and
  `dateRange=all`. A permit's `placeKey` then finds every permit at that address. A `full`
  permit carries `plannedProjectId` when Boardwalk has linked the two.

### Permit filters

The names match the map's permit view.

- **Region:** `states`, `counties`, `cities`, `bbox`, `placeKey`.
- **Work:** `workTypes` (for example `roofing`, `pool`, `new_home`, `solar_battery`),
  `workGroups`, `constructionTypes`, `trades`.
- **Building type:** `propertyTypes`, `propertySubtypes`, `assetClasses`, with the same labels as
  planned projects.
- **Status and timing:** `statuses`, `stages`, `whyNow` (for example `issued_no_inspections`,
  `finals_next`), and `dateField` (`applied`, `issued`, or `activity`) with `dateRange` or
  `dateFrom` and `dateTo`.
- **Size and value:** `valueMin`, `valueMax`, `valueBands`, `sqftMin`, `sqftMax`, `unitsMin`,
  `unitsMax`.
- **Who:** `contractor` (a name), `contractorKeys` (from the contractor roll-up),
  `hasContractor`, `hasCompanyContact`, `hasAnyContact`.
- **Text:** one of `q`, `permitNumber`, or `contractor` per request.
- **Region:** `states`, `counties`, and `cities` are one Region filter: a permit in any listed
  place matches. Send only the narrowest place. A state sent with a county or city inside it
  only helps find that place (warning `region_parent_state_dropped`), which is how to pick the
  Utah one of several cities with the same name.
- **One address:** search `q` with the street address and `dateRange=all`, then search again
  with the best hit's `placeKey`, because cities write one address several ways. Around a
  planned project, prefer its street address over a map box.
- With no date filter, a permit search covers major and minor jobs with activity in the last
  12 months, and `meta.appliedDefaults` says so. `dateRange=all` removes the window. A
  `permitNumber` lookup searches every date.
- `get_permit_reference` (`GET /permits/reference`) lists every value. `GET /permits/coverage`
  lists the places that have permits today.

### Reading a permit

- Boardwalk captures and digitizes what each city or county publishes about a building permit.
  A blank field (contractor, job value, phone, inspections, applied date) means the record
  Boardwalk could read there did not show it, not that the city never had it. Some places
  publish only the issued permit, with no inspections or later status, and for some permits
  Boardwalk has only the city's list entry (`detailLevel: "summary"`), so the city's own page
  may say more. That is normal in public permit records, and a later filing can fill a gap
  once Boardwalk captures it. Tell the user what is known. If two fields disagree, say what
  each one says.
- A permit lists every party the city recorded, with the phones and emails printed on the
  permit, homeowners included. Private individuals are marked `isPrivateIndividual`. Permits
  have no separate contact lookup; Find Contact Info is for planned projects.
- A homeowner's phone and email may be withheld: `contactWithheld.withheld` is `upgrade` when
  homeowner phones and emails need a paid plan and the user's plan does not include them, or
  `acknowledge` until the user accepts the contact rules once in the Boardwalk web app.
  `hasPhone` and `hasEmail` say which exists. Contractors, licensed parties and company
  contacts are never withheld. When anything on a permit is withheld, every
  phone and email in its source text and its descriptions and notes (`textContactWithheld`)
  shows as `[phone withheld]` or `[email withheld]`,
  business lines included; quote business phones and emails from the permit's parties instead.
  Name every party; mention withholding only for a party that carries `contactWithheld`. The
  contact exists: never call it missing, and never guess a number.
- If the user wants to contact private individuals: no calls or texts without the user
  confirming they have that person's consent, no robocalls, AI or recorded voice calls, or
  bulk texts, and Do Not Call rules apply. Raise this when they ask for outreach, not on a
  plain lookup.
- Contractors: `analyze_permits` with the `contractors` metric counts them for free;
  `search_permit_contractors` (`GET /permits/contractors`) ranks them by permits, job value,
  or recent activity with the contact details their permits show. Pass a row's
  `contractorKey` to `search_permits` as `contractorKeys` to list that contractor's permits.
- The text a permit was read from: `get_permit` with `includeSourceText` (REST
  `GET /permits/{permitId}/source-text`). It counts with the permit record, so a permit
  already delivered unchanged costs nothing more; a permit with no text on file is a `404`
  and charges nothing.
- Permit alerts have the same limits as planned-project alerts: a place is required (at most
  5 states and 25 counties or cities, unless the plan is nationwide), and an account holds
  at most 25 active permit alerts, counted separately from planned-project alerts.
- A permit alert run started over the API or MCP (`run_permit_alert`) is charged, whoever
  created the alert: each new or changed permit it adds counts as one export record (1 credit
  on API Only and the trial). An alert created, or with filters edited, over the API is
  API-made, so its scheduled runs are charged the same way. Preview first and state the cost.
  Reading a permit alert's matches (`get_permit_alert_matches`) is priced like `search_permits`:
  a permit already delivered unchanged, including one a run billed, does not count again.
- Lists hold planned projects and permits side by side (`add_permits_to_list`,
  `get_list_permits`). `create_permit_export` queues a permit CSV from exactly one source:
  an alert, a list, permit ids, or a filter set. `list_exports` and `get_export` serve both
  datasets.

### What permits cost

The same plan area and the same allowance as planned projects. One permit is one record: on
County, State, and National each new or changed permit counts toward the shared monthly export
records; on API Only and the trial it is 1 credit. The free trial's 50 record credits cover
planned projects and permits together, in any mix. An unchanged permit already delivered does
not count again, on the website, the API, or MCP. Counting and analyzing permits do not use
credits.

- **Permit coverage is Utah at launch.** Outside the plan area, or in a place without permits
  yet, say so plainly: building permits are new on Boardwalk, Utah is first, and more areas
  are being added through the end of 2026. Never say there are no permits there; offer
  planned projects in that place.
- **Job value is what the permit states.** `analyze_permits` returns `valueKnown`, the share of
  permits that stated one; give that share next to any value total.

## Company research

Boardwalk researches the outside parties on planned projects and building permits: the
company behind an LLC or a printed name, its parent company, and the people who work on the
project. It is the same research the Boardwalk web app shows on a project or permit page, and
it rides on `standard` and `full` records of both datasets (not `compact`).

- `companyResearch.status` says whether the research has run: `researched` (Researched), `nothing_found` (Researched, nothing found), `not_researched` (Not researched yet).
  `researchedOn` is the day it last ran. Not researched yet is not a gap in the filing; say
  the research has not reached that record.
- `researchedParties` lists each party the research tied to the record: `name`,
  `entityType` (`company` or `person`), `role` and `roleLabel`, `trade`, `foundVia`
  (`our_filing` (Named on the filing), `our_data` (From Boardwalk records), `web` (Found on the web)), `namesOnFilings` (how the filings wrote it), `sourceSite`, `whyItFits`, and
  `companyId`: the researched company the party is or, for a person, works for. A person
  also carries `title`, `linkedinUrl`, `email`, `personalEmail`, and `phone`. A company carries its general inbox in `email`.
- `researchedCompanies` holds each checked company once, including a researched person's
  employer: `officialName`, `description`, `localOfficeName`, `website`, `linkedinUrl`,
  `phone` (`phoneScope` is `market` for the local office line or `company` for the main
  line), `legalNames` (the LLCs it files under), `namesOnRecord`, `howWeKnow` (the proof
  that ties it, with `howWeKnowDocumentId` and `howWeKnowPage` when it came from a
  filing), and `parentCompanyName` and `parentCompanyWebsite`. Join a party to its company
  on `companyId`.
- Names, roles, descriptions, and websites go to every caller. A company's LinkedIn page and
  phone and a person's LinkedIn and emails ship when contacts are open on the record (a paid
  plan, a live trial, or a project the user unlocked); otherwise they are empty and
  `contactsWithheldReason` is `free_tier`. `suppressed` means the person asked Boardwalk to
  remove their contact details: give the name and role, never a contact value.
- When a checked company exists, `companyEnrichment` on planned projects and the CSV company
  columns use it, so every surface names the same company.
- Reading research never costs anything extra: it does not change a record's content hash,
  so a record already delivered unchanged is still free when its research updates.
- MCP list tools (`get_projects`, `search_permits`, alert and list pulls) carry
  `companyResearch` and a short `researchedLite` list (name, role, company, website) on each
  card, without emails or phones. `get_project` and `get_permit` carry the full research.
- CSV exports of planned projects and permits carry a Company research column group: status,
  researched on, researched companies, and researched people.

## The loop to follow

1. `GET /locations/states` (free). Confirm the market exists.
2. `GET /projects/count` with the real filters (free). It prices the whole match set:
   `billable.credits` is what delivering the entire set would cost, and
   `affordability.affordable` says whether that fits the balance. `affordable: false`
   does not mean your next search call fails. It means the balance runs out somewhere
   inside the set, and `affordability.maxAffordableRecords` is how many records you can
   still collect.
3. `GET /projects/search` with the identical filters. The count response hands you
   that URL in `links.search`. On API Only, each paid page is delivered whole while
   it fits the balance; a page that does not fit is refused with a quote and no charge.
   Trial pages may be shortened; check `meta.trialTruncation`.
4. Read `meta.creditsRemaining` from the response rather than tracking a local counter.
   The credit pool is shared with the account's other users and surfaces.

On `402 insufficient_credits`, retry with `limit` set to
`error.details.quote.affordableRecordCount` (that buys what the balance still affords),
or follow `error.details.quote.countUrl`, a free URL that re-prices the exact filter set
that was refused, and narrow from there. A `credits_exhausted` refusal carries no quote:
the balance cannot pay for even one record.

## Limitations you should tell your user about

- **Coverage is per-state, not nationwide.** `GET /locations/states` is free and lists exactly
  what exists. Check it before promising a market.
- **Filtered values are rechecked before billing.** A record with no current value for a
  requested status, building type, or measurement constraint is excluded rather than delivered
  as a null-valued match. Do not add a client-side null-value workaround.
- **An empty field means we could not establish the value**, not that the value is zero. We
  do not guess.
- **Building permits carry what each city or county publishes.** Many leave out the
  contractor, job value, phone, inspections, or applied date, and some publish only the issued
  permit. A blank field means the record Boardwalk could read did not show it, not that the
  city never had it; `detailLevel: "summary"` is the city's list entry only. If two fields
  disagree, say what each one says.
- **Free-trial keys are the one truncation exception.** A paid page is delivered whole or
  refused whole, so a short page on a paid key always means that is all that matched. A
  short page on a trial key may instead be the trial budget; check `meta.trialTruncation`.
- **Find Contact Info is opt-in.** Use `POST /enrichments` with purchased project IDs for a
  live lookup. On synchronous `GET /projects/search`, `include=contacts` requires
  `contactOptions={"cachedOnly":true}` and only reads stored contacts; it never starts a lookup.
  Live lookups are available to paid plans and live trials (the trial has 5 lookups,
  counted separately from the 50 record credits). Unpaid keys get
  `402 enrichment_requires_paid_plan`. Do not enrich the rest
  of the catalog.
- **Contacts we already hold need no toggle.** Email, phone, and LinkedIn ship on paid
  plans and on the records a live trial spends a credit on, and a trial's CSV carries
  them inside its 50-row sample. Free reads on a trial key withhold them;
  `contactsWithheldReason: "free_tier"` says so.

---

_Generated from the Boardwalk API contract. Do not edit by hand. Run `bin/console app:export-docs`._
