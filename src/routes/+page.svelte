<script>

    import "../assets/global.css";
    import { onMount } from 'svelte';
	import Select from 'svelte-select';
	import mapboxgl from "mapbox-gl";
	import cmaSummary from '../assets/cma-summary.json';

	mapboxgl.accessToken = 'pk.eyJ1Ijoic2Nob29sb2ZjaXRpZXMiLCJhIjoiY2w2Z2xhOXprMTYzczNlcHNjMnNvdGlmNCJ9.lOgVHrajc1L-LlU0as2i2A';

	const cmaData = cmaSummary.filter(item => item.Rank < 11);
	console.log(cmaData);

	let cmaAll = cmaData
        .sort((a, b) => b.pop2021 - a.pop2021)
        .map(item => item.CMANAME)


	let isContentVisible = true;
    function toggleContent() {
        isContentVisible = !isContentVisible;
    }

	let isMunicipalChecked = true;
    function toggleMunicipal() {
        isMunicipalChecked = !isMunicipalChecked;
        if (isMunicipalChecked) {
            map.setPaintProperty('metro-mindset-csd-2021-border', 'line-opacity', 1);
        } 
        else {
            map.setPaintProperty('metro-mindset-csd-2021-border', 'line-opacity', 0);
        }
    };

	onMount(() => {
		map = new mapboxgl.Map({
			container: 'map', 
			style: 'mapbox://styles/schoolofcities/cllwjnzlf016j01p71qq5eskt',
			center: [-79.6, 43.9], 
			zoom: 9,
			maxZoom: 11,
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
	});

	function cmaSelect(e) {
		console.log(e)
	};

</script>



<svelte:head>
	<link href='https://api.mapbox.com/mapbox-gl-js/v2.10.0/mapbox-gl.css' rel='stylesheet' />
</svelte:head>



<main>

	<div id="content">

        <h1>Metropolitan Mindset Maps</h1>

            <div id="content-wrapper" style="display: {isContentVisible ? 'block' : 'none'};">
            <div class="bar"></div>

            <p>
                meow meow meow
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
                    --height="20px"
                    --item-color="black"
                    --border-radius="0"
                    --border="1px"
                    --list-border-radius="0px"
                    --font-size="15px"
                    --max-height="30px"
                    --item-is-active-color="black"
                    --item-is-active-bg="lightgrey"
                />
            </div>

            <div class="bar"></div>

            <div id="satellite-switch">
                <p>
                    <input type="checkbox" on:change={toggleMunicipal} checked>
                    Municipal Boundaries
                </p>
            </div>

            <div class="bar"></div>

        <div class="bar"></div>

            <p>
                This map was built by ..... Code is on <a href="">GitHub</a>
            </p>

        </div>

        <div id="hide" on:click={toggleContent}>
            {isContentVisible ? "Click here to hide this panel" : "Click here to show details about this map"}
        </div>

    </div>

	<div id="map"></div>

</main>



<style>

	:global(body) {
		padding: 0px;
		margin: 0px;
		background-color: var(--brandDarkBlue);
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
        font-size: 27px;
        font-family: TradeGothicBold;
        margin: 0px;
        padding: 10px;
        padding-bottom: 5px;
        color: var(--brandDarkBlue);
    }

    p {
        font-size: 12px;
        font-family: RobotoRegular;
        line-height: 15px;
        margin: 0px;
        padding: 0px;
        padding-left: 10px;
        padding-right: 10px;
        padding-bottom: 5px;
        padding-top: 5px;
        color: var(--brandGray80);
    }

    a {
        color: var(--brandMedBlue);
    }

    #content {
        width: 300px;
        position: absolute;
        top: 5px;
        left: 5px;
        background-color: white; 
        border: solid 1px lightgrey;
        border-radius: 5px;
        z-index: 1; 
    }

    .bar {
        height: 1px;
        width: 290px;
        background-color: var(--brandDarkBlue);
        padding: 0px;
        margin: 0px;
        margin-left: 5px;
        opacity: 0.25;
    }
	
</style>
