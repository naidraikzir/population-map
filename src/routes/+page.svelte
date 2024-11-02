<script lang="ts">
	import type { FeatureCollection } from 'geojson';
	import type { Map as TMap } from 'maplibre-gl';
	import { Popup } from 'maplibre-gl';
	import papaparse from 'papaparse';
	import turfCenter from '@turf/center';
	import Map from '$lib/Map.svelte';
	import csv from '$lib/populasi.csv?raw';
	import { debounce } from '$lib/utils';

	const indoGeoJsonUrl =
		'https://raw.githubusercontent.com/eppofahmi/geojson-indonesia/refs/heads/master/provinsi/all_maps_state_indo.geojson';

	const LAYER_ID = 'province-extrusion';
	const COLOR_DIVIDER = 75_000_000;
	const HEIGHT_DIVIDER = 50;
	const TIMEOUT = 50;

	let map = $state<TMap>();
	let indonesiaGeoJSON = $state<FeatureCollection>();
	let years = $state<number[]>([]);
	let selectedYear = $state<number>();
	let dataBuilt = $state(false);
	let readyToDraw = $derived(!!map && dataBuilt);
	let extrusionAdded = $state(false);
	let popup = $state<Popup>();

	const debouncedPopup = debounce((name, year, center, value) => {
		if (!map || !popup) return;
		popup
			.setLngLat(center)
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
			.addTo(map);
	}, TIMEOUT);

	$effect(() => {
		fetchIndonesiaGeoJSON();
	});

	$effect(() => {
		if (readyToDraw) {
			const center = turfCenter(indonesiaGeoJSON as FeatureCollection)?.geometry.coordinates as [
				number,
				number
			];
			map?.on('moveend', addExtrusion);
			map?.flyTo({
				speed: 1.5,
				pitch: 25,
				center,
				zoom: 4.5,
				essential: true,
				easing: (t) => (t < 0.5 ? 8 * t * t * t * t : 1 - 8 * --t * t * t * t)
			});
		}
	});

	$effect(() => {
		if (extrusionAdded) {
			map?.setPaintProperty(LAYER_ID, 'fill-extrusion-color', [
				'rgb',
				['*', 255, ['/', ['get', `${selectedYear}`], COLOR_DIVIDER]],
				['-', 255, ['*', 255, ['/', ['get', `${selectedYear}`], COLOR_DIVIDER]]],
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
		popup = new Popup({
			closeButton: false,
			closeOnClick: false,
			className: 'bg-black/25 p-3 rounded-md shadow-md backdrop-blur-lg text-white'
		});
	});

	function onMapReady(m: TMap) {
		map = m;
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

		const { data }: { data: { Provinsi: string }[] } = papaparse.parse(csv, {
			header: true,
			dynamicTyping: true
		});

		indonesiaGeoJSON = {
			...indonesiaGeoJSON,
			features: indonesiaGeoJSON?.features.map((feature) => ({
				...feature,
				properties: {
					...feature.properties,
					...(data.find((d) => d.Provinsi.toUpperCase() === feature.properties?.name) || {})
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
			},
			paint: {
				'fill-extrusion-color': [
					'rgb',
					['*', 255, ['/', ['get', `${selectedYear}`], COLOR_DIVIDER]],
					['-', 255, ['*', 255, ['/', ['get', `${selectedYear}`], COLOR_DIVIDER]]],
					0
				],
				'fill-extrusion-height': ['/', ['get', `${selectedYear}`], HEIGHT_DIVIDER],
				'fill-extrusion-base': 100,
				'fill-extrusion-opacity': 0.75
			}
		});
		map?.on('mousemove', LAYER_ID, onMouseMove);
		map?.on('mouseleave', LAYER_ID, onMouseLeave);
		extrusionAdded = true;
	}

	function onMouseMove(e: any) {
		if (!map || !popup || !selectedYear) return;
		map.getCanvas().style.cursor = 'pointer';
		const name = e.features[0].properties.Provinsi;
		const value = e.features[0].properties[selectedYear];
		const center = turfCenter(e.features[0].geometry as FeatureCollection)?.geometry
			.coordinates as [number, number];
		debouncedPopup(name, selectedYear, center, value);
	}

	function onMouseLeave() {
		if (!map) return;
		map.getCanvas().style.cursor = '';
		setTimeout(() => {
			if (!popup) return;
			popup.remove();
		}, TIMEOUT + 50);
	}
</script>

<Map {onMapReady} />

<div
	class={'fixed bottom-4 left-4 z-10 w-60 ' +
		' flex flex-col justify-between gap-3 rounded-lg bg-black/25 p-4 pt-2 backdrop-blur-lg'}
>
	<div class="text-white drop-shadow-lg">{selectedYear}</div>
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
			' [&::-webkit-slider-thumb]:border-4 [&::-webkit-slider-thumb]:border-solid [&::-webkit-slider-thumb]:border-slate-400 [&::-webkit-slider-thumb]:shadow-slate-500'}
	/>
</div>

<style>
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
			background: rgba(0, 0, 0, 0.5);
			backdrop-filter: blur(1em);
		}

		.maplibregl-ctrl-group button + button {
			border: none;
		}

		.maplibregl-ctrl-attrib.maplibregl-compact {
			background: rgba(0, 0, 0, 0.5);
			backdrop-filter: blur(1em);
		}

		.maplibregl-ctrl-attrib-inner {
			color: rgba(255, 255, 255, 0.5);
		}
	}
</style>
