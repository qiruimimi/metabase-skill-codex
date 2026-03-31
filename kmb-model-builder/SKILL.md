---
name: kmb-model-builder
description: Create Metabase Models for KMB from SQL or a migration plan. Use it after SQL analysis when you already know the target collection and the model SQL is ready.
metadata:
  short-description: Create KMB models from analyzed SQL
---

# KMB Model Builder

Use this skill only after analysis is complete.

## Inputs

Required:

- `collection_id`
- one of `sql_file`, `sql`, or `config_file`

Optional:

- `name`
- `database`
- `--verify`

If the SQL or config is not ready yet, go back to `$kmb-sql-analyzer`.

## Command

```bash
python3 scripts/create_model.py --name "Model: 用户访问" --config-file migration_plan.json --collection 485 --verify
```

## Expectations

- Use analyzed Model SQL, not raw aggregate SQL.
- In KMB collection listings, assets marked as `dataset` are still read and referenced through `/api/card/{id}`. Treat them as card-backed datasets in downstream work.
- Keep Model scope on reusable detail-level data: wide-table joins, dimension preprocessing, standardized date fields, and helper fields. Do not put chart-specific aggregation SQL directly into the Model unless the task explicitly calls for a one-off native card.
- If you add a join, table, or field that was not already proven by the source graph SQL, schema-check it first. Do not assume a field exists on a table by name memory alone.
- Normalize every physical table reference to `catalog.database.table` before creating the Model. Do not carry raw `database.table` references from Space SQL into KMB unchanged.
- For T+1 fact tables, do not emit future-facing date rows. If the model expands `day/week/month` helper dates from a T+1 partition such as `ds`, enforce `stat_date <= ds_time` or the equivalent business bound before publishing the Model.
- If the SQL hits any `_s_d` table, the Model must include both:
  - `STR_TO_DATE(ds, '%Y%m%d') AS ds_time`
  - `WHERE ds = DATE_FORMAT(DATE_SUB(CURRENT_DATE, INTERVAL 1 DAY), '%Y%m%d')`
- The builder script now enforces this S-table rule and will refuse to create an unsafe model.
- When multiple charts reuse the same business grain, prefer one shared Model and let `$kmb-question-builder` split the downstream MBQL cards.
- Verify the created card can be queried when the task is a migration or delivery task.
- Treat `exceed big query scan_rows limit` on an intentionally wide base Model as a soft result only when the SQL itself is valid and the downstream filtered delivery cards will still be query-verified.
- Return the created `model_id`.

## Real patterns

- `collection 114` card `2208`: MBQL question joins a reusable dataset card and adds weekly breakout on top.
- `collection 18` cards `124/127`: multiple MBQL cards reuse one shared `card__121` base rather than rebuilding the base query repeatedly.

## Preflight

Use this before creating a model if the SQL is hand-written:

```bash
python3 scripts/create_model.py --name "Model: foo" --sql-file model.sql --collection 485 --validate-only
```

Then apply this preflight before the real create:

- confirm every newly referenced table exists in the intended `catalog.database`
- confirm every newly referenced field exists on that exact table
- confirm reusable base Models keep only the shared grain and shared helper fields
- confirm any expected dashboard filters are documented if the base Model is intentionally broad

## Shared dependency

This script uses the shared package from `../kmb-metabase/scripts/lib/kmb`.

## Hand-off

- Next step for charts: `$kmb-question-builder`
- Full migration: `$kmb-migration`
