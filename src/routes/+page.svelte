<script>
	import mapboxgl from "mapbox-gl";
	import "../../node_modules/mapbox-gl/dist/mapbox-gl.css";
	import { onMount } from "svelte";
	import * as d3 from "d3";

	mapboxgl.accessToken = "pk.eyJ1Ijoia2NuYWthbXUiLCJhIjoiY21vMWh5b21wMGpraDJwb2dqa2VxOXZyaiJ9.5lakJOlvNf_vRlvpSLiIqg";

	let map;
	let stations = [];
	let trips;
	let departures;
	let arrivals;

	async function initMap() {
		// We assign a value to the `map` variable here
		// but the variable should be defined outside the 
		// scope of this function 
		map = new mapboxgl.Map({
			container: 'map',
			center: [-71.09415, 42.36027],
			zoom: 12,
			style: "mapbox://styles/mapbox/streets-v12",
		});
		await new Promise(resolve => map.on("load", resolve));
		map.addSource("boston_route", {
			type: "geojson",
			data: "https://bostonopendata-boston.opendata.arcgis.com/datasets/boston::existing-bike-network-2022.geojson",
		});

		map.addLayer({
			id: "boston_bike", // A name for our layer (up to you)
			type: "line", // one of the supported layer types, e.g. line, circle, etc.
			source: "boston_route", // The id we specified in `addSource()`
			paint: {
				"line-color": "#2ca25f",
				"line-width": 3,
				"line-opacity": 0.4
			},
		});

		map.addSource("cambridge_route", {
			type: "geojson",
			data: "https://raw.githubusercontent.com/cambridgegis/cambridgegis_data/main/Recreation/Bike_Facilities/RECREATION_BikeFacilities.geojson",
		});

		map.addLayer({
			id: "cambridge_bike", // A name for our layer (up to you)
			type: "line", // one of the supported layer types, e.g. line, circle, etc.
			source: "cambridge_route", // The id we specified in `addSource()`
			paint: {
				"line-color": "#2ca25f",
				"line-width": 3,
				"line-opacity": 0.4
			},
		});
	}

	async function getStations() {
		stations = await d3.csv("https://vis-society.github.io/labs/9/data/bluebikes-stations.csv")
		trips = await d3.csv("https://vis-society.github.io/labs/9/data/bluebikes-traffic-2024-03.csv")
				// console.log(trips)
		departures = d3.rollup(trips, v => v.length, d => d.start_station_id);
		arrivals = d3.rollup(trips, v => v.length, d => d.end_station_id);
		let totalTraffic = new Map();
		for (let [stationId, depCount] of departures) {
			totalTraffic.set(stationId, depCount + (arrivals.get(stationId) || 0));
		}

		for (let [stationId, arrCount] of arrivals) {
			if (!totalTraffic.has(stationId)) {
				totalTraffic.set(stationId, arrCount);
			}
		}

		stations = stations.map(station => {
			let id = station.Number;
			station.arrivals = arrivals.get(id) ?? 0;
			station.departures = departures.get(id) ?? 0;
			station.totalTraffic = totalTraffic.get(id) ?? 0;
			return station;
		});
	}



	function getCoords (station) {
		let point = new mapboxgl.LngLat(+station.Long, +station.Lat);
		let {x, y} = map.project(point);
		return {cx: x, cy: y};
	}


	onMount(() => {
		initMap();
		getStations();
	})

	let mapViewChanged = 0;
	$: map?.on("move", evt => mapViewChanged++);

	$: radiusScale = d3.scaleSqrt()
		.domain([0, d3.max(stations, d => d.totalTraffic) || 0])
		.range([0, 25]);

	let timeFilter = -1;
	$: timeFilterLabel = new Date(0, 0, 0, 0, timeFilter)
					.toLocaleString("en", {timeStyle: "short"});

</script>


<header>
	<h1>Bikewatching</h1>
	<!-- <p>This is an immersive, interactive map visualization of bike traffic in the Boston area during different times of the day.</p> -->
	<label>
        Filter by time:
        <input type="range" min="-1" max="1440" bind:value={timeFilter} />
        {#if timeFilter !== -1}
            <time style="display: block">
                {timeFilterLabel}
            </time>
        {:else}
            <em style="display: block">(any time)</em>
        {/if}
    </label>
</header>


<div id="map">
    <svg>
		{#key mapViewChanged}
			{#each stations as station}
				<circle { ...getCoords(station) } r={radiusScale(station.totalTraffic)} fill="steelblue" />
			{/each}
		{/key}
		
	</svg>
</div>







<style>
	@import url("$lib/global.css");
	#map {
		flex: 1;
		background-color: blue;
	}

	#map svg {
		position: absolute;
		z-index: 1;
		width: 100%;
		height: 100%;
		pointer-events: none;
	}

	#map circle {
		background-color: steelblue;
		fill-opacity: 60%;
		stroke: white;
	}

	header {
		display: flex;
		gap: 1em;
		align-items: baseline;
		margin-bottom: 20px;
	}

	label {
		margin-left: auto;
	}
</style>