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
  const mapStyle = `https://api.maptiler.com/maps/streets-v2/style.json?key=${MAPTILER_KEY}`;

  // State
  let mapContainer: HTMLDivElement;
  let map = $state<MapInstance | null>(null);
  let dataSource = $state<GeoJSONSource | null>(null);
  let trafficVisible = true; // New: track visibility

  // Methods for data visualization
  const getColorForCount = (count: number): string => {
    if (count > 100000) return '#d73027';
    if (count > 1000) return '#fc8d59';
    if (count > 0) return '#fee08b';
    return '#d9d9d9';
  };

  const getWidthForCount = (count: number): number => {
    if (count > 100000) return 6;
    if (count > 1000) return 4;
    if (count > 0) return 2;
    return 1;
  };

  const styleTrafficGeoJSON = (geojson: GeoJSON.FeatureCollection): GeoJSON.FeatureCollection => {
    return {
      ...geojson,
      features: geojson.features
        .filter(f => f.geometry?.type === 'LineString' && f.properties?.segmentProbeCounts?.[0]?.probeCount >= 0)
        .map(f => {
          const count = f.properties.segmentProbeCounts[0].probeCount;
          return {
            ...f,
            properties: {
              ...f.properties,
              probeCount: count,
              color: getColorForCount(count),
              width: getWidthForCount(count)
            }
          };
        })
    };
  };

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
      type: 'line',
      source: 'visualization-data',
      paint: {
        'line-color': ['get', 'color'],
        'line-width': ['get', 'width'],
        'line-opacity': 0.8
      },
      layout: {
        visibility: trafficVisible ? 'visible' : 'none'
      }
    });
  };

  // Toggle method for traffic layer
  const toggleTrafficLayer = () => {
    if (!map?.getLayer('visualization-layer')) return;
    trafficVisible = !trafficVisible;
    map.setLayoutProperty('visualization-layer', 'visibility', trafficVisible ? 'visible' : 'none');
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

      map.on('load', async () => {
        const res = await fetch('/data/portsmouth-traffic.geojson');
        const rawData: GeoJSON.FeatureCollection = await res.json();
        const styledGeoJSON = styleTrafficGeoJSON(rawData);
        addDataLayer(styledGeoJSON);
      });

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

<button on:click={toggleTrafficLayer} class="toggle-btn">
  {trafficVisible ? 'Hide Traffic Layer' : 'Show Traffic Layer'}
</button>

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

  .toggle-btn {
    margin-top: 0.5rem;
    padding: 0.4rem 0.8rem;
    font-size: 0.9rem;
    border-radius: 4px;
    background: #007cbf;
    color: white;
    border: none;
    cursor: pointer;
  }

  .toggle-btn:hover {
    background: #005e9c;
  }
</style>
