<script lang="ts">
	import maplibre, { type Map as TMap } from 'maplibre-gl';
	import 'maplibre-gl/dist/maplibre-gl.css';

	let { onMapReady }: { onMapReady: (m: TMap) => void } = $props();
	let mapView = $state<TMap>();

	$effect(() => {
		if (document.querySelector('#map')) {
			mapView = new maplibre.Map({
				container: 'map',
				style: 'https://tiles.openfreemap.org/styles/liberty',
				center: [0, 0],
				zoom: 4.5,
				attributionControl: {
					compact: true
				}
			});
		}
	});

	$effect(() => {
		if (mapView) {
			mapView.addControl(
				new maplibre.NavigationControl({
					visualizePitch: true
				})
			);

			mapView.on('load', () => {
				onMapReady(mapView as TMap);
			});
		}
	});
</script>

<div id="map" class="h-dvh"></div>
