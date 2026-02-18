<script lang="ts">
  import type { TMovie } from "../types";
  import * as d3 from "d3";
  // define the props of the Bar component
  type Props = {
    movies: TMovie[];
    progress?: number;
    width?: number;
    height?: number;
  };
  // progress is 100 by default unless specified
  let { movies, progress = 100, width = 500, height = 400 }: Props = $props();

  let selectedGenre: string = $state();

  // processing the data; $derived is used to create a reactive variable that updates whenever the dependent variables change
  const yearRange = $derived(d3.extent(movies.map((d) => d.year)));

  function getUpYear(yearRange: [undefined, undefined] | [Date, Date]) {
    if (!yearRange[0]) return new Date();
    const timeScale = d3.scaleTime().domain(yearRange).range([0, 100]);
    return timeScale.invert(progress);
  }
  const upYear: Date = $derived(getUpYear(yearRange!));

  function getGenreNums(movies: TMovie[], upYear: Date) {
    let res: { [genre: string]: number } = {};
    movies
      .filter((movie) => movie.year <= upYear)
      .forEach((movie) => {
        movie.genres.forEach((genre: string) => {
          res[genre] = (res[genre] || 0) + 1;
        });
      });
    return res;
  }

  const genreNums = $derived(getGenreNums(movies, upYear));

  // drawing the bar chart

  const margin = {
    top: 15,
    bottom: 50,
    left: 30,
    right: 10,
  };

  let usableArea = {
    top: margin.top,
    right: width - margin.right,
    bottom: height - margin.bottom,
    left: margin.left,
  };

  const xScale = $derived(
    // tip: use d3.scaleBand() to create a band scale for the x-axis
    d3
      .scaleBand()
      .range([usableArea.left, usableArea.right])
      .domain(Object.keys(genreNums))
      .padding(0.2),
  );

  const yScale = $derived(
    // tip: use d3.scaleLinear() to create a linear scale for the y-axis
    d3
      .scaleLinear()
      .range([usableArea.bottom, usableArea.top])
      .domain([0, d3.max(Object.values(genreNums)) || 0]),
  );

  const xBarwidth: number = $derived(xScale.bandwidth());

  let xAxis: any = $state(),
    yAxis: any = $state();

  function updateAxis() {
    d3.select(xAxis)
      .call(d3.axisBottom(xScale))
      .selectAll("text")
      .attr("transform", "rotate(45)")
      .style("text-anchor", "start");

    // tip:
    // similar to the x-axis, create a y-axis using d3.axisLeft() and bind it to the yAxis variable
    d3.select(yAxis)
      .call(d3.axisLeft(yScale));
    }

  // the $effect function is used to run a function whenever the reactive variables change, also known as a side effect
  $effect(() => {   
    updateAxis();
  });

  //Q1
  type RankRow = { 
    year: number; 
    rank: number; 
    genre: string 
};

  const rankRows: RankRow[] = $derived.by(() => {
    if (!movies.length) return [];
    const yearGenreCount = new Map<number, Map<string, number>>();
    for (const movie of movies) {
      const year = movie.year instanceof Date ? movie.year.getFullYear() : Number(movie.year);
      if (!year || isNaN(year)) continue;
      if (!yearGenreCount.has(year)) yearGenreCount.set(year, new Map());
      const gMap = yearGenreCount.get(year)!;
      for (const g of movie.genres) {
        const genre = g.trim();
        if (!genre) continue;
        gMap.set(genre, (gMap.get(genre) ?? 0) + 1);
      }
    }
    const rows: RankRow[] = [];
    for (const [year, gMap] of yearGenreCount.entries()) {
      const sorted = [...gMap.entries()].sort((a, b) => b[1] - a[1]);
      for (let i = 0; i < Math.min(3, sorted.length); i++) {
        rows.push({ year, rank: i + 1, genre: sorted[i][0] });
      }
    }
    return rows;
  });

  type Segment = { genre: string; rank: number; startYear: number; endYear: number };

  const segments: Segment[] = $derived.by(() => {
    const segs: Segment[] = [];
    for (let rank = 1; rank <= 3; rank++) {
      const rr = rankRows.filter((r) => r.rank === rank).sort((a, b) => a.year - b.year);
      if (!rr.length) continue;
      let segStart = rr[0].year;
      let segGenre = rr[0].genre;
      for (let i = 1; i < rr.length; i++) {
        const prev = rr[i - 1];
        const curr = rr[i];
        if (curr.genre !== prev.genre || curr.year !== prev.year + 1) {
          segs.push({ genre: segGenre, rank, startYear: segStart, endYear: prev.year });
          segStart = curr.year;
          segGenre = curr.genre;
        }
      }
      const last = rr[rr.length - 1];
      segs.push({ genre: segGenre, rank, startYear: segStart, endYear: last.year });
    }
    return segs;
  });

  const allGenres: string[] = $derived(
    [...new Set(rankRows.map((r) => r.genre))].sort()
  );
  const allYears: number[] = $derived(
    [...new Set(rankRows.map((r) => r.year))].sort((a, b) => a - b)
  );
  const minYear: number = $derived(allYears[0] ?? 1945);
  const maxYear: number = $derived(allYears[allYears.length - 1] ?? 2023);

  const ROW_H = 26;
  const ROW_PAD = 6;
  const gMargin = { top: 52, right: 138, bottom: 40, left: 148 };

  let ganttWidth: number = $state(1100);
  let ganttContainerEl: HTMLDivElement | undefined = $state();

  $effect(() => {
    if (!ganttContainerEl) return;
    const ro = new ResizeObserver(() => {
      ganttWidth = ganttContainerEl!.clientWidth || 1100;
    });
    ro.observe(ganttContainerEl);
    return () => ro.disconnect();
  });

  const innerW: number = $derived(ganttWidth - gMargin.left - gMargin.right);
  const innerH: number = $derived(allGenres.length * (ROW_H + ROW_PAD));
  const ganttHeight: number = $derived(innerH + gMargin.top + gMargin.bottom);

  const gXScale = $derived(
    d3.scaleLinear().domain([minYear, maxYear + 1]).range([0, innerW])
  );
  const gYScale = $derived(
    d3.scaleBand().domain(allGenres).range([0, innerH]).padding(0.18)
  );
  const yearTicks: number[] = $derived(
    d3.range(Math.ceil(minYear / 5) * 5, maxYear + 1, 5)
  );

  let gXAxisEl: SVGGElement | undefined = $state();
  let gYAxisEl: SVGGElement | undefined = $state();

  $effect(() => {
    if (!gXAxisEl || !allYears.length) return;
    d3.select(gXAxisEl)
      .call(
        d3.axisBottom(gXScale)
          .tickValues(yearTicks)
          .tickFormat(d3.format("d") as any)
      );
  });

  $effect(() => {
    if (!gYAxisEl || !allGenres.length) return;
    d3.select(gYAxisEl)
      .call(d3.axisLeft(gYScale));
  });

  const rankColors = ["#e63946", "#f4a261", "#457b9d"];
  const rankLabels = ["1st Place", "2nd Place", "3rd Place"];

  type Tooltip = { visible: boolean; x: number; y: number; text: string };
  let tooltip: Tooltip = $state({ visible: false, x: 0, y: 0, text: "" });

  function onBarMove(e: MouseEvent, seg: Segment) {
    tooltip = {
      visible: true,
      x: e.offsetX + 14,
      y: e.offsetY - 14,
      text: `${seg.genre}  ·  Rank #${seg.rank}  ·  ${seg.startYear}${seg.endYear !== seg.startYear ? "–" + seg.endYear : ""}`,
    };
  }
  function onBarLeave() {
    tooltip = { ...tooltip, visible: false };
  }


//Q2
  type CoMatrix = Map<string, Map<string, number>>;

  const { hmGenres, hmMatrix } = $derived.by(() => {
    if (!movies.length) return { hmGenres: [] as string[], hmMatrix: new Map() as CoMatrix };

    const coMatrix: CoMatrix = new Map();
    for (const movie of movies) {
      const gs = movie.genres.map((g) => g.trim()).filter(Boolean);
      for (let i = 0; i < gs.length; i++) {
        for (let j = 0; j < gs.length; j++) {
          if (i === j) continue;
          const a = gs[i], b = gs[j];
          if (!coMatrix.has(a)) coMatrix.set(a, new Map());
          coMatrix.get(a)!.set(b, (coMatrix.get(a)!.get(b) ?? 0) + 1);
        }
      }
    }

    // Sort genres by total co-occurrence count descending
    const genreSet = new Set<string>();
    coMatrix.forEach((_, g) => genreSet.add(g));
    const genreList = [...genreSet].sort((a, b) => {
      const sumA = [...(coMatrix.get(a)?.values() ?? [])].reduce((s, v) => s + v, 0);
      const sumB = [...(coMatrix.get(b)?.values() ?? [])].reduce((s, v) => s + v, 0);
      return sumB - sumA;
    });

    return { hmGenres: genreList, hmMatrix: coMatrix };
  });

  const hmMaxVal = $derived.by(() => {
    let m = 0;
    hmMatrix.forEach((row) => row.forEach((v) => { if (v > m) m = v; }));
    return m;
  });

  const HM_MARGIN = { top: 120, right: 20, bottom: 20, left: 120 };
  let hmWidth: number = $state(900);
  let hmContainerEl: HTMLDivElement | undefined = $state();

  $effect(() => {
    if (!hmContainerEl) return;
    const ro = new ResizeObserver(() => {
      hmWidth = hmContainerEl!.clientWidth || 900;
    });
    ro.observe(hmContainerEl);
    return () => ro.disconnect();
  });

  const hmCell = $derived(
    Math.min(36, Math.floor((hmWidth - HM_MARGIN.left - HM_MARGIN.right) / Math.max(hmGenres.length, 1)))
  );
  const hmInnerW = $derived(hmCell * hmGenres.length);
  const hmInnerH = $derived(hmCell * hmGenres.length);
  const hmSvgHeight = $derived(hmInnerH + HM_MARGIN.top + HM_MARGIN.bottom);

  const hmColorScale = $derived(
    d3.scaleSequential().domain([0, hmMaxVal]).interpolator(d3.interpolateBlues)
  );

  const hmBandScale = $derived(
    d3.scaleBand().domain(hmGenres).range([0, hmInnerW]).padding(0.05)
  );

  let hmXAxisEl: SVGGElement | undefined = $state();
  let hmYAxisEl: SVGGElement | undefined = $state();

  $effect(() => {
    if (!hmXAxisEl || !hmGenres.length) return;
    d3.select(hmXAxisEl)
      .call(d3.axisTop(hmBandScale).tickSize(0))
      .call((ax) => {
        ax.select(".domain").remove();
        ax.selectAll("text")
          .attr("transform", "rotate(-45)")
          .style("text-anchor", "start")
          .attr("dy", "-0.4em")
          .attr("dx", "0.6em")
          .attr("font-size", "11px");
      });
  });

  $effect(() => {
    if (!hmYAxisEl || !hmGenres.length) return;
    d3.select(hmYAxisEl)
      .call(d3.axisLeft(hmBandScale).tickSize(0))
      .call((ax) => {
        ax.select(".domain").remove();
        ax.selectAll("text").attr("font-size", "11px").attr("dx", "-4px");
      });
  });

  let hmTooltip: Tooltip = $state({ visible: false, x: 0, y: 0, text: "" });

  function onCellMove(e: MouseEvent, a: string, b: string) {
    const val = hmMatrix.get(a)?.get(b) ?? 0;
    hmTooltip = {
      visible: true,
      x: e.offsetX + 14,
      y: e.offsetY - 14,
      text: val > 0 ? `${a} + ${b}: ${val} movies` : `${a} + ${b}: never co-occur`,
    };
  }
  function onCellLeave() {
    hmTooltip = { ...hmTooltip, visible: false };
  }

  let hmHighlighted: string | null = $state(null);

  function cellOpacity(a: string, b: string): number {
    if (!hmHighlighted) return 1;
    return a === hmHighlighted || b === hmHighlighted ? 1 : 0.15;
  }
</script>

<h3>
  The Distribution of Genres {yearRange[0]?.getFullYear()} - {yearRange[1]?.getFullYear()}
</h3>

{#if movies.length > 0}
  <svg {width} {height}>
    <g class="bars">
      {#each Object.entries(genreNums) as [genre, cnt]}
        <g class={genre}>
          <!-- tip: draw bars here using the svg <rect/> component -->
          <rect
            width={xBarwidth}
            height={yScale(0) - yScale(cnt)}
            x={xScale(genre)}
            y={yScale(cnt)}
            fill="#449900"
            class="bar"
            opacity={selectedGenre && selectedGenre !== genre ? 0.3 : 1}
            onmouseover={() => {
              selectedGenre = genre;
            }}
            onmouseout={() => {
              selectedGenre = "";
            }}
          /> 


          <text
            x={xScale(genre)! + xBarwidth / 2}
            y={yScale(cnt) - 5}
            font-size="12"
            text-anchor="middle"
          >
          <!-- tip: the text below should change with the hover on interaction -->
            {selectedGenre === genre ? cnt : ""} 
          </text>
        </g>
      {/each}
    </g>

    <g transform="translate(0, {usableArea.bottom})" bind:this={xAxis} />
    <g transform="translate({usableArea.left}, 0)" bind:this={yAxis} />
  </svg>
{/if}


<!-- Q1 -->
<h3>How Top 3 Genres Change Over Time (1945–2023)</h3>

{#if movies.length > 0}
  <div class="gantt-container" bind:this={ganttContainerEl}>

    {#if tooltip.visible}
      <div class="tooltip" style="left:{tooltip.x}px; top:{tooltip.y}px">
        {tooltip.text}
      </div>
    {/if}

    <svg width={ganttWidth} height={ganttHeight}>
      <defs>
        <clipPath id="gantt-clip">
          <rect width={innerW} height={innerH} />
        </clipPath>
      </defs>

      <g transform="translate({gMargin.left},{gMargin.top})">

        <!-- Light horizontal row stripes (like grid lines) -->
        {#each allGenres as genre, i}
          <rect
            x={0} y={gYScale(genre)}
            width={innerW} height={gYScale.bandwidth()}
            fill={i % 2 === 0 ? "#f5f5f5" : "white"}
          />
        {/each}

        <!-- Vertical grid every 5 years -->
        {#each yearTicks as yr}
          <line
            x1={gXScale(yr)} x2={gXScale(yr)}
            y1={0} y2={innerH}
            stroke="#e0e0e0" stroke-width={1}
          />
        {/each}

        <!-- Gantt bars -->
        <g clip-path="url(#gantt-clip)">
          {#each segments as seg}
            {@const bx = gXScale(seg.startYear) + 1}
            {@const bw = Math.max(2, gXScale(seg.endYear + 1) - gXScale(seg.startYear) - 2)}
            {@const by = (gYScale(seg.genre) ?? 0) + gYScale.bandwidth() * 0.1}
            {@const bh = gYScale.bandwidth() * 0.8}
            <g
              onmousemove={(e) => onBarMove(e, seg)}
              onmouseleave={onBarLeave}
              role="img"
              aria-label="{seg.genre} rank {seg.rank} {seg.startYear}–{seg.endYear}"
            >
              <rect
                x={bx} y={by} width={bw} height={bh}
                fill={rankColors[seg.rank - 1]}
                class="gantt-bar"
              />
              {#if bw > 26}
                <text
                  x={bx + bw / 2} y={by + bh / 2}
                  text-anchor="middle" dominant-baseline="middle"
                  fill="white" font-size="10" font-weight="600"
                  pointer-events="none"
                >
                  #{seg.rank}
                </text>
              {/if}
            </g>
          {/each}
        </g>

        <!-- Axes -->
        <g bind:this={gXAxisEl} transform="translate(0,{innerH})" />
        <g bind:this={gYAxisEl} />

      </g>

      <!-- Legend -->
      {#each rankLabels as label, i}
        <rect
          x={ganttWidth - gMargin.right + 18}
          y={gMargin.top + i * 26}
          width={16} height={13}
          fill={rankColors[i]}
        />
        <text
          x={ganttWidth - gMargin.right + 40}
          y={gMargin.top + i * 26 + 11}
          font-size="12"
        >{label}</text>
      {/each}
    </svg>

  </div>
{/if}


<h3>Genre Co-occurrence — Which Genres Appear Together?</h3>
<p class="chart-subtitle">
  Each cell shows how many movies share both genres. Click a cell to highlight a genre's row & column.
</p>

{#if movies.length > 0}
  <div class="hm-container" bind:this={hmContainerEl}>

    {#if hmTooltip.visible}
      <div class="tooltip" style="left:{hmTooltip.x}px; top:{hmTooltip.y}px">
        {hmTooltip.text}
      </div>
    {/if}

    <svg width={hmWidth} height={hmSvgHeight}>
      <g transform="translate({HM_MARGIN.left},{HM_MARGIN.top})">

        <!-- Cells -->
        {#each hmGenres as rowG}
          {#each hmGenres as colG}
            {@const val = hmMatrix.get(rowG)?.get(colG) ?? 0}
            {@const isDiag = rowG === colG}
            <rect
              x={hmBandScale(colG)}
              y={hmBandScale(rowG)}
              width={hmBandScale.bandwidth()}
              height={hmBandScale.bandwidth()}
              fill={isDiag ? "#eeeeee" : val === 0 ? "#fafafa" : hmColorScale(val)}
              stroke="#e8e8e8"
              stroke-width={0.5}
              opacity={cellOpacity(rowG, colG)}
              class="hm-cell"
              onmousemove={(e) => onCellMove(e, rowG, colG)}
              onmouseleave={onCellLeave}
              onclick={() => { hmHighlighted = hmHighlighted === rowG ? null : rowG; }}
              role="img"
              aria-label="{rowG} and {colG}: {val}"
            />
            {#if hmCell >= 24 && val > 0 && !isDiag}
              <text
                x={(hmBandScale(colG) ?? 0) + hmBandScale.bandwidth() / 2}
                y={(hmBandScale(rowG) ?? 0) + hmBandScale.bandwidth() / 2}
                text-anchor="middle"
                dominant-baseline="middle"
                font-size={hmCell >= 30 ? "10" : "8"}
                fill={val > hmMaxVal * 0.6 ? "white" : "#333"}
                pointer-events="none"
                opacity={cellOpacity(rowG, colG)}
              >{val}</text>
            {/if}
          {/each}
        {/each}

        <!-- Axes -->
        <g bind:this={hmXAxisEl} />
        <g bind:this={hmYAxisEl} />

      </g>

      <!-- Color legend -->
      <defs>
        <linearGradient id="hm-legend-grad" x1="0" x2="1" y1="0" y2="0">
          <stop offset="0%" stop-color={hmColorScale(0)} />
          <stop offset="100%" stop-color={hmColorScale(hmMaxVal)} />
        </linearGradient>
      </defs>
      <rect x={hmWidth - 184} y={18} width={160} height={12} fill="url(#hm-legend-grad)" stroke="#ccc" stroke-width={0.5} />
      <text x={hmWidth - 184} y={44} font-size="10" fill="#555">0</text>
      <text x={hmWidth - 184 + 160} y={44} font-size="10" fill="#555" text-anchor="end">{hmMaxVal}</text>
      <text x={hmWidth - 184 + 80} y={14} font-size="10" fill="#555" text-anchor="middle">co-occurrences</text>
    </svg>

  </div>
{/if}



<style>
  .bar {
    transition:
      y 0.1s ease,
      height 0.1s ease,
      width 0.1s ease; /* Smooth transition for height */
  }


/* Q1 */
  .gantt-container {
    position: relative;
    overflow-x: auto;
  }

  .gantt-bar {
    cursor: pointer;
    transition: opacity 0.15s;
  }
  .gantt-bar:hover {
    opacity: 0.75;
  }

  .tooltip {
    position: absolute;
    background: white;
    border: 1px solid #ccc;
    border-radius: 4px;
    padding: 4px 10px;
    font-size: 12px;
    pointer-events: none;
    white-space: nowrap;
    z-index: 10;
    box-shadow: 0 1px 6px #0002;
  }

  /* Q2 */
  .chart-subtitle {
    font-size: 13px;
    color: #666;
    margin: -6px 0 10px;
  }

  .hm-container {
    position: relative;
    overflow-x: auto;
  }

  .hm-cell {
    cursor: pointer;
    transition: opacity 0.15s;
  }
  .hm-cell:hover {
    stroke: #333 !important;
    stroke-width: 1.5px !important;
  }
</style>