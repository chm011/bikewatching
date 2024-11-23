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


function getCoords(station) {
        let point = new mapboxgl.LngLat(+station.Long, +station.Lat);
        let { x, y } = map.project(point);
        return { cx: x, cy: y };
    }

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
      Lat: +d.Lat,
      Long: +d.Long,
    }));
    $: map?.on('move', (evt) => mapViewChanged++);
    
   trips = await d3.csv('https://dsc-courses.github.io/dsc106-2025-wi/labs/lab08/data/bluebikes-traffic-2024-03.csv',(d) => ({
        rid_id: d.rid_id,
        bike_type: d.bike_type,
        started_at: d.started_at,
        ended_at: d.ended_at,
        start_station_id: d.start_station_id,
        end_station_id: d.end_station_id,
        is_member: d.is_member
    }));

    departures = d3.rollup(
        trips,
        (v) => v.length,
        (d) => d.start_station_id
    );


    stations = stations.map((station) => {
        let id = station.Number;
        stations.arrivals = arrivals.get(id) ?? 0;
        station.departures = departures.get(id) ?? 0;
        station.totalTraffic = station.arrivals + station.departures;
        return station;
    });
    $: radiusScale = d3
        .scaleSqrt()
        .domain([0, d3.max(stations, (d) => d.totalTraffic)])
        .range([0, 25]);
  });


</script>

<div id="map">
    {#key mapViewChanged}
    <svg id="station-points">
        {#each stations as station}
            <circle {...getCoords(station)} r='5' fill='steelblue' />
        {/each}
    </svg>
    {/key}
</div>


<style>
@import url('$lib/global.css');
</style>


Bikewatching 

