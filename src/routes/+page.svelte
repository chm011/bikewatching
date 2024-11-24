<script>
import mapboxgl from 'mapbox-gl';
import '../../node_modules/mapbox-gl/dist/mapbox-gl.css';
mapboxgl.accessToken = 'pk.eyJ1IjoiY2htMDExIiwiYSI6ImNtM3RvdmQxNzBhczUya3EyNHV0OW53M2QifQ.TMIB_EDaTwnRvXBF85do4Q'
import * as d3 from 'd3';

import { onMount } from 'svelte';

let map;
let foo;
let stations = [];
let mapViewChanged = 0;
let trips = [];
let arrivals;
let departures;
let timeFilter =-1;
let filteredTrips = [];
let filteredArrival = [];
let filteredDepartures = [];
let filteredStations = [];

let departuresByMinute = Array.from({ length: 1440 }, () => []);
let arrivalsByMinute = Array.from({ length: 1440 }, () => []);
let stationFlow = d3.scaleQuantize().domain([0, 1]).range([0, 0.5, 1]);

function getCoords(station) {
        let point = new mapboxgl.LngLat(+station.Long, +station.Lat);
        let { x, y } = map.project(point);
        console.log(`Coords for station${station.NAME}: x=${x}, y=${y}`);
        return { cx: x, cy: y };
    }

function minutesSinceMidnight(date) {
  return date.getHours() * 60 + date.getMinutes();
}
function filterByMinute(tripsByMinute, minute) {
  // Normalize both to the [0, 1439] range
  // % is the remainder operator: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Remainder
  console.log("Minute:", minute);
  let minMinute = (minute - 60 + 1440) % 1440;
  let maxMinute = (minute + 60) % 1440;

  if (minMinute > maxMinute) {
    let beforeMidnight = tripsByMinute.slice(minMinute);
    let afterMidnight = tripsByMinute.slice(0, maxMinute);

    return beforeMidnight.concat(afterMidnight).flat();
  } else {
    return tripsByMinute.slice(minMinute, maxMinute).flat();
  }
}

$: map?.on('move', (evt) => mapViewChanged++);

$: radiusScale = d3
    .scaleSqrt()
    .domain([0, d3.max(timeFilter === -1 ? stations : filteredStations, (d) => d.totalTraffic)]) 
    .range(timeFilter === -1 ? [3, 25] : [0, 25]); 

$: timeFilterLabel = new Date(0, 0, 0, 0, timeFilter).toLocaleString('en', {
  timeStyle: 'short',});    


$: filteredDepartures = d3.rollup(
   timeFilter !== -1? filterByMinute(departuresByMinute, timeFilter) : trips,
    (v) => v.length,
    (d) => d.start_station_id
    );

$: filteredArrivals = d3.rollup(
    timeFilter !== -1 ? filterByMinute(arrivalsByMinute, timeFilter) : trips,
    (v) => v.length,
    (d) => d.end_station_id
    );

$: filteredStations = stations.map((station) => {
    let id = station.Number;
    let arrivalsCount = filteredArrivals.get(id) ?? 0;
    let departuresCount = filteredDepartures.get(id) ?? 0;

    station = { ...station }; // Clone the station object
        station.arrivals = arrivalsCount;
        station.departures = departuresCount;
        station.totalTraffic = arrivalsCount + departuresCount;
        return station;
    });

onMount(async () => {
    map = new mapboxgl.Map({
      container: 'map', // ID of the HTML element for the map
      style: 'mapbox://styles/chm011/cm3tpnwup00bu01sva3jn38bw', // Your custom Mapbox style
      center: [-71.1138879, 42.3586452], // Longitude, Latitude
      zoom: 12, // Initial zoom level
    });

    // Wait for the map to load
    await new Promise((resolve) => map.on('load', resolve));

    // Add a GeoJSON source
    map.addSource('boston_route', {
      type: 'geojson',
      data: 'https://bostonopendata-boston.opendata.arcgis.com/datasets/boston::existing-bike-network-2022.geojson?outSR=%7B%22latestWkid%22%3A3857%2C%22wkid%22%3A102100%7D',
    });

    // Add a layer to render the GeoJSON data
    map.addLayer({
      id: 'boston-route-layer', // A unique name for the layer
      type: 'line', // Specify the layer type as a line
      source: 'boston_route', // Link the layer to the source ID
      paint: {
        'line-color': '#06402B', //green for green line
        'line-width': 3, 
        'line-opacity': 0.4, 
      },
    });
    map.addSource('cambridge_route', {
      type: 'geojson',
      data: 'https://raw.githubusercontent.com/cambridgegis/cambridgegis_data/main/Recreation/Bike_Facilities/RECREATION_BikeFacilities.geojson',
    });

    map.addLayer({
      id: 'cambridge-route-layer', // A unique name for the layer
      type: 'line', // Specify the layer type as a line
      source: 'cambridge_route', // Link the layer to the source ID
      paint: {
        'line-color': '#C21807', //red for red line
        'line-width': 3, 
        'line-opacity': 0.4, 
      },
    });

    
    stations = await d3.csv('https://dsc-courses.github.io/dsc106-2025-wi/labs/lab08/data/bluebikes-stations.csv',(d) => ({
        Number: d.Number,
        NAME: d.NAME,
        Lat: +d.Lat,
        Long: +d.Long,
        seasonal_status: d.seasonal_status,
        municipality: d.municipality,
        total_docks: d.total_docks

    }));

    trips = await d3
        .csv("https://dsc-courses.github.io/dsc106-2025-wi/labs/lab08/data/bluebikes-traffic-2024-03.csv")
        .then(data => {
	    for (let trip of data) {
		    trip.started_at = new Date(trip.started_at);
            trip.ended_at = new Date(trip.ended_at);

            let startedMinutes = minutesSinceMidnight(trip.started_at);
            departuresByMinute[startedMinutes].push(trip);
   
            let endedMinutes = minutesSinceMidnight(trip.ended_at);
            arrivalsByMinute[endedMinutes].push(trip);

	        }
	return data;
    });


    departures = d3.rollup(
        trips,
        (v) => v.length,
        (d) => d.start_station_id
    );

    arrivals = d3.rollup(
        trips,
        (v) => v.length,
        (d) => d.end_station_id
    );

    stations = stations.map((station) => {
        let id = station.Number;
        station.arrivals = arrivals.get(id) ?? 0;
        station.departures = departures.get(id) ?? 0;
        station.totalTraffic = station.arrivals + station.departures;
        return station;
    });


});






</script>
<header>
    <h1> <a src='favicon.svg'></a> Bikewatching</h1>
    <label>
        Filter by time:
        <input type="range" id = "time-slider" min="-1" max="1440" bind:value={timeFilter}/>
        {#if timeFilter === -1}
        <em>(any time)</em>
        {:else}
        <time>{timeFilterLabel}</time>
        {/if}

    </label>
</header>

<div id="map">
    {#key mapViewChanged}
    <svg id="station-points">
        {#each filteredStations as station}
            <circle {...getCoords(station)} 
            r={radiusScale(station.totalTraffic)}
            style="--departure-ratio: { stationFlow(station.departures / station.totalTraffic) }">
                <title>
                    {station.totalTraffic} trips ({station.departures} departures, {station.arrivals} arrivals)
                </title>
            </circle>
        {/each}
    </svg>
    {/key}
</div>
<div class="legend">
    <div style="--departure-ratio: 1">More departures</div>
    <div style="--departure-ratio: 0.5">Balanced</div>
    <div style="--departure-ratio: 0">More arrivals</div>
  </div>


<style>
@import url('$lib/global.css');
</style>


