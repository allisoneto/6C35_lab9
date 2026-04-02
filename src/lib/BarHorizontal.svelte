<script>
    import * as d3 from 'd3';

    export let data = [];
    export let height = 300;
    export let title = "Lines per Language";
    let width = 400;

    $: fontSizeScaler = height / 300;
    $: offsetScaler = height / 500;

    // Wider left margin to accommodate Language labels on y-axis;
    // smaller bottom margin since x-axis tick labels are short numerics (no rotation needed)
    $: margin = { 
        top: Math.round(40 * fontSizeScaler), 
        right: Math.round(50 * fontSizeScaler), 
        bottom: Math.round(60 * fontSizeScaler), 
        left: Math.round(60 * fontSizeScaler) 
    };

    $: innerWidth  = width  - margin.left - margin.right;
    $: innerHeight = height - margin.top  - margin.bottom;

    let xAxis, yAxis;

    // xScale: linear — maps quantitative values to horizontal pixels
    $: xScale = d3.scaleLinear()
        .domain([0, d3.max(data, d => d.value) || 1])
        .range([0, innerWidth]);

    // yScale: band — maps category labels to vertical positions
    $: yScale = d3.scaleBand()
        .domain(data.map(d => d.label))
        .range([0, innerHeight])
        .padding(0.2);

    $: colorScale = d3.scaleOrdinal(d3.schemeTableau10)
        .domain(data.map(d => d.label));

    // xAxis at bottom, yAxis at left
    // $: if (xAxis && yAxis) {
    //     d3.select(xAxis).call(
    //         d3.axisBottom(xScale)
    //             .tickFormat(d => Number.isInteger(d) ? d : "")
    //             .tickValues(d3.range(0, (d3.max(data, d => d.value) || 0) + 1))
    //     );
    //     d3.select(yAxis).call(d3.axisLeft(yScale));
    // }
    $: if (xAxis && yAxis) {
        d3.select(xAxis).call(d3.axisBottom(xScale));
        d3.select(yAxis).call(d3.axisLeft(yScale));
    }

    $: maxBar = d3.greatest(data, d => d.value);

</script>

<div class="container" style="--font-size-scaler: {fontSizeScaler}">
    <svg viewBox="0 0 {width} {height}">

        <g transform="translate({margin.left}, {margin.top})">
            {#each data as d}
                <!-- x=0: all bars start at the left axis; width grows rightward -->
                <rect
                    x={0}
                    y={yScale(d.label)}
                    width={xScale(d.value)}
                    height={yScale.bandwidth()}
                    fill={colorScale(d.label)}
                />
            {/each}

            {#if maxBar}
                <!-- highlight outline around the longest bar -->
                <rect
                    x={0}
                    y={yScale(maxBar.label)}
                    width={xScale(maxBar.value)}
                    height={yScale.bandwidth()}
                    fill="none"
                    stroke="currentColor"
                    stroke-width="2"
                />
                <!-- leader line extending down from the bar tip -->
                <line
                    x1={xScale(maxBar.value / 2)}
                    y1={yScale(maxBar.label) + yScale.bandwidth()}
                    x2={xScale(maxBar.value / 2)}
                    y2={yScale(maxBar.label) + yScale.bandwidth() + 30 * offsetScaler}
                    stroke="currentColor"
                    stroke-width="1"
                />
                <!-- annotation text below the end of leader line -->
                <text
                    x={xScale(maxBar.value / 2)}
                    y={yScale(maxBar.label) + yScale.bandwidth() + 35 * offsetScaler}
                    text-anchor="middle"
                    dominant-baseline="hanging"
                    class="annotation">
                    Most lines of code
                </text>
            {/if}

        </g>
        
        <text
            x={margin.left + innerWidth / 2}
            y={margin.top / 2}
            text-anchor="middle"
            class="chart-title">
            {title}
        </text>

        <!-- xAxis sits at the bottom of the inner plot area -->
        <g transform="translate({margin.left}, {margin.top + innerHeight})"
            bind:this={xAxis} class="tick-label"/>
        <!-- yAxis sits at the left edge of the inner plot area -->
        <g transform="translate({margin.left}, {margin.top})"
            bind:this={yAxis} class="tick-label"/>

        <!-- x-axis label -->
        <text
            x={margin.left + innerWidth / 2}
            y={margin.top + innerHeight + margin.bottom - 5}
            text-anchor="middle"
            class="axis-label">
            Number of Lines
        </text>

        <!-- y-axis label (rotated) -->
        <text
            x={-(margin.top + innerHeight / 2)}
            y={margin.left - 45}
            text-anchor="middle"
            transform="rotate(-90)"
            class="axis-label">
            Language
        </text>

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
        font-size: calc(var(--font-size-scaler, 1) * 1em);
        font-weight: bold;
        fill: currentColor;
    }

    .axis-label {
        font-size: calc(var(--font-size-scaler, 1) * 0.8em);
        fill: currentColor;
    }

    .annotation {
        font-size: calc(var(--font-size-scaler, 1) * 0.7em);
        fill: black;
        font-style: italic;
    }

    .tick-label {
        font-size: calc(var(--font-size-scaler, 1) * 0.8em);
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
        margin-top: 1.5rem;
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
