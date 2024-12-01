---
toc: false
---

<h1>Bluesky graph</h1>

<div class="card">
${vis}
</div>

<style>
  /* Optional: Add a subtle hover effect to the cards */
  .card:hover {
    transform: translateY(-5px);
    transition: transform 0.3s;
    box-shadow: 0 8px 16px rgba(0, 0, 0, 0.2);
  }
  .card-title {
  word-break: break-word; /* Default for regular breaks */
  overflow-wrap: anywhere; /* Ensures breaking is flexible */
}

.card-title  {
  word-break: keep-all; /* Ensures full word retained */
}
</style>


<div class="container my-4">
  <div class="row row-cols-1 row-cols-sm-2 row-cols-md-3 row-cols-lg-4 g-4">
    ${_.chain(data)
      .sortBy(d => d.followers_count)
      .map(d => html`
        <div class="col">
          <div class="card h-100 shadow-sm">
            <div class="d-flex justify-content-center mt-3">
              <img src="${d.avatar_url}" class="rounded-circle" alt="${d.handle}" style="width: 100px; height: 100px; object-fit: cover;">
            </div>
            <div class="card-body text-center">
              <h5 class="card-title">@${d.handle}</h5>
              <p class="card-text">
                <span class="badge bg-primary">
                  Followers: ${d.followers_count}
                </span>
              </p>
            </div>
          </div>
        </div>
      `)
      .value()}
  </div>
</div>





```js
display(vis)
```

```js
const vis = CustomForceGraph(graph, {
  nodeId: (d) => d.id,
  nodeTitle: (d) => `${d.id}\nFollowers: ${d.followers_count}`,
  nodeRadius: (d) => Math.sqrt(d.followers_count) / 50 + 5, // Adjust scaling as needed
  nodeStrength: -30,
  linkStrength: 0.1,
  width: width,
  height: width,
  nodeImage: (d) => d.avatar_url // Use avatar images
})
```


```js

let data = await d3.json(
  "https://raw.githubusercontent.com/elaval/BskyGraph/refs/heads/main/user_data_with_connections.json"
);

data = data.filter((d) => !d.handle.match(/bsky.app/));

const graph = ({
  nodes: data.map((d) => ({
    id: d.handle,
    followers_count: d.followers_count,
    avatar_url: d.avatar_url
    // Add any other attributes you need
  })),
  links: data.flatMap(
    (d) =>
      d.connections &&
      d.connections
        .filter((d) => data.map((d) => d.handle).includes(d))
        .map((targetHandle) => ({
          source: d.handle,
          target: targetHandle
        }))
  )
})

```

```js
/*
  Based on https://observablehq.com/@d3/force-directed-graph-component
*/
function CustomForceGraph(
  {
    nodes, // an iterable of node objects (typically [{id}, …])
    links // an iterable of link objects (typically [{source, target}, …])
  },
  {
    nodeId = (d) => d.id, // given d in nodes, returns a unique identifier (string)
    nodeImage,
    nodeGroup, // given d in nodes, returns an (ordinal) value for color
    nodeGroups, // an array of ordinal values representing the node groups
    nodeTitle, // given d in nodes, a title string
    nodeFill = "currentColor", // node stroke fill (if not using a group color encoding)
    nodeStroke = "#fff", // node stroke color
    nodeStrokeWidth = 1.5, // node stroke width, in pixels
    nodeStrokeOpacity = 1, // node stroke opacity
    nodeRadius = 5, // node radius, in pixels
    nodeStrength,
    linkSource = ({ source }) => source, // given d in links, returns a node identifier string
    linkTarget = ({ target }) => target, // given d in links, returns a node identifier string
    linkStroke = "#999", // link stroke color
    linkStrokeOpacity = 0.6, // link stroke opacity
    linkStrokeWidth = 1.5, // given d in links, returns a stroke width in pixels
    linkStrokeLinecap = "round", // link stroke linecap
    linkStrength,
    colors = d3.schemeTableau10, // an array of color strings, for the node groups
    width = 640, // outer width, in pixels
    height = 400, // outer height, in pixels
    invalidation // when this promise resolves, stop the simulation
  } = {}
) {
  // Compute values.
  const N = d3.map(nodes, nodeId).map(intern);
  const R = typeof nodeRadius !== "function" ? null : d3.map(nodes, nodeRadius);
  const LS = d3.map(links, linkSource).map(intern);
  const LT = d3.map(links, linkTarget).map(intern);
  if (nodeTitle === undefined) nodeTitle = (_, i) => N[i];
  const T = nodeTitle == null ? null : d3.map(nodes, nodeTitle);
  const G = nodeGroup == null ? null : d3.map(nodes, nodeGroup).map(intern);
  const W =
    typeof linkStrokeWidth !== "function"
      ? null
      : d3.map(links, linkStrokeWidth);
  const L = typeof linkStroke !== "function" ? null : d3.map(links, linkStroke);
  const I = nodeImage == null ? null : d3.map(nodes, nodeImage);

  // Replace the input nodes and links with mutable objects for the simulation.
  nodes = d3.map(nodes, (_, i) => ({ id: N[i], r: R[i] }));
  links = d3.map(links, (_, i) => ({ source: LS[i], target: LT[i] }));

  // Compute default domains.
  if (G && nodeGroups === undefined) nodeGroups = d3.sort(G);

  // Construct the scales.
  const color = nodeGroup == null ? null : d3.scaleOrdinal(nodeGroups, colors);

  // Construct the forces.
  const forceNode = d3.forceManyBody();
  const forceLink = d3.forceLink(links).id(({ index: i }) => N[i]);
  if (nodeStrength !== undefined) forceNode.strength(nodeStrength);
  if (linkStrength !== undefined) forceLink.strength(linkStrength);

  const simulation = d3
    .forceSimulation(nodes)
    .force("link", forceLink)
    .force("charge", forceNode)
    .force("center", d3.forceCenter())
    .on("tick", ticked);

  const svg = d3
    .create("svg")
    .attr("width", width)
    .attr("height", height)
    .attr("viewBox", [-width / 2, -height / 2, width, height])
    .attr("style", "max-width: 100%; height: auto; height: intrinsic;");

  const link = svg
    .append("g")
    .attr("stroke", typeof linkStroke !== "function" ? linkStroke : null)
    .attr("stroke-opacity", linkStrokeOpacity)
    .attr(
      "stroke-width",
      typeof linkStrokeWidth !== "function" ? linkStrokeWidth : null
    )
    .attr("stroke-linecap", linkStrokeLinecap)
    .selectAll("line")
    .data(links)
    .join("line");

  const node = svg
    .append("g")
    .attr("fill", nodeFill)
    .attr("stroke", nodeStroke)
    .attr("stroke-opacity", nodeStrokeOpacity)
    .attr("stroke-width", nodeStrokeWidth)
    .selectAll("g.node")
    .data(nodes)
    .join("g")
    .attr("class", "node")
    .call(drag(simulation));

  // Append a circle to each node group
  node
    .append("circle")
    .attr("r", nodeRadius)
    .attr("fill", nodeFill)
    .attr("stroke", nodeStroke)
    .attr("stroke-width", nodeStrokeWidth);

  // Append an image to each node group, if nodeImage is provided
  if (I) {
    node
      .append("image")
      .attr("href", ({ index: i }) => I[i]) // 'href' instead of 'xlink:href' for SVG2
      .attr("width", ({ index: i }) => R[i] * 2)
      .attr("height", ({ index: i }) => R[i] * 2)
      .attr("x", ({ index: i }) => -R[i])
      .attr("y", ({ index: i }) => -R[i])
      .attr(
        "clip-path",
        ({ index: i }) => `circle(${R[i]}px at ${R[i]}px ${R[i]}px)`
      ); // Optional: Clip to circle
  }

  if (W) link.attr("stroke-width", ({ index: i }) => W[i]);
  if (L) link.attr("stroke", ({ index: i }) => L[i]);
  if (G) node.attr("fill", ({ index: i }) => color(G[i]));
  if (R) node.select("circle").attr("r", ({ index: i }) => R[i]);
  if (T) node.append("title").text(({ index: i }) => T[i]);
  if (invalidation != null) invalidation.then(() => simulation.stop());

  function intern(value) {
    return value !== null && typeof value === "object"
      ? value.valueOf()
      : value;
  }

  function ticked() {
    link
      .attr("x1", (d) => d.source.x)
      .attr("y1", (d) => d.source.y)
      .attr("x2", (d) => d.target.x)
      .attr("y2", (d) => d.target.y);

    node.attr("transform", (d) => `translate(${d.x},${d.y})`);
  }

  function drag(simulation) {
    function dragstarted(event) {
      if (!event.active) simulation.alphaTarget(0.3).restart();
      event.subject.fx = event.subject.x;
      event.subject.fy = event.subject.y;
    }

    function dragged(event) {
      event.subject.fx = event.x;
      event.subject.fy = event.y;
    }

    function dragended(event) {
      if (!event.active) simulation.alphaTarget(0);
      event.subject.fx = null;
      event.subject.fy = null;
    }

    return d3
      .drag()
      .on("start", dragstarted)
      .on("drag", dragged)
      .on("end", dragended);
  }

  return Object.assign(svg.node(), { scales: { color } });
}
```
