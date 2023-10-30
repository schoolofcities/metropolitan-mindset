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

    let mapLayers = ["Street Map", "Satellite", "Population Density", "Dwellings Density", "Median Household Income",  "Percent Rent", "Percent in Core Housing Need", "Percent Recent Immigrant"];
    let mapSelected = "Street Map";

    const choropleths = {
        "population-density": {
            name: "Population Density",
            breaks: [500, 3000, 6000],
            colours: ["#fffef8", "#fbefb5", "#f7dd66", "#f1c500"],
            // "colours": ["#fcfcfc", "#a4dcd4", "#4ebdad", "#00a189"]
        },
        "median-household-income": {
            name: "Median Household Income",
            breaks: [50000, 75000, 100000, 150000],
            colours: ["#e07260", "#e6b8a9", "#eaeee0", "#a8cfcf", "#3d9cb3"],
            //colours: ["#EDF1F7", "#C3D1E5", "#6F91C1", "#375681", "#1C2C42"],
        },
        "dwellings-density": {
            name: "Dwellings Density",
            breaks: [100, 1000, 2000],
            colours: ["#fffef8", "#fbefb5", "#f7dd66", "#f1c500"],
        },
        "perc-rent": {
            name: "% of Renter",
            breaks: [0.2, 0.35, 0.5, 0.7],
            colours: ["#FAF8F2", "#E9E1C7", "#D0BE86", "#B59A46", "#74632D"],
        },
        "perc-corehous-need": {
            name: "% of Core Housing Need",
            breaks: [0.06, 0.1, 0.16, 0.2],
            colours: ["#EBDEE0", "#C9A6AB", "#A76E76", "#74474D", "#3C2528"],
            //colours: ["#ECF3EF", "#B5D0C1", "#7EAD93", "#518066", "#2E493A"],
        },
        "perc-rec-immig": {
            name: "% of Recent Immigrant",
            breaks: [0.05, 0.085, 0.13, 0.2],
            colours: ["#F5EDE9", "#D9BAAB", "#BD876D", "#8F5A41", "#513325"],
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
        } else if (layer === "Median Household Income") {
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
                            ["!=", ["get", "MedHhldIncome"], null],
                            [
                                "step",
                                ["get", "MedHhldIncome"],
                                choropleths["median-household-income"].colours[0],
                                choropleths["median-household-income"].breaks[0],
                                choropleths["median-household-income"].colours[1],
                                choropleths["median-household-income"].breaks[1],
                                choropleths["median-household-income"].colours[2],
                                choropleths["median-household-income"].breaks[2],
                                choropleths["median-household-income"].colours[3],
                                choropleths["median-household-income"].breaks[3],
                                choropleths["median-household-income"].colours[4],
                            ],
                            "#cbcbcb",
                        ],
                    },
                },
                "transitStops"
            );
        } else if (layer === "Dwellings Density") {
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
                            ["!=", ["get", "Dwellings"], null],
                            [
                                "step",
                                ["/", ["get", "Dwellings"], ["get", "Area"]],
                                choropleths["dwellings-density"].colours[0],
                                choropleths["dwellings-density"].breaks[0],
                                choropleths["dwellings-density"].colours[1],
                                choropleths["dwellings-density"].breaks[1],
                                choropleths["dwellings-density"].colours[2],
                                choropleths["dwellings-density"].breaks[2],
                                choropleths["dwellings-density"].colours[3],
                            ],
                            "#cbcbcb",
                        ],
                    },
                },
                "transitStops"
            );
        } else if (layer === "% of Renter") {
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
                            ["!=", ["get", "PerRent"], null],
                            [
                                "step",
                                ["get", "PerRent"],
                                choropleths["perc-rent"].colours[0],
                                choropleths["perc-rent"].breaks[0],
                                choropleths["perc-rent"].colours[1],
                                choropleths["perc-rent"].breaks[1],
                                choropleths["perc-rent"].colours[2],
                                choropleths["perc-rent"].breaks[2],
                                choropleths["perc-rent"].colours[3],
                                choropleths["perc-rent"].breaks[3],
                                choropleths["perc-rent"].colours[4],
                            ],
                            "#cbcbcb",
                        ],
                    },
                },
                "transitStops"
            );
        } else if (layer === "% of Core Housing Need") {
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
                            ["!=", ["get", "PerInCoreNeed"], null],
                            [
                                "step",
                                ["get", "PerInCoreNeed"],
                                choropleths["perc-corehous-need"].colours[0],
                                choropleths["perc-corehous-need"].breaks[0],
                                choropleths["perc-corehous-need"].colours[1],
                                choropleths["perc-corehous-need"].breaks[1],
                                choropleths["perc-corehous-need"].colours[2],
                                choropleths["perc-corehous-need"].breaks[2],
                                choropleths["perc-corehous-need"].colours[3],
                                choropleths["perc-corehous-need"].breaks[3],
                                choropleths["perc-corehous-need"].colours[4],
                            ],
                            "#cbcbcb",
                        ],
                    },
                },
                "transitStops"
            );
        } else if (layer === "% of Recent Immigrant") {
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
                            ["!=", ["get", "PerRecImmig"], null],
                            [
                                "step",
                                ["get", "PerRecImmig"],
                                choropleths["perc-rec-immig"].colours[0],
                                choropleths["perc-rec-immig"].breaks[0],
                                choropleths["perc-rec-immig"].colours[1],
                                choropleths["perc-rec-immig"].breaks[1],
                                choropleths["perc-rec-immig"].colours[2],
                                choropleths["perc-rec-immig"].breaks[2],
                                choropleths["perc-rec-immig"].colours[3],
                                choropleths["perc-rec-immig"].breaks[3],
                                choropleths["perc-rec-immig"].colours[4],
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
            center: [-100, 55],
            zoom: 2.5,
            maxZoom: 12,
            minZoom: 1,
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
                        "circle-color": "#8a8783",
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
                        "line-color": "#8a8783",
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
            This map is still under construction :) Add brief blurb about this project here
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
                    --font-size="14.45px"
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
                    --font-size="14.45px"
                    --max-height="30px"
                    --item-is-active-color="#0D534D"
                    --item-is-active-bg="#6FC7EA"
                />
            </div>

            <div class="bar" />

            <div class="legend">
                {#if mapSelected === "Population Density"}

                <div id="legend-wrapper">
                    <svg id="legend-svg" height="110">
                    
                        <rect class="legend-box" x="10" y="10" width="15" height="15" fill="{choropleths["population-density"].colours[3]}" />
                        <text x="30" y="22" class="legend-text" font-size="12" >6,000 people/km2 and up</text>
                            
                        <rect class="legend-box" x="10" y="30" width="15" height="15" fill="{choropleths["population-density"].colours[2]}" />
                        <text x="30" y="42" class="legend-text" font-size="12" >3,000 to 6,000 people/km2</text>
    
                        <rect class="legend-box" x="10" y="50" width="15" height="15" fill="{choropleths["population-density"].colours[1]}" />
                        <text x="30" y="62" class="legend-text" font-size="12" >500 to 3,000 people/km2</text>
    
                        <rect class="legend-box" x="10" y="70" width="15" height="15" fill="{choropleths["population-density"].colours[0]}" />
                        <text x="30" y="82" class="legend-text" font-size="12" >Less than 500 people/km2</text>
    
                        <rect class="legend-box" x="10" y="90" width="15" height="15" fill="#D0D1C9" />
                        <text x="30" y="102" class="legend-text" font-size="12" >No Data</text>
    
                    </svg>
                </div>

                {/if}

                {#if mapSelected === "Median Household Income"}
                
                <div id="legend-wrapper">
                    <svg id="legend-svg" height="145">
                    
                        <rect class="legend-box" x="10" y="10" width="15" height="15" fill="{choropleths["median-household-income"].colours[4]}" />
                        <text x="30" y="22" class="legend-text" font-size="12" >$150,000 and up</text>

                        <rect class="legend-box" x="10" y="30" width="15" height="15" fill="{choropleths["median-household-income"].colours[3]}" />
                        <text x="30" y="42" class="legend-text" font-size="12" >$100,000 to $150,000</text>
                            
                        <rect class="legend-box" x="10" y="50" width="15" height="15" fill="{choropleths["median-household-income"].colours[2]}" />
                        <text x="30" y="62" class="legend-text" font-size="12" >$75,000 to $100,000</text>
    
                        <rect class="legend-box" x="10" y="70" width="15" height="15" fill="{choropleths["median-household-income"].colours[1]}" />
                        <text x="30" y="82" class="legend-text" font-size="12" >$50,000 to $75,000</text>
    
                        <rect class="legend-box" x="10" y="90" width="15" height="15" fill="{choropleths["median-household-income"].colours[0]}" />
                        <text x="30" y="102" class="legend-text" font-size="12" >Less than $50,000</text>
    
                        <rect class="legend-box" x="10" y="110" width="15" height="15" fill="#D0D1C9" />
                        <text x="30" y="122" class="legend-text" font-size="12" >No Data</text>

                        <text x="10" y="142" class="legend-text" font-size="12" >(all $ values are based on after-tax income)</text>
                        
                        </svg>
                </div>

                {/if}

                {#if mapSelected === "Dwellings Density"}
                
                <div id="legend-wrapper">
                    <svg id="legend-svg" height="110">

                        <rect class="legend-box" x="10" y="10" width="15" height="15" fill="{choropleths["dwellings-density"].colours[3]}" />
                        <text x="30" y="22" class="legend-text" font-size="12" >2,000 dwellings/km2 and up</text>
                            
                        <rect class="legend-box" x="10" y="30" width="15" height="15" fill="{choropleths["dwellings-density"].colours[2]}" />
                        <text x="30" y="42" class="legend-text" font-size="12" >1,000 to 2,000 dwellings/km2</text>
    
                        <rect class="legend-box" x="10" y="50" width="15" height="15" fill="{choropleths["dwellings-density"].colours[1]}" />
                        <text x="30" y="62" class="legend-text" font-size="12" >100 to 1,000 dwellings/km2</text>
    
                        <rect class="legend-box" x="10" y="70" width="15" height="15" fill="{choropleths["dwellings-density"].colours[0]}" />
                        <text x="30" y="82" class="legend-text" font-size="12" >Less than 100 dwellings/km2</text>
    
                        <rect class="legend-box" x="10" y="90" width="15" height="15" fill="#D0D1C9" />
                        <text x="30" y="102" class="legend-text" font-size="12" >No Data</text>
    
                    </svg>
                    
                </div>

                {/if}

                {#if mapSelected === "% of Renter"}
                
                <div id="legend-wrapper">
                    <svg id="legend-svg" height="125">
                    
                        <rect class="legend-box" x="10" y="10" width="15" height="15" fill="{choropleths["perc-rent"].colours[4]}" />
                        <text x="30" y="22" class="legend-text" font-size="12" >70% and up</text>

                        <rect class="legend-box" x="10" y="30" width="15" height="15" fill="{choropleths["perc-rent"].colours[3]}" />
                        <text x="30" y="42" class="legend-text" font-size="12" >50% to 70%</text>
                            
                        <rect class="legend-box" x="10" y="50" width="15" height="15" fill="{choropleths["perc-rent"].colours[2]}" />
                        <text x="30" y="62" class="legend-text" font-size="12" >35% to 50%</text>
    
                        <rect class="legend-box" x="10" y="70" width="15" height="15" fill="{choropleths["perc-rent"].colours[1]}" />
                        <text x="30" y="82" class="legend-text" font-size="12" >20% to 35%</text>
    
                        <rect class="legend-box" x="10" y="90" width="15" height="15" fill="{choropleths["perc-rent"].colours[0]}" />
                        <text x="30" y="102" class="legend-text" font-size="12" >less than 20%</text>
    
                        <rect class="legend-box" x="10" y="110" width="15" height="15" fill="#D0D1C9" />
                        <text x="30" y="122" class="legend-text" font-size="12" >No Data</text>
                        
                        </svg>
                </div>

                {/if}

                {#if mapSelected === "% of Core Housing Need"}
                
                <div id="legend-wrapper">
                    <svg id="legend-svg" height="125">
                    
                        <rect class="legend-box" x="10" y="10" width="15" height="15" fill="{choropleths["perc-corehous-need"].colours[4]}" />
                        <text x="30" y="22" class="legend-text" font-size="12" >20% and up</text>

                        <rect class="legend-box" x="10" y="30" width="15" height="15" fill="{choropleths["perc-corehous-need"].colours[3]}" />
                        <text x="30" y="42" class="legend-text" font-size="12" >16% to 20%</text>
                            
                        <rect class="legend-box" x="10" y="50" width="15" height="15" fill="{choropleths["perc-corehous-need"].colours[2]}" />
                        <text x="30" y="62" class="legend-text" font-size="12" >10% to 16%</text>
    
                        <rect class="legend-box" x="10" y="70" width="15" height="15" fill="{choropleths["perc-corehous-need"].colours[1]}" />
                        <text x="30" y="82" class="legend-text" font-size="12" >0.6% to 10%</text>
    
                        <rect class="legend-box" x="10" y="90" width="15" height="15" fill="{choropleths["perc-corehous-need"].colours[0]}" />
                        <text x="30" y="102" class="legend-text" font-size="12" >less than 0.6%</text>
    
                        <rect class="legend-box" x="10" y="110" width="15" height="15" fill="#D0D1C9" />
                        <text x="30" y="122" class="legend-text" font-size="12" >No Data</text>
                        
                        </svg>
                </div>

                {/if}


                {#if mapSelected === "% of Recent Immigrant"}
                
                <div id="legend-wrapper">
                    <svg id="legend-svg" height="125">
                    
                        <rect class="legend-box" x="10" y="10" width="15" height="15" fill="{choropleths["perc-rec-immig"].colours[4]}" />
                        <text x="30" y="22" class="legend-text" font-size="12" >20% and up</text>

                        <rect class="legend-box" x="10" y="30" width="15" height="15" fill="{choropleths["perc-rec-immig"].colours[3]}" />
                        <text x="30" y="42" class="legend-text" font-size="12" >13% to 20%</text>
                            
                        <rect class="legend-box" x="10" y="50" width="15" height="15" fill="{choropleths["perc-rec-immig"].colours[2]}" />
                        <text x="30" y="62" class="legend-text" font-size="12" >0.85% to 13%</text>
    
                        <rect class="legend-box" x="10" y="70" width="15" height="15" fill="{choropleths["perc-rec-immig"].colours[1]}" />
                        <text x="30" y="82" class="legend-text" font-size="12" >0.5% to 0.85%</text>
    
                        <rect class="legend-box" x="10" y="90" width="15" height="15" fill="{choropleths["perc-rec-immig"].colours[0]}" />
                        <text x="30" y="102" class="legend-text" font-size="12" >less than 0.5%</text>
    
                        <rect class="legend-box" x="10" y="110" width="15" height="15" fill="#D0D1C9" />
                        <text x="30" y="122" class="legend-text" font-size="12" >No Data</text>
                        
                        </svg>
                </div>

                {/if}

                <p id="note">
                    Map created by Jeff Allen and Irene Chang at the School of Cities. 
                    Data sources: Statistics Canada, OpenStreetMap, Mapbox
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

    #note {
        font-size: 10px;
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
        line-height: 14px;
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
        fill: var(--brandDarkBlueFade);
    }
</style>
