<script>

    import "../assets/global.css";
    import { onMount } from 'svelte';
	import Select from 'svelte-select';
	import mapboxgl from "mapbox-gl";
    import * as topojson from "topojson-client";

	import cmaSummary from '../assets/cma-summary.json';
	import transitLines from '../assets/transit-lines-canada.geo.json';
    import transitStops from '../assets/transit-stops-canada.geo.json';
	import municipalLabels from '../assets/csd-2021-centroids.geo.json';

    mapboxgl.accessToken = 'pk.eyJ1Ijoic2Nob29sb2ZjaXRpZXMiLCJhIjoiY2w2Z2xhOXprMTYzczNlcHNjMnNvdGlmNCJ9.lOgVHrajc1L-LlU0as2i2A';

    let isContentVisible = true;
    function toggleContent() {
        isContentVisible = !isContentVisible;
    }


    // Changing the CMA

	let cmaSelected = 'Toronto';
	let cmauidSelected = 535;

	const cmaData = cmaSummary.filter(item => item.Rank < 11);
	console.log(cmaData);

	let filteredData;
	$: filteredData = cmaData.filter(item => item.CMAUID === cmauidSelected)[0];
	$: console.log(filteredData);

	let cmaAll = cmaData
        .sort((a, b) => b.pop2021 - a.pop2021)
        .map(item => item.CMANAME)


    function cmaSelect(e) {
		cmaSelected = e.detail.value;
		let filteredData = cmaData.filter(item => item.CMANAME === cmaSelected)[0];
		
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
			false
		]
		map.setFilter('metro-mindset-csd-2021-border', cmaFilter)
		map.setFilter('metro-mindset-cma-2021-border', cmaFilter)
		map.setFilter('metro-mindset-cma-2021-background', cmaFilter)
		map.setFilter('municipalLabels', cmaFilter)
	};



    // Add / remove municipal borders

	let isMunicipalChecked = true;
    function toggleMunicipal() {
        isMunicipalChecked = !isMunicipalChecked;
        if (isMunicipalChecked) {
            map.setPaintProperty('metro-mindset-csd-2021-border', 'line-opacity', 1);
			map.setPaintProperty('municipalLabels', 'text-opacity', 1);
        } 
        else {
            map.setPaintProperty('metro-mindset-csd-2021-border', 'line-opacity', 0);
			map.setPaintProperty('municipalLabels', 'text-opacity', 0);
        }
    };



    // Changing the map layer

	let mapLayers = ["Street Map", "Satellite", "Population Density"];
	let mapSelected = "Street Map"

    function layerSelect(e) {
		$: mapSelected = e.detail.value;
		
		if (mapSelected === "Street Map") {
			map.setPaintProperty('mapbox-satellite', 'raster-opacity', 0.011);
            // remove CT layer
		}
		if (mapSelected === "Satellite") {
			map.setPaintProperty('mapbox-satellite', 'raster-opacity', 0.798);
            // remove CT layer
		}
        if (mapSelected === "Population Density") {
            console.log(mapSelected);
            // if CT is null
            //   add CT
            //   join to tabular data

        }
	}

    let ct;
    async function loadCensusTract(cmauid) {
        if (cmauid === cmauidSelected) { // Check the passed cmauid
            try {
                console.log(cmauid);
                const response = await fetch(`ct-${cmauid}-2021.topo.json`);
                ct = await response.json();
                ct = topojson.feature(ct, "mtl-ct-2021");

                // load in and join tabular data

                console.log(ct)
            } catch (error) {
                console.error('Error loading CT data files:', error);
            }
        }
    }

    $: {
        loadCensusTract(cmauidSelected);
    }






    // Map setup

	let map;

	onMount(() => {
		map = new mapboxgl.Map({
			container: 'map', 
			style: 'mapbox://styles/schoolofcities/cllwjnzlf016j01p71qq5eskt',
			center: [-79.6, 43.9], 
			zoom: 9,
			maxZoom: 12,
			minZoom: 5,
			projection: 'globe',
			scrollZoom: true,
			attributionControl: false
		});
		
		const scale = new mapboxgl.ScaleControl({
			maxWidth: 100,
			unit: 'metric'
			});
		map.addControl(scale, 'bottom-right');

		map.addControl(new mapboxgl.NavigationControl(), 'bottom-right');

		map.on('load', function () {
            map.addLayer({
            id: 'municipalLabels',
            type: 'symbol',
            source: {
                type: 'geojson',
                data: municipalLabels
            },
            layout: {
				'text-field': '{CSDNAME}', 
				'text-font': ['Roboto Regular'], 
				'text-size': [
					'interpolate',
					['linear'],
					['zoom'],
					9, 12, 
					18, 40 
				],
				'text-anchor': 'center'
			},
			paint: {
				'text-color': '#AB1368',
				'text-halo-color': 'rgba(255, 255, 255, 0.65)', 
				'text-halo-width': 2,
				'text-halo-blur': 1	
			}
            });

			map.setFilter('municipalLabels', [
				"match",
				["get", "CMAUID"],
				['535'],
				true,
				false
			])
        });

		map.on('load', function () {
            map.addLayer({
            id: 'transitStops',
            type: 'circle',
            source: {
                type: 'geojson',
                data: transitStops
            },
            paint: {
                'circle-radius': [
                    'interpolate',
                    ['linear'],
                    ['zoom'],
                    5, 0,
                    6, 1,
                    20, 7
                ],
                'circle-color': '#8DBF2E',
            }
            }, 'bridge-minor-case');
        });

        map.on('load', function () {
            map.addLayer({
            id: 'transitLines',
            type: 'line',
            source: {
                type: 'geojson',
                data: transitLines
            },
            paint: {
                'line-width': [
                    'interpolate',
                    ['linear'],
                    ['zoom'],
                    5, 0,
                    6, 1,
                ],
                'line-color': '#8DBF2E',
            }
            }, 'bridge-minor-case');
        });

	});



	

</script>



<svelte:head>
	<link href='https://api.mapbox.com/mapbox-gl-js/v2.14.1/mapbox-gl.css' rel='stylesheet' />
</svelte:head>



<main>

	<div id="content">

        <h1>Metropolitan Mindset</h1>

		<div class="bar"></div>

		<p>This map is still under construction :) Add brief blurb about this project here</p>

		<div id="municipality-toggle">
			<h2>
				<input type="checkbox" on:change={toggleMunicipal} checked>
				Municipal Borders
			</h2>
		</div>     

        <div id="content-wrapper" style="display: {isContentVisible ? 'block' : 'none'};">
			
			<p>
				<i>Select to zoom to a metropolitan area:</i>
			</p>

            <div class="bar"></div>

            <div id="select-wrapper">
                <Select 
                    id="select"
                    items={cmaAll} 
                    value={"Toronto"} 
                    clearable={false} 
                    showChevron={true} 
                    on:input={cmaSelect}
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
            
            <div class="bar"></div>

			<p>
				Number of Municipalities: <span id="number">{filteredData["Total_Number_of_Municipalities"]}</span><br>

				Total Land Area (sq.km): <span id="number">{Math.round(filteredData["Land_Area_(sq_km)"]).toLocaleString()}</span><br>

				GDP (2019) in mllions: <span id="number">${Math.round(filteredData["GDP_2019_(millions)"]).toLocaleString()}</span><br>

				Population (2021): <span id="number">{Math.round(filteredData["Population_2021"]).toLocaleString()}</span><br>

				% of Population in Central City: <span id="number">{filteredData["Proportion_of_Population_In_Central"]}%</span>
			</p>

			<p>
				<i>Select to switch between map layers:</i>
			</p>

			<div class="bar"></div>

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

			<div class="bar"></div>

			<p>Map created by Jeff Allen at the School of Cities. Data sources: Statistics Canada, OpenStreetMap, Mapbox</p>

			<div class="bar"></div>

        </div>

        <div id="hide" on:click={toggleContent}>
            <i>{isContentVisible ? "Click here to hide this panel" : "Click here to show details about this map"}</i>
        </div>

    </div>

	<div id="map"></div>

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

	input[type=checkbox]{
		accent-color: var(--brandPink);  
	}

</style>
