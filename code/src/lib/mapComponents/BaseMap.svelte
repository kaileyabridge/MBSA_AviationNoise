<script>
    import mapboxgl from "mapbox-gl";
    import "../../../node_modules/mapbox-gl/dist/mapbox-gl.css";
    import { onMount } from "svelte";
    import {calculateBoundingBox} from "$lib/mapComponents/mapUtils.js";
    import AircraftAnimation from "$lib/aircraftComponents/Aircraft.svelte"; // New Import for MBSA
    import {
        computePosition,
        autoPlacement,
        offset,
    } from '@floating-ui/dom';
    

    mapboxgl.accessToken = "pk.eyJ1IjoicmZpb3Jpc3RhIiwiYSI6ImNsdWQwcDd0aDFkengybG85eW00eDJqdzEifQ.smRFd5P2IKrDHr5HGsfrGw";

    let showAircraftAnimation = true;
    // Define a flight path - customize this based on your needs
    const flightPath = [
        // Starting from what appears to be Logan Airport
        [-71.0202, 42.3643], // Logan Airport
        [-71.0502, 42.3843], // Point 1
        [-71.0802, 42.4043], // Point 2
        [-71.1102, 42.3943], // Point 3
        [-71.1402, 42.3743], // Point 4
        [-71.1702, 42.3543], // Point 5
        [-71.2002, 42.3443], // Point 6
        [-71.0202, 42.3643]  // Back to Logan
    ];
    // Function to start/stop the aircraft animation
    function toggleAircraftAnimation() {
        showAircraftAnimation = !showAircraftAnimation;
    }

    let map;
    let mapViewChanged = 0;

    export let municipalities = [];
    export let selectedMunicipality = null;
    export let comparisonMode = false;
    export let explorationMode = false;
    export let parcelFiles = [];

    const mapContainerID = "svg-map-container";

    // Tooltip stuff
    let hoveredIndex = -1;
    let showTooltip = false;
    $: hoveredMunicipality = municipalities[hoveredIndex] ?? {};
    let municipalityTooltip;
    let tooltipPosition = {x: 0, y: 0};

    const overallCenter = [-71.05672511293635, 42.35885643076469]
    const overallZoom = 9

    const baseCenter = [-71.29451, 42.36027];
    const baseZoom = 10;

    onMount(async () => {
        map = new mapboxgl.Map({
            container: 'map',
            center: baseCenter,
            zoom: baseZoom,
            style: 'mapbox://styles/smpeter/cluqd5hft05en01qqc4mxa1kd',
            attributionControl: false,
            interactive: true
        });

        await new Promise(resolve => map.on("load", resolve));

        map.addSource("MBTALines", {
            type: "geojson",
            data: "/data/mbta_community_lines.geojson",
        });

        // Add a button to toggle the animation
        const toggleButton = document.createElement('button');
        toggleButton.textContent = 'Toggle Flight Animation';
        toggleButton.className = 'mapboxgl-ctrl mapboxgl-ctrl-group aircraft-toggle-btn';
        toggleButton.onclick = toggleAircraftAnimation;
        
        const customControl = document.createElement('div');
        customControl.className = 'mapboxgl-ctrl-top-right';
        customControl.appendChild(toggleButton);
        map.getContainer().appendChild(customControl);
    })

    $: map?.on("move", evt => mapViewChanged++);

    /**
     * Fits the map bounds with specified padding.
     * @param {Object} bounds - The bounds to fit the map to.
     * @param {(number|Object)} padding - The padding as a number or an object with optional top, bottom, left, and right properties.
     */
    function fitBounds(bounds, padding = { padding: 20 }) {
        map.fitBounds(bounds, padding);
    }

    function projectPolygonCoordinates(coordinates) {
        return coordinates.map(coord => {
            let { cx, cy } = getCoords({ Lat: coord[1], Long: coord[0] });
            return `${cx},${cy}`;
        }).join(' ');
    }

    function getCoords(station) {

        const longitude = parseFloat(station.Long);
        const latitude = parseFloat(station.Lat);

        if (isNaN(longitude) || isNaN(latitude)) {
            console.error("Invalid coordinates for station:", station);
            return { cx: 0, cy: 0 }; // You might want to handle this case differently
        }

        let point = new mapboxgl.LngLat(longitude, latitude);
        let {x, y} = map?.project(point);
        return {cx: x, cy: y};
    }


    function zoomToEntireMap(withSidePanel=true) {
        if (withSidePanel) {
            map?.flyTo({
                center: baseCenter,
                zoom: baseZoom
            })
        } else {
            map?.flyTo({
                center: overallCenter,
                zoom: overallZoom
            })
        }

    }

    function zoomToMunicipality(municipality) {
        const bounds = calculateBoundingBox(selectedMunicipality?.Geometries);
        fitBounds(bounds, {
            padding: explorationMode ? 50 : {top: 20, bottom: 20, left: 1150, right: 20}
        });
    }


    let loadedParcels = [];

    function toggleStationParcels (station) {
        if (!loadedParcels.some(s => s.Name === station.Name)) {
            console.log(`Adding parcel layer for ${station.Name}`)
            if (parcelFiles.filter(e => e.StopName == station.Name).length > 0) {
                map.addSource(station.Name, {
                    type: "geojson",
                    data: parcelFiles.filter(e => e.StopName == station.Name)[0].FileName,
                });

                map.addLayer({
                    id: station.Name,
                    type: "fill",  // Use 'fill' type for polygon layers
                    source: station.Name,
                    paint: {
                        "fill-color": [
                            'case',
                            ["to-boolean", ['get', 'mustUpzone']], '#f39034',  // true
                            ['!', ["to-boolean", ['get', 'mustUpzone']]], '#789f4f',  // false
                            '#ffffff'
                        ],  // Correct property for setting the fill color of polygons
                        "fill-outline-color": "black"  // Sets the color of the outline
                    },
                });
                loadedParcels.push(station);
            } else {
                console.log("Parcel data not available");
            }
        } else {
            console.log("Removing parcel layer")
            map.removeLayer(station.Name);
            map.removeSource(station.Name);
            loadedParcels = loadedParcels.filter(s => s.Name !== station.Name);
        }
    }

    //
    $: {
        if (selectedMunicipality) {
            
            zoomToMunicipality(selectedMunicipality);
        } else {

        }
    }


    $: {
        if (comparisonMode && selectedMunicipality) {
            zoomToMunicipality(selectedMunicipality);
        }
    }

    $: {
        if (explorationMode) {
            map.boxZoom.enable();
            map.scrollZoom.enable();
            map.dragPan.enable();
            map.dragRotate.enable();
            map.keyboard.enable();
            map.doubleClickZoom.enable();
            map.touchZoomRotate.enable();
            map.addControl(new mapboxgl.NavigationControl());
            selectedMunicipality = undefined;
            zoomToEntireMap(false);
        }
    }

    $: filteredMunicipalities = (selectedMunicipality && !explorationMode) ? municipalities.filter(m => {
        return m.Name == selectedMunicipality.Name
    }) : municipalities;

    async function dotInteraction (index, evt) {
        let hoveredDot = evt.target;
        hoveredIndex = index;
        if (!selectedMunicipality || explorationMode){
            if (evt.type === "mouseenter" || evt.type === "focus") {
                // dot hovered
                // cursor = {x: evt.x, y: evt.y};
                showTooltip = true;
                tooltipPosition = await computePosition(hoveredDot, municipalityTooltip, {
                    strategy: "fixed", // because we use position: fixed
                    middleware: [
                        offset(5), // spacing from tooltipComponent to dot
                        autoPlacement() // see https://floating-ui.com/docs/autoplacement
                    ],
                });
            }
            else if (evt.type === "mouseleave" || evt.type === "blur") {
                // dot unhovered
                // hoveredIndex = -1;
                showTooltip = false;
            }
            else if (evt.type === "click" || evt.type === "keyup" && evt.key === "Enter") {
                if (explorationMode && selectedMunicipality && selectedMunicipality.Name === municipalities[index].Name) {
                    selectedMunicipality = undefined;
                    showTooltip = true;
                } else {
                    selectedMunicipality = municipalities[index];
                    showTooltip = false;
                }
            }
        }
    }

</script>

<div id="map">
    <svg
            id={mapContainerID}
    >
        {#key mapViewChanged}
            {#each filteredMunicipalities as municipality, index}
                {#if municipality.Geometries.type === "Polygon"}
                    <polygon
                            id={ `polygon-${index}` }
                            points={
                                municipality.Geometries.coordinates.length ?
                                projectPolygonCoordinates(municipality.Geometries.coordinates[0]) : ""
                            }
                            fill="#a9987a"
                            stroke="black"
                            stroke-width="1"
                            opacity={municipality.Name == selectedMunicipality?.Name ? '0.6' : (!explorationMode ? '0.5' : '0.3')}
                            class:municipality
                            on:mouseenter={(evt) => dotInteraction(index, evt)}
                            on:mouseleave={(evt) => dotInteraction(index, evt)}
                            on:focus={(evt) => dotInteraction(index, evt)}
                            on:blur={(evt) => dotInteraction(index, evt)}
                            on:click={(evt) => dotInteraction(index, evt)}
                            on:keyup={(evt) => dotInteraction(index, evt)}
                    >
<!--                        <title> { municipality.Name } </title>-->
                    </polygon>
                {:else if municipality.Geometries.type === 'MultiPolygon'}
                    {#each municipality.Geometries.coordinates as geometry}
                        <polygon
                                id={ `polygon-${index}` }
                                points={
                                    municipality.Geometries.coordinates.length ?
                                    projectPolygonCoordinates(geometry[0]) : ""
                                }
                                fill="#a9987a"
                                stroke="black"
                                stroke-width="1"
                                opacity={municipality.Name == selectedMunicipality?.Name ? '0.6' : (!explorationMode ? '0.5' : '0.3')}
                                class:municipality
                                on:mouseenter={(evt) => dotInteraction(index, evt)}
                                on:mouseleave={(evt) => dotInteraction(index, evt)}
                                on:focus={(evt) => dotInteraction(index, evt)}
                                on:blur={(evt) => dotInteraction(index, evt)}
                                on:click={(evt) => dotInteraction(index, evt)}
                                on:keyup={(evt) => dotInteraction(index, evt)}
                        >
<!--                            <title> { municipality.Name } </title>-->
                        </polygon>
                    {/each}
                {/if}
            {/each}
        {/key}
    </svg>
</div>

<!-- Add the aircraft animation component -->
{#if map}
    <AircraftAnimation 
        {map} 
        active={showAircraftAnimation} 
        flightPath={flightPath}
    />
{/if}

<dl class="info"
    hidden={!showTooltip}
    style="top: {tooltipPosition.y}px; left: {tooltipPosition.x}px"
    bind:this={municipalityTooltip}
    role="tooltip" >
    <dt>Municipality</dt>
    <dd>{ hoveredMunicipality.Name }</dd>
</dl>

<style>
    @import url("$lib/global.css");

    /* Your existing styles */
    
    :global(.aircraft-toggle-btn) {
        margin: 10px;
        padding: 8px 16px;
        background-color: white;
        border: 1px solid #ccc;
        border-radius: 4px;
        cursor: pointer;
    }
    
    :global(.aircraft-toggle-btn:hover) {
        background-color: #f0f0f0;
    }

    #map {
        flex: 1;
    }

    #map svg {
        width: 100%;
        height: 100%;
        pointer-events: none;
        position: absolute;
        z-index: 2;
        cursor: grab;}

    #map svg .municipality {
            pointer-events: auto;
            cursor: pointer;
        }


    dl.info {
        z-index: 2;
        display: grid;
        grid-template-columns: auto auto; /* Define two columns */
        grid-auto-rows: auto; /* This will create a new row for each term/description pair */
        gap: 0.5em; /* Adjust the gap between items */
        align-items: start;
        position: fixed; /* Ensure it's positioned in relation to the SVG or a relative container */
        top: 10px;
        left: 10px;
        background-color: rgba(10, 0, 0, 0.4); /* Semi-transparent background */
        backdrop-filter: blur(10px);
        border-radius: 5px;
        box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1); /* Soft shadow for better readability */
        font-size: 0.9em;
        padding: 1em;
        transition-duration: 500ms;
        transition-property: opacity, visibility;

        font-family: 'Montserrat', sans-serif;
        visibility: visible;
        width: 250px;
        color: #a9987a;

        &[hidden]:not(:hover, :focus-within) {
            opacity: 0;
            visibility: hidden;
        }
    }

    dl.info dt {
        font-weight: bold; /* Makes text bold */
    }
</style>
