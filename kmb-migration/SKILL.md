---
name: kmb-migration
description: Run the full Space-to-KMB migration workflow. Use it when the user wants an end-to-end migration from a Space page into KMB assets including collection preparation, SQL extraction, analysis, model creation, questions, visualization, dashboard assembly, and acceptance checks.
metadata:
  short-description: Orchestrate full Space to KMB migration
---

# KMB Migration

This is the workflow skill that orchestrates the other KMB skills.

## Required inputs

- `page_id`
- one target strategy:
  - existing `target_collection_id`, or
  - create-new-collection strategy with parent location

Default behavior if unspecified:

- preserve the original page's card order and primary chart types
- for Space-to-KMB migration with no explicit KMB target directory, default to replicating the Space directory path under KMB `545-root`

## Target collection priority

Resolve the target collection in this order:

1. If the user explicitly provides a KMB destination such as `target_collection_id`, a target collection path, or an equivalent KMB parent directory, use it directly.
2. Otherwise, if the task is a Space-to-KMB migration, default to KMB root collection `545` and replicate the Space page directory hierarchy under that root.
3. If the task is not a Space migration, do not apply the `545-root` default.

Treat “no explicit KMB target” as the absence of any user-provided KMB destination, not only the absence of `target_collection_id`.

## Workflow

1. Use `$kmb-space-query` to inspect the page, extract source SQL for each graph, and capture the page's full Space path.
2. Resolve the target collection strategy using the priority rules above.
3. If no explicit KMB target was provided for a Space migration, remove the fixed prefix `spaceId为1140的root页面` from the page path, then use `$kmb-collection-builder` to create or reuse each collection level in order under KMB root collection `545`.
4. Use the resolved leaf collection as the `target_collection_id` for all downstream assets.
5. Use `$kmb-sql-analyzer` on each SQL and produce `migration_plan.json`.
6. Use `$kmb-model-builder --validate-only` first, then create the model layer and query-verify it.
7. Use `$kmb-question-builder --verify` to create questions or cards and query-verify each one.
8. Use `$kmb-viz-config` to restore chart behavior.
9. Use `$kmb-dashboard-builder` to assemble the dashboard.
10. Produce an asset mapping and run acceptance gates.

Do not skip these hand-offs. Each stage should consume the artifact from the previous stage rather than rebuilding the intent from memory.

## Execution notes

- Before creating anything in the target collection, inspect whether it already contains partial migration assets. If it does, either archive the failed partials first or stop and confirm reuse strategy.
- For Space migration with no explicit KMB destination, the default routing root is KMB collection `545`. Do not create migrated assets directly under `545`; first resolve or create the replicated directory path, then write assets into the leaf collection.
- Before treating any existing KMB asset as reusable or “already migrated”, verify that it sits on the formal `545-root` branch. Ignore same-name assets that only exist in test or historical branches.
- Space path resolution priority is:
  - live MCP / page query result
  - fallback to the offline `path` from `$kmb-space-query`
- Strip the fixed Space root prefix `spaceId为1140的root页面` before building the KMB collection hierarchy.
- Replicate every remaining path segment in order under `545`, including the page name as the leaf collection name, unless the user explicitly specifies a different KMB destination.
- If the selected page is a container page with no charts but has child pages, do not silently recurse by default. First follow the task scope:
  - if the task is page-only, create or reuse the formal container collection and stop there
  - if the task explicitly includes descendants, then continue migration into child pages
- Treat repeated Space graphs as separate delivery artifacts even when they share one SQL or one MBQL logic path. Reuse analysis and model work, but still produce distinct `card_id`s so chart styling and dashboard placement stay independent.
- Prefer live Tesseract MCP for page and parameter config. If MCP is unavailable in the current Codex session, fall back to `space-data/page_map.json` and record the fallback reason in `migration_plan.json`.
- Even when MCP is available, `TEXT` graphs may still require offline extraction from `space-data/page_map.json`; record that limited fallback explicitly instead of marking the whole page as offline-driven.
- For pages with dynamic time grain (`日/周/月`) plus dashboard filters, push the date-grain expansion into the Model and keep Dashboard date filters mapped to a true date field such as `signup_day_time`.
- For pages that need threshold-style exploration such as “n 天内转化”, prefer Model-side helper fields over native Question SQL so the Question layer can stay in MBQL.
- Do not convert a whole page to native SQL just to recover dashboard parameter linkage. First ask which parameters can be expressed as Model fields and MBQL dimension mappings, then isolate only the truly algorithmic exceptions.
- If a card must remain native, use Metabase native template tags (`{{start_ds}}`, `{{end_ds}}`, `{{date_type}}`) in the SQL itself. Do not generate KMB native SQL that still uses `@start_ds` / `@date_type` placeholders, because Dashboard parameter mappings target template tags, not raw engine session variables.
- For native exception cards with date filters, prefer `type: date` template tags for date inputs and `type: text` only for categorical or threshold-like inputs such as `funnel_time`.
- Acceptance for a mixed MBQL/native page requires more than `status == "completed"`: verify that the native cards still return rows for a representative historical range after Dashboard parameter mappings are applied, or record a concrete business reason why a fixed range is expected to be empty.
- For detail-table cards, create the base `display: table` card with the minimum working MBQL payload first. If custom table column config is needed, add it only after the base card is created successfully; do not block the whole migration on an optional table visualization payload.
- When validating shortly after midnight or during partition refresh windows, treat `status == "completed"` with `0` rows as a valid execution result unless the page is expected to be non-empty for a fixed historical range. Record the runtime timing context instead of failing the migration immediately.
- Persist the runtime artifact set in the workspace, not just in the final message: raw SQL, analyzer outputs, final `migration_plan.json`, and `asset_mapping.json`.
- Record the resolved collection routing in `migration_plan.json`: `root_collection_id`, `space_path`, `resolved_collection_path_segments`, `intermediate_collection_ids`, and `final_target_collection_id`.
- For a concrete execution checklist and known pitfalls from page 55839, read `../kmb-metabase/references/migration-55839-checklist.md` and `../kmb-metabase/references/migration-lessons.md` before rerunning a failed migration.
- For the default `545-root` routing details, read `../kmb-metabase/references/migration-directory-routing.md`.

## Handoff contract

Use `../kmb-metabase/references/skill-handoffs.md` as the source of truth for:

- which artifact each child skill must produce
- which next skill consumes that artifact
- which jumps are forbidden

## Required outputs

Track and report at least:

- extracted SQL files or equivalent raw SQL artifacts
- `migration_plan.json`
- `root_collection_id`
- `space_path`
- `resolved_collection_path_segments`
- `intermediate_collection_ids`
- `final_target_collection_id`
- `collection_id`
- `model_id`
- `question_id` or `card_id`
- `dashboard_id`
- asset mapping between Space and KMB resources

## Acceptance gates

Do not call the migration complete unless all required checks pass.

- Cards reference KMB models or cards, not raw Space tables
- Card queries run successfully
- Sampled data matches the original SQL closely enough for the task
- Dashboard card count, order, and major chart behavior are preserved
- Dashboard parameters exist and each expected dashcard mapping is present
- When no explicit KMB destination was provided, the resolved `collection_id` must sit under KMB root collection `545`, and its hierarchy must match the Space path after removing the fixed root prefix
- The leaf collection name must match the final segment of the Space page path unless the user explicitly overrides the target
- All downstream Models, cards, and dashboards must be created in the resolved leaf collection, not directly under `545`
- Every created Model and card has passed `/api/card/{id}/query`
- Treat query failures caused by invalid SQL, missing columns, unknown tables/databases, malformed MBQL, or API 4xx/5xx validation errors as hard failures.
- A same-name dashboard or collection that exists only outside the formal `545-root` path does not satisfy the acceptance gate.
- Treat `exceed big query scan_rows limit` on an intentionally wide unfiltered base Model as a soft validation result only when the SQL is otherwise valid and the intended dashboard-level filters are documented in `migration_plan.json`.
- If a base Model is soft-passed because of `scan_rows limit`, downstream filtered cards still must pass `/api/card/{id}/query`.
- `status == "completed"` with `0` rows is acceptable during active partition refresh windows when the query shape is otherwise valid; log the timing assumption in `migration_plan.json`.
- `migration_plan.json` records fallback sources, runtime IDs, and validation results

## Stop and escalate when

- SQL cannot be mapped safely into Model and Question layers
- graph types or parameter mappings cannot be reconstructed confidently
- data consistency checks fail

## References

- MCP and fallback parameter mapping: `../kmb-metabase/references/mcp-parameter-mapping.md`
- Default 545-root routing: `../kmb-metabase/references/migration-directory-routing.md`
- Page 55839 execution checklist: `../kmb-metabase/references/migration-55839-checklist.md`
- Recent migration lessons: `../kmb-metabase/references/migration-lessons.md`
