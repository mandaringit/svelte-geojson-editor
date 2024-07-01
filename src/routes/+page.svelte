<script lang="ts">
	import '../assets/custom.css';
	import { onMount } from 'svelte';

	let map: google.maps.Map;
	let container$: HTMLDivElement;

	let drawingManager: google.maps.drawing.DrawingManager;
	let polygons: google.maps.Polygon[] = [];
	let rectangles: google.maps.Rectangle[] = [];
	let points: google.maps.Marker[] = [];

	let drawingMode: google.maps.drawing.OverlayType | null = null;

	let mapLoaded = false;

	$: {
		console.log(drawingMode);
	}

	$: {
		console.log(polygons);
	}

	onMount(async () => {
		const { Map } = (await google.maps.importLibrary('maps')) as google.maps.MapsLibrary;
		const { DrawingManager } = (await google.maps.importLibrary(
			'drawing'
		)) as google.maps.DrawingLibrary;

		mapLoaded = true;

		map = new Map(container$, {
			center: { lat: 37.7749, lng: -122.4194 },
			zoom: 8,
			disableDefaultUI: true,
			clickableIcons: false
		});

		drawingManager = new DrawingManager({
			drawingMode: google.maps.drawing.OverlayType.POLYGON,
			drawingControl: false,
			drawingControlOptions: {
				position: google.maps.ControlPosition.TOP_CENTER,
				drawingModes: [
					google.maps.drawing.OverlayType.POLYGON,
					// google.maps.drawing.OverlayType.RECTANGLE,
					google.maps.drawing.OverlayType.MARKER
					// google.maps.drawing.OverlayType.CIRCLE,
					// google.maps.drawing.OverlayType.POLYLINE,
				]
			},
			markerOptions: {
				// icon: 'https://developers.google.com/maps/documentation/javascript/examples/full/images/beachflag.png',
				draggable: true
			},
			// circleOptions: {
			// 	fillColor: '#ffff00',
			// 	fillOpacity: 1,
			// 	strokeWeight: 5,
			// 	clickable: false,
			// 	editable: true,
			// 	zIndex: 1
			// },
			polygonOptions: {
				fillColor: 'black',
				fillOpacity: 0.3,
				strokeWeight: 4,
				strokeColor: 'black',
				editable: true,
				zIndex: 1
			}
			// rectangleOptions: {
			// 	fillColor: 'black',
			// 	fillOpacity: 0.3,
			// 	strokeWeight: 4,
			// 	strokeColor: 'black',
			// 	editable: true,
			// 	zIndex: 1
			// }
		});

		google.maps.event.addListener(
			drawingManager,
			'overlaycomplete',
			(e: google.maps.drawing.OverlayCompleteEvent) => {
				if (e.type === google.maps.drawing.OverlayType.POLYGON) {
					const polygon = e.overlay as google.maps.Polygon;

					// google.maps.event.addListener(polygon.getPath(), 'set_at', () => {
					// 	console.log('set_at', polygon.getPath().getArray());
					// });

					// google.maps.event.addListener(polygon.getPath(), 'insert_at', () => {
					// 	console.log('insert_at', polygon.getPath().getArray());
					// });

					polygons = [...polygons, polygon as google.maps.Polygon];
				} else if (e.type === google.maps.drawing.OverlayType.RECTANGLE) {
					const rectangle = e.overlay as google.maps.Rectangle;

					rectangles = [...rectangles, rectangle as google.maps.Rectangle];
				} else if (e.type === google.maps.drawing.OverlayType.MARKER) {
					const point = e.overlay as google.maps.Marker;

					points = [...points, point as google.maps.Marker];
				}

				drawingManager.setDrawingMode(null);
			}
		);

		google.maps.event.addListener(drawingManager, 'drawingmode_changed', () => {
			drawingMode = drawingManager.getDrawingMode();
		});

		console.log(drawingManager.getDrawingMode());

		drawingManager.setDrawingMode(google.maps.drawing.OverlayType.POLYGON);
		drawingManager.setMap(map);
	});

	function uploadGeoJson() {
		const input = document.createElement('input');
		input.type = 'file';
		input.accept = '.geojson';

		input.click();
		input.onchange = async (e) => {
			const file = (e.target as HTMLInputElement).files?.[0];
			if (!file) return;

			const text = await file.text();
			const geoJson = JSON.parse(text);
			console.log(geoJson);
			geoJson.features.forEach((feature: any) => {
				if (feature.geometry.type === 'Polygon') {
					const path = feature.geometry.coordinates[0].map((coord: [number, number]) => {
						return { lat: coord[1], lng: coord[0] };
					});

					const polygon = new google.maps.Polygon({
						paths: path,
						fillColor: 'black',
						fillOpacity: 0.3,
						strokeWeight: 4,
						strokeColor: 'black',
						editable: true,
						zIndex: 1
					});

					polygon.setMap(map);
					polygons = [...polygons, polygon];
				} else if (feature.geometry.type === 'Point') {
					const position = {
						lat: feature.geometry.coordinates[1],
						lng: feature.geometry.coordinates[0]
					};

					const point = new google.maps.Marker({
						position,
						draggable: true
					});

					point.setMap(map);
					points = [...points, point];
				}
			});
		};
	}

	function exportGeoJson() {
		const geoJson = {
			type: 'FeatureCollection',
			features: []
		};

		const polygonFeatures = polygons.map((polygon) => {
			const path = polygon
				.getPath()
				.getArray()
				.map((latLng) => {
					return [latLng.lng(), latLng.lat()];
				})
				.concat([[polygon.getPath().getAt(0).lng(), polygon.getPath().getAt(0).lat()]]);

			return {
				type: 'Feature',
				properties: {},
				geometry: {
					type: 'Polygon',
					coordinates: [path]
				}
			};
		});

		const rectangleFeatures = rectangles.map((rectangle) => {
			const bounds = rectangle.getBounds();
			if (!bounds) return;

			const ne = bounds.getNorthEast();
			const sw = bounds.getSouthWest();

			const path = [
				[ne.lng(), ne.lat()],
				[sw.lng(), ne.lat()],
				[sw.lng(), sw.lat()],
				[ne.lng(), sw.lat()],
				[ne.lng(), ne.lat()]
			];

			return {
				type: 'Feature',
				properties: {},
				geometry: {
					type: 'Polygon',
					coordinates: [path]
				}
			};
		});

		const pointFeatures = points.map((point) => {
			const position = point.getPosition();
			if (!position) return;

			return {
				type: 'Feature',
				properties: {},
				geometry: {
					type: 'Point',
					coordinates: [position.lng(), position.lat()]
				}
			};
		});

		geoJson.features = [...polygonFeatures, ...rectangleFeatures, ...pointFeatures];

		const blob = new Blob([JSON.stringify(geoJson, undefined, 2)], { type: 'application/json' });
		const url = URL.createObjectURL(blob);

		const a = document.createElement('a');
		a.href = url;
		a.download = `export.geojson`;
		a.click();
	}

	function boundAll() {
		const bounds = new google.maps.LatLngBounds();
		polygons.forEach((polygon) => {
			polygon.getPath().forEach((latLng) => {
				bounds.extend(latLng);
			});
		});
		rectangles.forEach((rectangle) => {
			const bounds = rectangle.getBounds();
			if (!bounds) return;

			const ne = bounds.getNorthEast();
			const sw = bounds.getSouthWest();

			bounds.extend(ne);
			bounds.extend(sw);
		});

		points.forEach((point) => {
			const position = point.getPosition();
			if (!position) return;

			bounds.extend(position);
		});

		if (bounds.isEmpty()) {
			map.setCenter({ lat: 37.7749, lng: -122.4194 });
			map.setZoom(8);
		} else {
			map.fitBounds(bounds);
		}
	}
</script>

<div class="container">
	<div class="left">
		<div id="map" bind:this={container$}></div>

		{#if mapLoaded}
			<div class="buttons">
				<button
					class:on={drawingMode === google.maps.drawing.OverlayType.POLYGON}
					on:click={() => drawingManager.setDrawingMode(google.maps.drawing.OverlayType.POLYGON)}
				>
					Polygon
				</button>
				<!-- <button
				class:on={drawingMode === google.maps.drawing.OverlayType.RECTANGLE}
				on:click={() => drawingManager.setDrawingMode(google.maps.drawing.OverlayType.RECTANGLE)}
			>
				Rectangle
			</button> -->
				<button
					class:on={drawingMode === google.maps.drawing.OverlayType.MARKER}
					on:click={() => drawingManager.setDrawingMode(google.maps.drawing.OverlayType.MARKER)}
				>
					Point
				</button>
			</div>
		{/if}
	</div>
	<div class="right">
		<div>
			<button on:click={() => boundAll()}>전체 보기</button>
			<button
				on:click={() => {
					polygons.forEach((polygon) => {
						polygon.setMap(null);
					});
					rectangles.forEach((rectangle) => {
						rectangle.setMap(null);
					});
					points.forEach((point) => {
						point.setMap(null);
					});

					polygons = [];
					rectangles = [];
					points = [];
				}}>초기화</button
			>
			<button on:click={() => uploadGeoJson()}>Upload(.geojson)</button>
			<button on:click={() => exportGeoJson()}>Export</button>
		</div>
		<ul>
			{#each polygons as item, i}
				<li>
					Polygon {i}
					<button
						on:click={() => {
							item.setMap(null);
							polygons = polygons.filter((p) => p !== item);
						}}>제거</button
					>
					<button
						on:click={() => {
							item.setOptions({
								fillColor: 'red',
								strokeColor: 'red'
							});
						}}>색변경</button
					>
					<button
						on:click={() => {
							const bounds = new google.maps.LatLngBounds();
							item.getPath().forEach((latLng) => {
								bounds.extend(latLng);
							});
							map.fitBounds(bounds);
						}}>뷰</button
					>
				</li>
			{/each}
		</ul>
		<!-- <ul>
			{#each rectangles as item, i}
				<li>
					Rectangle {i}
					<button
						on:click={() => {
							item.setMap(null);
							rectangles = rectangles.filter((r) => r !== item);
						}}>제거</button
					>
					<button
						on:click={() => {
							const bounds = item.getBounds();
							if (!bounds) return;

							const ne = bounds.getNorthEast();
							const sw = bounds.getSouthWest();

							bounds.extend(ne);
							bounds.extend(sw);

							map.fitBounds(bounds);
						}}>뷰</button
					>
				</li>
			{/each}
		</ul> -->
		<ul>
			{#each points as item, i}
				<li>Point {i} <button>제거</button> <button>뷰</button></li>
			{/each}
		</ul>
	</div>
</div>

<style>
	.container {
		width: 100%;
		height: 100%;
		display: grid;
		grid-template-columns: 1fr 300px;
	}

	#map {
		width: 100%;
		height: 100%;
		margin: 0;
		padding: 0;
	}

	.left {
		position: relative;
	}

	.right {
		display: grid;
		grid-template-rows: 50px 1fr 1fr 50px;
	}

	.buttons {
		position: absolute;
		top: 10px;
		right: 10px;
		z-index: 1000;
	}

	button {
		background-color: white;
		border: 1px solid black;
		border-radius: 3px;
		padding: 5px;
		margin: 5px;
	}

	button.on {
		color: white;
		background-color: black;
	}
</style>
