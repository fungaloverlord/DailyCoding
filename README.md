<!-- _includes/graph-aside.html
     Drop into your Jekyll layout with {% include graph-aside.html %}
     Reads from /assets/graph.json produced by build_graph.rb
     Requires: nothing external beyond D3 v7 (loaded below via CDN)
-->

<aside class="doc-graph" aria-label="Document relationship graph">

  <header class="graph-header">
    <span class="graph-title">
      <svg width="12" height="12" viewBox="0 0 12 12" fill="none" aria-hidden="true">
        <circle cx="2"  cy="2"  r="1.5" fill="currentColor"/>
        <circle cx="10" cy="2"  r="1.5" fill="currentColor"/>
        <circle cx="6"  cy="10" r="1.5" fill="currentColor"/>
        <line x1="2"  y1="2"  x2="10" y2="2"  stroke="currentColor" stroke-width="0.75"/>
        <line x1="2"  y1="2"  x2="6"  y2="10" stroke="currentColor" stroke-width="0.75"/>
        <line x1="10" y1="2"  x2="6"  y2="10" stroke="currentColor" stroke-width="0.75"/>
      </svg>
      Graph
    </span>
    <button class="graph-toggle" aria-expanded="true" aria-controls="graph-canvas-wrap" title="Collapse graph">
      <svg width="10" height="10" viewBox="0 0 10 10" fill="none" aria-hidden="true">
        <path d="M1 3l4 4 4-4" stroke="currentColor" stroke-width="1.5" stroke-linecap="round"/>
      </svg>
    </button>
  </header>

  <div class="graph-search-wrap">
    <svg class="graph-search-icon" width="11" height="11" viewBox="0 0 11 11" fill="none" aria-hidden="true">
      <circle cx="4.5" cy="4.5" r="3.5" stroke="currentColor" stroke-width="1.2"/>
      <line x1="7.5" y1="7.5" x2="10" y2="10" stroke="currentColor" stroke-width="1.2" stroke-linecap="round"/>
    </svg>
    <input
      id="graph-search"
      class="graph-search"
      type="search"
      placeholder="Filter nodes…"
      aria-label="Filter graph nodes"
      autocomplete="off"
    />
  </div>

  <div id="graph-canvas-wrap" class="graph-canvas-wrap">
    <svg id="graph-canvas" aria-label="Document link graph" role="img"></svg>
    <div id="graph-tooltip" class="graph-tooltip" role="tooltip" aria-hidden="true"></div>
    <p class="graph-empty" id="graph-empty" hidden>No matching documents.</p>
  </div>

  <footer class="graph-legend">
    <span class="legend-dot legend-dot--current"></span><span>current</span>
    <span class="legend-dot legend-dot--linked"></span><span>linked</span>
    <span class="legend-dot legend-dot--other"></span><span>other</span>
  </footer>

</aside>

<!-- ─── Styles ──────────────────────────────────────────────────────────────── -->
<style>
  /* Tokens — override these in your site's CSS if needed */
  .doc-graph {
    --g-bg:          #0f1117;
    --g-surface:     #181c27;
    --g-border:      #2a2f3d;
    --g-text:        #8892a4;
    --g-text-hi:     #c9d1df;
    --g-accent:      #e8a838;
    --g-accent-dim:  #7a5518;
    --g-linked:      #4a90d9;
    --g-linked-dim:  #1e3a5a;
    --g-other:       #3a4255;
    --g-edge:        #2e3548;
    --g-edge-hi:     #4a90d9;
    --g-missing:     #6b3030;
    --g-radius:      6px;
    --g-font:        "JetBrains Mono", "Fira Code", "Cascadia Code", ui-monospace, monospace;
    --g-width:       240px;
    --g-height:      300px;
  }

  .doc-graph {
    font-family: var(--g-font);
    font-size: 11px;
    background: var(--g-bg);
    border: 1px solid var(--g-border);
    border-radius: var(--g-radius);
    color: var(--g-text);
    width: var(--g-width);
    overflow: hidden;
    user-select: none;
  }

  /* Header */
  .graph-header {
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: 8px 10px 7px;
    border-bottom: 1px solid var(--g-border);
    letter-spacing: 0.08em;
    text-transform: uppercase;
    font-size: 10px;
    color: var(--g-text);
  }
  .graph-title {
    display: flex;
    align-items: center;
    gap: 6px;
  }
  .graph-toggle {
    background: none;
    border: none;
    color: var(--g-text);
    cursor: pointer;
    padding: 2px 4px;
    border-radius: 3px;
    display: flex;
    align-items: center;
    transition: color 0.15s, background 0.15s;
  }
  .graph-toggle:hover { background: var(--g-border); color: var(--g-text-hi); }
  .graph-toggle[aria-expanded="false"] svg { transform: rotate(-90deg); }
  .graph-toggle svg { transition: transform 0.2s ease; }

  /* Search */
  .graph-search-wrap {
    position: relative;
    border-bottom: 1px solid var(--g-border);
  }
  .graph-search-icon {
    position: absolute;
    left: 10px;
    top: 50%;
    transform: translateY(-50%);
    color: var(--g-text);
    pointer-events: none;
  }
  .graph-search {
    width: 100%;
    box-sizing: border-box;
    background: transparent;
    border: none;
    color: var(--g-text-hi);
    font-family: var(--g-font);
    font-size: 11px;
    padding: 7px 10px 7px 28px;
    outline: none;
  }
  .graph-search::placeholder { color: var(--g-text); opacity: 0.6; }
  .graph-search::-webkit-search-cancel-button { display: none; }

  /* Canvas wrap */
  .graph-canvas-wrap {
    position: relative;
    height: var(--g-height);
    overflow: hidden;
    background:
      radial-gradient(ellipse at 30% 40%, #1a2236 0%, transparent 60%),
      var(--g-bg);
  }
  /* Collapsed state */
  .graph-canvas-wrap[hidden] { display: none; }

  #graph-canvas {
    width: 100%;
    height: 100%;
    display: block;
  }

  /* Tooltip */
  .graph-tooltip {
    position: absolute;
    background: var(--g-surface);
    border: 1px solid var(--g-border);
    border-radius: 4px;
    padding: 4px 8px;
    font-size: 11px;
    color: var(--g-text-hi);
    pointer-events: none;
    white-space: nowrap;
    opacity: 0;
    transform: translateY(2px);
    transition: opacity 0.12s, transform 0.12s;
    z-index: 10;
    max-width: 180px;
    overflow: hidden;
    text-overflow: ellipsis;
  }
  .graph-tooltip.visible {
    opacity: 1;
    transform: translateY(0);
  }

  /* Empty state */
  .graph-empty {
    position: absolute;
    inset: 0;
    display: flex;
    align-items: center;
    justify-content: center;
    margin: 0;
    color: var(--g-text);
    font-size: 11px;
  }

  /* Legend */
  .graph-legend {
    display: flex;
    align-items: center;
    gap: 6px;
    padding: 6px 10px;
    border-top: 1px solid var(--g-border);
    font-size: 10px;
    letter-spacing: 0.04em;
    color: var(--g-text);
  }
  .legend-dot {
    width: 7px;
    height: 7px;
    border-radius: 50%;
    flex-shrink: 0;
  }
  .legend-dot--current { background: var(--g-accent); }
  .legend-dot--linked  { background: var(--g-linked); }
  .legend-dot--other   { background: var(--g-other); border: 1px solid var(--g-border); }

  /* D3 node/edge styles (applied via JS, but kept here for reference) */
  .g-node circle  { cursor: pointer; transition: filter 0.15s; }
  .g-node:hover circle { filter: brightness(1.3); }
  .g-node text    {
    fill: var(--g-text);
    font-family: var(--g-font);
    font-size: 9px;
    pointer-events: none;
    dominant-baseline: middle;
  }
  .g-edge { stroke: var(--g-edge); stroke-width: 1; }
  .g-edge.highlighted { stroke: var(--g-edge-hi); stroke-width: 1.5; }
</style>

<!-- ─── Script ──────────────────────────────────────────────────────────────── -->
<script src="https://cdnjs.cloudflare.com/ajax/libs/d3/7.9.0/d3.min.js"
        integrity="sha512-vc58zeoONBCkfGnl2SWIF/6BIhpinZQkMlRIfhYvJWk2IHK8AMlwHrNUmKD3+lmNHSJR5Vw2kbBHJVPKaHnQ=="
        crossorigin="anonymous"
        referrerpolicy="no-referrer"></script>

<script>
(function () {
  "use strict";

  // ── Config ────────────────────────────────────────────────────────────────
  const GRAPH_JSON   = "/assets/graph.json";
  const CURRENT_PATH = (typeof PAGE_PATH !== "undefined")
    ? PAGE_PATH                                 // set by Jekyll: {{ page.path }}
    : document.location.pathname.replace(/^\/|\.html$/g, "").replace(/^docs\//, "");

  const NODE_R       = { current: 6, linked: 4.5, other: 3.5 };
  const COLORS       = {
    current : "var(--g-accent)",
    linked  : "var(--g-linked)",
    other   : "var(--g-other)",
    missing : "var(--g-missing)",
  };

  // ── DOM refs ──────────────────────────────────────────────────────────────
  const aside    = document.querySelector(".doc-graph");
  const svgEl    = document.getElementById("graph-canvas");
  const wrap     = document.getElementById("graph-canvas-wrap");
  const tooltip  = document.getElementById("graph-tooltip");
  const searchEl = document.getElementById("graph-search");
  const emptyEl  = document.getElementById("graph-empty");
  const toggleBtn= aside.querySelector(".graph-toggle");

  // ── Collapse toggle ───────────────────────────────────────────────────────
  toggleBtn.addEventListener("click", () => {
    const expanded = toggleBtn.getAttribute("aria-expanded") === "true";
    toggleBtn.setAttribute("aria-expanded", String(!expanded));
    wrap.hidden = expanded;
    if (!expanded) simulation?.alpha(0.3).restart();
  });

  // ── Load graph data ───────────────────────────────────────────────────────
  let allNodes = [], allEdges = [];
  let simulation, svgRoot, edgeSel, nodeSel;

  fetch(GRAPH_JSON)
    .then(r => { if (!r.ok) throw new Error(r.statusText); return r.json(); })
    .then(data => {
      allNodes = data.nodes || [];
      allEdges = data.edges || [];
      buildGraph(allNodes, allEdges);
    })
    .catch(err => {
      wrap.innerHTML = `<p class="graph-empty" style="position:absolute;inset:0;display:flex;align-items:center;justify-content:center;margin:0;opacity:.5;">
        Could not load graph.json</p>`;
      console.warn("[graph-aside]", err);
    });

  // ── Build / rebuild force graph ───────────────────────────────────────────
  function buildGraph(nodes, edges) {
    const W = svgEl.clientWidth  || 240;
    const H = svgEl.clientHeight || 300;

    // Classify nodes relative to current page
    const linkedIds = new Set(
      edges.filter(e => e.from === CURRENT_PATH || e.to === CURRENT_PATH)
           .flatMap(e => [e.from, e.to])
    );

    nodes.forEach(n => {
      n._class = n.id === CURRENT_PATH ? "current"
               : linkedIds.has(n.id)  ? "linked"
               : n.missing            ? "missing"
               : "other";
    });

    // Mutable copies for simulation
    const simNodes = nodes.map(n => ({ ...n }));
    const idIdx    = Object.fromEntries(simNodes.map((n, i) => [n.id, i]));
    const simEdges = edges
      .filter(e => idIdx[e.from] !== undefined && idIdx[e.to] !== undefined)
      .map(e => ({ source: idIdx[e.from], target: idIdx[e.to] }));

    // Clear previous
    d3.select(svgEl).selectAll("*").remove();

    svgRoot = d3.select(svgEl)
      .attr("viewBox", `0 0 ${W} ${H}`)
      .call(d3.zoom()
        .scaleExtent([0.4, 4])
        .on("zoom", e => svgRoot.select("g.root").attr("transform", e.transform))
      );

    const root = svgRoot.append("g").attr("class", "root");

    // Arrow marker for directed edges
    svgRoot.append("defs").append("marker")
      .attr("id", "arrow")
      .attr("viewBox", "0 -3 6 6")
      .attr("refX", 12)
      .attr("markerWidth", 6)
      .attr("markerHeight", 6)
      .attr("orient", "auto")
      .append("path")
        .attr("d", "M0,-3L6,0L0,3")
        .attr("fill", "var(--g-edge)");

    edgeSel = root.append("g").attr("class", "edges")
      .selectAll("line")
      .data(simEdges)
      .join("line")
        .attr("class", "g-edge")
        .attr("marker-end", "url(#arrow)");

    nodeSel = root.append("g").attr("class", "nodes")
      .selectAll("g")
      .data(simNodes)
      .join("g")
        .attr("class", "g-node")
        .call(d3.drag()
          .on("start", dragStart)
          .on("drag",  dragging)
          .on("end",   dragEnd))
        .on("mouseenter", onNodeEnter)
        .on("mouseleave", onNodeLeave)
        .on("click", onNodeClick);

    nodeSel.append("circle")
      .attr("r", n => NODE_R[n._class] ?? 3.5)
      .attr("fill", n => COLORS[n._class])
      .attr("stroke", n => n._class === "current" ? "rgba(232,168,56,0.35)" : "none")
      .attr("stroke-width", 4);

    nodeSel.append("text")
      .text(n => truncate(n.label, 18))
      .attr("x", n => (NODE_R[n._class] ?? 3.5) + 4)
      .attr("y", 0)
      .style("fill", n => n._class === "current" ? "var(--g-text-hi)" : "var(--g-text)")
      .style("font-weight", n => n._class === "current" ? "600" : "400");

    simulation = d3.forceSimulation(simNodes)
      .force("link",    d3.forceLink(simEdges).distance(55).strength(0.6))
      .force("charge",  d3.forceManyBody().strength(-60))
      .force("center",  d3.forceCenter(W / 2, H / 2))
      .force("collide", d3.forceCollide(10))
      .on("tick", tick);

    // Centre on current node after layout settles
    simulation.on("end", () => centreOnCurrent(simNodes, W, H));
  }

  function tick() {
    edgeSel
      .attr("x1", d => d.source.x)
      .attr("y1", d => d.source.y)
      .attr("x2", d => d.target.x)
      .attr("y2", d => d.target.y);
    nodeSel.attr("transform", d => `translate(${d.x},${d.y})`);
  }

  function centreOnCurrent(nodes, W, H) {
    const cur = nodes.find(n => n.id === CURRENT_PATH);
    if (!cur || cur.x == null) return;
    const dx = W / 2 - cur.x, dy = H / 2 - cur.y;
    svgRoot.select("g.root")
      .transition().duration(400)
      .attr("transform", `translate(${dx},${dy})`);
  }

  // ── Drag ──────────────────────────────────────────────────────────────────
  function dragStart(event, d) {
    if (!event.active) simulation.alphaTarget(0.3).restart();
    d.fx = d.x; d.fy = d.y;
  }
  function dragging(event, d) { d.fx = event.x; d.fy = event.y; }
  function dragEnd(event, d) {
    if (!event.active) simulation.alphaTarget(0);
    d.fx = null; d.fy = null;
  }

  // ── Hover ─────────────────────────────────────────────────────────────────
  function onNodeEnter(event, d) {
    // Highlight connected edges
    edgeSel.classed("highlighted", e =>
      e.source.id === d.id || e.target.id === d.id);

    // Tooltip
    tooltip.textContent = d.label;
    tooltip.classList.add("visible");
    tooltip.removeAttribute("aria-hidden");
    positionTooltip(event);
  }
  function onNodeLeave() {
    edgeSel.classed("highlighted", false);
    tooltip.classList.remove("visible");
    tooltip.setAttribute("aria-hidden", "true");
  }
  function positionTooltip(event) {
    const rect = svgEl.getBoundingClientRect();
    const asideRect = aside.getBoundingClientRect();
    const x = event.clientX - asideRect.left + 8;
    const y = event.clientY - asideRect.top  - 28;
    tooltip.style.left = Math.min(x, asideRect.width - 190) + "px";
    tooltip.style.top  = Math.max(y, 4) + "px";
  }

  // ── Click → navigate ─────────────────────────────────────────────────────
  function onNodeClick(event, d) {
    if (event.defaultPrevented) return; // drag
    if (d.path && !d.missing) window.location.href = d.path;
  }

  // ── Search / filter ───────────────────────────────────────────────────────
  let filterTimer;
  searchEl.addEventListener("input", () => {
    clearTimeout(filterTimer);
    filterTimer = setTimeout(applyFilter, 120);
  });

  function applyFilter() {
    const q = searchEl.value.trim().toLowerCase();
    if (!q) {
      buildGraph(allNodes, allEdges);
      emptyEl.hidden = true;
      return;
    }
    // Always keep current node and its neighbours
    const matched = new Set(
      allNodes
        .filter(n => n.label.toLowerCase().includes(q) || n.id.toLowerCase().includes(q))
        .map(n => n.id)
    );
    // Pull in current page regardless
    matched.add(CURRENT_PATH);

    const filteredNodes = allNodes.filter(n => matched.has(n.id));
    const filteredEdges = allEdges.filter(e => matched.has(e.from) && matched.has(e.to));

    emptyEl.hidden = filteredNodes.length > 1;
    buildGraph(filteredNodes, filteredEdges);
  }

  // ── Utility ───────────────────────────────────────────────────────────────
  function truncate(str, max) {
    return str && str.length > max ? str.slice(0, max - 1) + "…" : str;
  }

})();
</script>

<!--
  ════════════════════════════════════════════════════════════════════════════
  Jekyll integration notes
  ════════════════════════════════════════════════════════════════════════════

  1.  Copy this file to  _includes/graph-aside.html

  2.  In your layout (e.g. _layouts/default.html), expose the current page
      path so the JS knows which node to highlight:

        <script>const PAGE_PATH = "{{ page.path | remove_first: '_docs/' | remove: '.md' }}";</script>
        {% include graph-aside.html %}

  3.  Run  ruby build_graph.rb  before  jekyll build  (or add it to your
      Makefile / GitHub Actions workflow):

        - name: Build graph
          run: ruby build_graph.rb

        - name: Build site
          run: bundle exec jekyll build

  4.  CSS width / height are controlled by two custom properties on .doc-graph.
      Override in your site stylesheet:

        .doc-graph {
          --g-width:  280px;
          --g-height: 360px;
        }

  5.  If your docs live at a different permalink, adjust the `url` field
      produced by build_graph.rb and the PAGE_PATH Liquid expression above.

  ════════════════════════════════════════════════════════════════════════════
-->
