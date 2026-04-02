<script>
    import * as d3 from 'd3';
    import {
        computePosition,
        autoPlacement,
        offset,
    } from '@floating-ui/dom';

    import BarHorizontal from "$lib/BarHorizontal.svelte";

    let width = 500;
    let height = 300;

    export let commits = [];
    export let locData = [];

    commits = d3.sort(commits, d => -d.totalLines);

    // let margin = { top: 20, right: 20, bottom: 30, left: 60 };
    let margin = { top: 40, right: 50, bottom: 30, left: 60 };

    let innerWidth  = width  - margin.left - margin.right;
    let innerHeight = height - margin.top  - margin.bottom;

    let usableArea = {
        top: margin.top,
        right: width - margin.right,
        bottom: height - margin.bottom,
        left: margin.left
    };
    usableArea.width = usableArea.right - usableArea.left;
    usableArea.height = usableArea.bottom - usableArea.top;

    let xAxis, yAxis;
    let yAxisGridlines;

    $: [minDate, maxDate] = d3.extent(commits.map(d => d.date));
    $: maxDateFudged = new Date(maxDate);
    $: maxDateFudged.setDate(maxDateFudged.getDate() + 1);

    $: xScale = d3.scaleTime()
                .domain([minDate, maxDateFudged])
                .range([usableArea.left, usableArea.right])
                .nice();

    $: yScale = d3.scaleLinear()
                .domain([24, 0])
                .range([usableArea.bottom, usableArea.top]);
    
    $: rScale = d3.scaleSqrt()
                .domain([0, d3.max(commits, d => d.totalLines)])
                .range([2, 15]);


    // $: colorScale = d3.scaleOrdinal(d3.schemeTableau10)
    //     .domain(data.map(d => d.label));
    
    $: if (xAxis && yAxis && yAxisGridlines) {
        d3.select(xAxis).call(d3.axisBottom(xScale).ticks(8)); // reduce number of ticks
        d3.select(yAxis).call(d3.axisLeft(yScale).tickFormat(d => String(d % 24).padStart(2, "0") + ":00"));

        d3.select(yAxisGridlines).call(
            d3.axisLeft(yScale)
                .tickFormat(() => "")
                .tickSize(-usableArea.width)
        );
    }

    let hoveredIndex = -1;
    $: hoveredCommit = commits[hoveredIndex] ?? hoveredCommit ?? {};

    let commitTooltip;
    let tooltipPosition = {x: 0, y: 0};
    let clickedCommits = [];

    async function dotInteraction (index, evt) {
        let hoveredDot = evt.target;
        if (evt.type === "mouseenter") {
            hoveredIndex = index;
            tooltipPosition = await computePosition(hoveredDot, commitTooltip, {
                strategy: "fixed", // because we use position: fixed
                middleware: [
                    offset(5), // spacing from tooltip to dot
                    autoPlacement() // see https://floating-ui.com/docs/autoplacement
                ],
            });        }
        else if (evt.type === "mouseleave") {
            hoveredIndex = -1
        }
        else if (evt.type === "click") {
            let commit = commits[index]
            if (!clickedCommits.includes(commit)) {
                // Add the commit to the clickedCommits array
                clickedCommits = [...clickedCommits, commit];
            }
            else {
                    // Remove the commit from the array
                    clickedCommits = clickedCommits.filter(c => c !== commit);
            }
        }

    }

    $: selectedLines = (clickedCommits.length > 0 ? clickedCommits.flatMap(d => d.lines) : locData);
    $: selectedCounts = d3.rollup(selectedLines, v => v.length, d => d.type);
    $: allTypes = Array.from(new Set(locData.map(d => d.type)));
    $: barData = allTypes.map(type =>  ({ label: String(type), value: selectedCounts.get(type) || 0 }));



    // console.log(commits);
    // $: maxBar = d3.greatest(data, d => d.value);


</script>

<div class="container">
    <svg viewBox="0 0 {width} {height}">

        <text
            x={margin.left + innerWidth / 2}
            y={margin.top / 2}
            text-anchor="middle"
            class="chart-title">
            Commits by Time of Day
        </text>

        <g class="gridlines" transform="translate({usableArea.left}, 0)" bind:this={yAxisGridlines} />
        <g transform="translate(0, {usableArea.bottom})"
            bind:this={xAxis} class="tick-label"/>
        <g transform="translate({usableArea.left}, 0)"
            bind:this={yAxis} class="tick-label"/>

        <!-- x-axis label -->
        <text
            x={margin.left + innerWidth / 2}
            y={margin.top + innerHeight + margin.bottom + 10}
            text-anchor="middle"
            class="axis-label">
            Time of Day
        </text>

        <!-- y-axis label -->
        <!-- <text
            x={-(margin.top + innerHeight / 2)}
            y={margin.left - 40}
            text-anchor="middle"
            transform="rotate(-90)"
            class="axis-label">
            Number of Commits
        </text> -->

        <g class="dots">
        {#each commits as commit, index }
            <circle 
                class:selected={ clickedCommits.includes(commit) }
                on:click={ evt => dotInteraction(index, evt) }
                on:mouseenter={evt => dotInteraction(index, evt)}
                on:mouseleave={evt => dotInteraction(index, evt)}
                cx={ xScale(commit.datetime) }
                cy={ yScale(commit.hourFrac) }
                r={rScale(commit.totalLines)}
                fill="steelblue"
            />
            
        {/each}
        </g>
            
    </svg>

    <dl class="info tooltip" hidden={hoveredIndex === -1} bind:this={commitTooltip} style="top: {tooltipPosition.y}px; left: {tooltipPosition.x}px">
        <dt>Commit</dt>
        <dd><a href="{ hoveredCommit.url }" target="_blank">{ hoveredCommit.id }</a></dd>
    
        <dt>Date</dt>
        <dd>{ hoveredCommit.datetime?.toLocaleString("en", {dateStyle: "full"}) }</dd>
    
        <dt>Time</dt>
        <dd>{ hoveredCommit.datetime?.toLocaleTimeString("en") }</dd>
    
        <dt>Author</dt>
        <dd>{ hoveredCommit.author }</dd>
    
        <dt>Lines edited</dt>
            <dd>{ hoveredCommit.totalLines }</dd>
    </dl>

    <!-- <ul class="legend">
        {#each data as d}
            <li style="--color: {colorScale(d.label)}">
                <span class="swatch"></span>
                {d.label} <em>({d.value})</em>
            </li>
        {/each}
    </ul> -->
</div>
<div class="container">
    <BarHorizontal
        data={barData}
        height={150}
        title={clickedCommits.length > 0 ? "Lines of Code: Selected Commits" : "Lines of Code: Website Breakdown"}
    />
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

    .tick-label {
        font-size: 0.5em;
    }

    .gridlines {
        stroke: currentColor;
        stroke-opacity: 0.2;
        stroke-width: 0.6;
    }

    .dots {
        fill-opacity: 0.8;
    }

    circle {
        transition: 200ms;
        fill: var(--color-accent-med);
        &:hover {
            fill: var(--color-accent);
        }
    }
    .selected {
        fill: var(--color-accent);
    }

    dl.info {
        display: grid;
        grid-template-columns: max-content auto;
        gap: 0.2em 1em;
        margin: 0;
        transition-duration: 300ms;
        transition-property: opacity, visibility;

        &[hidden]:not(:hover, :focus-within) {
            opacity: 0;
            visibility: hidden;
        }
    }
    dl.info dt {
        margin: 0;
        color: #666;
        font-weight: normal;
        opacity: 0.8;
    }
    dl.info dd {
        margin: 0;
        font-weight: bold;
    }
    .tooltip {
        position: fixed;
        top: 1em;
        left: 1em;
        background: oklch(97% 0% 0 / 0.85);
        color: black;
        box-shadow: 0 4px 16px 0 oklch(0% 0 0 / 0.16);
        border-radius: 8px;
        padding: 0.8em 1.2em;
        z-index: 10;
        backdrop-filter: blur(4px);
        pointer-events: none;
    }

    .container {
        display: flex;
        gap: 1.5rem;
        margin-top: 1.5rem;
        width: 100%;
    }

    /* .annotation {
        font-size: 0.7em;
        fill: black;
        font-style: italic;
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
    } */

</style>