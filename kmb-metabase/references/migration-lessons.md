# Migration Lessons

Short notes from real Space-to-KMB migrations. Read this only when a migration hits non-obvious workflow or tooling issues.

## Page 55839

### What worked

- One shared Model can support multiple Space graphs when the graphs differ only by aggregation, breakout, threshold, or chart type.
- Repeated graphs should still become separate KMB cards so visualization and dashboard placement remain independent.
- For “日/周/月” switching, expanding the grain in the Model is safer than trying to rebuild the original `CASE` logic in every Question.
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
- If multiple same-name collections exist under one parent, prefer the first active candidate and ignore archived trees during routing.

### Suggested defaults

- Add an explicit cleanup step for misrouted collection trees after a failed migration; “move to Trash” is the supported cleanup path in Metabase/KMB, not permanent deletion.
- For native-source rebuilds, fully qualify non-default schemas early instead of waiting for verification to expose `Unknown table` errors.
