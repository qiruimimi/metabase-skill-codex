---
name: kmb-space-query
description: Query the internal Space dataset used by KMB migration. Use it to search pages, inspect a page or graph, extract a graph's SQL, or browse the offline tree before any modeling or migration work.
metadata:
  short-description: Search Space pages, graphs, and SQL
---

# KMB Space Query

Use this skill for upstream Space discovery only.

## Inputs

Use whichever identifier the user already gave.

- `keyword` for `search`
- `page_id` for `page`
- `graph_id` for `graph` or `sql`
- `tree` intent for browsing the directory tree

If the user already gave a `page_id` or `graph_id`, do not ask follow-up questions before running the matching query.

## Commands

Run from this skill directory.

```bash
python3 scripts/space_sql_mapper.py search "转化"
python3 scripts/space_sql_mapper.py page 34433
python3 scripts/space_sql_mapper.py graph 41236
python3 scripts/space_sql_mapper.py sql 41236
python3 scripts/space_sql_mapper.py tree
```

## Output requirements

Return the identifiers needed downstream.

- `search`: page candidates with `page_id` and page name
- `page`: page metadata, graph inventory, and complete page path
- `graph`: graph metadata and type
- `sql`: complete SQL text without truncating filters or parameter logic
- `tree`: navigable directory structure

For migration work, always preserve the full Space page `path` when it is available. That path is the default source for collection routing under `$kmb-migration`.

## Hand-off

- If the next step is modeling, hand off to `$kmb-sql-analyzer`.
- If the task is an end-to-end migration, hand off to `$kmb-migration` after extracting the needed page and graph context.

## Do not

- Do not design models, questions, or dashboards here.
- Do not treat a fuzzy search hit as a confirmed page selection.
