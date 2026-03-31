---
name: kmb-dashboard-builder
description: Create and assemble KMB dashboards from existing cards, including dashcard layout and parameter mappings. Use it after questions and visualization are ready.
metadata:
  short-description: Build dashboards and dashcard layout
---

# KMB Dashboard Builder

Use this skill for dashboard assembly only.

## Inputs

Required:

- dashboard `name`
- `collection_id`
- cards layout config with `card_id`, `row`, `col`, `size_x`, and `size_y`

Optional:

- dashboard parameter config

## Commands

```bash
python3 scripts/create_dashboard.py --name "投放数据分析" --collection 485
python3 scripts/add_cards.py --dashboard-id 5001 --config-file cards_config.json
```

## Requirements

- Use negative temporary IDs for new dashcards.
- Do not use a nonexistent `/api/dashboard/:id/cards` endpoint.
- If parameter mapping is involved, verify the `target` structure carefully.
- Support both `["dimension", ...]` and `["variable", ["template-tag", ...]]` target styles. Real dashboards may mix both in one page.
- Do not assume every dashcard has a `card_id`; some dashboards include `card_id = null` text or explanation blocks that must be preserved.
- For `card_id = null` text blocks, convert Space HTML/rich-text content into KMB-compatible Markdown before writing `visualization_settings.text`. Do not paste raw HTML tables or styled spans directly into KMB.
- If a migrated text block becomes taller after Markdown conversion, reflow the downstream dashcard rows explicitly instead of keeping the old coordinates.
- A single dashboard parameter may map to different temporal fields or grains across cards; preserve the per-card mapping instead of forcing one global target.
- Return the created `dashboard_id`.
- Consume `card_id` values from `$kmb-question-builder` and, when present, use the post-style-update `card_id` from `$kmb-viz-config`.

If needed, read `../kmb-metabase/references/dashboard-configs.md`.
Also read `../kmb-metabase/references/parameter-mapping-patterns.md` when dashboard filters coordinate MBQL and native cards.
Also use `../kmb-metabase/references/skill-handoffs.md` if the pipeline has multiple owners.

## Hand-off

- For full migration delivery, return control to `$kmb-migration`.
