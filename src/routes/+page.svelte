<script>
    import * as d3 from "d3";
    import { onMount } from "svelte";
    import BinGraph from "./BinGraph.svelte";
    import DotAnimation from "./DotAnimation.svelte";
    import EvictionTime from "./EvictionTime.svelte";
    import {
        computePosition,
        autoPlacement,
        offset
    } from '@floating-ui/dom';

    let evict_dataRAW = [];
    let dataRAW  = [];
    let data = [];
    let temp_data = [];
    let temp_dataRAW = [];
    let width = 800, height = 275; // changed the height of the graph from 600 to 450
    let yScale = d3.scaleLinear();
    let xScaleHousehold = d3.scaleBand();
    let xScaleRace = d3.scaleBand();
    let xScaleElder = d3.scaleBand();
    let xScaleCorp = d3.scaleBand();
    let yAxisGridlines;
    let brushedSelection;
    let selectedEvictions;
    let hasSelection;
    let selectedLines;
    let family_bins = ["family", "non-family", "mixed"];
    let race_bins = ["Black", "White", "Latino", "Other"];
    let elder_bins = ["some elders", "no elders"];
    let corp_bins = ["low", "medium", "high"];
    let text_description = ["Majority, more than 50%, of the tenants in the house are a particular race.", 
    "The tenants in the house are either all family members, not related at all, or a mix of family members and non family members.", 
    "The housing property either contains some elderly people or no elderly people.", 
    "The housing properties in the area have a majority of low, medium, or high corporate ownership."];
    let box_plot_stats_household_array = [];
    let box_plot_stats_race_array = [];
    let box_plot_stats_elder_array = [];
    let box_plot_stats_corp_array = [];
    let metric_to_graph = "Family Type";
    let checkbox_pushed = false;
    let hoveredIndex = -1;
    $: hoveredEviction = text_description[hoveredIndex]?? {};
    let tooltipPosition = {x:0, y:0};
    let evictionTooltip;
    const format = d3.format(".1~%");
    let selectedYear = 2000;

   //let customColors = ['#ECE8A4', '#744665', '#1F5452', '#46A09E'];

    let rangePercentage;

    $: rangePercentage = ((selectedYear - 2000) / (2023 - 2000)) * 100;

    // defining axes
    let margin = {top: 10, right: 10, bottom: 30, left:40};
    let usableArea = {
        top: margin.top,
        bottom: height - margin.bottom,
        left: margin.left,
        right: width - margin.right
    };
    usableArea.width = usableArea.right - usableArea.left;
    usableArea.height = usableArea.bottom - usableArea.top;

    // creating axes on the page
    $: {
        d3.select(yAxisGridlines).call(d3.axisLeft(yScale).tickSize(-usableArea.width));
    }

    onMount(async() => {
        dataRAW  = await d3.csv("binned_data.csv", row=> ({
            ...row,
            mhi: Number(row.mhi),
            eviction_rate: Number(row.eviction_rate)
        }));

        temp_dataRAW = await d3.csv("updated_monthly.csv", row=> ({
            ...row,
            mhi: Number(row.mhi),
            eviction_rate: Number(row.eviction_rate),
            year: Number(row.year),
            Month: Number(row.Month),
            filings: Number(row.filings)
        }));

        evict_dataRAW = await d3.csv("fake_eviction.csv", row=> ({
            ...row,
            year: Number(row.year),
            eviction_rate: Number(row.eviction_rate),
        }));

        const observerOptions = {
            threshold: 0.02
        };

        const observer = new IntersectionObserver((entries) => {
            entries.forEach(entry => {
                const index = sections.indexOf(entry.target);
                if (index !== -1) {
                    visibleSections[index] = entry.isIntersecting;
                }
            });
        }, observerOptions);

        sections.forEach(section => {
            if (section) {
                observer.observe(section);
            }
        });

        return () => {
            sections.forEach(section => {
                if (section) {
                    observer.unobserve(section);
                }
            });
        };
        
        // temp_data

    });

    // $: data  = radio_button_pushed ? dataRAW.filter((d) => d.month >= 4).filter((d) => d.month < 11).filter((d) => d.family_bins !== '').filter((d) => d.eviction_rate < 1) : dataRAW.filter((d) => d.family_bins !== '').filter((d) => d.eviction_rate < 1).filter((d) => d.mhi > minVal ).filter((d) => d.mhi < maxVal);
    $: evict_data = evict_dataRAW.filter((d) => d.year <= selectedYear);
    $: data = dataRAW.filter((d) => d.family_bins !== '').filter((d) => d.eviction_rate < 1).filter((d) => d.mhi > minVal ).filter((d) => d.mhi < maxVal);
    $: temp_data = isChecked ? temp_dataRAW.filter((d) => d.family_bins !== '').filter((d) => d.eviction_rate < 1).filter((d) => d.majority_race !== '').filter((d) => d.month >= 4).filter((d) => d.month < 11).filter((d) => d.mhi > minVal).filter((d) => d.mhi < maxVal) : temp_dataRAW.filter((d) => d.family_bins !== '').filter((d) => d.eviction_rate < 1).filter((d) => d.majority_race !== '').filter((d) => d.mhi > minVal).filter((d) => d.mhi < maxVal);
    $: selectedEvictions = brushedSelection ? data.filter(isDataSelected) : [];    
    $: hasSelection = brushedSelection && selectedEvictions.length > 0;
    $: selectedLines = (hasSelection ? selectedEviction: data).flatMap(d => d.mhi);
    $: yScale = yScale.domain([0, 0.2]).range([usableArea.bottom, usableArea.top]); // might need to switch values currently domain = [0, height], range = [0, 24] 
    
    // Setting X scale
    $: xScaleHousehold = xScaleHousehold.domain(family_bins).range( [usableArea.left, usableArea.right] ).padding(0.2);    
    $: xScaleRace = xScaleRace.domain(race_bins).range( [usableArea.left, usableArea.right] ).padding(0.2);
    $: xScaleElder = xScaleElder.domain(elder_bins).range( [usableArea.left, usableArea.right] ).padding(0.2);
    $: xScaleCorp = xScaleCorp.domain(corp_bins).range( [usableArea.left, usableArea.right] ).padding(0.2);

    // Computing box plots for family binned data
    $: box_plot_stats_household_array = computeStats(family_bins, "family", data);

    // Computing box plots for race binned data
    $: box_plot_stats_race_array = computeStats(race_bins, "race", data);

    // Computing box plots for elder binned data
    $: box_plot_stats_elder_array = computeStats(elder_bins, "elder", data);

    // Computing box plots for corporate ownership binned data
    $: box_plot_stats_corp_array = computeStats(corp_bins, "corporate", data);

    async function textInteraction (index, evt){
        let hoveredDot = evt.target;
        
        if (evt.type === "mouseenter" || evt.type === "focus")
        {
            hoveredIndex = index;
            tooltipPosition = await computePosition(hoveredDot, evictionTooltip, {
                strategy: "fixed",
                middleware: [
                    offset(10),
                    autoPlacement()
                ],
            });
        }

        else if (evt.type === "mouseleave" || evt.type === "blur")
        {
            hoveredIndex = -1;
        }
    } 

    // compute boxplots for each category for a bin (family, race, elder, corporate ownership)
    function computeStats(bins, metric, data)
    {
        let summary_stats = [];
        let data_filtered;

        for(let bin of bins)
        {
            // checks if metric to graph is associated with household type, family/non family/mixed
            if(metric === "family")
            {
                data_filtered = data.filter((d) => d.family_bins === bin);
            }

            // checks if metric to graph is associated with race, white/black/latino/other
            else if(metric === "race")
            {
                data_filtered = data.filter((d) => d.majority_race === bin);
            }
            
            // checks if metric to graph is associated with elder, some elders/no elders
            else if(metric === "elder")
            {
                data_filtered = data.filter((d) => d.elder_bins === bin);
            }

            // assumes the metric to graph is associated with corporate ownership, low/medium/high
            else
            {
                data_filtered = data.filter((d) => d.corp_bins === bin);
            }
            
            let summary_stat = calculate_box_plot(data_filtered);
            summary_stats.push(summary_stat);
        }
        return summary_stats;
    }

    function calculate_box_plot(binned_data)
    {   
        let eviction_rate_array = [];

        for (let d of binned_data)
        {
            eviction_rate_array.push(d.eviction_rate);
        }

        let quantile1 = d3.quantile(eviction_rate_array, 0.25);
        let quantile2 = d3.quantile(eviction_rate_array, 0.5);
        let quantile3 = d3.quantile(eviction_rate_array, 0.75);
        let innerQuantileRange = quantile3 - quantile1;
        let minimum = quantile1 - 1.5*innerQuantileRange;
        let maximum = quantile3 + 1.5*innerQuantileRange;
        let summary_stats = {q1: quantile1, q2: quantile2, q3:quantile3, 
            innerQuantile:innerQuantileRange, min: d3.max([minimum, 0]), 
            max:d3.min([maximum, height]) }
            // makes sure that the min and max lines do not exceed the bounds of the plot

        return summary_stats
    } 

    let isChecked = false;

    function toggleCheckbox() {
        isChecked = !isChecked;
        console.log(isChecked);
    }

    ////////////////// JS for the income slider //////////////////

    let minVal = 60000;
    let maxVal = 100000;
    const minGap = 5000;
    const sliderMinValue = 60000;
    const sliderMaxValue = 100000;
    const stepValue = 100;

    // Reactive styling for slider track
    $: minPercent = ((minVal - sliderMinValue) / (sliderMaxValue - sliderMinValue)) * 100;
    $: maxPercent = ((maxVal - sliderMinValue) / (sliderMaxValue - sliderMinValue)) * 100;

    // Enforce the minimum gap dynamically, broken
    $: {
        if (maxVal - minVal < minGap) {
            // If decreasing maxVal or increasing minVal, adjust the other to maintain the gap
            if (maxVal - minVal < minGap) {
                maxVal = minVal + minGap;
            }

            if (minVal < sliderMinValue) {
                minVal = sliderMinValue;
                maxVal = minVal + minGap;
            }
            if (maxVal > sliderMaxValue) {
                maxVal = sliderMaxValue;
                minVal = maxVal - minGap;
            }
        }
    }

    // How to handle typed inputs
    function handleMinInput(event) {
        const inputVal = Math.max(Number(parseMoney(event.target.value)), sliderMinValue);
        minVal = Math.min(inputVal, maxVal - minGap);
    }

    // How to handle typed inputs
    function handleMaxInput(event) {
        const inputVal = Math.min(Number(parseMoney(event.target.value)), sliderMaxValue);
        maxVal = Math.max(inputVal, minVal + minGap);
    }

    // Checks with user has hit enter to change slider
    function checkEnterKey(event, handler) {
        if (event.key === 'Enter') {
        handler(event);
        event.target.blur(); // Optionally blur the input after processing to mimic the blur behavior
        }
    }

    // Function to format numbers as money
    function formatIncome(value) {
        return value.toLocaleString('en-US');
    }

    // Function to parse formatted money back to number
    function parseMoney(value) {
        return parseInt(value.replace(/[\$,]/g, ''), 10);
    }

    //////////////////////////////////////////////////////////////

    let sectionsData = [
        {
            title: 'Winter evictions are especially dangerous',
            content: 'What challenges do evictees face? Let’s put ourselves in the shoes of an evictee. You are xx% more likely to be a person, xx% likely to have a family. No matter what time of year, evictions are cruel and traumatizing. Do you have a place to go? Many do not, and shelters fill up, especially during winter. As people seek wealth, shelters fill up.',
            style: 'left'
        },
        {
            title: 'Boston lacks cold-weather renter protections',
            content: 'France ensures its families aren’t put out into the cold, by providing a “winter break” measure banning evictions between November and March. The United States lacks such a blanket protection, leaving the responsibility to a handful of municipalities and states like Chicago and Washington D.C. to provide some, inconsistent level of protection. While statewide or national legislation would be the best approach, Boston can work quickly to protect its population now. Boston’s local Boards of Health can declare states of emergency that prevent winter evictions.',
            style: 'right'
        },

        {
            title: 'Winter evictions are especially dangerous',
            content: 'Subfreezing temperatures can occur more than 90 days a year in Boston. Yet, despite ranking as one of the coldest major cities in America, Boston lacks eviction protection during winter months. Families can be evicted in the dead of winter, with nowhere to go. Evictions are always cruel and traumatizing, but winter evictions are especially dangerous–The National Coalition for Homelessness estimates that 700 people experiencing homelessness die from hypothermia each year.',
            style: 'left'
        },
        {
            title: 'Boston lacks cold-weather renter protections',
            content: 'France ensures its families aren’t put out into the cold, by providing a “winter break” measure banning evictions between November and March. The United States lacks such a blanket protection, leaving the responsibility to a handful of municipalities and states like Chicago and Washington D.C. to provide some, inconsistent level of protection. While statewide or national legislation would be the best approach, Boston can work quickly to protect its population now. Boston’s local Boards of Health can declare states of emergency that prevent winter evictions.',
            style: 'right'
        },

        {
            title: 'Winter evictions are especially dangerous',
            content: 'Subfreezing temperatures can occur more than 90 days a year in Boston. Yet, despite ranking as one of the coldest major cities in America, Boston lacks eviction protection during winter months. Families can be evicted in the dead of winter, with nowhere to go. Evictions are always cruel and traumatizing, but winter evictions are especially dangerous–The National Coalition for Homelessness estimates that 700 people experiencing homelessness die from hypothermia each year.',
            style: 'left'
        },
        {
            title: 'Boston lacks cold-weather renter protections',
            content: 'France ensures its families aren’t put out into the cold, by providing a “winter break” measure banning evictions between November and March. The United States lacks such a blanket protection, leaving the responsibility to a handful of municipalities and states like Chicago and Washington D.C. to provide some, inconsistent level of protection. While statewide or national legislation would be the best approach, Boston can work quickly to protect its population now. Boston’s local Boards of Health can declare states of emergency that prevent winter evictions.',
            style: 'right'
        },
        // Add more sections as needed
    ];

    let sections = []; // Array to hold references to each section
    let visibleSections = new Array(sectionsData.length).fill(false); // Array to track visibility, initially false


</script>

<style>

     /* CSS for Income Selector */

     .double-slider-box {
        margin: 0;
        box-sizing: border-box;
        display: grid;
        max-width: 100%;
        /* background-color: rgb(207, 207, 207); */
        padding: 20px 40px;
        border-radius: 1em;
        font-family: 'Helvetica Neue';
    }

    h3.range-title{

        text-align: center;

    }

    p.range-description {

        text-align: center;
        margin-bottom: 1em;
        margin-top: -0.5em;

    }

    .range-slider {

        position: relative;
        width: 100%;
        height: 5px;
        margin: 30px 0;
        background-color: rgb(114, 114, 114);

    }

    .slider-track {

        height: 100%;
        position: absolute;
        background-color: #46A09E;

    }

    .range-slider input {

        position: absolute;
        width: 100%;
        background: none;
        pointer-events: none;
        top: 50%;
        transform: translateY(-60%);
        appearance: none;

    }

    input[type="range"]::-webkit-slider-thumb {

        height: 20px;
        width: 20px;
        border-radius: 50%;
        border: 3px solid #FFFFFF;
        background: #FFFFFF;
        pointer-events: auto;
        appearance: none;
        cursor: pointer;
        box-shadow: 0 0.125em 0.5625em rgba(0, 0, 0, 0.25)

    }

    /* this is the same thing, but for the Firefox (uncessary?) */
    input[type="range"]::-moz-slider-thumb {

        height: 25px;
        width: 25px;
        border-radius: 50%;
        border: 3px solid #FFFFFF;
        background: #FFFFFF;
        pointer-events: auto;
        cursor: pointer;
        -moz-appearance: none;
        box-shadow: 0 0.125rem 0.5625rem rgba(0, 0, 0, 0.25)

    }

    .input-box {

        display: flex;

    }

    .min-box, 
    .max-box {

        width: 50%;

    }

    .min-box {

        padding-right: 0.5em;
        margin-right: 0.5em;

    }

    .input-wrap {

        position: relative;
        display: flex;
        flex-wrap: wrap;
        align-items: stretch;
        width: 100%;
        
    }

    .input-addon {

        display: flex;
        align-items: center;
        padding: 0.625em 1em;
        font-size: 1em;
        font-weight: 400;
        line-height: 1.5;
        color: rgb(0, 0, 0);
        text-align: center;
        white-space: nowrap;
        background-color: rgb(181, 181, 181);
        border: 1px solid rgb(0, 0, 0);
        border-radius: 0.5em;
        border-top-right-radius: 0;
        border-bottom-right-radius: 0;

    }

    .input_field-min-input, 
    .input_field-max-input{

        margin-left: -1px;
        padding: 0.425em;
        font-size: 1em;
        border-radius: 0.5em;
        position: relative;
        flex: 1 1 auto;
        width: 1%;
        min-width: 0;
        color: rgb(0, 0, 0);
        background-color: white;
        background-clip: padding-box;
        border: 1px solid black;
        border-top-left-radius: 0;
        border-bottom-left-radius: 0;
    }

    /* CSS for Income Selector */

    .meta {
        font-family: 'Helvetica Neue';
        text-align: center;
        margin-top: 150px;
    }

    .subtitle {
        font-family: 'Helvetica Neue';
        text-align: center;
        margin-top: 0px;
        font-size: 20px;
    }

    .narrative1 {
        .section {
            width: 100%;
            padding: 30px;
            box-sizing: border-box;
            background-color: white;
            color: black;
            margin-bottom: 40px;
            display: flex;
            flex-direction: column;
            margin-top: 100px;
            margin-bottom: 100px;
            border-radius: 20px;
            opacity: 0;  /* Start with sections hidden */
            transition: opacity 1s ease-out;
        }
            
        .section-right {
            text-align: right;
            margin-left: auto;
            margin-right: 100px;
            background-color: #f2f2f2;
            width: 70%;

        }

        .section-left {
            text-align: left;
            margin-right: auto;
            margin-left: 100px;
            background-color: #f2f2f2;
            width: 70%;

        }

        h3, p {
            margin: 0;
            font-size: 30px; /* for h3 */
            font-size: 20px; /* for p */
            line-height: 1.5;
        }

        p, a{

            margin-top: 10px;
            margin-bottom: 10px;

        }

        img {
            display: block;
            margin-left: auto;
            margin-right: auto;
            width: 60%;
            margin-top: 30px;
            margin-bottom: 30px;
            box-shadow: 0 4px 8px 0 rgba(0, 0, 0, 0.2), 0 6px 20px 0 rgba(0, 0, 0, 0.19);
            border-radius: 20px;
            }

        .pic-des {

            font-size: 14px;
            text-align: center;

        }

    }

    .bottom-text {

        font-family: 'Helvetica Neue';
        text-align: center;
        margin: 150px;
        font-size: 100;

    }

    .fadeInLeft {
        animation: fadeInLeft 1s ease-out forwards;
    }

    .fadeInRight {
        animation: fadeInRight 1s ease-out forwards;
    }

    @keyframes fadeInLeft {
        from { opacity: 0; transform: translateX(-200px); }
        to { opacity: 1; transform: translateX(0); }
    }

    @keyframes fadeInRight {
        from { opacity: 0; transform: translateX(200px); }
        to { opacity: 1; transform: translateX(0); }
    }
    

    .chart-container {
        width: 100%; /* or specific width as per your design */
        padding: 20px; /* Adjust as needed */
        box-sizing: border-box;
    }

    .year-slider {
        -webkit-appearance: none; /* Removes default styling for sliders in WebKit browsers */
        width: 100%; /* Slider takes full width of its container */
        height: 8px; /* Sets the slider track height */
        background: #ddd; /* Light grey background for the slider track */
        outline: none; /* Removes the outline on focus */
        border-radius: 5px; /* Optional: rounds the corners of the slider track */
        position: relative;
        margin: 10px 0; /* Adds some space around the slider */
    }

    /* Styles for the thumb (the draggable part of the slider) */
    .year-slider::-webkit-slider-thumb {
        -webkit-appearance: none;
        appearance: none;
        width: 20px; /* Width of the thumb */
        height: 20px; /* Height of the thumb */
        background: #8a8a8a; /* Thumb color */
        cursor: pointer; /* Cursor changes to pointer when hovering over the thumb */
        border-radius: 50%; /* Makes the thumb circular */
    }

    .year-slider::-moz-range-thumb {
        width: 20px;
        height: 20px;
        background: #744665;
        cursor: pointer;
        border-radius: 50%;
    }

    /* Styles for the track that fills as the thumb is moved (for WebKit browsers) */
    .year-slider::-webkit-slider-runnable-track {
        width: 100%;
        height: 20px;
        background: linear-gradient(to right, #46A09E 0%, #46A09E var(--range), #ddd var(--range), #ddd 100%);
        border-radius: 5px;
    }

    /* Firefox does not support styling the filled part separately using only CSS as WebKit does,
    so we need to employ a little trick using JavaScript to adjust a pseudo-element based on the slider's value */
    .year-slider::-moz-range-progress {
        background-color: #46A09E; /* Filled track color */
        height: 8px; /* Same as the track height */
        border-radius: 5px 0 0 5px; /* Rounded corners on the left side of the filled track */
    }

    /* Optional: style for the range track that remains unfilled */
    .year-slider::-moz-range-track {
        background: #ddd;
        height: 8px;
        border-radius: 5px;
    }

    /* let customColors = ['#ECE8A4', '#744665', '#1F5452', '#46A09E']; */
</style>

<h3 class="meta">
    <strong style="color: #46A09E;">Interactive Voices:</strong>
    Mapping the Movement for Eviction Reform and Advocacy
</h3>
<h4 class="subtitle">
    An informative exposé by <font color="#46A09E">Olivia Dias</font>, <font color="#46A09E">Sarah Gurev</font>, <font color="#46A09E">Ted Letsou</font>, and <font color="#46A09E">Protyasha Nishat</font>, MIT 6.C35
</h4>


<div class="full_page">

    <div class="narrative1">
        <!-- <div bind:this={leftSection} class="fade section section-left" class:fadeInLeft={leftVisible}>
                <h3 style="color: #744665">Winter evictions are especially dangerous</h3>
                <p>
                    Subfreezing temperatures can occur more than 90 days a year in Boston. 
                    Yet, despite ranking as one of the coldest major cities in America, Boston lacks eviction protection during winter months. 
                    Families can be evicted in the dead of winter, with nowhere to go. 
                    Evictions are always cruel and traumatizing, but winter evictions are especially dangerous–The National Coalition for Homelessness estimates that 700 people experiencing homelessness die from hypothermia each year. 
                </p>
            </div>

            <div bind:this={rightSection} class="fade section section-right" class:fadeInRight={rightVisible}>
                <h3 style="color: #744665">Boston lacks cold-weather renter protections </h3>
                <p>
                    France ensures its families aren’t put out into the cold, by providing a “winter break” measure banning evictions between November and March. 
                    The United States lacks such a blanket protection, leaving the responsibility to a handful of municipalities and states like Chicago and Washington D.C. to provide some, inconsistent level of protection. 
                    While statewide or national legislation would be the best approach, Boston can work quickly to protect its population now. 
                    Boston’s local Boards of Health can declare states of emergency that prevent winter evictions. 

                </p>
            </div>
            
            <div bind:this={leftSection} class="fade section section-left" class:fadeInLeft={leftVisible}>
                <h3 style="color: #744665">Winter evictions are especially dangerous</h3>
                <p>
                    Subfreezing temperatures can occur more than 90 days a year in Boston. 
                    Yet, despite ranking as one of the coldest major cities in America, Boston lacks eviction protection during winter months. 
                    Families can be evicted in the dead of winter, with nowhere to go. 
                    Evictions are always cruel and traumatizing, but winter evictions are especially dangerous–The National Coalition for Homelessness estimates that 700 people experiencing homelessness die from hypothermia each year. 
                </p>
            </div>

            <div bind:this={rightSection} class="fade section section-right" class:fadeInRight={rightVisible}>
                <h3 style="color: #744665">Boston lacks cold-weather renter protections </h3>
                <p>
                    France ensures its families aren’t put out into the cold, by providing a “winter break” measure banning evictions between November and March. 
                    The United States lacks such a blanket protection, leaving the responsibility to a handful of municipalities and states like Chicago and Washington D.C. to provide some, inconsistent level of protection. 
                    While statewide or national legislation would be the best approach, Boston can work quickly to protect its population now. 
                    Boston’s local Boards of Health can declare states of emergency that prevent winter evictions. 

                </p>
            </div> -->




            <!-- {#each sectionsData as section, index}
                <div bind:this={sections[index]}
                    class="section {section.style === 'left' ? 'section-left' : 'section-right'}"
                    class:fadeInLeft={section.style === 'left' && visibleSections[index]}
                    class:fadeInRight={section.style === 'right' && visibleSections[index]}>
                    <h3 style="color: #744665">{section.title}</h3>
                    <p>{section.content}</p>
                    <div class="chart-container">
                        <EvictionTime data={evict_data} />
                        <input type="range" bind:value={selectedYear} min="2000" max="2023" class="year-slider" style="--range: {rangePercentage}%;">
                    </div>
                </div>
            {/each} -->
            <div bind:this={sections[0]}
                class="section section-left"
                class:fadeInLeft={visibleSections[0]}>
                <h3 style="color: #744665">Millions of families in the US are impacted by the eviction crisis in the US.</h3>
                <p>According to the Eviction Lab, landlords filed more than 1.1 million eviction cases. 
                    In fact, eviction has been increasing over the past years throughout the whole US and Massachusetts is no exception to that. 
                    Move the slider below to check how eviction has changed in MA from 2020 to 2023:</p>
                <div class="chart-container">
                    <EvictionTime data={evict_data} />
                    <input type="range" bind:value={selectedYear} min="2000" max="2023" class="year-slider" style="--range: {rangePercentage}%;">
                </div>
            </div>

            <div bind:this={sections[1]}
                class="section section-right"
                class:fadeInRight={visibleSections[1]}>
                <h3 style="color: #744665">What challenges do evictees face?</h3>
                <p>Let’s put ourselves in the shoes of an evictee. You are xx% more likely to be a person, xx% likely to have a family.</p>
                <p>No matter what time of year, evictions are cruel and traumatizing. </p>
                <p>Do you have a place to go?</p>
                <p>Many do not, and shelters fill up, especially during winter. As people seek wealth, shelters fill up. </p>
                <img src="images/image1.png" alt="">
                <p class="pic-des"><i style="color: #8a8a8a">A man experiencing homeless stands beside a pile of his belongings, at a homeless encampment near the Charles River, January 24, 2024. The weather dropped below freezing. 
                </i></p>
                <a class="pic-des" style="color: #8a8a8a" href="https://www.wgbh.org/news/local/2024-03-06/after-mass-and-cass-crackdown-homeless-community-cast-out-into-the-shadows-of-boston">Tori Bedford/GBH news 
                </a>

                
            </div>


             <div bind:this={sections[2]}
                class="section section-left"
                class:fadeInLeft={visibleSections[2]}>
                <h3 style="color: #744665">Who does this affect?</h3>
                <p><i style="color: #46A09E">“I meditate–that’s how I deal with this weather. I’ve sat in the Polar Vortex, every Polar Vortex we’ve had...if you meditate, you become warmer.” - Davie</i></p>
                <p><i style="color: #8a8a8a">Davie is a European immigrant living on the streets of Cambridge for two years. He prefers to sleep at the Harvard Square Homeless shelter, but beds are lotteried, and on cold nights when more people are vying for shelter, Davie is left to camp outdoors. </i></p>
                <p>Davie is not alone. While most homeless shelters in Cambridge typically at full capacity, there is no more room for the greater influx of people during cold weather. </p>
                <img src="images/image2.png" alt="" >
                <p class="pic-des"><i style="color: #8a8a8a">A man experiencing homeless rests in a bus stop, seeking shelter from a winter storm. Dec. 17, 2020 in Massachusetts.
                </i></p>
                <p class="pic-des" style="color: #8a8a8a">Elise Amendola/AP</p>
            </div>

            <div bind:this={sections[3]}
                class="section section-right"
                class:fadeInRight={visibleSections[3]}>
                <h3 style="color: #744665">What challenges do evictees face?</h3>
                <p><i style="color: #46A09E">“Particularly in the winter, there’s always more need than there are beds.” - Shannon</i></p>
                <p><i style="color: #8a8a8a">Shannon, Mary Shannon Thomas, a Cambridge social worker who specializes in homelessness.</i> </p>
                <p>Even still, shelters aren’t the first choice for everyone. Many shelters are gender-specific, so families can’t stay together. And communal settings are difficult - with concerns over safety, noise, and spread of illnesses, particularly with the rise of Covid. </p>
                <p><i style="color: #46A09E">“One of the guys I worked with has PTSD, a shelter for him is one of the most terrifying places you could put him. There’s too much noise, there’s too many people, there’s too much stuff going on in the environment, and for him, being outside where it’s 15 degrees feels safer than being in a crowded shelter.” - Shannon.</i></p>
                <img src="images/image3.png" alt="">
                <p class="pic-des" style="color: #8a8a8a">The Boston Globe</p>
            </div>

            <div bind:this={sections[4]}
                class="section section-left"
                class:fadeInLeft={visibleSections[4]}>
                <h3 style="color: #744665">Who does this affect?</h3>
                <p>And so many people become homeless. <b>Several studies showed that 14% to 47% of homeless families have been evicted.</b> The number of people experiencing homelessness in Boston is increasing dramatically, rising 17.2% - from 4,439 people in 2022 to 5,202 people in 2023. </p>
                <p><i style="color: #46A09E">“I had a spot where I could cut into a little cubby corner and stay inconspicuous. I didn't really have any problems about anybody bothering me there at night.” - Ishmael Rodriguez </i></p>
                <p><i style="color: #8a8a8a">Ishmael is a 57-year old man who slept in a tent for almost a year near the intersection of Massachusetts Ave and Melnea Cass Boulevard, Boston’s longtime epicenter of homelessness. He’s been homeless since losing his Dorchester apartment in 2021. </i></p>   
            </div>

            <div bind:this={sections[5]}
                class="section section-right"
                class:fadeInRight={visibleSections[5]}>
                <h3 style="color: #744665">What challenges do evictees face?</h3>
                <p>People facing unsheltered homelessness are at heightened risk during winter due to exposure to the elements outdoors. Unhoused Bostonians who sleep in the street are 10 times more likely to die than those with shelter. 
                </p>
                <p>Cold temperatures can lead to hypothermia, a condition where the body loses heat faster than it can generate, and frostbite. Those experiencing homelessness often lack essential resources such as insulated winter clothing and access to warm shelter, which is particularly concerning during nighttime when temperatures plummet below freezing.
                </p>
             </div>

             <div bind:this={sections[6]}
                class="section section-left"
                class:fadeInLeft={visibleSections[6]}>
                <h3 style="color: #744665">But things can change. A winter moratorium on eviction can protect people. 
                </h3>
                <p><i style="color: #46A09E"> “I was a landlord for many years. Even though I followed state laws regarding this eviction, I came to realize these laws are unjust and need to be changed.” -Gerry Willis </i></p>
                <p><i style="color: #8a8a8a"> Garry is a former landlord who owned multiple properties. He evicted a male tenant who had received multiple noise complaints. The evicted tenant died in a heatless garage.     </i></p>   
            </div>
            
            <div bind:this={sections[7]}
                class="section section-right"
                class:fadeInRight={visibleSections[7]}>
                <h3 style="color: #744665">Winter evictions are especially dangerous
                </h3>
                <p>Subfreezing temperatures can occur more than 90 days a year in Boston. Yet, despite ranking as one of the coldest major cities in America, Boston lacks eviction protection during winter months. Families can be evicted in the dead of winter, with nowhere to go. Evictions are always cruel and traumatizing, but winter evictions are especially dangerous–The National Coalition for Homelessness estimates that 700 people experiencing homelessness die from hypothermia each year. 
                </p>
            </div>

            <div bind:this={sections[8]}
                class="section section-left"
                class:fadeInLeft={visibleSections[8]}>
                <h3 style="color: #744665">Boston lacks cold-weather renter protections
                </h3>
                <p>France ensures its families aren’t put out into the cold, by providing a “winter break” measure banning evictions between November and March. The United States lacks such a blanket protection, leaving the responsibility to a handful of municipalities and states like Chicago and Washington D.C. to provide some, inconsistent level of protection. While statewide or national legislation would be the best approach, Boston can <b>work quickly to protect its population now.</b>  Boston’s local Boards of Health can declare states of emergency that prevent winter evictions. 
                </p>
            </div>
            
            <div class="bottom-text">
                <h3 style = "color: #744665" > <b>Boston’s lack of winter protection greatly affects historically discriminated groups: families, the elderly, and people of color. 
                    A moratorium on winter evictions would benefit these groups the most. 
                </b></h3> 
            </div>
    </div>

    


    <div class="selection_column section-left" bind:this={sections[9]} 
        class:fadeInLeft={visibleSections[9]}>
        <div class="double-slider-box">
            <h3 class="range-title">Looking at evictions from groups with matched incomes</h3>
            <p class="range-description">Adjust slider to subset regions by average income</p>
            <div class="range-slider">
                <span class="slider-track" style="left: {minPercent + 1}%; right: {100 - maxPercent}%"></span>
                <input type="range" bind:value={minVal} class="min-val" min={sliderMinValue} max={sliderMaxValue} step={stepValue}>
                <input type="range" bind:value={maxVal} class="max-val" min={sliderMinValue} max={sliderMaxValue} step={stepValue}>
            </div>
            <div class="input-box">
                <div class="min-box">
                    <div class="input-wrap">
                        <span class="input-addon">$</span>
                        <input type="text" class="input_field-min-input"
                               value={formatIncome(minVal)} 
                               on:blur={handleMinInput}
                               on:keydown={(event) => checkEnterKey(event, handleMinInput)}>
                    </div>
                </div>
                <div class="max-box">
                    <div class="input-wrap">
                        <span class="input-addon">$</span>
                        <input type="text" class="input_field-max-input"
                               value={formatIncome(maxVal)} 
                               on:blur={handleMaxInput}
                               on:keydown={(event) => checkEnterKey(event, handleMaxInput)}>
                    </div>
                </div>
            </div>
        </div>

        <dl id="eviction-tooltip" class="info tooltip" 
            hidden={hoveredIndex === -1}
            bind:this={evictionTooltip}
            style="top:{tooltipPosition.y}px; left:{tooltipPosition.x}px">
            <dt>Metric Description</dt>
            <dd> { hoveredEviction } </dd>
        </dl>
        
        <div class="metric_selection">
            <dl class="family_selection">
                <input type="button" value="Family" 
                on:click={ function() {metric_to_graph= "Family"} }
                on:mouseenter= {evt=> textInteraction(1, evt)}
                on:mouseleave={evt => textInteraction(1, evt)}>
            </dl>
            <dl class="race_selection">
                <input type="button" value="Race" 
                on:click={ function() {metric_to_graph= "Race"} }
                on:mouseenter= {evt=> textInteraction(0, evt)}
                on:mouseleave={evt => textInteraction(0, evt)}>

            </dl>
            <dl class="elderly_selection">
                <input type="button" value="Elder" 
                on:click={ function() {metric_to_graph= "Elderly"} }
                on:mouseenter= {evt=> textInteraction(2, evt)}
                on:mouseleave={evt => textInteraction(2, evt)}>
            </dl>
        
            <dl class="corporate_selection">
                <input type="button" value="Corporate" 
                on:click={ function() {metric_to_graph= "Corporate"} }
                on:mouseenter= {evt=> textInteraction(3, evt)}
                on:mouseleave={evt => textInteraction(3, evt)}>
            </dl>
        </div>

        <div class="simulate_winter_ban">
            <input type="checkbox" bind:checked={isChecked} on:click={toggleCheckbox}>
            <label><h3>Simulate Winter Ban</h3></label>
            <section>
                <p>
                    Click on the <em>"Simulate Winter Ban"</em> button to see how the eviction rate changes 
                    with the removal of evictions during winter months. Additionally, you will see
                    which demographics are most impacted by winter bans.
                </p>
            </section>
        </div>
    </div>
    
    <div class="full_graph">
        {#if metric_to_graph.includes("Family")}
            <BinGraph binned_data={box_plot_stats_household_array} yScale={yScale}
                      xScale={xScaleHousehold} metric={metric_to_graph} 
                      bin_type={family_bins} data={data} text={["Families (especially with children) increase the risk of eviction. "]}/>
        {/if}   
        
        {#if metric_to_graph.includes("Race")}
            <BinGraph binned_data={box_plot_stats_race_array} yScale={yScale}
                      xScale={xScaleRace} metric={metric_to_graph}
                      bin_type={race_bins} data={data} text={["Eviction rates are lowest in regions with a majority white population.  "]}/>
        {/if}

        {#if metric_to_graph.includes("Elder")}
            <BinGraph binned_data={box_plot_stats_elder_array} yScale={yScale}
                      xScale={xScaleElder} metric={metric_to_graph}
                      bin_type={elder_bins} data={data} text={["Homogenous regions without any elderly population experience far lower eviction "]}/>
        {/if}
        {#if metric_to_graph.includes("Corporate")}
            <BinGraph binned_data={box_plot_stats_corp_array} yScale={yScale}
                      xScale={xScaleCorp} metric={metric_to_graph}
                      bin_type={corp_bins} data={data} text={["Low Corporate Ownership"]}/>
        {/if}

        
        
    </div>

    <!-- <div class="metric_selection2">
        <dl class="fam_select2">
            <input type="button" value="Family" 
            on:click={ function() {metric_to_graph= "Family"} }
            on:mouseenter= {evt=> textInteraction(1, evt)}
            on:mouseleave={evt => textInteraction(1, evt)}>
        </dl>
        <dl class="race_selection2">
            <input type="button" value="Race" 
            on:click={ function() {metric_to_graph= "Race"} }
            on:mouseenter= {evt=> textInteraction(0, evt)}
            on:mouseleave={evt => textInteraction(0, evt)}>

        </dl>
        <dl class="elderly_selection2">
            <input type="button" value="Elder" 
            on:click={ function() {metric_to_graph= "Elderly"} }
            on:mouseenter= {evt=> textInteraction(2, evt)}
            on:mouseleave={evt => textInteraction(2, evt)}>
        </dl>
    
        <dl class="corporate_selection2">
            <input type="button" value="Corporate" 
            on:click={ function() {metric_to_graph= "Corporate"} }
            on:mouseenter= {evt=> textInteraction(3, evt)}
            on:mouseleave={evt => textInteraction(3, evt)}>
        </dl>
    </div> -->
<!-- </div> -->

    <div class="eviction_animation">
        {#if metric_to_graph.includes("Family")}
            <DotAnimation data={temp_data} text={"Thousands of people are evicted from their homes each year in Boston in subfreezing weather, disproportionately affecting already disadvantaged groups.  A winter eviction moratorium stops people from being evicted into freezing temperatures outside, especially benefitting already disadvantaged groups. "}
            bins={family_bins} metric={metric_to_graph} />
        {/if}

        {#if metric_to_graph.includes("Race")}
            <DotAnimation data={temp_data} text={"Thousands of people are evicted from their homes each year in Boston in subfreezing weather, disproportionately affecting already disadvantaged groups.  A winter eviction moratorium stops people from being evicted into freezing temperatures outside, especially benefitting already disadvantaged groups. "}
            bins={race_bins} metric={metric_to_graph} />
        {/if}

        {#if metric_to_graph.includes("Elder")}
            <DotAnimation data={temp_data} text={"Thousands of people are evicted from their homes each year in Boston in subfreezing weather, disproportionately affecting already disadvantaged groups.  A winter eviction moratorium stops people from being evicted into freezing temperatures outside, especially benefitting already disadvantaged groups. "}
            bins={elder_bins} metric={metric_to_graph} />
        {/if}

        {#if metric_to_graph.includes("Corporate")}
            <DotAnimation data={temp_data} text={"Thousands of people are evicted from their homes each year in Boston in subfreezing weather, disproportionately affecting already disadvantaged groups.  A winter eviction moratorium stops people from being evicted into freezing temperatures outside, especially benefitting already disadvantaged groups. "}
            bins={corp_bins} metric={metric_to_graph} />
        {/if}
        
    </div>

</div>


<!-- TEXT for each Bin and Category: -->
    <!-- Mixed: ["Families (especially with children) increase the risk of eviction. "]
    Non-Fam: ["Eviction rates are lower in non-family households. "]
    FamilyType: ["Families (especially with children) increase the risk of eviction. "]
    White: ["Eviction rates are lowest in regions with a majority white population.  "]
    Black: ["Eviction rates are highest in regions with a majority black population, more than twice as high as majority white regions. "]
    Latino: ["Eviction rates are higher in regions with a majority latino population."]
    Other: ["Eviction rates are higher in neighborhoods that are not majority white."]
    No Elderly: ["Homogenous regions without any elderly population experience far lower eviction "]
    Some Elderly: ["Regions with some elderly population are much more at risk of evictions."]
    Corporate Ownership Low: ["Low Corporate Ownership"]
    Corporate Ownership Medium: ["Medium Corporate Ownership"]
    Corporate Ownership High: ["High Corporate Ownership"] -->