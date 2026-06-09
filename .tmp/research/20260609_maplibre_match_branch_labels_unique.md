# MapLibre match expression error research

Date: 2026-06-09

## Symptom

Browser console reports:

```text
Error: layers.kzone-to.paint.fill-color[152]: Branch labels must be unique.
Error: layers.kzone-to.paint.fill-color[150]: Branch labels must be unique.
```

The failing layer is `kzone-to` in `src/routes/+page.svelte`.

## Relevant Code

`src/routes/+page.svelte:36-47` builds a MapLibre `match` expression:

```ts
const pairs = ods.flatMap((od) => {
	const t = max > 0 ? od.count / max : 0;
	const g = Math.round(165 * (1 - t));
	return [od.to | 0, `rgb(255,${g},0)`];
});
return ['match', ['get', 'kzone'], ...pairs, 'rgba(0,0,0,0)'];
```

`src/routes/+page.svelte:57-66` maps DuckDB rows with:

```ts
to: Number(r['着地'])
```

## Findings

The `着地` column is not always a pure numeric zone code. It also contains aggregate or "other" labels such as:

```text
7000 東京区部
00-- 東京区部（その他）
8888 全計
8700 圏域外合計（含不明）
```

Examples from `static/d-1.parquet` for `発地='0010'` include both pure zone codes and aggregate labels. Pure numeric destination rows exist, but non-numeric destination rows are mixed into the same query result.

Counts for `目的種類='計'`:

```text
pure numeric 着地 rows: 92288
non-pure 着地 rows: 13392
```

For many origins, multiple destination rows are non-pure. Example:

```text
発地 0010 has 25 non-pure 着地 rows
```

Because `Number('7000 東京区部')` and `Number('00-- 東京区部（その他）')` both produce `NaN`, the generated `match` expression can contain multiple `NaN` branch labels. MapLibre stores label keys by string conversion internally, so repeated `NaN` labels violate the unique-label requirement.

## MapLibre Constraint

Context7 lookup could not be completed because `ctx7` is not installed in this environment:

```text
zsh: command not found: ctx7
```

Local installed package source confirms the same rule:

- `node_modules/@maplibre/maplibre-gl-style-spec/src/reference/v8.json` documents that each `match` label must be unique.
- `node_modules/@maplibre/maplibre-gl-style-spec/src/expression/definitions/match.ts` raises `Branch labels must be unique.` when a label key repeats.

## Root Cause

The frontend query returns rows that are not valid map feature zone destinations, then `Number(r['着地'])` collapses those values to `NaN`. The paint expression is then built from invalid OD destination labels.

The correct contract should be: the map highlight layer only receives OD rows whose destination is a concrete numeric zone code that can match the `kzone` vector-tile property.

