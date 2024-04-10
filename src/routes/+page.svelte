<script>
    import mapboxgl from "mapbox-gl";
    import "../../node_modules/mapbox-gl/dist/mapbox-gl.css";
    mapboxgl.accessToken =
        "pk.eyJ1IjoiazgwY2hlbiIsImEiOiJjbGdlYTRkZzUyaW1kM2VsaW56bzF0OHRzIn0.7DzuSnwWXOjfBMwTAhlUqg";

    import { onMount } from "svelte";
    import * as d3 from "d3";

    const layer_style_paint = {
        "fill-color": "green",
        "fill-opacity": 0.5,
        "line-color": "green",
        "line-opacity": 0.4,
        "line-width": 3,
    };

    let stations = [];
    let map;
    let mapViewChanged = 0;

    let trips = [];

    let arrivals;
    let departures;

    let radiusScale;

    let timeFilter = -1;
    let timeFilterLabel;
    // let filteredTrips;

    let filteredArrivals;
    let filteredDepartures;
    let filteredStations;

    let departuresByMinute = Array.from({ length: 1440 }, () => []);
    let arrivalsByMinute = Array.from({ length: 1440 }, () => []);

    let stationFlow = d3.scaleQuantize().domain([0, 1]).range([0, 0.5, 1]);

    function minutesSinceMidnight(date) {
        return date.getHours() * 60 + date.getMinutes();
    }

    onMount(async () => {
        map = new mapboxgl.Map({
            container: "map",
            style: "mapbox://styles/mapbox/streets-v12",
            center: [-71.09062477150395, 42.361997951771265],
            zoom: 12,
        });

        await new Promise((resolve) => map.on("load", resolve));

        map.addSource("boston_route", {
            type: "geojson",
            data: "https://bostonopendata-boston.opendata.arcgis.com/datasets/boston::existing-bike-network-2022.geojson?outSR=%7B%22latestWkid%22%3A3857%2C%22wkid%22%3A102100%7D",
        });

        map.addSource("cambridge_route", {
            type: "geojson",
            data: "https://raw.githubusercontent.com/cambridgegis/cambridgegis_data/main/Recreation/Bike_Facilities/RECREATION_BikeFacilities.geojson",
        });

        map.addLayer({
            id: "bikelanes_boston", // A name for our layer (up to you)
            type: "line", // one of the supported layer types, e.g. line, circle, etc.
            source: "boston_route", // The id we specified in `addSource()`
            paint: layer_style_paint,
        });

        map.addLayer({
            id: "bikelanes_cambridge", // A name for our layer (up to you)
            type: "line", // one of the supported layer types, e.g. line, circle, etc.
            source: "cambridge_route", // The id we specified in `addSource()`
            paint: layer_style_paint,
        });

        // load data with d3.csv()
        stations = await d3.csv(
            "https://vis-society.github.io/labs/8/data/bluebikes-stations.csv",
        );

        trips = await d3
            .csv(
                "https://vis-society.github.io/labs/8/data/bluebikes-traffic-2024-03.csv",
            )
            .then((trips) => {
                for (let trip of trips) {
                    trip.started_at = new Date(trip.started_at);
                    trip.ended_at = new Date(trip.ended_at);

                    let startedMinutes = minutesSinceMidnight(trip.started_at);
                    departuresByMinute[startedMinutes].push(trip);

                    let endedMinutes = minutesSinceMidnight(trip.ended_at);
                    arrivalsByMinute[endedMinutes].push(trip);
                }
                return trips;
            });
        departures = d3.rollup(
            trips,
            (v) => v.length,
            (d) => d.start_station_id,
        );

        arrivals = d3.rollup(
            trips,
            (v) => v.length,
            (d) => d.end_station_id,
        );

        stations = stations.map((station) => {
            let id = station.Number;
            station.arrivals = arrivals.get(id) ?? 0; // d3.rollup returns a Map object, which we can then use d3.map.get() to get the value. In our case arrivals is a Map object with key as station id and value as number of arrivals
            station.departures = departures.get(id) ?? 0;
            station.totalTraffic = station.arrivals + station.departures;
            return station;
        });

        filteredStations = stations;
        filteredArrivals = arrivals;
        filteredDepartures = departures;
    });

    function getCoords(station) {
        let point = new mapboxgl.LngLat(+station.Long, +station.Lat);
        let { x, y } = map.project(point);
        return { cx: x, cy: y };
    }

    $: map?.on("move", (evt) => mapViewChanged++);

    $: radiusScale = d3
        .scaleSqrt()
        .domain([0, d3.max(filteredStations, (d) => d.totalTraffic)])
        .range([0, 25]);

    $: timeFilterLabel = new Date(0, 0, 0, 0, timeFilter).toLocaleString("en", {
        timeStyle: "short",
    });
    $: filteredTrips =
        timeFilter === -1
            ? trips
            : trips.filter((trip) => {
                  let startedMinutes = minutesSinceMidnight(trip.started_at);
                  let endedMinutes = minutesSinceMidnight(trip.ended_at);
                  return (
                      Math.abs(startedMinutes - timeFilter) <= 60 ||
                      Math.abs(endedMinutes - timeFilter) <= 60
                  );
              });

    $: filteredArrivals = d3.rollup(
        filteredTrips,
        (v) => v.length,
        (d) => d.end_station_id,
    );

    $: filteredDepartures = d3.rollup(
        filteredTrips,
        (v) => v.length,
        (d) => d.start_station_id,
    );

    // $: filteredDepartures = departuresByMinute
    //     .slice(timeFilter - 60, timeFilter + 60)
    //     .flat();

    // $: filteredArrivals = arrivalsByMinute
    //     .slice(timeFilter - 60, timeFilter + 60)
    //     .flat();

    $: filteredStations = stations.map((station) => {
        let updatedStation = { ...station };
        let id = updatedStation.Number;
        updatedStation.arrivals = filteredArrivals.get(id) ?? 0;
        updatedStation.departures = filteredDepartures.get(id) ?? 0;
        updatedStation.totalTraffic =
            updatedStation.arrivals + updatedStation.departures;
        return updatedStation;
    });
</script>

<header>
    <h1>Cyclogist</h1>
    <label>
        Filter by time
        <input type="range" min="-1" max="1440" bind:value={timeFilter} />
        <time>{timeFilterLabel}</time>
        {#if timeFilter === -1}
            <em>(start time)</em>
        {/if}
    </label>
</header>
<div id="map">
    <svg>
        {#key mapViewChanged}
            {#each filteredStations as station}
                <circle
                    cx={getCoords(station).cx}
                    cy={getCoords(station).cy}
                    r={radiusScale(station.totalTraffic)}
                    fill="steelblue"
                    style="--departure-ratio: {stationFlow(
                        station.departures / station.totalTraffic,
                    )}"
                >
                    <title
                        >{station.totalTraffic} trips ({station.departures} departures,
                        {station.arrivals} arrivals)</title
                    >
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
        background-color: #fed702;
    }

    #map svg {
        position: absolute;
        z-index: 1;
        width: 100%;
        height: 100%;
        pointer-events: none;
    }

    #map svg circle {
        fill-opacity: 0.5;
        pointer-events: auto;

        --color-departures: steelblue;
        --color-arrivals: darkorange;
        --color: color-mix(
            in oklch,
            var(--color-departures) calc(100% * var(--departure-ratio)),
            var(--color-arrivals)
        );
        fill: var(--color);
    }

    #map circle {
        --color-departures: steelblue;
        --color-arrivals: darkorange;
        --color: color-mix(
            in oklch,
            var(--color-departures) calc(100% * var(--departure-ratio)),
            var(--color-arrivals)
        );
        fill: var(--color);
    }

    .legend > div,
    circle {
        --color-departures: steelblue;
        --color-arrivals: darkorange;
        --color: color-mix(
            in oklch,
            var(--color-departures) calc(100% * var(--departure-ratio)),
            var(--color-arrivals)
        );
    }

    .legend {
        display: flex;
        gap: 1px;
    }

    .legend div {
        flex: 1;
        background-color: var(--color);
        padding: 0.1rem 1rem;

        color: white;
        font-weight: bold;
        font-size: small;
        &:nth-child(2) {
            text-align: center;
        }
        &:nth-child(3) {
            text-align: right;
        }
    }

    header {
        display: flex;
        gap: 1em;
        align-items: baseline;
    }
    label {
        margin-left: auto;
    }
    time {
        display: block;
    }
    em {
        display: block;
    }
</style>
