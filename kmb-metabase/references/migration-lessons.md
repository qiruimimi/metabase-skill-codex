# Migration Lessons

Short notes from real Space-to-KMB migrations. Read this only when a migration hits non-obvious workflow or tooling issues.

## Page 55839

### What worked

- One shared Model can support multiple Space graphs when the graphs differ only by aggregation, breakout, threshold, or chart type.
- Repeated graphs should still become separate KMB cards so visualization and dashboard placement remain independent.
- For page `55839`'s specific “日/周/月” switching logic, expanding the grain in the Model was safer than trying to rebuild the original `CASE` logic in every Question. Do not generalize this to every time-series page when native Metabase temporal grouping on one true date field is sufficient.
- For threshold exploration like “n 天内新签率”, a helper field in the Model plus Dashboard parameter mapping keeps the Question in MBQL.

### What failed first

- `POST /api/card` for a Question failed with HTTP 400 when `display` and `visualization_settings` were omitted.
- Relying on Space copy chains alone was not enough to reconstruct the full delivery; the final migration still needed an explicit asset map from `graph_id -> card_id`.
- In a Codex session where Tesseract MCP was not actually registered, assuming live MCP availability caused planning drift. The migration succeeded only after explicitly falling back to `space-data/page_map.json`.

### Guardrails to keep

- Check target collection cleanliness before creating assets; archive partial runs instead of piling new assets on top.
- Save raw SQL, analyzer outputs, final `migration_plan.json`, and `asset_mapping.json` in the workspace during execution.
- Record fallback reason and source when using offline parameter config.
- Verify each created card with `/api/card/{id}/query`, not just the Model.

### Suggested defaults

- Dashboard date filters should target a real date field such as `*_day_time`, not the display grain field.
- When a page mixes tables and charts from the same SQL, create one Question per displayed graph rather than trying to reuse one card with multiple visual states.

## Page 55909

### What worked

- Rebuilding the page on the correct active collection path was straightforward once collection matching switched from `parent_id` to `location`.
- One shared Model still covered the full page after adding the missing fully qualified table names for the subscription plan and FX tables.
- Archiving the two failed leaf collections before rerun kept the final asset map clean and avoided mixing valid and invalid card IDs.

### What failed first

- Matching collections by `parent_id` created parallel directory trees (`606/.../609` and `610/.../613`) instead of reusing the intended path.
- Treating `/api/card/{id}/query` as success whenever it returned a JSON body caused a false positive; the first rebuilt Model still had `status: "failed"` because one table name was not fully qualified.
- Trying `DELETE /api/collection/{id}` on KMB returned HTTP 404 because this deployment does not expose a collection delete endpoint.

### Guardrails to keep

- For collection routing, match same-name siblings by `location == parent.location + parent_id + "/"`, not by `parent_id`.
- Before rerun, archive failed page assets in the wrong leaf collection; after the leaf is empty, archive the wrong root collection tree by `PUT /api/collection/{id}` with `archived: true`.
- When verifying Models or Questions, require `status == "completed"` and treat `status == "failed"` as a hard failure even if the response body contains rows, metadata, or SQL text.
- If multiple same-name collections exist under one parent, ignore archived trees first, then confirm ownership from live contents and recorded asset metadata before reusing any active candidate.

### Suggested defaults

- Add an explicit cleanup step for misrouted collection trees after a failed migration; “move to Trash” is the supported cleanup path in Metabase/KMB, not permanent deletion.
- For native-source rebuilds, fully qualify non-default schemas early instead of waiting for verification to expose `Unknown table` errors.

## Pages 55865 and 56698

### What worked

- Reusing a validated parent-page detail Model shape from page `57302` reduced rebuild risk for page `55865` and its child page `56698`.
- Treating graph-level hidden defaults such as `openposition2` as fixed filters made it possible to rebuild 22 child-page cards from one shared Model without losing page intent.
- Rebuilding the 3 `TEXT` graphs as dashboard text cards preserved the source dashboard card count and order.

### What failed first

- Historical workspace artifacts incorrectly marked page `55865` as `container_only`, but live Space truth showed 10 SQL graphs. Trusting the old artifact would have skipped the parent dashboard entirely.
- Two active same-name collections (`624` and `625`) existed under the formal `支付漏斗` parent. Name-only routing was not enough to decide reuse.
- `PUT /api/dashboard/{id}` and `PUT /api/card/{id}` hit intermittent SSL transport errors (`EOF`, `bad record mac`) even though the business payloads were valid.

### Guardrails to keep

- When live page truth conflicts with old migration artifacts, trust live MCP / page-query truth and record the conflict in the new `migration_plan.json`.
- If multiple active same-name collections exist under one formal parent, confirm actual page ownership from existing dashboards, models, and asset mappings before reusing anything.
- Preserve graph-level fixed defaults, including hidden ones, in `migration_plan.json` as `graph_id -> fixed_filters`.
- For pages with `TEXT` graphs, record `text_graph_ids` and the fallback source used to rebuild them.
- For transport-layer KMB API failures, retry first and inspect partial assets before rerunning the whole page.

### Suggested defaults

- Treat `TEXT` graphs as first-class dashboard artifacts, not optional notes.
- Put finite retry wrappers around `POST /api/card`, `PUT /api/card/{id}`, and `PUT /api/dashboard/{id}` in long migration scripts.

## Pages 54573, 55725, and 55726

### What worked after correction

- One shared revenue detail Model still supported the parent page and both child pages after the Model was reduced back to a single true payment date field.
- Weekly charts behaved correctly once each Question used a weekly MBQL temporal breakout on `pay_success_day_time` instead of breaking out on an unbinned display date.
- Default visible data was stable once the Question itself carried a relative recent-weeks filter on the same true date field used for breakout.

### What failed first

- Adding a `date_expanded` layer (`date_type`, `stat_date`, `UNION ALL` for day/week/month) was unnecessary for this page family and created avoidable complexity in both the Model and the Questions.
- Breaking out by an unbinned date field made Metabase render the weekly chart at day grain, so only Monday points held values and the chart looked sparse or wrong.
- A quick text replacement while removing the expansion layer broke the closing `joined_base` CTE, which then surfaced only at Question verification time.

### Guardrails to keep

- Before adding Model-side date expansion, first check whether Metabase can express the source requirement directly with one real date field plus `temporal-unit`.
- If the intended chart grain is weekly or monthly, inspect the final Question payload and verify that the `breakout` contains the temporal unit, not just a bare date field.
- Keep the Dashboard date parameter mapped to the same true date field used by the Question breakout whenever possible.
- After refactoring a shared Model SQL block, inspect the final saved SQL for balanced CTE boundaries before attributing failures to MBQL.

### Suggested defaults

- Prefer one real date field in the Model and let Questions choose `day/week/month` with native Metabase temporal grouping.
- Reserve `date_type` or other expanded-grain helper fields for pages whose business logic truly requires separate precomputed grain rows or other non-standard calendar semantics.
