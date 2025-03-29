<script lang="ts">
  import maplibregl, { type Map as MapInstance, type GeoJSONSource, type StyleSpecification } from 'maplibre-gl';
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
  const mapStyle = 'https://api.maptiler.com/maps/streets-v2/style.json?key=faQwRnzRLoIsPK7HN5b4';

  // State
  let mapContainer: HTMLDivElement;
  let map = $state<MapInstance | null>(null);
  let dataSource = $state<GeoJSONSource | null>(null);

  // Methods for data visualization
  const addDataLayer = (geojsonData: GeoJSON.FeatureCollection) => {
    if (!map) return;

    // Remove existing source if it exists
    if (map.getSource('visualization-data')) {
      map.removeLayer('visualization-layer');
      map.removeSource('visualization-data');
    }

    // Add the data source
    map.addSource('visualization-data', {
      type: 'geojson',
      data: geojsonData
    });

    // Get the source for future updates
    dataSource = map.getSource('visualization-data') as GeoJSONSource;

    // Add a layer to visualize the data
    map.addLayer({
      id: 'visualization-layer',
      type: 'circle',
      source: 'visualization-data',
      paint: {
        'circle-radius': 6,
        'circle-color': '#007cbf',
        'circle-opacity': 0.7
      }
    });
  };

  // Method to update existing data
  const updateData = (geojsonData: GeoJSON.FeatureCollection) => {
    if (dataSource) {
      dataSource.setData(geojsonData);
    }
  };

  $effect(() => {
    if (mapContainer && !map) {
      map = new maplibregl.Map({
        container: mapContainer,
        style: mapStyle,
        center: initialCenter,
        zoom: initialZoom
      });

      // Add navigation controls (zoom in/out, compass)
      map.addControl(new maplibregl.NavigationControl(), 'top-right');

      // Add scale control
      map.addControl(new maplibregl.ScaleControl(), 'bottom-left');

      // Clean up on destroy
      return () => {
        map?.remove();
      };
    }
  });
</script>

<div 
  bind:this={mapContainer} 
  style="width: {width}; height: {height};"
  class="map-container"
/>

<style>
  .map-container {
    position: relative;
    border-radius: 4px;
    overflow: hidden;
  }

  /* Ensure the map container has a background while loading */
  .map-container::before {
    content: '';
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    background: #f0f0f0;
    z-index: -1;
  }
</style> 