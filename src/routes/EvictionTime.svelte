<script>
    import * as d3 from "d3";
    export let data = [];
    export let raw_data = [];

    let svg;
    let width = 800, height = 275;
    let margin = {top: 20, right: 30, bottom: 70, left: 70}; // Adjusted margins to provide space for labels

    // will need to incorporate the months in the x axis 
    // Set the scales with fixed domains
    // d3.scaleTime().domain(d3.extent(commits, d => d.datetime))

    $: xScale = d3.scaleTime()
                 .domain(d3.extent(raw_data, d => d.date)) // Set fixed domain for years
                 .range([margin.left, width - margin.right])
                 .nice();

    $: yScale = d3.scaleLinear()
                 .domain([0, d3.max(raw_data, d => d.total_filings)]) // Set fixed domain for eviction rate
                 .range([height - margin.bottom, margin.top]);
    // $: data = d3.filter( (raw_data) => d.year <= selectedYear);
    $: {
        if (svg) {
            // Render x-axis
            d3.select(svg)
              .select(".x-axis")
              .attr("transform", `translate(0,${height - margin.bottom})`)
            //   .call(d3.axisBottom(xScale).tickFormat(d3.format("d")));
            .call(d3.axisBottom(xScale).tickFormat(d3.timeFormat("%B %Y")));

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
              .text("Date");

            // Add Y-axis label
            d3.select(svg)
              .select(".y-label")
              .attr("transform", "rotate(-90)")
              .attr("y", 15)
              .attr("x", 0 - (height / 2.5))
              .style("text-anchor", "middle")
              .text("Total Eviction Filings");
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
            <circle cx={xScale(d.date)} cy={yScale(d.total_filings)} r="5" fill="#744665"></circle>
        {/each}
    </svg>
</div>