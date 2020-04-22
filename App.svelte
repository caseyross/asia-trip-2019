<map>
	<div id='cesium-root'>
	</div>
	<cite>
		globe tech and world heightmaps by <a href='https://cesium.com/cesiumjs/'>CESIUM</a><br>
		2d map imagery by <a href='https://www.mapbox.com/'>MAPBOX</a> & <a href='https://www.openstreetmap.org/about'>OPENSTREETMAP</a>
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
		background: black;
	}
	img {
		max-height: 100%;
		max-width: 100%;
	}
</style>

<script>
	import { onMount } from 'svelte'
	import photoData from './photoData.json'
	let viewer = {}
	const photosInView = []
	let imgUrl = null
	photoData.features.sort((a, b) => (a.properties.date < b.properties.date) - 0.5)
	Cesium.Ion.defaultAccessToken = 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJqdGkiOiI4NmZjNDk1ZS02Mjc1LTQ5YzgtODdhMy02MGMxMzA4MDg2MjIiLCJpZCI6MjIxOTcsInNjb3BlcyI6WyJhc3IiXSwiaWF0IjoxNTgyMzkyMDMxfQ.g2xsKoYIO5TwP2arlDG2YseVce2vhsuxfaWoWcEouNo' // steal my bus pass I dare you
	onMount(() => {
		viewer = new Cesium.Viewer(
			'cesium-root',
			{
				// 2d map tiles
				imageryProvider: new Cesium.MapboxStyleImageryProvider({
					username: 'caseyross',
					styleId: 'ck73luvkf18551irtcwd95kwm',
					accessToken: 'pk.eyJ1IjoiY2FzZXlyb3NzIiwiYSI6ImNrNzNpeTlyeDBka2EzbW11bWQ1Yno3bmwifQ.VF1A9UvUCKC8y1kFMEZ4pw'
				}),
				// heightmap
				terrainProvider: Cesium.createWorldTerrain(),
				// turn off lots of stuff we don't need
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
		// draw the points and lines on the map
		const now = Cesium.JulianDate.now()
		viewer.entities.suspendEvents()
		viewer.dataSources.add(
			Cesium.GeoJsonDataSource.load(
				photoData,
				{
					markerSize: 0,
				}
			)
		)
		let prevEntity = viewer.dataSources.get(0).entities.values[0]
		for (const entity of viewer.dataSources.get(0).entities.values) {
			entity.point = {
				pixelSize: 6,
				color: Cesium.Color.WHITE,
				outlineWidth: 1,
				outlineColor: Cesium.Color.BLACK,
    		}
			viewer.entities.add(
				new Cesium.Entity({
					polyline: {
						positions: [
							prevEntity.position.getValue(now),
							entity.position.getValue(now),
						],
						clampToGround: true,
						width: 2,
						material: Cesium.Color.RED
					}
				})
			)
			prevEntity = entity
		}
		viewer.entities.resumeEvents()
		// look at japan
		viewer.scene.camera.setView({
			destination:
				Cesium.Cartesian3.fromDegrees(
					135, // long
					35, // lat
					5000000 // height (m)
				)
		})
		// show the pic when we click on a point
		const eventHelper = new Cesium.EventHelper()
		eventHelper.add(viewer.selectedEntityChanged,
			() => 
				viewer.selectedEntity && viewer.selectedEntity.point
				? imgUrl = viewer.selectedEntity.properties.url.getValue(now)
				: imgUrl = null
		)
		// recalc ambient pics when we drag the map
		eventHelper.add(viewer.scene.camera.moveEnd,
			() => {
				const viewRectangle = viewer.scene.camera.computeViewRectangle(viewer.scene.globe.ellipsoid)
				const latView = [viewRectangle.west * 180 / Math.PI, viewRectangle.east * 180 / Math.PI]
				const lonView = [viewRectangle.south * 180 / Math.PI, viewRectangle.north * 180 / Math.PI]
				photosInView.length = 0;
				for (const photo of photoData.features) {
					if (
						photo.geometry.coordinates[0] > latView[0] &
						photo.geometry.coordinates[0] < latView[1] &
						photo.geometry.coordinates[1] > lonView[0] &
						photo.geometry.coordinates[1] < lonView[1]
					) {
						photosInView.push(photo)
					}
				}
		})
		// don't zoom in on double click
		viewer.screenSpaceEventHandler.removeInputAction(Cesium.ScreenSpaceEventType.LEFT_DOUBLE_CLICK)
	})
</script>