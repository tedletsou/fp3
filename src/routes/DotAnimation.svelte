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
    let width = 900;
    let progress_bar_width = 900;
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
    // let summer_dots_count = [];
    // let spring_dots_count = [];
    // let fall_dots_count = [];
    // let winter_dots_count = [];
    let categoryScale = d3.scaleBand();
    // let sum = 0;


    // calculate the total number of winter, fall, spring, and summer evictions
    // and then given the index

    // Define your custom colors
    // '#46A09E', yellow 
    let customColors = ['#46A09E', '#744665', '#d43264', '#DECE00']; // Example colors: red, green, blue, yellow
    //let customColors = ['#ECE8A4', '#744665', '#1F5452', '#DECE00']; // Example colors: red, green, blue, yellow

    
    // Create the ordinal scale with your custom colors
    let fillColor = d3.scaleOrdinal(customColors);
    let monthColor = d3.scaleOrdinal(["blue",  "pink", "red", "orange" ]);
    const format = d3.format(".1~%");

    $: yScale = yScale.domain(temp_bins).range([usableArea.bottom, usableArea.top]);
    $: {
        d3.select(yAxisGridlines).call(d3.axisRight(yScale).tickSize(width-100));
    }
    $: summer_dots_count = new Array(bins.length).fill(0);
    $: spring_dots_count = new Array(bins.length).fill(0);
    $: fall_dots_count = new Array(bins.length).fill(0);
    $: winter_dots_count = new Array(bins.length).fill(0);
    $: categoryScale = categoryScale.domain(bins).range([0, bins.length]);


    // determines the category type for each bin
    function categoryType(data, metric)
    {
        // checks if graphing family bin
        if(metric.includes("Family"))
        {
            return data.family_bins;
        }

        // checks if graphing race bin
        if(metric.includes("Race"))
        {
            return data.majority_race;
        }

        // checks if graphing elder bin
        if(metric.includes("Elder"))
        {
            return data.elder_bins;
        }

        // checks if graphing corporate bin
        if(metric.includes("Corporate"))
        {
            return data.corp_bins;
        }
    }

    async function dotInteraction (metric, data, index, evt){
        let hoveredDot = evt.target;
        
        // checks if the animation has ended for the dot
        if (evt.type === "animationend")
        {
            // gets the corresponding index for the category
            let category_type = categoryType(data, metric);           
            let category_index = categoryScale( category_type);

            // checks if eviction was during summer
            if ( convertMonthToTemp(data.month) === "Summer")
            {
                summer_dots_count[category_index] +=1;
            }

            // checks if eviction was during spring
            else if (convertMonthToTemp(data.month) === "Spring")
            {
                spring_dots_count[category_index] +=1;
            }

            // checks if eviction was during fall
            else if (convertMonthToTemp(data.month) === "Fall")
            {
                fall_dots_count[category_index] +=1;
            }

            // checks if eviction was during winter
            if (convertMonthToTemp(data.month) === "Winter")
            {
                winter_dots_count[category_index] +=1;
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

    function computeSum(season_array)
    {
        let sum = 0;

        for (let element of season_array)
        {
            sum+= element;
        }

        return sum;
    }

    // calculates the proportions from the previous categories for the dot animation
    function previousHeight(index, season_array)
    {
        let total_proportions = 0;
        
        for (let i = index; i > 0; i--)
        {
            total_proportions += (season_array[i-1] / computeSum(season_array));
        }

        return total_proportions;
    }

</script>
<style>

@keyframes moveCircles{
    
    100%{
        opacity: 100%;
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
    animation-delay: calc(var(--index) * 100ms);
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
        margin-left: 300px;

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

rect.Winter
{
    height: calc(var(--winter_height) * 50px);
    /* transition: opacity 5s ease-in-out 1s; */

    animation: rectangleTransitions 6s ease-in;
    /* animation-delay: 1s; */
}   

rect.Summer
{
    height: calc(var(--summer_height) *50px);
    /* transition: opacity 5s ease-in-out 1s; */
    animation: rectangleTransitions 6s ease-in;
    /* animation-delay: 1s; */
}

rect.Spring
{
    height: calc(var(--spring_height) * 50px);
    /* transition: opacity 5s ease-in-out 1s; */
    animation: rectangleTransitions 6s ease-in;
    /* animation-delay: 1s; */
}

rect.Fall
{
    height: calc(var(--fall_height) * 50px);
    /* transition: opacity 5s ease-in-out 1s; */

    animation: rectangleTransitions 6s ease-in;
    /* animation-delay: 1s; */
}

svg{
    margin-bottom: 20px;
}

.percent_labels{
    color: var(--season_color);
}

</style>
<dl class="animation_container">
    <section class="bin_legend">
        {#each bins as bin}
            <label>{bin}</label>
            <div style="background-color:{fillColor(bin)}"></div>
        {/each}
    </section>

    <!-- <dl id="eviction-tooltip" class="info tooltip" 
        hidden={hoveredIndex === -1}
        bind:this={evictionTooltip}
        style="top:{tooltipPosition.y}px; left:{tooltipPosition.x}px">
        <dt>Eviction Rate</dt>
        <dd> { format(hoveredEviction.eviction_rate, ".3f") } </dd>
    
        <dt>Eviction Month</dt>
        <dd>{ hoveredEviction.month }</dd>
    
        <dt>Eviction Year</dt>
        <dd> { hoveredEviction.year } </dd>
       
    </dl> -->

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
                        --xposition:{width - 100 -Math.random() * 50};
                        --index:{index};
                        --yposition:{ yScale( convertMonthToTemp(d.month) ) + yScale.bandwidth() / 2 + Math.random()*40};
                        --seasonColor: { monthColor(convertMonthToTemp(d.month)) };"
                        on:mouseenter= {evt=> dotInteraction(metric, d, index, evt)}
                        on:mouseleave={evt => dotInteraction(metric, d, index, evt)} 
                        on:animationend={evt => dotInteraction(metric, d, index, evt)}
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
                
                {#each temp_bins as season}
                    {#each bins as category, i}
                        {#if season === "Winter"}
                            <rect
                                class="{season}"
                                id="{category}"
                                x= { progress_bar_width }
                                y={yScale(season) + previousHeight(i, winter_dots_count)*50}
                                width={boxWidth}
                                style="
                                --winter_height:{ winter_dots_count[i] / computeSum(winter_dots_count) };
                                --summer_height: { summer_dots_count[i] / computeSum(summer_dots_count) };
                                --spring_height: { spring_dots_count[i] / computeSum(spring_dots_count) };
                                --fall_height: { fall_dots_count[i] / computeSum(fall_dots_count) };
                                "
                                fill={fillColor(category)}
                            />
                        {/if}
                        {#if season === "Fall"}
                            <rect
                                class="{season}"
                                id="{category}"
                                x={ progress_bar_width }
                                y={yScale(season) + previousHeight(i, fall_dots_count)*50}
                                width={boxWidth}
                                style="
                                --winter_height:{ winter_dots_count[i] / computeSum(winter_dots_count) };
                                --summer_height: { summer_dots_count[i] / computeSum(summer_dots_count) };
                                --spring_height: { spring_dots_count[i] / computeSum(spring_dots_count) };
                                --fall_height: { fall_dots_count[i] / computeSum(fall_dots_count) };
                                "
                                fill={fillColor(category)}
                            />
                        {/if}
                        {#if season === "Spring"}
                            <rect
                                class="{season}"
                                id="{category}"
                                x={ progress_bar_width }
                                y={yScale(season) + previousHeight(i, spring_dots_count)*50}
                                width={boxWidth}
                                style="
                                --winter_height:{ winter_dots_count[i] / computeSum(winter_dots_count) };
                                --summer_height: { summer_dots_count[i] / computeSum(summer_dots_count) };
                                --spring_height: { spring_dots_count[i] / computeSum(spring_dots_count) };
                                --fall_height: { fall_dots_count[i] / computeSum(fall_dots_count) };
                                "
                                fill={fillColor(category)}
                            />
                        {/if}

                        {#if season === "Summer"}
                        <rect
                            class="{season}"
                            id="{category}"
                            x={ progress_bar_width }
                            y={yScale(season) + previousHeight(i, summer_dots_count)*50}
                            width={boxWidth}
                            style="
                            --winter_height:{ winter_dots_count[i] / computeSum(winter_dots_count) };
                            --summer_height: { summer_dots_count[i] / computeSum(summer_dots_count) };
                            --spring_height: { spring_dots_count[i] / computeSum(spring_dots_count) };
                            --fall_height: { fall_dots_count[i] / computeSum(fall_dots_count) };
                            "
                            fill={fillColor(category)}
                        />
                    {/if}
            
                        
                    {/each}
                {/each}
            </svg>
        </section>
        <section class="temp_legend">
            
            <!-- COULD REMOVE THIS  -->
            {#each bins as bin, i}
                <div 
                    class="bin_circles"
                    style="background-color:{fillColor(bin)}"
                />
            {/each}

            <!-- END OF SECTION TO POTENTIALLY REMOVE -->
    
            {#each temp_bins.reverse() as b, index}

                {#each bins as bin, i}
    
                    {#if b === "Summer"}
                        <dd class="percent_labels" style="--season_color: {fillColor(bin)};">{summer_dots_count[i]}</dd>
                    {/if}
                    {#if b === "Spring"}
                        <dd class="percent_labels" style="--season_color: {fillColor(bin)};">{spring_dots_count[i]}</dd>
                    {/if}

                    {#if b === "Fall"}
                        <dd class="percent_labels" style="--season_color: {fillColor(bin)};">{fall_dots_count[i]}</dd>
                    {/if}

                    {#if b === "Winter"}
                        <dd class="percent_labels" style="--season_color: {fillColor(bin)};">{winter_dots_count[i]}</dd>
                    {/if}

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