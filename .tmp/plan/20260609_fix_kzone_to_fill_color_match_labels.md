# Fix kzone-to fill-color match labels

Date: 2026-06-09

## Contract

For the `kzone-to` highlight layer:

1. `ods` contains only concrete OD rows whose `from`, `to`, and `count` are finite numbers.
2. `to` is an integer destination zone code matching the vector tile `kzone` property domain.
3. `toColorExpr` never emits duplicate or non-finite `match` labels.
4. Aggregate rows such as `7000 東京区部`, `8888 全計`, `8700 圏域外合計（含不明）`, and `00-- ...（その他）` are not rendered as destination zones unless a future contract explicitly maps them to geometries.
5. If the clicked feature has no finite integer `kzone`, the click handler fails explicitly by logging an error and skips the query.

## Proposed Change

Scope: `src/routes/+page.svelte`.

1. Validate `feature.properties.kzone` before querying.
2. Change the DuckDB query so it returns typed fields only:
   - `TRY_CAST(発地 AS INTEGER) AS from_zone`
   - `TRY_CAST(着地 AS INTEGER) AS to_zone`
   - `計 AS count`
3. Add `TRY_CAST(着地 AS INTEGER) IS NOT NULL` to exclude aggregate/non-zone destination rows.
4. Map rows from the typed query result instead of parsing raw Japanese column names in the UI mapping.
5. Add a small pure helper for building the color expression from already-normalized rows. It should throw on duplicate/non-finite destination labels rather than silently fallback.
6. Keep the visual encoding unchanged: max-count normalized orange ramp and transparent fallback.

## Verification

Before implementation:

```text
- Do not edit .env files.
- Do not add dependency packages.
- Do not run destructive git commands.
- TDD target: OD row normalization and MapLibre expression construction behavior.
```

After implementation:

```bash
bun run check
bun run lint
bun run build
```

Frontend verification:

Use the `playwright-cli` skill after starting the dev server. Confirm:

1. Clicking a zone no longer logs `Branch labels must be unique`.
2. `kzone-to` still highlights destination zones.
3. Console/network output does not show new errors from the click path.

## Approval Needed

Please approve before implementation. The planned behavior intentionally excludes aggregate/non-zone OD rows from the rendered destination layer.

