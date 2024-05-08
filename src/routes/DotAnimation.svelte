<script>
    import * as d3 from "d3";
    import {
        computePosition,
        autoPlacement,
        offset
    } from '@floating-ui/dom';
    export let text = "";
    export let data = [];
    export let bins = [];
    export let metric = "";

    let temp_bins = ["Summer", "Spring", "Fall", "Winter"];
    let margin = {top: 10, right: 10, bottom: 30, left:40};
    let height= 400;
    let width = 600;
    let startX = 5;
    let startY = height/4 ;
    let hoveredIndex = -1;
    let boxWidth = 20; 
    let boxHeight = 50;
    $: hoveredEviction = data[hoveredIndex]?? {};
    let tooltipPosition = {x:0, y:0};
    let evictionTooltip;
    let yAxis;
    let yAxisGridlines;
    let usableArea = {
        top: margin.top,
        bottom: height - margin.bottom,
        left: margin.left,
        right: width - margin.right
    };
    let yScale = d3.scaleBand();
    let summer_dots = 0;
    let spring_dots = 0;
    let fall_dots = 0;
    let winter_dots = 0;
    let season_dot;

    $: season_dot = summer_dots;

    // Define your custom colors
    let customColors = ['#ECE8A4', '#744665', '#1F5452', '#46A09E']; // Example colors: red, green, blue, yellow

    // Create the ordinal scale with your custom colors
    let fillColor = d3.scaleOrdinal(customColors);
    let monthColor = d3.scaleOrdinal(["red", "pink", "orange", "blue"]);
    const format = d3.format(".1~%");

    $: yScale = yScale.domain(temp_bins).range([usableArea.bottom, usableArea.top]);
    $: {
        d3.select(yAxisGridlines).call(d3.axisRight(yScale).tickSize(width-100));
    }

    async function dotInteraction (data, index, evt){
        let hoveredDot = evt.target;
        
        if (evt.type === "animationend")
        {
            if ( convertMonthToTemp(data.month) === "Summer")
            {
                console.log("I am a summer eviction!!!");
                summer_dots +=1;
                // console.log(summer_dots);
                // console.log();
            }

            else if (convertMonthToTemp(data.month) === "Spring")
            {
                console.log("I am a spring eviction!!!");
                spring_dots +=1;
                // console.log(spring_dots);
                // console.log();
            }

            else if (convertMonthToTemp(data.month) === "Fall")
            {
                console.log("I am a fall eviction!!!");
                fall_dots +=1;
                // console.log(fall_dots);
                // console.log();
            }

            if (convertMonthToTemp(data.month) === "Winter")
            {
                console.log("I am a winter eviction!!!");
                winter_dots +=1;
                // console.log(winter_dots);
                // console.log();
            }
            
        }

        if (evt.type === "mouseenter" || evt.type === "focus")
        {
            hoveredIndex = index;
            tooltipPosition = await computePosition(hoveredDot, evictionTooltip, {
                strategy: "fixed",
                middleware: [
                    offset(20),
                    autoPlacement()
                ],
            });
        }

        else if (evt.type === "mouseleave" || evt.type === "blur")
        {
            hoveredIndex = -1;
        }
    }

    function dataColoring(d, bin_type, i)
    {
        if (bin_type.includes("Family"))
        {
            return fillColor(d.family_bins);
        }

        else if (bin_type.includes("Race"))
        {
            return fillColor(d.majority_race);
        }

        else if (bin_type.includes("Elder"))
        {
            return fillColor(d.elder_bins);
        }

        else if (bin_type.includes("Corporate"))
        {
            return fillColor(d.corp_bins)
        }
    }

    // classifies the month attribute from the dataset to a temperature
    // which will be used to graph the y position of the data point
    function convertMonthToTemp(month)
    {
        // temperature bins: ["Summer", "Spring", "Fall", "Winter"]
        // Fall Months
        if((9 <= month) && (month < 11))
        {
            return "Fall";
        }
        // Winter Months
        if (((1 <= month ) && (month < 4)) || (11 <= month))
        {
            return "Winter";
        }

        // Spring Months
        if ((4 <= month) && (month < 6 ))
        {
            return "Spring";
        }

        // Summer Months
        if(6 <= month < 9)
        {
            return "Summer";
        }
    }
    function updateDotCount(dot, index)
    {
        
        num_dots +=1;
    }

</script>
<style>

@keyframes moveCircles{
    
    100%{
        /* fill: var(--seasonColor); */
        opacity: 100%;
        /* transform: scale(1.5); */
        /* opacity: 0%; */
    }
    to{
        /* final x position of cirlce */
        cx: var(--xposition);
        /* final x position of cirlce */
        cy: var(--yposition);
        
            /* changes the color of the circle to match the season */
        /* fill: var(--seasonColor); */
    }
}
.moving_dots{
    opacity: 75%;
    /* animation: moveCircles 20s  ease-in-out; */
    animation: moveCircles 5s ease-in-out; 
    animation-delay: calc(var(--index) * 200ms);
    animation-iteration-count: 1;
    animation-fill-mode: forwards;
}
.description{
    /* border: 2px solid black; */
    border-radius:10px;
    padding:10px;
    /* box-shadow:5px 5px 5px grey; */
    margin-right:10px;
    row-gap:1fr;
    margin-top:15px;
}

.bin_legend{
    display: grid;
    grid-template-rows: 10fr 10fr;
    padding:50px;
    margin-bottom:20px;
    border-bottom:2px solid black;
    /* box-shadow: 5px 5px 5px grey; */
    opacity:75%;
    border-radius:1px;
    margin-right:20px;
    margin-left:30px;
    
    label{
        grid-row:1;
        padding:10px;
    }
    div{
        grid-row:2;
        padding:5px;
        margin-left:15px;
        border-radius:40px;
        /* background-color:red; */
        border:2px solid black;
        width:10px;
        height:10px;
    }
}

.entire_visualization{
    display:grid;
    grid-template-columns: 10fr 10fr;
    margin-bottom:20px;
    
    .visual{
        width:700px;
        height:300px;
        /* background-color:blue; */
        opacity:80%;
        grid-column:1;
        margin-bottom:80px;
        padding-bottom: 30px;
        /* margin-right: 20px; */
        /* padding:5px; */
    }

    .temp_legend{
        font-size: 20px;
        font-family: 'Helvetica Neue';
        display:grid;
        grid-template-rows: auto auto auto auto;
        margin-right: 20px;

            div{
                grid-row:1;
                border-radius:50px;
                background-color:red;
                border:2px solid black;
                width:10px;
                height:10px;
                subgrid-row:1;
            }  
        .percent_labels{
            grid-row: span 3;
        }
    }
}

svg{
    margin-bottom: 20px;
    
}

</style>
<dl class="animation_container">
    <section class="bin_legend">
        {#each bins as bin}
            <label>{bin}</label>
            <div style="background-color:{fillColor(bin)}"></div>
        {/each}
    </section>

    <dl id="eviction-tooltip" class="info tooltip" 
        hidden={hoveredIndex === -1}
        bind:this={evictionTooltip}
        style="top:{tooltipPosition.y}px; left:{tooltipPosition.x}px">
        <dt>Eviction Rate</dt>
        <dd> { format(hoveredEviction.eviction_rate, ".3f") } </dd>
    
        <dt>Eviction Month</dt>
        <dd>{ hoveredEviction.month }</dd>
    
        <dt>Eviction Year</dt>
        <dd> { hoveredEviction.year } </dd>
       
    </dl>

    <div class="entire_visualization">
        <section class="visual">
            <!-- x, y, width, height -->
            <!-- viewBox="0 0 180 140" -->
            <svg  
                width={width+50} height={height}> 
                <g class="gridlines" transform="translate({usableArea.left}, 0)" bind:this={yAxisGridlines} />
                <g transform="translate({usableArea.top})" bind:this={yAxis}/>
                <!-- INSERT AN IMAGE ELEMENT TO  -->
                {#each data as d, index}
                    <circle
                        class="moving_dots"
                        cx={ startX + Math.random()*60}
                        cy={ startY + Math.random()*200}
                        r="5"
                        fill={dataColoring(d, metric, index)}
                        stroke="white"
                        style="
                        --xposition:{500-Math.random() * 50};
                        --index:{index};
                        --yposition:{ yScale( convertMonthToTemp(d.month) ) + yScale.bandwidth() / 2 + Math.random()*40};
                        --seasonColor: { monthColor(convertMonthToTemp(d.month)) };"
                        on:mouseenter= {evt=> dotInteraction(d, index, evt)}
                        on:mouseleave={evt => dotInteraction(d, index, evt)} 
                        on:animationend={evt => dotInteraction(d, index, evt)}
                    />
                    <!-- <polygon
                        class="moving_dots"
                        points="1, -1 2, -2 2, -5 1, -6 -2, -6 -3,
                        -5 -3, -2 -2, -1 -3, 7 -1, 7 -1, 4 0, 4 0, 7 2, 7"
                        transform="translate(-7, 0"
                        fill={dataColoring(d, metric, index)}
                        stroke="white"
                        on:mouseenter= {evt=> dotInteraction(index, evt)}
                        on:mouseleave={evt => dotInteraction(index, evt)} 
                    /> -->
                {/each}
                
                {#each temp_bins as bin}
                    <rect
                    x={ 600 }
                    y={ yScale(bin) } 
                    width={boxWidth}
                    height={boxHeight}
                    stroke="black"
                    fill={monthColor(bin)}
                    />
                {/each}
            </svg>
        </section>
        <section class="temp_legend">
            {#each bins as bin, i}
                <!-- <dt>{bin}</dt> -->
                <div 
                    class="bin_circles"
                    style="background-color:{fillColor(bin)}"
                />
    
                {#each temp_bins as b, index}
                    <!-- <dd class="percent_labels">(bin, temp): ({index}, {i})%</dd> -->
                    {console.log(bin)}
                    {#if b === "Summer"}
                        <dd class="percent_labels">{summer_dots} Summer</dd>
                    {/if}

                    {#if b === "Spring"}
                        <dd class="percent_labels">{spring_dots}Spring</dd>
                    {/if}

                    {#if b === "Fall"}
                        <dd class="percent_labels">{fall_dots}Fall</dd>
                    {/if}

                    {#if b === "Winter"}
                        <dd class="percent_labels">{winter_dots}Winter</dd>
                    {/if}
                    {:else}
                    <!-- <dd class="percent_labels">%</dd> -->

                    {console.log(bin)}
                    
                {/each}
            {/each}
        </section>
    </div>
    
    <section class="description">
        <p>
            {text}
        </p>
    </section>
</dl>