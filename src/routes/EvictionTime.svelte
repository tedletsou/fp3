<script>
    import * as d3 from "d3";
    export let data = [];

    let svg;
    let width = 800, height = 275;
    let margin = {top: 20, right: 30, bottom: 70, left: 70}; // Adjusted margins to provide space for labels

    // Set the scales with fixed domains
    $: xScale = d3.scaleLinear()
                 .domain([2000, 2023]) // Set fixed domain for years
                 .range([margin.left, width - margin.right]);

    $: yScale = d3.scaleLinear()
                 .domain([0, 12]) // Set fixed domain for eviction rate
                 .range([height - margin.bottom, margin.top]);

    $: {
        if (svg) {
            // Render x-axis
            d3.select(svg)
              .select(".x-axis")
              .attr("transform", `translate(0,${height - margin.bottom})`)
              .call(d3.axisBottom(xScale).tickFormat(d3.format("d")));

            // Render y-axis
            d3.select(svg)
              .select(".y-axis")
              .attr("transform", `translate(${margin.left},0)`)
              .call(d3.axisLeft(yScale));

            // Add X-axis label
            d3.select(svg)
              .select(".x-label")
              .attr("transform", `translate(${width / 2}, ${height - 20})`)
              .style("text-anchor", "middle")
              .text("Year");

            // Add Y-axis label
            d3.select(svg)
              .select(".y-label")
              .attr("transform", "rotate(-90)")
              .attr("y", 15)
              .attr("x", 0 - (height / 2.5))
              .style("text-anchor", "middle")
              .text("Eviction Rate (%)");
        }
    }
</script>


<style>
    svg {
        font-family: 'Helvetica', sans-serif;
    }
    .tick text {
        fill: #000;
        font-size: 20px; /* Set the desired font size for tick labels */
    }
    text {
        fill: #000;
        font-size: 20px; /* General text, e.g., axis labels */
    }
</style>

<div>
    <svg bind:this={svg} viewBox={`0 0 ${width} ${height}`}>
        <!-- Axes groups -->
        <g class="x-axis" />
        <g class="y-axis" />
        <text class="x-label"></text> <!-- X-axis label -->
        <text class="y-label"></text> <!-- Y-axis label -->
        <!-- Data points -->
        {#each data as d}
            <circle cx={xScale(d.year)} cy={yScale(d.eviction_rate)} r="5" fill="#744665"></circle>
        {/each}
    </svg>
</div>
