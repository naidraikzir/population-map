<script lang="ts">
	import maplibre, { type Map as TMap } from 'maplibre-gl';
	import 'maplibre-theme/icons.lucide.css';
	import 'maplibre-theme/modern.css';

	type Props = {
		onMapReady: (m: TMap) => void;
	};

	let div: HTMLDivElement;
	let { onMapReady }: Props = $props();
	let mapView = $state() as TMap;

	$effect(() => {
		if (div) {
			mapView = new maplibre.Map({
				container: 'map',
				style: 'https://tiles.openfreemap.org/styles/dark',
				center: [0, 0],
				zoom: 4.5,
				attributionControl: {
					compact: true
				}
			});
		}
	});

	$effect(() => {
		mapView.addControl(
			new maplibre.NavigationControl({
				visualizePitch: true
			})
		);

		mapView.on('load', () => {
			onMapReady(mapView);
		});
	});
</script>

<div id="map" class="dark h-dvh" bind:this={div}></div>
