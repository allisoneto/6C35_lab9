<script>
    import * as d3 from 'd3';

    let width = 400;
    let height = 200;

    // let data = [
    //     { label: "A", value: 10 },
    //     { label: "B", value: 20 },
    //     { label: "C", value: 15 },
    //     { label: "D", value: 8 },
    //     { label: "E", value: 25 }
    // ];

    export let data = [];



    // let margin = { top: 20, right: 20, bottom: 30, left: 60 };
    let margin = { top: 40, right: 50, bottom: 30, left: 60 };

    let innerWidth  = width  - margin.left - margin.right;
    let innerHeight = height - margin.top  - margin.bottom;

    let xAxis, yAxis;


    $: xScale = d3.scaleBand()
        .domain(data.map(d => d.label))
        .range([0, innerWidth])
        .padding(0.2);

    $: yScale = d3.scaleLinear()
        .domain([0, d3.max(data, d => d.value) || 1])
        .range([innerHeight, 0]);

    $: colorScale = d3.scaleOrdinal(d3.schemeTableau10)
        .domain(data.map(d => d.label));
    
    $: if (xAxis && yAxis) {
        d3.select(xAxis).call(d3.axisBottom(xScale));
        d3.select(yAxis).call(
            d3.axisLeft(yScale)
                .tickFormat(d => Number.isInteger(d) ? d : "")
                .tickValues(d3.range(0, d3.max(data, d => d.value) + 1))
        );
    }

    $: maxBar = d3.greatest(data, d => d.value);


</script>

<div class="container">
    <svg viewBox="0 0 {width} {height}">

        <text
        x={margin.left + innerWidth / 2}
        y={margin.top / 2}
        text-anchor="middle"
        class="chart-title">
        Projects per Year
    </text>

        <g transform="translate({margin.left}, {margin.top + innerHeight})"
            bind:this={xAxis} />
        <g transform="translate({margin.left}, {margin.top})"
            bind:this={yAxis} />

        <!-- x-axis label -->
        <text
            x={margin.left + innerWidth / 2}
            y={margin.top + innerHeight + margin.bottom + 10}
            text-anchor="middle"
            class="axis-label">
            Year
        </text>

        <!-- y-axis label -->
        <text
            x={-(margin.top + innerHeight / 2)}
            y={margin.left - 40}
            text-anchor="middle"
            transform="rotate(-90)"
            class="axis-label">
            Number of Projects
        </text>

        <g transform="translate({margin.left}, {margin.top})">
            {#each data as d}
                <rect
                    x={xScale(d.label)}
                    y={yScale(d.value)}
                    width={xScale.bandwidth()}
                    height={innerHeight - yScale(d.value)}
                    fill={colorScale(d.label)}
                />
            {/each}
            {#if maxBar}
                <!-- highlight outline around the tallest bar -->
                <rect
                    x={xScale(maxBar.label)}
                    y={yScale(maxBar.value)}
                    width={xScale.bandwidth()}
                    height={innerHeight - yScale(maxBar.value)}
                    fill="none"
                    stroke="currentColor"
                    stroke-width="2"
                />
                <!-- leader line -->
                <line
                    x1={xScale(maxBar.label) + xScale.bandwidth()}
                    y1={yScale(maxBar.value) + (innerHeight - yScale(maxBar.value)) / 2}
                    x2={xScale(maxBar.label) + xScale.bandwidth() + 30}
                    y2={yScale(maxBar.value) + (innerHeight - yScale(maxBar.value)) / 2}
                    stroke="currentColor"
                    stroke-width="1"
                />
                <!-- annotation text at end of leader line -->
                <text
                    x={xScale(maxBar.label) + xScale.bandwidth() + 35}
                    y={yScale(maxBar.value) + (innerHeight - yScale(maxBar.value)) / 2}
                    dominant-baseline="middle"
                    class="annotation">
                    Year with most projects
                </text>
            {/if}

        </g>
    </svg>

    <ul class="legend">
        {#each data as d}
            <li style="--color: {colorScale(d.label)}">
                <span class="swatch"></span>
                {d.label} <em>({d.value})</em>
            </li>
        {/each}
    </ul>
</div>


<style>
    svg {
        max-width: 100%;
        height: auto;
        overflow: visible;
    }

    .chart-title {
        font-size: 1em;
        font-weight: bold;
        fill: currentColor;
    }

    .axis-label {
        font-size: 0.8em;
        fill: currentColor;
    }

    .annotation {
        font-size: 0.7em;
        fill: black;
        font-style: italic;
    }


    .container {
        display: flex;
        gap: 1.5rem;
        margin-top: 1.5rem;
        width: 100%;
    }

    .legend {
        flex: 1;
        list-style: none;
        padding: 0;
        margin: 0;
    }

    .legend li {
        display: flex;
        align-items: center;
        gap: 0.5em;
    }

    .swatch {
        width: 1em;
        height: 1em;
        background: var(--color);
        margin-right: 0.5em;
        border-radius: 3px;
    }

</style>