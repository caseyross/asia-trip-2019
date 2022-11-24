<div id='main'>
	<map id='cesium-root'>
	</map>
	<nav bind:this={thumbnailContainer}>
		{#each photosInView as photo}
			<img class='thumbnail-image' style='width: {thumbnailSize}px; height: {thumbnailSize}px' src="./photos/{thumbnailSize > 191 ? 'full' : 'thumb'}/{photo.properties.url}" on:click={() => selectedPhoto = photo.properties.url}>
		{/each}
		{#if selectedPhoto}
			<figure on:click={() => {selectedPhoto = ''; viewer.selectedEntity = null}}>
				<img src='./photos/full/{selectedPhoto}' id='closeup-image'>
			</figure>
		{/if}
	</nav>
	<cite>
		built with <a href='https://cesium.com/cesiumjs/'>cesium</a> and <a href='https://svelte.dev/'>svelte</a>
	</cite>
</div>

<style>
	#main {
		height: 100%;
		display: flex;
		color: white;
		font-size: 12px;
	}
	map {
		flex: 1 1 auto;
		height: 100%;
	}
	nav {
		position: relative;
		flex: 0 0 1008px;
		height: 100%;
		overflow: auto;
		background: #222;
	}
	.thumbnail-image {
		display: inline-block;
		vertical-align: top;
		box-sizing: border-box;
		border:	1px solid black;
		background: gray;
		object-fit: cover;
		cursor: zoom-in;
		opacity: 0.5;
	}
	.thumbnail-image:hover {
		opacity: 1;
	}
	figure {
		position: absolute;
		top: 0;
		height: 100%;
		width: 100%;
		display: flex;
		justify-content: center;
		align-items: center;
		background: #222;
	}
	#closeup-image {
		max-height: 100%;
		max-width: 100%;
		cursor: zoom-out;
	}
	cite {
		position: absolute;
		bottom: 0;
		left: 0;
		padding: 8px;
	}
</style>

<script>
	import { onMount } from 'svelte'
	import photoData from './photoData.geojson'
	const photosInView = []
	let thumbnailContainer = {}
	let thumbnailSize = 64
	let selectedPhoto = null
	let viewer = {}
	photoData.features.sort((a, b) => (a.properties.date > b.properties.date) - 0.5)
	Cesium.Ion.defaultAccessToken = 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJqdGkiOiI4NmZjNDk1ZS02Mjc1LTQ5YzgtODdhMy02MGMxMzA4MDg2MjIiLCJpZCI6MjIxOTcsInNjb3BlcyI6WyJhc3IiXSwiaWF0IjoxNTgyMzkyMDMxfQ.g2xsKoYIO5TwP2arlDG2YseVce2vhsuxfaWoWcEouNo' // steal my bus pass I dare you
	onMount(() => {
		viewer = new Cesium.Viewer(
			'cesium-root',
			{
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
		// draw the points on the map
		viewer.entities.suspendEvents()
		viewer.dataSources.add(
			Cesium.GeoJsonDataSource.load(
				photoData,
				{
					markerSize: 0
				}
			)
		)
		for (const entity of viewer.dataSources.get(0).entities.values) {
			entity.point = {
				scaleByDistance: new Cesium.NearFarScalar(0, 1, 10000, 0.25),
				pixelSize: 20,
				color: Cesium.Color.RED,
				outlineWidth: 1,
				outlineColor: Cesium.Color.BLACK,
    		}
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
		let prevSelectedEntity = {
			point: {}
		}
		const now = Cesium.JulianDate.now()
		const eventHelper = new Cesium.EventHelper()
		eventHelper.add(viewer.selectedEntityChanged,
			() => {
				prevSelectedEntity.point.color = Cesium.Color.BLACK
				prevSelectedEntity.point.outlineColor = Cesium.Color.WHITE
				if(viewer.selectedEntity && viewer.selectedEntity.point) {
					selectedPhoto = viewer.selectedEntity.properties.url.getValue(now)
					viewer.selectedEntity.point.color = Cesium.Color.YELLOW
					prevSelectedEntity = viewer.selectedEntity
				} else {
					selectedPhoto = ''
				}
			}
		)
		// recalc ambient pics & thumbnail size when we drag the map
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
				let numThumbnailsInRow = 0;
				do {
					numThumbnailsInRow ++;
					thumbnailSize = Math.floor(thumbnailContainer.clientWidth / numThumbnailsInRow);
				} while(Math.floor(thumbnailContainer.clientWidth / thumbnailSize) * Math.floor(thumbnailContainer.clientHeight / thumbnailSize) < photosInView.length)
		})
		// don't zoom in on double click
		viewer.screenSpaceEventHandler.removeInputAction(Cesium.ScreenSpaceEventType.LEFT_DOUBLE_CLICK)
	})
</script>