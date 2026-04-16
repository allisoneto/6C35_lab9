<script>
    import mapboxgl from "mapbox-gl";
    import "../../node_modules/mapbox-gl/dist/mapbox-gl.css";
    import * as d3 from "d3";
    import { onMount } from "svelte";

    mapboxgl.accessToken = "pk.eyJ1IjoiYWxsaXNvbmV0byIsImEiOiJjbW8wcXRxNzkwYjduMndweXRoMGlvMnVjIn0.VuzO_BFOqhcY_2Tq543HEg";

    let map;
    let stations = [];
    let mapViewChanged = 0;
    let timeFilter = -1;
    let selectedStation = null;
    let isochrone = null;

    const departuresByMinute = Array.from({length: 1440}, () => []);
    const arrivalsByMinute = Array.from({length: 1440}, () => []);
    let allDepartures = new Map();
    let allArrivals = new Map();

    $: map?.on("move", () => mapViewChanged++);

    $: timeFilterLabel = new Date(0, 0, 0, 0, timeFilter)
        .toLocaleString("en", {timeStyle: "short"});

    $: filteredDepartures = timeFilter === -1
        ? allDepartures
        : d3.rollup(filterByMinute(departuresByMinute, timeFilter), v => v.length, d => d.start_station_id);

    $: filteredArrivals = timeFilter === -1
        ? allArrivals
        : d3.rollup(filterByMinute(arrivalsByMinute, timeFilter), v => v.length, d => d.end_station_id);

    $: filteredStations = stations.map(station => {
        const id = station.Number;
        const arr = filteredArrivals.get(id) ?? 0;
        const dep = filteredDepartures.get(id) ?? 0;
        return { ...station, arrivals: arr, departures: dep, totalTraffic: arr + dep };
    });

    $: radiusScale = d3.scaleSqrt()
        .domain([0, d3.max(filteredStations, d => d.totalTraffic) || 0])
        .range([0, 25]);

    $: stationFlow = d3.scaleQuantize()
        .domain([0, 1])
        .range([0, 0.5, 1]);

    $: if (selectedStation) {
        getIso(+selectedStation.Long, +selectedStation.Lat);
    } else {
        isochrone = null;
    }

    function minutesSinceMidnight(date) {
        return date.getHours() * 60 + date.getMinutes();
    }

    function filterByMinute(tripsByMinute, minute) {
        let minMinute = (minute - 60 + 1440) % 1440;
        let maxMinute = (minute + 60) % 1440;
        if (minMinute > maxMinute) {
            return [...tripsByMinute.slice(minMinute), ...tripsByMinute.slice(0, maxMinute)].flat();
        }
        return tripsByMinute.slice(minMinute, maxMinute).flat();
    }

    function getCoords(station) {
        let point = new mapboxgl.LngLat(+station.Long, +station.Lat);
        let {x, y} = map.project(point);
        return {cx: x, cy: y};
    }

    function geoJSONPolygonToPath(feature) {
        const path = d3.path();
        for (const ring of feature.geometry.coordinates) {
            for (let i = 0; i < ring.length; i++) {
                const [lng, lat] = ring[i];
                const {x, y} = map.project([lng, lat]);
                if (i === 0) path.moveTo(x, y);
                else path.lineTo(x, y);
            }
            path.closePath();
        }
        return path.toString();
    }

    async function getIso(lon, lat) {
        const minutes = [5, 10, 15, 20];
        const contourColors = ["03045e", "0077b6", "00b4d8", "90e0ef"];
        const params = new URLSearchParams({
            contours_minutes: minutes.join(","),
            contours_colors: contourColors.join(","),
            polygons: "true",
            access_token: mapboxgl.accessToken,
        });
        const url = `https://api.mapbox.com/isochrone/v1/mapbox/cycling/${lon},${lat}?${params}`;
        const query = await fetch(url, {method: "GET"});
        isochrone = await query.json();
    }

    const bikeLineStyle = {
        "line-color": "#32D400",
        "line-width": 3,
        "line-opacity": 0.4,
    };

    async function initMap() {
        map = new mapboxgl.Map({
            container: "map",
            center: [-71.09415, 42.36027],
            zoom: 12,
            style: "mapbox://styles/mapbox/streets-v12",
        });
        await new Promise(resolve => map.on("load", resolve));

        map.addSource("boston_route", {
            type: "geojson",
            data: "https://bostonopendata-boston.opendata.arcgis.com/datasets/boston::existing-bike-network-2022.geojson?outSR=%7B%22latestWkid%22%3A3857%2C%22wkid%22%3A102100%7D",
        });
        map.addLayer({
            id: "boston_bike_lanes",
            type: "line",
            source: "boston_route",
            paint: bikeLineStyle,
        });

        map.addSource("cambridge_route", {
            type: "geojson",
            data: "https://raw.githubusercontent.com/cambridgegis/cambridgegis_data/main/Recreation/Bike_Facilities/RECREATION_BikeFacilities.geojson",
        });
        map.addLayer({
            id: "cambridge_bike_lanes",
            type: "line",
            source: "cambridge_route",
            paint: bikeLineStyle,
        });

        stations = await d3.csv("https://vis-society.github.io/labs/9/data/bluebikes-stations.csv");

        let trips = await d3.csv("https://vis-society.github.io/labs/9/data/bluebikes-traffic-2024-03.csv", trip => {
            trip.started_at = new Date(trip.started_at);
            trip.ended_at = new Date(trip.ended_at);
            return trip;
        });

        for (let trip of trips) {
            departuresByMinute[minutesSinceMidnight(trip.started_at)].push(trip);
            arrivalsByMinute[minutesSinceMidnight(trip.ended_at)].push(trip);
        }
        allDepartures = d3.rollup(trips, v => v.length, d => d.start_station_id);
        allArrivals = d3.rollup(trips, v => v.length, d => d.end_station_id);
    }

    onMount(() => {
        initMap();
    });
</script>

<h1>Boston Bike Traffic</h1>

<label>
    Filter by time:
    <input type="range" min="-1" max="1440" bind:value={timeFilter} />
    {#if timeFilter !== -1}
        <time>{timeFilterLabel}</time>
    {:else}
        <em>(any time)</em>
    {/if}
</label>

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
                <circle
                    {...getCoords(station)}
                    r={radiusScale(station.totalTraffic)}
                    style="--departure-ratio: {station.totalTraffic > 0 ? stationFlow(station.departures / station.totalTraffic) : 0.5}"
                    class={station?.Number === selectedStation?.Number ? "selected" : ""}
                    on:mousedown={() => selectedStation = selectedStation?.Number !== station?.Number ? station : null}
                >
                    <title>{station.totalTraffic} trips ({station.departures} departures, {station.arrivals} arrivals)</title>
                </circle>
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
    }

    #map svg {
        position: absolute;
        z-index: 1;
        width: 100%;
        height: 100%;
        pointer-events: none;
    }

    #map svg circle, .legend > div {
        --color-departures: steelblue;
        --color-arrivals: darkorange;
        --color: color-mix(
            in oklch,
            var(--color-departures) calc(100% * var(--departure-ratio)),
            var(--color-arrivals)
        );
        fill: var(--color);
    }

    #map svg circle {
        fill-opacity: 0.6;
        stroke: white;
        pointer-events: auto;
    }

    #map svg:has(circle.selected) circle:not(.selected) {
        opacity: 0.3;
    }

    path {
        pointer-events: auto;
    }

    .legend {
        display: flex;
        gap: 0.5em;
        margin-top: 0.5em;
    }

    .legend > div {
        flex: 1;
        padding: 0.4em;
        text-align: center;
        background-color: var(--color);
        color: white;
    }
</style>
