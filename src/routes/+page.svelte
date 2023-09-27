<script>
    // libraries
    import "../assets/global.css";
    import { onMount } from "svelte";
    import Select from "svelte-select";
    import mapboxgl from "mapbox-gl";
    import * as topojson from "topojson-client";
    import { csv } from "d3-fetch";
    mapboxgl.accessToken =
        "pk.eyJ1Ijoic2Nob29sb2ZjaXRpZXMiLCJhIjoiY2w2Z2xhOXprMTYzczNlcHNjMnNvdGlmNCJ9.lOgVHrajc1L-LlU0as2i2A";
    // data
    import cmaSummary from "../assets/cma-summary.json";
    import cmaPoints from "../assets/cma-points.geo.json";
    import transitLines from "../assets/transit-lines-canada.geo.json";
    import transitStops from "../assets/transit-stops-canada.geo.json";
    import municipalLabels from "../assets/csd-2021-centroids.geo.json";

    // initial variables
    let cmaSelected = "Toronto";
    let cmauidSelected = 535;
    let map;
    let ctDataTable;

    // toggle for if the panel is visible or not

    let isContentVisible = true;
    function toggleContent() {
        isContentVisible = !isContentVisible;
    }

    // loading CMA summary data, also converting it to geojson for map points

    const cmaData = cmaSummary.filter((item) => item.Rank < 11);

    // Changing the CMA - zooming to new CMA

    let filteredData;
    $: filteredData = cmaData.filter(
        (item) => item.CMAUID === cmauidSelected
    )[0];

    let cmaAll = cmaData
        .sort((a, b) => b.pop2021 - a.pop2021)
        .map((item) => item.CMANAME);

    function cmaSelectDropDown(e) {
        cmaSelectMapUpdate(e.detail.value);
    }

    function cmaSelectMapUpdate(cmaname) {
        cmaSelected = cmaname;
        let filteredData = cmaData.filter(
            (item) => item.CMANAME === cmaSelected
        )[0];

        cmauidSelected = filteredData.CMAUID;
        let cmaX = filteredData.x;
        let cmaY = filteredData.y;

        map.setZoom(9);
        map.setBearing(0);
        map.setPitch(0);
        map.panTo([cmaX, cmaY]);

        const cmaFilter = [
            "match",
            ["get", "CMAUID"],
            [cmauidSelected.toString()],
            true,
            false,
        ];
        map.setFilter("metro-mindset-csd-2021-border", cmaFilter);
        map.setFilter("metro-mindset-cma-2021-border", cmaFilter);
        map.setFilter("metro-mindset-cma-2021-background", cmaFilter);
        map.setFilter("municipalLabels", cmaFilter);
    }

    // Add or remove municipal borders

    let isMunicipalChecked = true;
    function toggleMunicipal() {
        isMunicipalChecked = !isMunicipalChecked;
        if (isMunicipalChecked) {
            map.setPaintProperty(
                "metro-mindset-csd-2021-border",
                "line-opacity",
                1
            );
            map.setPaintProperty("municipalLabels", "text-opacity", 1);
        } else {
            map.setPaintProperty(
                "metro-mindset-csd-2021-border",
                "line-opacity",
                0
            );
            map.setPaintProperty("municipalLabels", "text-opacity", 0);
        }
    }

    // Changing the map layer

    let mapLayers = ["Street Map", "Satellite", "Population Density"];
    let mapSelected = "Street Map";

    const choropleths = {
        "population-density": {
            name: "Population Density",
            breaks: [500, 3000, 6000],
            colours: ["#fffef8", "#fbefb5", "#f7dd66", "#f1c500"],
            // "colours": ["#fcfcfc", "#a4dcd4", "#4ebdad", "#00a189"]
        }
    };

    console.log(choropleths["population-density"].colours[0])

    

    function layerSelect(e) {
        $: mapSelected = e.detail.value;
        layerSet(cmauidSelected, mapSelected);
    }

    function layerSet(cmauid, layer) {
        console.log(cmauid, layer);
        if (layer === "Street Map") {
            map.setPaintProperty("mapbox-satellite", "raster-opacity", 0.011);
            try {
                map.removeLayer("ctPolygon");
                map.removeSource("ctPolygon");
            } catch {}
            // remove CT layer
        } else if (layer === "Satellite") {
            map.setPaintProperty("mapbox-satellite", "raster-opacity", 0.798);
            try {
                map.removeLayer("ctPolygon");
                map.removeSource("ctPolygon");
            } catch {}
            // remove CT layer
        } else if (layer === "Population Density") {
            try {
                map.removeLayer("ctPolygon");
                map.removeSource("ctPolygon");
            } catch {}
            map.setPaintProperty("mapbox-satellite", "raster-opacity", 0.011);
            map.addLayer(
                {
                    id: "ctPolygon",
                    type: "fill",
                    source: {
                        type: "geojson",
                        data: ctPolygon,
                    },
                    paint: {
                        "fill-outline-color": "white",
                        "fill-opacity": 0.881,
                        "fill-color": [
                            "case",
                            ["!=", ["get", "Population"], null],
                            [
                                "step",
                                ["/", ["get", "Population"], ["get", "Area"]],
                                choropleths["population-density"].colours[0],
                                choropleths["population-density"].breaks[0],
                                choropleths["population-density"].colours[1],
                                choropleths["population-density"].breaks[1],
                                choropleths["population-density"].colours[2],
                                choropleths["population-density"].breaks[2],
                                choropleths["population-density"].colours[3],
                            ],
                            "#cbcbcb",
                        ],
                    },
                },
                "transitStops"
            );
        }
    }

    // getting the ctPolygon data and join ctData for the selected CMA

    let ctPolygon;
    async function loadCensusTract(cmauid) {
        if (cmauid === cmauidSelected) {
            try {
                const response = await fetch(`ct-${cmauid}-2021.topo.json`);
                ctPolygon = await response.json();

                ctPolygon = topojson.feature(ctPolygon, `ct-${cmauid}-2021`);

                // join tabular data
                const dataMap = new Map();
                ctDataTable.forEach((item) => {
                    dataMap.set(item.ctuid, item);
                });
                const joinedData = ctPolygon.features.map((feature) => {
                    const ctuid = feature.properties.ctuid;
                    const matchingData = dataMap.get(ctuid);
                    return {
                        ...feature,
                        properties: {
                            ...feature.properties,
                            ...matchingData,
                        },
                    };
                });

                ctPolygon.features = joinedData;

                layerSet(cmauid, mapSelected);
            } catch (error) {
                console.error("Error loading CT data files:", error);
            }
        }
    }

    // called each time the CMA changes
    $: {
        loadCensusTract(cmauidSelected);
    }

    // Map setup and loading ct data table - happens on initial load of the page

    onMount(() => {
        csv("/ct-data.csv")
            .then((data) => {
                data.forEach((row) => {
                    for (const key in row) {
                        if (key !== "ctuid") {
                            row[key] = parseFloat(row[key]);
                        }
                    }
                });
                ctDataTable = data;
            })
            .catch((error) => {
                console.error("Error loading CSV data:", error);
            });

        map = new mapboxgl.Map({
            container: "map",
            style: "mapbox://styles/schoolofcities/cllwjnzlf016j01p71qq5eskt",
            center: [-79.6, 43.9],
            zoom: 4,
            maxZoom: 12,
            minZoom: 3,
            projection: "globe",
            scrollZoom: true,
            attributionControl: false,
        });

        const scale = new mapboxgl.ScaleControl({
            maxWidth: 100,
            unit: "metric",
        });
        map.addControl(scale, "bottom-right");

        map.addControl(new mapboxgl.NavigationControl(), "bottom-right");

        map.on("load", function () {
            map.addLayer({
                id: "municipalLabels",
                type: "symbol",
                source: {
                    type: "geojson",
                    data: municipalLabels,
                },
                layout: {
                    "text-field": "{CSDNAME}",
                    "text-font": ["Roboto Regular"],
                    "text-size": [
                        "interpolate",
                        ["linear"],
                        ["zoom"],
                        9,
                        12,
                        18,
                        40,
                    ],
                    "text-anchor": "center",
                },
                paint: {
                    "text-color": "#AB1368",
                    "text-halo-color": "rgba(255, 255, 255, 0.65)",
                    "text-halo-width": 2,
                    "text-halo-blur": 1,
                    "text-opacity": [
                        "interpolate",
                        ["linear"],
                        ["zoom"],
                        5.99,
                        0,
                        6,
                        1,
                    ],
                },
            });

            map.setFilter("municipalLabels", [
                "match",
                ["get", "CMAUID"],
                ["535"],
                true,
                false,
            ]);
        });

        map.on("load", function () {
            map.addLayer(
                {
                    id: "transitStops",
                    type: "circle",
                    source: {
                        type: "geojson",
                        data: transitStops,
                    },
                    paint: {
                        "circle-radius": [
                            "interpolate",
                            ["linear"],
                            ["zoom"],
                            5,
                            0,
                            6,
                            1,
                            20,
                            10,
                        ],
                        "circle-color": "#8DBF2E",
                    },
                },
                "bridge-minor-case"
            );
        });

        map.on("load", function () {
            map.addLayer(
                {
                    id: "transitLines",
                    type: "line",
                    source: {
                        type: "geojson",
                        data: transitLines,
                    },
                    paint: {
                        "line-width": [
                            "interpolate",
                            ["linear"],
                            ["zoom"],
                            5,
                            0,
                            6,
                            1,
                            10,
                            2,
                        ],
                        "line-color": "#8DBF2E",
                    },
                },
                "bridge-minor-case"
            );
        });

        map.on("load", function () {
            map.addLayer({
                id: "cmaPoints",
                type: "circle",
                source: {
                    type: "geojson",
                    data: cmaPoints,
                },
                paint: {
                    "circle-radius": 6,
                    "circle-color": "#1E3765",
                    "circle-stroke-width": 2,
                    "circle-stroke-color": "#fff",
                    "circle-opacity": [
                        "interpolate",
                        ["linear"],
                        ["zoom"],
                        4.99,
                        1,
                        5,
                        0,
                    ],
                    "circle-stroke-opacity": [
                        "interpolate",
                        ["linear"],
                        ["zoom"],
                        4.99,
                        1,
                        5,
                        0,
                    ],
                },
            });
        });

        map.on("mouseenter", "cmaPoints", () => {
            map.getCanvas().style.cursor = "pointer";
        });

        map.on("mouseleave", "cmaPoints", () => {
            map.getCanvas().style.cursor = "";
        });

        map.on("click", "cmaPoints", (e) => {
            cmaSelectMapUpdate(e.features[0].properties.cmaname);
            $: layerSet(cmauidSelected, mapSelected);
        });

        map.on("mouseenter", "metro-mindset-cma-2021-bg-all-fill", (e) => {
            if (cmaSelected !== e.features[0].properties.CMANAME) {
                map.getCanvas().style.cursor = "pointer";
            }
        });

        map.on("mouseleave", "metro-mindset-cma-2021-bg-all-fill", (e) => {
            map.getCanvas().style.cursor = "";
        });

        map.on("click", "metro-mindset-cma-2021-bg-all-fill", (e) => {
            if (e.features[0].properties.CMANAME !== cmaSelected) {
                console.log(e.features[0]);
                cmaSelectMapUpdate(e.features[0].properties.CMANAME);
                console.log(cmauidSelected, mapSelected);
                $: layerSet(cmauidSelected, mapSelected);
            }
        });

        loadCensusTract(cmauidSelected);
    });
</script>

<svelte:head>
    <link
        href="https://api.mapbox.com/mapbox-gl-js/v2.14.1/mapbox-gl.css"
        rel="stylesheet"
    />
</svelte:head>

<main>
    <div id="content">
        <h1>Metropolitan Mindset</h1>

        <div class="bar" />

        <p>
            This map is still under construction :) Add brief blurb about this
            project here
        </p>

        <div id="municipality-toggle">
            <h2>
                <input type="checkbox" on:change={toggleMunicipal} checked />
                Municipal Borders
            </h2>
        </div>

        <div
            id="content-wrapper"
            style="display: {isContentVisible ? 'block' : 'none'};"
        >
            <p>
                <i>Select to zoom to a metropolitan area:</i>
            </p>

            <div class="bar" />

            <div id="select-wrapper">
                <Select
                    id="select"
                    items={cmaAll}
                    value={cmaSelected}
                    clearable={false}
                    showChevron={true}
                    on:input={cmaSelectDropDown}
                    --background="white"
                    --selected-item-color="#6D247A"
                    --height="22px"
                    --item-color="#6D247A"
                    --border-radius="0"
                    --border="1px"
                    --list-border-radius="0px"
                    --font-size="16px"
                    --max-height="30px"
                    --item-is-active-color="#0D534D"
                    --item-is-active-bg="#6FC7EA"
                />
            </div>

            <div class="bar" />

            <p>
                Number of Municipalities: <span id="number"
                    >{filteredData["Total_Number_of_Municipalities"]}</span
                ><br />

                Total Land Area (sq.km):
                <span id="number"
                    >{Math.round(
                        filteredData["Land_Area_(sq_km)"]
                    ).toLocaleString()}</span
                ><br />

                GDP (2019) in mllions:
                <span id="number"
                    >${Math.round(
                        filteredData["GDP_2019_(millions)"]
                    ).toLocaleString()}</span
                ><br />

                Population (2021):
                <span id="number"
                    >{Math.round(
                        filteredData["Population_2021"]
                    ).toLocaleString()}</span
                ><br />

                % of Population in Central City:
                <span id="number"
                    >{filteredData[
                        "Proportion_of_Population_In_Central"
                    ]}%</span
                >
            </p>

            <p>
                <i>Select to switch between map layers:</i>
            </p>

            <div class="bar" />

            <div id="select-wrapper">
                <Select
                    id="select"
                    items={mapLayers}
                    value={"Street Map"}
                    clearable={false}
                    showChevron={true}
                    on:input={layerSelect}
                    --background="white"
                    --selected-item-color="#6D247A"
                    --height="22px"
                    --item-color="#6D247A"
                    --border-radius="0"
                    --border="1px"
                    --list-border-radius="0px"
                    --font-size="16px"
                    --max-height="30px"
                    --item-is-active-color="#0D534D"
                    --item-is-active-bg="#6FC7EA"
                />
            </div>

            <div class="bar" />

            <div class="legend">
                {#if mapSelected === "Population Density"}
                
                <div id="legend-wrapper">
                    <svg id="legend-svg" height="130">
                    
                        <rect class="legend-box" x="10" y="30" width="15" height="15" fill="{choropleths["population-density"].colours[3]}" />
                        <text x="30" y="42" class="legend-text" font-size="12" >6,000 people/km2 and up</text>
                            
                        <rect class="legend-box" x="10" y="50" width="15" height="15" fill="{choropleths["population-density"].colours[2]}" />
                        <text x="30" y="62" class="legend-text" font-size="12" >3,000 to 6,000 people/km2</text>
    
                        <rect class="legend-box" x="10" y="70" width="15" height="15" fill="{choropleths["population-density"].colours[1]}" />
                        <text x="30" y="82" class="legend-text" font-size="12" >500 to 3,000 people/km2</text>
    
                        <rect class="legend-box" x="10" y="90" width="15" height="15" fill="{choropleths["population-density"].colours[0]}" />
                        <text x="30" y="102" class="legend-text" font-size="12" >less than 500 people/km2</text>
    
                        <rect class="legend-box" x="10" y="110" width="15" height="15" fill="#D0D1C9" />
                        <text x="30" y="122" class="legend-text" font-size="12" >No Data: </text>
    
                    </svg>
                </div>

                {/if}

                <p>
                    Map created by Jeff Allen at the School of Cities. Data
                    sources: Statistics Canada, OpenStreetMap, Mapbox
                </p>
            </div>

            <div class="bar" />
        </div>

        <div id="hide" on:click={toggleContent}>
            <i
                >{isContentVisible
                    ? "Click here to hide this panel"
                    : "Click here to show details about this map"}</i
            >
        </div>
    </div>

    <div id="map" />
</main>

<style>
    :global(body) {
        padding: 0px;
        margin: 0px;
    }

    main {
        margin: auto;
        width: 100%;
        height: 100%;
    }

    #map {
        height: 100%;
        width: 100%;
        position: absolute;
        z-index: -99;
    }

    h1 {
        font-size: 24px;
        font-family: TradeGothicBold;
        margin: 0px;
        padding: 10px;
        padding-bottom: 5px;
        font-weight: 600;
        color: var(--brandPurple);
    }

    h2 {
        font-size: 15px;
        font-family: Roboto;
        font-weight: 500;
        margin: 0px;
        padding: 10px;
        padding-bottom: 5px;
        padding-top: 5px;
        color: var(--brandPink);
    }

    p {
        font-size: 13px;
        font-family: RobotoRegular;
        line-height: 15px;
        margin: 0px;
        padding: 0px;
        padding-left: 10px;
        padding-right: 10px;
        padding-bottom: 6px;
        padding-top: 6px;
        opacity: 1;
        color: var(--brandDarkBlueFade);
    }

    a {
        color: var(--brandMedBlue);
    }

    #content {
        width: 250px;
        position: absolute;
        top: 1px;
        left: 1px;
        background-color: white;
        border: solid 1px lightgrey;
        border-radius: 0px;
        z-index: 1;
    }

    #number {
        color: var(--brandMedGreen);
        /* font-size: 15px; */
    }

    .bar {
        height: 1px;
        width: 245px;
        background-color: var(--brandDarkBlue);
        padding: 0px;
        margin: 0px;
        margin-left: 5px;
        opacity: 0.25;
    }

    #hide {
        font-family: RobotoRegular;
        font-size: 12px;
        height: 18px;
        text-align: left;
        background-color: none;
        padding-top: 3px;
        padding-left: 10px;
        opacity: 0.8;
        color: var(--brandDarkBlue);
    }

    #hide:hover {
        background-color: #f5f5f5;
        cursor: pointer;
    }

    input[type="checkbox"] {
        accent-color: var(--brandPink);
    }

    #legend {
        background-color: white;
    }

    .legend-box {
        stroke: rgb(181, 181, 181);
        stroke-width: 0.5px;
    }

    .legend-bar {
        fill: #efefef;
        stroke: #cecece;
        stroke-width: 0.5px;
    }

    .legend-text {
        font-family: RobotoRegular;
        fill: var(--brandGray80);
    }
</style>
