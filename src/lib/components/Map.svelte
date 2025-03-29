<script lang="ts">
	import maplibregl, { type Map as MapInstance, type GeoJSONSource } from 'maplibre-gl';
	import 'maplibre-gl/dist/maplibre-gl.css';
  import boatIcon from '$lib/assets/boat-icon.svg';

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
  let trafficVisible = $state(true);
  let airQualityVisible = $state(true);
  let routeVisible = $state(true);
	let boatVisible = $state(true);
	let heatmapVisible = $state(true);
	let topoVisible = $state(true);
	let riskZonesVisible = $state(true);
	let isFlying = $state(false);
	let routeCoordinates = $state<[number, number][]>([]);

	// Route waypoints
	const routeWaypoints = [
		[-1.0732275, 50.8093439],
		[-1.0620259, 50.7924269],
		[-1.0930822, 50.7802554]
	] as [number, number][];

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
				.filter(
					(f) =>
						f.geometry?.type === 'LineString' &&
						f.properties?.segmentProbeCounts?.[0]?.probeCount >= 0
				)
				.map((f) => {
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
		map.setLayoutProperty(
			'air-quality-layer',
			'visibility',
			airQualityVisible ? 'visible' : 'none'
		);
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
				geometry: { type: 'Point', coordinates: [-1.087, 50.8] }
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
				geometry: { type: 'Point', coordinates: [-1.063, 50.791] }
			},
			{
				type: 'Feature',
				properties: { status: 'good', name: 'Hilsea Lines Nature Reserve' },
				geometry: { type: 'Point', coordinates: [-1.0731, 50.8281] }
			}
		]
	};

	const mockBoatData: GeoJSON.FeatureCollection = {
		type: 'FeatureCollection',
		features: [
			{
				type: 'Feature',
				properties: {
					orientation: 45  // Heading northeast
				},
				geometry: {
					coordinates: [-1.1046954744105904, 50.78089568162707],
					type: 'Point'
				}
			},
			{
				type: 'Feature',
				properties: {
					orientation: 180  // Heading south
				},
				geometry: {
					coordinates: [-1.1128571003072807, 50.8043329453744],
					type: 'Point'
				}
			},
			{
				type: 'Feature',
				properties: {
					orientation: 270  // Heading west
				},
				geometry: {
					coordinates: [-1.0949586313444968, 50.81135748920212],
					type: 'Point'
				}
			}
		]
	};

	const addBoatLayer = () => {
		if (!map) return;

		map.addSource('boat-data', {
			type: 'geojson',
			data: mockBoatData
		});

		map.addLayer({
			id: 'boat-layer',
			type: 'symbol',
			source: 'boat-data',
			layout: {
				'icon-image': 'boat-icon',
				'icon-size': 1,
				'icon-allow-overlap': true,
				'icon-ignore-placement': true,
				'icon-rotate': ['get', 'orientation'],
				'icon-rotation-alignment': 'map'
			}
		});
	};

	const toggleBoatLayer = () => {
		if (!map?.getLayer('boat-layer')) return;
		boatVisible = !boatVisible;
		map.setLayoutProperty('boat-layer', 'visibility', boatVisible ? 'visible' : 'none');
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
					'good',
					'#2ECC71',
					'moderate',
					'#F1C40F',
					'poor',
					'#E74C3C',
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

	const toggleRouteLayer = () => {
		if (!map?.getLayer('route-layer')) return;
		routeVisible = !routeVisible;
		map.setLayoutProperty('route-layer', 'visibility', routeVisible ? 'visible' : 'none');
	};

	const fetchSnappedRoute = async () => {
		if (!map) return;

		const res = await fetch('https://api.openrouteservice.org/v2/directions/foot-walking/geojson', {
			method: 'POST',
			headers: {
				'Authorization': '5b3ce3597851110001cf62485c0d23547a23458d9833ec420c38f0fa',
				'Content-Type': 'application/json'
			},
			body: JSON.stringify({
				coordinates: [
					[-1.0732275,50.8093439],
					[-1.0620259,50.7924269],
					[-1.0930822,50.7802554]
				]
			})
		});

		const data = await res.json();
		
		// Store route coordinates for the fly-through
		if (data.features[0].geometry.type === 'LineString') {
			routeCoordinates = data.features[0].geometry.coordinates as [number, number][];
		}

		const geojson: GeoJSON.FeatureCollection = {
			type: 'FeatureCollection',
			features: [{
				type: 'Feature',
				properties: { name: 'Snapped walking route with waypoint' },
				geometry: data.features[0].geometry
			}]
		};

		if (map.getSource('route')) {
			(map.getSource('route') as GeoJSONSource).setData(geojson);
		} else {
			map.addSource('route', {
				type: 'geojson',
				data: geojson
			});

			map.addLayer({
				id: 'route-layer',
				type: 'line',
				source: 'route',
				paint: {
					'line-color': '#3498db',
					'line-width': 4,
					'line-dasharray': [2, 2]
				},
				layout: {
					visibility: routeVisible ? 'visible' : 'none'
				}
			});
		}
	};

	const mockRiskZones: GeoJSON.FeatureCollection = {
		type: 'FeatureCollection',
		features: [
			// M275 Corridor - Major pollution zone
			{
				type: 'Feature',
				properties: { 
					risk: 'severe',
					description: 'M275 Corridor - Heavy traffic pollution'
				},
				geometry: {
					type: 'Polygon',
					coordinates: [[
						[-1.0949, 50.8334],  // Outer ring
						[-1.0899, 50.8334],
						[-1.0849, 50.8284],
						[-1.0849, 50.8134],
						[-1.0899, 50.8084],
						[-1.0949, 50.8084],
						[-1.0999, 50.8134],
						[-1.0999, 50.8284],
						[-1.0949, 50.8334]
					]]
				}
			},
			// Commercial Road Area
			{
				type: 'Feature',
				properties: { 
					risk: 'high',
					description: 'Commercial Road - High urban pollution'
				},
				geometry: {
					type: 'Polygon',
					coordinates: [[
						[-1.0920, 50.8050],
						[-1.0820, 50.8050],
						[-1.0820, 50.7950],
						[-1.0920, 50.7950],
						[-1.0920, 50.8050]
					]]
				}
			},
			// Portsmouth Harbor Industrial Zone
			{
				type: 'Feature',
				properties: { 
					risk: 'moderate',
					description: 'Harbor Industrial Area'
				},
				geometry: {
					type: 'Polygon',
					coordinates: [[
						[-1.1020, 50.8150],
						[-1.0920, 50.8150],
						[-1.0920, 50.8050],
						[-1.1020, 50.8050],
						[-1.1020, 50.8150]
					]]
				}
			},
			// Fratton Area
			{
				type: 'Feature',
				properties: { 
					risk: 'moderate',
					description: 'Fratton - Urban pollution zone'
				},
				geometry: {
					type: 'Polygon',
					coordinates: [[
						[-1.0773, 50.7995],
						[-1.0673, 50.7995],
						[-1.0673, 50.7895],
						[-1.0773, 50.7895],
						[-1.0773, 50.7995]
					]]
				}
			},
			// Cosham High Street
			{
				type: 'Feature',
				properties: { 
					risk: 'high',
					description: 'Cosham - Traffic congestion zone'
				},
				geometry: {
					type: 'Polygon',
					coordinates: [[
						[-1.0720, 50.8460],
						[-1.0620, 50.8460],
						[-1.0620, 50.8360],
						[-1.0720, 50.8360],
						[-1.0720, 50.8460]
					]]
				}
			}
		]
	};

	const addRiskZonesLayer = () => {
		if (!map) return;

		if (map.getSource('risk-zones')) {
			map.removeLayer('risk-zones-fill');
			map.removeLayer('risk-zones-outline');
			map.removeLayer('risk-zones-label');
			map.removeSource('risk-zones');
		}

		map.addSource('risk-zones', {
			type: 'geojson',
			data: mockRiskZones
		});

		// Add fill layer
		map.addLayer({
			id: 'risk-zones-fill',
			type: 'fill',
			source: 'risk-zones',
			paint: {
				'fill-color': [
					'match',
					['get', 'risk'],
					'severe', 'rgba(178,24,43,0.5)',
					'high', 'rgba(239,138,98,0.5)',
					'moderate', 'rgba(253,219,199,0.5)',
					'rgba(247,247,247,0.5)'
				],
				'fill-opacity': 0.6
			},
			layout: {
				visibility: riskZonesVisible ? 'visible' : 'none'
			}
		});

		// Add outline layer
		map.addLayer({
			id: 'risk-zones-outline',
			type: 'line',
			source: 'risk-zones',
			paint: {
				'line-color': [
					'match',
					['get', 'risk'],
					'severe', '#b2182b',
					'high', '#ef8a62',
					'moderate', '#fddbc7',
					'#f7f7f7'
				],
				'line-width': 2,
				'line-dasharray': [2, 2]
			},
			layout: {
				visibility: riskZonesVisible ? 'visible' : 'none'
			}
		});

		// Add labels
		map.addLayer({
			id: 'risk-zones-label',
			type: 'symbol',
			source: 'risk-zones',
			layout: {
				'text-field': ['get', 'description'],
				'text-size': 12,
				'text-anchor': 'center',
				'text-justify': 'center',
				'text-offset': [0, 0],
				visibility: riskZonesVisible ? 'visible' : 'none'
			},
			paint: {
				'text-color': '#000000',
				'text-halo-color': '#ffffff',
				'text-halo-width': 1
			}
		});
	};

	const toggleRiskZonesLayer = () => {
		if (!map?.getLayer('risk-zones-fill')) return;
		riskZonesVisible = !riskZonesVisible;
		const visibility = riskZonesVisible ? 'visible' : 'none';
		const layerIds = ['risk-zones-fill', 'risk-zones-outline', 'risk-zones-label'];
		for (const layerId of layerIds) {
			if (map) {
				map.setLayoutProperty(layerId, 'visibility', visibility);
			}
		}
	};

	const startFlythrough = () => {
		if (!map || isFlying || routeCoordinates.length === 0) return;
		
		isFlying = true;
		let currentIndex = 0;
		const totalPoints = routeCoordinates.length;
		const pointsPerBatch = Math.max(1, Math.floor(totalPoints / 30)); // Divide route into ~30 segments
		
		const flyToNext = () => {
			if (!map || currentIndex >= totalPoints) {
				isFlying = false;
				return;
			}

			const point = routeCoordinates[currentIndex];
			
			// Calculate bearing to next point for camera orientation
			let bearing = 0;
			if (currentIndex < totalPoints - 1) {
				const nextPoint = routeCoordinates[currentIndex + 1];
				bearing = getBearing(point, nextPoint);
			}

			// Calculate appropriate zoom based on segment location
			const zoomLevel = getAppropriateZoom(point);

			map.flyTo({
				center: point,
				zoom: zoomLevel,
				bearing: bearing,
				speed: 0.25,
				curve: 1,
				essential: true
			});

			currentIndex += pointsPerBatch;
			setTimeout(flyToNext, 1000); // Adjust timing for smooth movement
		};

		flyToNext();
	};

	// Helper function to calculate bearing between two points
	const getBearing = (start: [number, number], end: [number, number]): number => {
		const startLat = toRad(start[1]);
		const startLng = toRad(start[0]);
		const endLat = toRad(end[1]);
		const endLng = toRad(end[0]);

		const dLng = endLng - startLng;

		const y = Math.sin(dLng) * Math.cos(endLat);
		const x = Math.cos(startLat) * Math.sin(endLat) -
				Math.sin(startLat) * Math.cos(endLat) * Math.cos(dLng);

		let bearing = toDeg(Math.atan2(y, x));
		bearing = (bearing + 360) % 360;

		return bearing;
	};

	const toRad = (deg: number): number => (deg * Math.PI) / 180;
	const toDeg = (rad: number): number => (rad * 180) / Math.PI;

	// Helper function to determine appropriate zoom level based on location
	const getAppropriateZoom = (point: [number, number]): number => {
		// Check if point is in a high-detail area (like city center)
		const isInDetailArea = mockRiskZones.features.some(zone => {
			if (zone.geometry.type === 'Polygon') {
				return isPointInPolygon(point, zone.geometry.coordinates[0]);
			}
			return false;
		});

		return isInDetailArea ? 16 : 15;
	};

	// Helper function to check if a point is inside a polygon
	const isPointInPolygon = (point: [number, number], polygon: [number, number][]): boolean => {
		let inside = false;
		for (let i = 0, j = polygon.length - 1; i < polygon.length; j = i++) {
			const xi = polygon[i][0], yi = polygon[i][1];
			const xj = polygon[j][0], yj = polygon[j][1];

			const intersect = ((yi > point[1]) !== (yj > point[1])) &&
				(point[0] < (xj - xi) * (point[1] - yi) / (yj - yi) + xi);
			if (intersect) inside = !inside;
		}
		return inside;
	};

	// Custom control for fly-through
	class FlyThroughControl {
		private _map!: MapInstance;
		private _btn!: HTMLButtonElement;
		private _container!: HTMLDivElement;
		private _unsubscribe: (() => void) | undefined;

		onAdd(map: MapInstance): HTMLElement {
			this._map = map;
			this._container = document.createElement('div');
			this._container.className = 'maplibregl-ctrl maplibregl-ctrl-group';
			
			this._btn = document.createElement('button');
			this._btn.className = 'maplibregl-ctrl-icon';
			this._btn.type = 'button';
			this._btn.innerHTML = '✈️';
			this._btn.title = 'Start fly-through';
			
			// Add loading state indicator
			const loadingSpinner = document.createElement('div');
			loadingSpinner.className = 'fly-through-loading';
			loadingSpinner.style.display = 'none';
			
			this._btn.onclick = () => {
				if (!isFlying) {
					startFlythrough();
					this._btn.style.backgroundColor = '#e0e0e0';
				}
			};
			
			// Update button state based on flying status
			this.updateButtonState();
			
			this._container.appendChild(this._btn);
			this._container.appendChild(loadingSpinner);
			
			return this._container;
		}

		private updateButtonState() {
			// Create a reactive effect outside the class method
			$effect.root(() => {
				if (isFlying) {
					this._btn.style.backgroundColor = '#e0e0e0';
					this._btn.title = 'Flying...';
				} else {
					this._btn.style.backgroundColor = '';
					this._btn.title = 'Start fly-through';
				}
			});
		}

		onRemove(): void {
			if (this._unsubscribe) {
				this._unsubscribe();
			}
			this._container.parentNode?.removeChild(this._container);
		}
	}

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

				// Load boat icon
				if (!map) return;
				const img = new Image();
				img.onload = () => {
					if (map && !map.hasImage('boat-icon')) {
						map.addImage('boat-icon', img);
						// Add layers after ensuring the icon is loaded
						addDataLayer(styledGeoJSON);
						addAirQualityLayer();
						addBoatLayer();
						addRiskZonesLayer();
						fetchSnappedRoute();
					}
				};
				img.src = boatIcon;

				map?.addControl(
					new ToggleButtonControl('🚦', 'Toggle traffic layer', toggleTrafficLayer),
					'top-right'
				);
				map?.addControl(
					new ToggleButtonControl('🟢', 'Toggle air quality layer', toggleAirQualityLayer),
					'top-right'
				);
				map?.addControl(
					new ToggleButtonControl('🚢', 'Toggle boat layer', toggleBoatLayer),
					'top-right'
				);
				map?.addControl(
					new ToggleButtonControl('🚶', 'Toggle route layer', toggleRouteLayer),
					'top-right'
				);
				map?.addControl(
					new ToggleButtonControl('⚠️', 'Toggle risk zones', toggleRiskZonesLayer),
					'top-right'
				);
				map?.addControl(
					new FlyThroughControl(),
					'top-right'
				);
			});

			return () => {
				map?.remove();
			};
		}
	});
</script>

<div bind:this={mapContainer} style="width: {width}; height: {height};" class="map-container" />

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

	/* Fly-through control styles */
	.fly-through-loading {
		position: absolute;
		top: 50%;
		left: 50%;
		transform: translate(-50%, -50%);
		width: 16px;
		height: 16px;
		border: 2px solid #f3f3f3;
		border-top: 2px solid #3498db;
		border-radius: 50%;
		animation: spin 1s linear infinite;
		display: none;
	}

	@keyframes spin {
		0% { transform: translate(-50%, -50%) rotate(0deg); }
		100% { transform: translate(-50%, -50%) rotate(360deg); }
	}
</style>
