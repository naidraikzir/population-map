<script lang="ts">
	import type { FeatureCollection } from 'geojson';
	import type { Map as TMap, LngLatBoundsLike } from 'maplibre-gl';
	import { Popup } from 'maplibre-gl';
	import papaparse from 'papaparse';
	import turfBbox from '@turf/bbox';
	import Github from '$lib/Github.svelte';
	import Map from '$lib/Map.svelte';
	import csv from '$lib/populasi.csv?raw';

	type ProvinsiData = Record<string, string | number>;

	const indoGeoJsonUrl =
		'https://raw.githubusercontent.com/eppofahmi/geojson-indonesia/refs/heads/master/provinsi/all_maps_state_indo.geojson';

	const LAYER_ID = 'province-extrusion';
	const COLOR_DIVIDER = 75_000_000;
	const HEIGHT_DIVIDER = 50;

	let map = $state() as TMap;
	let indonesiaGeoJSON = $state<FeatureCollection>();
	let years = $state([]) as number[];
	let selectedYear = $state(0);
	let dataBuilt = $state(false);
	let readyToDraw = $derived(!!map && dataBuilt);
	let extrusionAdded = $state(false);
	let popup = $state() as Popup;
	let popupIsOpen = $state(false);
	let highlighted = $state({
		name: '',
		value: 0
	});

	$effect(() => {
		fetchIndonesiaGeoJSON();
	});

	$effect(() => {
		if (readyToDraw) {
			map?.on('moveend', addExtrusion);
			const bbox = turfBbox(indonesiaGeoJSON as FeatureCollection) as LngLatBoundsLike;
			map?.fitBounds(bbox, {
				speed: 1.5,
				pitch: 25,
				essential: true,
				easing: (t) => (t < 0.5 ? 8 * t * t * t * t : 1 - 8 * --t * t * t * t)
			});
		}
	});

	$effect(() => {
		if (extrusionAdded) {
			map?.setPaintProperty(LAYER_ID, 'fill-extrusion-color', [
				'rgb',
				['*', 150, ['/', ['get', `${selectedYear}`], COLOR_DIVIDER]],
				['-', 150, ['*', 150, ['/', ['get', `${selectedYear}`], COLOR_DIVIDER]]],
				0
			]);
			map?.setPaintProperty(LAYER_ID, 'fill-extrusion-height', [
				'/',
				['get', `${selectedYear}`],
				HEIGHT_DIVIDER
			]);
		}
	});

	$effect(() => {
		if (popupIsOpen) {
			updatePopup();
		}
	});

	function onMapReady(m: TMap) {
		map = m;
		initPopup();
	}

	async function fetchIndonesiaGeoJSON() {
		const res = await fetch(indoGeoJsonUrl);
		indonesiaGeoJSON = await res.json();
		buildData();
	}

	function buildData() {
		const { data: headers }: { data: string[][] } = papaparse.parse(csv);
		years = headers[0].slice(1).map((y) => Number(y));
		selectedYear = years[0];

		const { data }: { data: ProvinsiData[] } = papaparse.parse(csv, {
			header: true,
			dynamicTyping: true
		});

		indonesiaGeoJSON = {
			...indonesiaGeoJSON,
			features: indonesiaGeoJSON?.features.map((feature) => ({
				...feature,
				properties: {
					...feature.properties,
					...(data.find((d) => (d.Provinsi as string).toUpperCase() === feature.properties?.name) ||
						{})
				}
			}))
		} as FeatureCollection;

		dataBuilt = true;
	}

	function addExtrusion() {
		if (extrusionAdded) return;
		map?.addLayer({
			id: LAYER_ID,
			type: 'fill-extrusion',
			source: {
				type: 'geojson',
				data: indonesiaGeoJSON as FeatureCollection
			}
		});
		extrusionAdded = true;
		map?.on('mousemove', LAYER_ID, onMouseMove);
		map?.on('mouseleave', LAYER_ID, onMouseLeave);
	}

	function onMouseMove(e: any) {
		map.getCanvas().style.cursor = 'pointer';
		highlighted.name = e.features[0].properties.Provinsi;
		highlighted.value = e.features[0].properties[selectedYear];
		showPopup(highlighted.name, selectedYear, highlighted.value);
	}

	function onMouseLeave() {
		map.getCanvas().style.cursor = '';
		popup.remove();
	}

	function initPopup() {
		popup = new Popup({
			closeButton: false,
			closeOnClick: false,
			className: 'bg-black/25 p-3 rounded-md shadow-md backdrop-blur-lg text-white'
		})
			.trackPointer()
			.on('open', () => {
				popupIsOpen = true;
			})
			.on('close', () => {
				popupIsOpen = false;
			});
	}

	function showPopup(name: string, year: number, value: number) {
		popup
			.setHTML(
				`
					<div class="font-bold text-right text-xl">
						${name}
					</div>
					<div class="text-xs text-right mb-2">
						${year}
					</div>
					<div class="text-right text-sm">
						${Intl.NumberFormat().format(value)}
					</div>
				`
			)
			.setMaxWidth('auto')
			.addTo(map);
	}

	function updatePopup() {
		const provinsi = indonesiaGeoJSON?.features.find(
			(f) => f.properties?.Provinsi === highlighted.name
		);
		popup.setHTML(`
			<div class="font-bold text-right text-xl">
				${highlighted.name}
			</div>
			<div class="text-xs text-right mb-2">
				${selectedYear}
			</div>
			<div class="text-right text-sm">
				${Intl.NumberFormat().format(provinsi?.properties?.[selectedYear])}
			</div>
		`);
	}
</script>

<Map {onMapReady} />

<div
	class={'fixed bottom-[0.5rem] left-[0.5rem] z-10 w-60 ' +
		' flex flex-col justify-between gap-3 rounded-lg bg-black/25 p-4 pt-2 backdrop-blur-lg'}
	role="slider"
	aria-valuenow={selectedYear}
	tabindex={0}
	onclick={(e) => e.stopPropagation()}
	onkeydown={(e) => e.stopPropagation()}
>
	<div class="text-white drop-shadow-lg">{selectedYear}'s Population</div>
	<input
		type="range"
		bind:value={selectedYear}
		min={Math.min(...years)}
		max={Math.max(...years)}
		class={'h-1 w-full cursor-pointer appearance-none rounded-lg bg-white ' +
			' transition-all focus:outline-0 ' +
			' [&::-webkit-slider-thumb]:h-5 [&::-webkit-slider-thumb]:appearance-none ' +
			' [&::-webkit-slider-thumb]:w-5 [&::-webkit-slider-thumb]:rounded-full ' +
			' [&::-webkit-slider-thumb]:bg-white [&::-webkit-slider-thumb]:shadow-sm ' +
			' [&::-webkit-slider-thumb]:border-4 [&::-webkit-slider-thumb]:border-solid ' +
			' [&::-webkit-slider-thumb]:border-slate-400 [&::-webkit-slider-thumb]:shadow-slate-500'}
	/>
</div>

<a
	href="https://github.com/naidraikzir/population-map"
	class="fixed left-[0.5rem] top-[0.5rem] h-32 w-32 text-white"
>
	<Github />
</a>

<style lang="postcss">
	:global {
		.maplibregl-popup-content {
			background: transparent;
			box-shadow: none;
			padding: 0;
		}

		.maplibregl-popup-tip {
			display: none;
		}

		.maplibregl-ctrl-group {
			@apply bg-black/25 backdrop-blur-lg;
		}

		.maplibregl-ctrl-group button + button {
			border: none;
		}

		.maplibregl-ctrl-attrib.maplibregl-compact {
			@apply bg-black/25 backdrop-blur-lg;
		}

		.maplibregl-ctrl-attrib-inner {
			@apply text-white/50;
		}
	}
</style>
