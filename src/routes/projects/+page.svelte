<script>
  import projects from "$lib/projects.json";
  import Project from "$lib/Project.svelte";
  import ProjectNarrative from "$lib/ProjectNarrative.svelte";
  import Bar from "$lib/Bar.svelte";

  let years = projects.map(proj => proj.year)
  let validYears = years.filter(year => typeof year === "number");
  let range = validYears.length > 0 ? Math.max(...validYears) - Math.min(...validYears) + 1 : 0;
  
  import { onMount } from "svelte";
  import * as d3 from "d3";

  let rawData = [];
  let wrangled = [];
  let wrangled_percentages = [];
  
  onMount(async () => {
      rawData = await d3.json('/lab6_example.json');
      console.log(rawData);
      wrangled = d3.rollups(
          rawData,
          v => d3.sum(v, d => d.lines),
          d => d.language
      );
      wrangled_percentages = wrangled.map(d => d[1] / d3.sum(wrangled, d => d[1]));
      // console.log(wrangled);
  });

  $: barData = d3.rollups(projects, v => v.length, d => d.year)
    .map(([year, count]) => ({ label: String(year), value: count }));

  </script>

<!-- <section>
  <h2>Data wrangling result</h2>
  <pre>{JSON.stringify(wrangled, null, 2)}</pre>
  <pre>{JSON.stringify(wrangled_percentages, null, 2)}</pre>
</section> -->

<svelte:head>
  <title>Projects</title>
</svelte:head>

<h1>{projects.length} Projects over {range} Years</h1>

<Bar data={barData} />

<p>Scroll down to see my a timeline of my projects and how they've contributed to my professional and personal life</p>

<ProjectNarrative />

<p class="outro">Thanks for scrolling through my project story! Feel free to explore all of the projects at your leisure below.</p>


<h1>{projects.length} Projects</h1>

<div class="projects">
    {#each projects as p}
    <Project data={p} />
    {/each}
</div>