<script>

    import "../assets/global.css"
    import { onMount } from 'svelte'
	import mapboxgl from "mapbox-gl";

	mapboxgl.accessToken = 'pk.eyJ1Ijoic2Nob29sb2ZjaXRpZXMiLCJhIjoiY2w2Z2xhOXprMTYzczNlcHNjMnNvdGlmNCJ9.lOgVHrajc1L-LlU0as2i2A';

	let isContentVisible = true;
    function toggleContent() {
        isContentVisible = !isContentVisible;
    }

	let isChecked = false;
    function toggleCheckbox() {
        isChecked = !isChecked;
        if (isChecked) {
            map.setPaintProperty('mapbox-satellite', 'raster-opacity', 0.63);
        } 
        else {
            map.setPaintProperty('mapbox-satellite', 'raster-opacity', 0);
        }
    };

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
		
		map.addControl(new mapboxgl.NavigationControl(), 'top-right');

		const scale = new mapboxgl.ScaleControl({
			maxWidth: 100,
			unit: 'metric'
			});
		map.addControl(scale, 'bottom-right');
	});

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

            <!-- <div id="select-wrapper">
                <Select 
                    id="select"
                    items={cmaAll} 
                    value={cmaSelected} 
                    clearable={false} 
                    showChevron={true} 
                    on:input={handleSelect}
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
            </div> -->

            <div class="bar"></div>

            <div id="satellite-switch">
                <p>
                    <input type="checkbox" on:change={toggleCheckbox}>
                    Satellite View
                    <svg width="30" height="15" xmlns="http://www.w3.org/2000/svg">
                        <line x1="8" y1="10" x2="30" y2="10" stroke="#AB1368" stroke-width="1"/>
                        <circle cx="19" cy="10" r="3" fill="#AB1368"/>
                    </svg>
                    Major Transit Line
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
