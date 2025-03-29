<script lang="ts">
  import maplibregl, { type Map as MapInstance, type GeoJSONSource } from 'maplibre-gl';
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
  let trafficVisible = true;
  let airQualityVisible = true;

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
          const count = f.properties?.segmentProbeCounts[0].probeCount;
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

    if (map.getSource('visualization-data')) {
      map.removeLayer('visualization-layer');
      map.removeSource('visualization-data');
    }

    map.addSource('visualization-data', {
      type: 'geojson',
      data: geojsonData
    });

    dataSource = map.getSource('visualization-data') as GeoJSONSource;

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

  const toggleTrafficLayer = () => {
    if (!map?.getLayer('visualization-layer')) return;
    trafficVisible = !trafficVisible;
    map.setLayoutProperty('visualization-layer', 'visibility', trafficVisible ? 'visible' : 'none');
  };

  const toggleAirQualityLayer = () => {
    if (!map?.getLayer('air-quality-layer')) return;
    airQualityVisible = !airQualityVisible;
    map.setLayoutProperty('air-quality-layer', 'visibility', airQualityVisible ? 'visible' : 'none');
  };

  // Custom map control for toggling layers
  class ToggleButtonControl {
    private _map!: MapInstance;
    private _btn!: HTMLButtonElement;
    private _icon: string;
    private _title: string;
    private _toggleFn: () => void;

    constructor(icon: string, title: string, toggleFn: () => void) {
      this._icon = icon;
      this._title = title;
      this._toggleFn = toggleFn;
    }

    onAdd(map: MapInstance): HTMLElement {
      this._map = map;
      this._btn = document.createElement('button');
      this._btn.className = 'maplibregl-ctrl-icon';
      this._btn.type = 'button';
      this._btn.textContent = this._icon;
      this._btn.title = this._title;
      this._btn.onclick = () => this._toggleFn();

      const container = document.createElement('div');
      container.className = 'maplibregl-ctrl maplibregl-ctrl-group';
      container.appendChild(this._btn);
      return container;
    }

    onRemove(): void {
      this._btn?.parentNode?.removeChild(this._btn);
    }
  }

  // Air quality mock data
  type AirQualityStatus = 'good' | 'moderate' | 'poor';

  const mockAirQualityData: GeoJSON.FeatureCollection = {
    type: 'FeatureCollection',
    features: [
      {
        type: 'Feature',
        properties: { status: 'good' as AirQualityStatus, name: 'Southsea Common' },
        geometry: { type: 'Point', coordinates: [-1.091, 50.782] }
      },
      {
        type: 'Feature',
        properties: { status: 'poor' as AirQualityStatus, name: 'Commercial Road' },
        geometry: { type: 'Point', coordinates: [-1.087, 50.800] }
      },
      {
        type: 'Feature',
        properties: { status: 'moderate' as AirQualityStatus, name: 'Eastern Road' },
        geometry: { type: 'Point', coordinates: [-1.054, 50.825] }
      },
      {
        type: 'Feature',
        properties: { status: 'poor' as AirQualityStatus, name: 'Cosham High Street' },
        geometry: { type: 'Point', coordinates: [-1.067, 50.841] }
      },
      {
        type: 'Feature',
        properties: { status: 'poor', name: 'M275 Northern End (near Tipner)' },
        geometry: { type: 'Point', coordinates: [-1.0915, 50.8286] }
      },
      {
        type: 'Feature',
        properties: { status: 'poor', name: 'M275 South (near Rudmore Roundabout)' },
        geometry: { type: 'Point', coordinates: [-1.0849, 50.8134] }
      },
      {
        type: 'Feature',
        properties: { status: 'moderate', name: 'Fratton Bridge' },
        geometry: { type: 'Point', coordinates: [-1.0723, 50.7945] }
      },
      {
        type: 'Feature',
        properties: { status: 'moderate', name: 'North End (London Road)' },
        geometry: { type: 'Point', coordinates: [-1.0779, 50.8149] }
      },
      {
        type: 'Feature',
        properties: { status: 'good', name: 'Milton Park' },
        geometry: { type: 'Point', coordinates: [-1.0630, 50.7910] }
      },
      {
        type: 'Feature',
        properties: { status: 'good', name: 'Hilsea Lines Nature Reserve' },
        geometry: { type: 'Point', coordinates: [-1.0731, 50.8281] }
      },
    ]
  };

  const addAirQualityLayer = () => {
    if (!map) return;

    if (map.getSource('air-quality')) {
      map.removeLayer('air-quality-layer');
      map.removeSource('air-quality');
    }

    map.addSource('air-quality', {
      type: 'geojson',
      data: mockAirQualityData
    });

    map.addLayer({
      id: 'air-quality-layer',
      type: 'circle',
      source: 'air-quality',
      paint: {
        'circle-radius': 20,
        'circle-color': [
          'match',
          ['get', 'status'],
          'good', '#2ECC71',
          'moderate', '#F1C40F',
          'poor', '#E74C3C',
          '#bdc3c7'
        ],
        'circle-stroke-color': '#333',
        'circle-stroke-width': 1.5,
        'circle-opacity': 0.85
      },
      layout: {
        visibility: airQualityVisible ? 'visible' : 'none'
      }
    });
  };

  $effect(() => {
    if (mapContainer && !map) {
      map = new maplibregl.Map({
        container: mapContainer,
        style: mapStyle,
        center: initialCenter,
        zoom: initialZoom
      });

      map.addControl(new maplibregl.NavigationControl(), 'top-right');
      map.addControl(new maplibregl.ScaleControl(), 'bottom-left');

      map.on('load', async () => {
        const res = await fetch('/data/portsmouth-traffic.geojson');
        const rawData: GeoJSON.FeatureCollection = await res.json();
        const styledGeoJSON = styleTrafficGeoJSON(rawData);
        addDataLayer(styledGeoJSON);
        addAirQualityLayer();

        map?.addControl(new ToggleButtonControl('🚦', 'Toggle traffic layer', toggleTrafficLayer), 'top-right');
        map?.addControl(new ToggleButtonControl('🟢', 'Toggle air quality layer', toggleAirQualityLayer), 'top-right');
      });

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

  .maplibregl-ctrl-icon {
    font-size: 16px;
    padding: 2px;
  }
</style>
