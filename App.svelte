<map>
	<div id='cesium-root'>
	</div>
	<cite>
		globe tech by <a href='https://cesium.com/cesiumjs/'>CESIUM</a><br>
		map tiles by <a href='https://www.openstreetmap.org/about'>OPENSTREETMAP</a>
	</cite>
	<figure on:click={() => {imgUrl = null; viewer.selectedEntity = null}} style='visibility: {imgUrl ? "visible" : "hidden"}'>
		<img src='./photos/{imgUrl}'>
	</figure>
</map>

<style>
	map {
		display: block;
		height: 100%;
	}
	#cesium-root {
		height: 100%;
	}
	cite {
		position: absolute;
		bottom: 0;
		right: 0;
		color: white;
		font-size: 12px;
		line-height: 1.5;
		padding: 8px 16px;
		text-align: right;
	}
	figure {
		position: absolute;
		top: 0;
		height: 100%;
		width: 100%;
		display: flex;
		justify-content: center;
		align-items: center;
		background: gray;
	}
	img {
		max-height: 100%;
		max-width: 100%;
	}
</style>

<script>
	let viewer = {}
	let imgUrl = null
	import photoLocations from './photo-locations.json'
	photoLocations.features.sort((a, b) => (a.properties.date < b.properties.date) - 0.5)
	Cesium.Ion.defaultAccessToken = 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJqdGkiOiIzZmM1ZTU4Zi0yZDc3LTQxYWQtOWFhNS1jMmM3NGJkYTRlNjMiLCJpZCI6MjIxOTcsInNjb3BlcyI6WyJhc3IiXSwiYXNzZXRzIjpbMV0sImlhdCI6MTU4MjMyODk2MX0.wB2frl_bf8vBKYj8r3Gzsxn8IvN88qHodTT9PCQBufE' // steal my bus pass I dare you
	import { onMount } from 'svelte'
	onMount(() => {
		viewer = new Cesium.Viewer(
			'cesium-root',
			{
				imageryProvider: new Cesium.OpenStreetMapImageryProvider(),
				terrainProvider: Cesium.createWorldTerrain(),
				skyAtmosphere: false,
				animation: false,
				baseLayerPicker : false,
				fullscreenButton: false,
				homeButton: false,
				navigationHelpButton: false,
				sceneModePicker: false,
				timeline: false,
				geocoder: false,
				selectionIndicator: false,
				infoBox: false
			}
		)
		const now = Cesium.JulianDate.now()
		viewer.entities.suspendEvents()
		viewer.dataSources.add(
			Cesium.GeoJsonDataSource.load(
				photoLocations,
				{
					markerSize: 0,
				}
			)
		)
		let prevEntity = viewer.dataSources.get(0).entities.values[0]
		for (const entity of viewer.dataSources.get(0).entities.values) {
			entity.point = {
				pixelSize: 6,
				color: Cesium.Color.RED,
				outlineWidth: 1,
				outlineColor: Cesium.Color.BLACK
    		}
			viewer.entities.add(
				new Cesium.Entity({
					polyline: {
						positions: [
							prevEntity.position.getValue(now),
							entity.position.getValue(now),
						],
						clampToGround: true,
						width: 3,
						material: Cesium.Color.RED
					}
				})
			)
			prevEntity = entity
		}
		viewer.entities.resumeEvents()
		viewer.scene.camera.setView({
			destination:
				Cesium.Cartesian3.fromDegrees(
					135, // long
					35, // lat
					5000000 // height (m)
				)
		})
		const eventHelper = new Cesium.EventHelper()
		eventHelper.add(viewer.selectedEntityChanged,
			() => 
				viewer.selectedEntity && viewer.selectedEntity.point
				? imgUrl = viewer.selectedEntity.properties.url.getValue(now)
				: imgUrl = null
		)
		viewer.screenSpaceEventHandler.removeInputAction(Cesium.ScreenSpaceEventType.LEFT_DOUBLE_CLICK)
	})
</script>