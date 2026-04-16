<script>
	import mapboxgl from "mapbox-gl";
	import "../../node_modules/mapbox-gl/dist/mapbox-gl.css";
	import { onMount } from "svelte";
	import * as d3 from "d3";

	mapboxgl.accessToken = "pk.eyJ1Ijoia2NuYWthbXUiLCJhIjoiY21vMWh5b21wMGpraDJwb2dqa2VxOXZyaiJ9.5lakJOlvNf_vRlvpSLiIqg";

	let map;
	let stations = [];
	let trips = [];
	let departures;
	let arrivals;

	let stationFlow = d3.scaleQuantize()
		.domain([0, 1])
		.range([0, 0.5, 1]);

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
		trips = await d3.csv("https://vis-society.github.io/labs/9/data/bluebikes-traffic-2024-03.csv").then(trips => {
			for (let trip of trips) {
				trip.started_at = new Date(trip.started_at);
				trip.ended_at = new Date(trip.ended_at);
			}
			return trips;
		});
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

	function minutesSinceMidnight(date) {
		return date.getHours() * 60 + date.getMinutes();
	}

	$: filteredTrips = timeFilter === -1? trips : trips.filter(trip => {
		let startedMinutes = minutesSinceMidnight(trip.started_at);
		let endedMinutes = minutesSinceMidnight(trip.ended_at);
		return Math.abs(startedMinutes - timeFilter) <= 60
			|| Math.abs(endedMinutes - timeFilter) <= 60;
	});

	$: filteredDepartures = d3.rollup(filteredTrips, v => v.length, d => d.start_station_id);
	$: filteredArrivals = d3.rollup(filteredTrips, v => v.length, d => d.end_station_id);	

	$: filteredStations = stations.map(station => {
		const id = station.Number;
		const arr = filteredArrivals.get(id) ?? 0;
		const dep = filteredDepartures.get(id) ?? 0;
		return {
			...station,
			arrivals: arr,
			departures: dep,
			totalTraffic: arr + dep
		};
	});


	onMount(() => {
		initMap();
		getStations();
	})

	let mapViewChanged = 0;
	
	$: map?.on("move", evt => mapViewChanged++);

	$: radiusScale = d3.scaleSqrt()
		.domain([0, d3.max(filteredStations, d => d.totalTraffic) || 0])
   		.range(Number(timeFilter) === -1 ? [0, 25] : [3, 30]);

	let timeFilter = -1;
	$: timeFilterLabel = new Date(0, 0, 0, 0, timeFilter)
					.toLocaleString("en", {timeStyle: "short"});
	
	const urlBase = 'https://api.mapbox.com/isochrone/v1/mapbox/';
	const profile = 'cycling';
	const minutes = [5, 10, 15, 20];
	const contourColors = [
		"03045e",
		"0077b6",
		"00b4d8",
		"90e0ef"
	]
	let isochrone = null;

	async function getIso(lon, lat) {
		const base = `${urlBase}${profile}/${lon},${lat}`;
		const params = new URLSearchParams({
			contours_minutes: minutes.join(','),
			contours_colors: contourColors.join(','),
			polygons: 'true',
			access_token: mapboxgl.accessToken
		});
		const url = `${base}?${params.toString()}`;

		const query = await fetch(url, { method: 'GET' });
		isochrone = await query.json();
	}

	let selectedStation = null;

	function geoJSONPolygonToPath(feature) {
    const path = d3.path();
    const rings = feature.geometry.coordinates;

    for (const ring of rings) {
			for (let i = 0; i < ring.length; i++) {
				const [lng, lat] = ring[i];
				const { x, y } = map.project([lng, lat]);
				if (i === 0) path.moveTo(x, y);
				else path.lineTo(x, y);
			}
			path.closePath();
		}
		return path.toString();
	}


	$: if (selectedStation) {
		getIso(+selectedStation.Long, +selectedStation.Lat);
	} else {
		isochrone = null;
	}

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
			{#if isochrone}
				{#each isochrone.features as feature}
					<path
							d={geoJSONPolygonToPath(feature)}
							fill={feature.properties.fillColor}
							fill-opacity="0.2"
							stroke="#000000"
							stroke-opacity="0.5"
							stroke-width="1"
					>
						<title>{feature.properties.contour} minutes of biking</title>
					</path>
				{/each}
			{/if}
			{#each filteredStations as station}
				<circle { ...getCoords(station) } 
					r={radiusScale(station.totalTraffic)} 
					style="--departure-ratio: {stationFlow(
						station.totalTraffic ? station.departures / station.totalTraffic : 0.5
					)}"
					class={station?.Number === selectedStation?.Number ? "selected" : ""}
    					on:mousedown={() => selectedStation = selectedStation?.Number !== station?.Number ? station : null}
					/>
			{/each}
		{/key}
		
	</svg>
</div>

<div class="legend">
    <div style="--departure-ratio: 1">More departures</div>
    <div style="--departure-ratio: 0.5">Balanced</div>
    <div style="--departure-ratio: 0">More arrivals</div>
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

	#map svg circle {
		--color-departures: steelblue;
		--color-arrivals: darkorange;
		--color: color-mix(
			in oklch,
			var(--color-departures) calc(100% * var(--departure-ratio)),
			var(--color-arrivals)
		);
		fill: var(--color);
		transition: opacity 0.2s ease;
		pointer-events: auto;
	}


	#map svg:has(circle.selected) circle:not(.selected) {
		opacity: 0.3;
	}

	#map svg path {
		pointer-events: auto;
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


	.legend {
		display: flex;
		gap: 1px;
		margin-block: 1em;
	}

	.legend > div {
		--color-departures: steelblue;
		--color-arrivals: darkorange;
		--color: color-mix(
			in oklch,
			var(--color-departures) calc(100% * var(--departure-ratio)),
			var(--color-arrivals)
		);
		flex: 1;
		background: var(--color);
		padding: 0.5em 1.5em;
		color: white;
		font-size: 0.9rem;
	}

	.legend > div:first-child {
		text-align: left;
	}

	.legend > div:nth-child(2) {
		text-align: center;
	}

	.legend > div:last-child {
		text-align: right;
	}
	
</style>