<script lang="ts">
	import { DefaultMarker, MapLibre } from 'svelte-maplibre';
	import 'maplibre-gl/dist/maplibre-gl.css';

	// Props
	const {
		height = '400px',
		width = '100%',
		initialCenter = [-1.0678, 50.7979] as [number, number],
		initialZoom = 13,
		theme = 'light'
	} = $props<{
		height?: string;
		width?: string;
		initialCenter?: [number, number];
		initialZoom?: number;
		theme?: 'light' | 'dark';
	}>();

	// Environment variables
	const MAPTILER_KEY = import.meta.env.VITE_MAPTILER_KEY;

	// Map style based on theme
	const mapStyle = `https://api.maptiler.com/maps/streets-v2/style.json?key=${MAPTILER_KEY}`;


const markers: Array<{
  lngLat: [number, number];
  label: string;
  name: string;
}> = [
  {
    lngLat: [-1.0678, 50.7979],
    label: "CITY",
    name: "Portsmouth City Centre",
  },
  {
    lngLat: [-1.0955, 50.7952],
    label: "DOCK",
    name: "Portsmouth Historic Dockyard",
  },
  {
    lngLat: [-1.0873, 50.7891],
    label: "GUNS",
    name: "Portsmouth Museum of the Royal Navy",
  },
  {
    lngLat: [-1.1075, 50.7952],
    label: "SPINN",
    name: "Spinnaker Tower",
  },
  {
    lngLat: [-1.0941, 50.7934],
    label: "GUNW",
    name: "Gunwharf Quays",
  },
  {
    lngLat: [-1.0915, 50.7866],
    label: "SOUTH",
    name: "Portsmouth Southsea",
  },
  {
    lngLat: [-1.0638, 50.7989],
    label: "UNI",
    name: "University of Portsmouth",
  },
  {
    lngLat: [-1.0883, 50.7947],
    label: "VICT",
    name: "HMS Victory",
  }
];
</script>

<MapLibre style={mapStyle} center={initialCenter} zoom={initialZoom}>
  {#each markers as marker}
    <DefaultMarker lngLat={marker.lngLat} />
  {/each}
</MapLibre>
