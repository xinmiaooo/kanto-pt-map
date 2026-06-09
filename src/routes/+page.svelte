<script lang="ts">
	import { MapLibre, VectorTileSource, FillLayer, LineLayer } from 'svelte-maplibre-gl';
	import type { Map } from 'maplibre-gl';
	import { PMTilesProtocol } from '@svelte-maplibre-gl/pmtiles';
	import { base } from '$app/paths';
	import { instantiateDuckDb } from '$lib/duckdb';
	import { onMount } from 'svelte';
	import type { MapLayerMouseEvent } from 'maplibre-gl';
	import type { AsyncDuckDBConnection } from '@duckdb/duckdb-wasm';
	import type {
		ColorSpecification,
		DataDrivenPropertyValueSpecification,
		ExpressionSpecification
	} from '@maplibre/maplibre-gl-style-spec';

	type QueryKey = 'total' | 'work' | 'home' | 'school' | 'private';

	const queryLabels: Record<QueryKey, string> = {
		work: '通勤・業務',
		school: '通学',
		private: '私事',
		home: '帰宅',
		total: '計'
	};

	function buildQuery(key: QueryKey, kzone: number): string {
		const base = `SELECT * FROM parquet_scan('d-1.parquet') WHERE TRY_CAST(発地 AS INTEGER) = ${kzone} AND TRY_CAST(着地 AS INTEGER) IS NOT NULL`;
		switch (key) {
			case 'total':
				return `${base} AND 目的種類 = '計'`;
			case 'work':
				return `${base} AND (目的種類 = '自宅－勤務' OR 目的種類 = '勤務・業務' OR 目的種類 = '自宅－業務')`;
			case 'home':
				return `${base} AND 目的種類 = '帰宅'`;
			case 'school':
				return `${base} AND (目的種類 = '通学' OR 目的種類 = '自宅－通学')`;
			case 'private':
				return `${base} AND (目的種類 = '私事' OR 目的種類 = 'その他私事' OR 目的種類 = '自宅－私事')`;
		}
	}

	type Od = {
		from: number;
		to: number;
		purpose: string;
		count: number;
	};

	type JsonRow = {
		toJSON: () => Record<string, unknown>;
	};

	async function load_db() {
		const db = await instantiateDuckDb();
		await db.registerFileURL('d-1.parquet', `${window.location.origin}${base}/d-1.parquet`, 4, false);
		const conn = await db.connect();
		return conn;
	}

	let conn: AsyncDuckDBConnection | null = $state(null);

	onMount(async () => {
		try {
			conn = await load_db();
		} catch (e) {
			console.error('DuckDB error:', e);
		}
	});

	let ods: Od[] = $state([]);
	let fromZone: number | null = $state(null);
	let map: Map | undefined = $state(undefined);
	let selectedQuery: QueryKey = $state('total');

	$effect(() => {
		if (!map || fromZone === null) return;
		let rafId: number;
		let start: number | null = null;

		function pulse(ts: number) {
			if (start === null) start = ts;
			const t = ((ts - start) % 1200) / 1200;
			const opacity = 0.4 + 0.55 * Math.abs(Math.sin(t * Math.PI));
			map!.setPaintProperty('kzone-from', 'fill-opacity', opacity);
			rafId = requestAnimationFrame(pulse);
		}

		rafId = requestAnimationFrame(pulse);
		return () => cancelAnimationFrame(rafId);
	});

	let toZones = $derived(ods.map((od) => od.to));

	let toColorExpr: DataDrivenPropertyValueSpecification<ColorSpecification> = $derived(
		ods.length === 0
			? 'rgba(0,0,0,0)'
			: (() => {
					const max = Math.max(...ods.map((od) => od.count));
					const pairs = ods.flatMap((od) => {
						const t = max > 0 ? Math.log1p(od.count) / Math.log1p(max) : 0;
						const r = Math.round(173 * (1 - t));
						const g = Math.round(216 * (1 - t));
						const b = Math.round(230 * (1 - t) + 139 * t);
						return [od.to | 0, `rgb(${r},${g},${b})`];
					});
					return [
						'match',
						['get', 'kzone'],
						...pairs,
						'rgba(0,0,0,0)'
					] as unknown as ExpressionSpecification;
				})()
	);

	async function runQuery(kzone: number) {
		if (!conn) {
			console.error('DuckDB connection is not ready.');
			return;
		}
		try {
			const results = await conn.query(buildQuery(selectedQuery, kzone));
			const rows = results.toArray().map((r) => (r as JsonRow).toJSON());
			const mapped = rows
				.map(
					(r): Od => ({
						from: Number(r['発地']),
						to: Number(r['着地']),
						purpose: String(r['目的種類']),
						count: Number(r['計'])
					})
				)
				.filter(
					(od) => Number.isFinite(od.from) && Number.isFinite(od.to) && Number.isFinite(od.count)
				);
			const zoneMap = new Map<number, Od>();
			for (const od of mapped) {
				const existing = zoneMap.get(od.to);
				if (existing) {
					existing.count += od.count;
				} else {
					zoneMap.set(od.to, { ...od });
				}
			}
			ods = Array.from(zoneMap.values());
		} catch (err) {
			console.error('query error:', err);
		}
	}

	async function handleZoneClick(e: MapLayerMouseEvent) {
		const feature = e.features?.[0];
		if (!feature) return;
		const kzone = Number(feature.properties.kzone);
		fromZone = kzone;
		await runQuery(kzone);
	}

	async function selectQuery(key: QueryKey) {
		selectedQuery = key;
		if (fromZone !== null) {
			await runQuery(fromZone);
		}
	}
</script>

<PMTilesProtocol />
<MapLibre
	bind:map
	class="h-dvh"
	zoom={10}
	center={[139.762612, 35.67822]}
	style={{ version: 8, sources: {}, layers: [] }}
>
	<VectorTileSource
		id="kzone"
		url="{`pmtiles://${base}/H30_kzone.pmtiles`}"
		attribution="<a href='https://www.tokyo-pt.jp/data/01_01' target='_blank'>H30年東京都市圏パーソントリップ調査データ</a>"
	>
		<FillLayer
			id="kzone-fill"
			sourceLayer="H30_kzone"
			paint={{ 'fill-color': '#FBF6D9', 'fill-opacity': 0.7 }}
			onclick={handleZoneClick}
		/>
		<LineLayer
			id="kzone-line"
			sourceLayer="H30_kzone"
			paint={{ 'line-color': '#C7A317', 'line-opacity': 0.4 }}
		/>
		<FillLayer
			id="kzone-to"
			sourceLayer="H30_kzone"
			filter={toZones.length > 0
				? ['in', ['get', 'kzone'], ['literal', toZones]]
				: ['==', ['id'], -1]}
			paint={{ 'fill-color': toColorExpr, 'fill-opacity': 0.95 }}
		/>
		{#if fromZone !== null}
			<FillLayer
				id="kzone-from"
				sourceLayer="H30_kzone"
				filter={['==', ['get', 'kzone'], fromZone]}
				paint={{ 'fill-color': '#f0ffff', 'fill-opacity': 0.8 }}
			/>
		{/if}
	</VectorTileSource>
</MapLibre>

<div class="pointer-events-none absolute top-4 left-4 flex gap-2">
	{#each Object.entries(queryLabels) as [key, label]}
		<button
			class="text-m pointer-events-auto rounded-xl px-5 py-2 font-bold shadow transition-colors {selectedQuery ===
			key
				? 'bg-blue-700 text-white'
				: 'bg-white text-gray-700 hover:bg-gray-100'}"
			onclick={() => selectQuery(key as QueryKey)}
		>
			{label}
		</button>
	{/each}
</div>
