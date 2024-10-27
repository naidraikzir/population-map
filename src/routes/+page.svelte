<script lang="ts">
	/* global GeoJSON */
	import type { Map as TMap } from 'maplibre-gl';
	import turfCenter from '@turf/center';
	import Map from '$lib/Map.svelte';

	const GEOJSON =
		'https://raw.githubusercontent.com/ans-4175/peta-indonesia-geojson/master/indonesia-prov.geojson';

	let map = $state<TMap>();
	let indonesiaGeoJSON = $state<GeoJSON.FeatureCollection>();

	$effect(() => {
		fetchIndonesiaGeoJSON();
	});

	$effect(() => {
		if (map && indonesiaGeoJSON) {
			const center = turfCenter(indonesiaGeoJSON as GeoJSON.FeatureCollection)?.geometry
				.coordinates as [number, number];
			map?.flyTo({
				speed: 1.5,
				bearing: 0,
				center,
				zoom: 4.5,
				essential: true,
				easing: (t) => (t < 0.5 ? 8 * t * t * t * t : 1 - 8 * --t * t * t * t)
			});
		}
	});

	function onMapReady(m: TMap) {
		map = m;
	}

	async function fetchIndonesiaGeoJSON() {
		const res = await fetch(GEOJSON);
		indonesiaGeoJSON = await res.json();
	}
</script>

<Map {onMapReady} />
