# Going Dark
### Where do ships disappear from tracking? <br> 1 year of data from 2025-2026

```js
const gaps = FileAttachment("data/global_gaps_clean.json").json();
const land = FileAttachment("data/land-50m.json").json();
```

```js
const landGeo = topojson.feature(land, land.objects.land);
```

```js
const height = width / 1.8;

// Natural Earth projection
const projection = d3.geoNaturalEarth1()
  .fitSize([width, height], {type: "Sphere"});

const path = d3.geoPath(projection);
```

```js
// Color encoding for ship types
const classColor = {
  fishing:        "#f86666",
  cargo:          "#f8d66f",
  passenger:      "#55caf1",
  carrier:        "#bd80fa",
  seismic_vessel: "#78f5a2",
  other:          "#b9b9b9"
};

function bucket(c) {
  return classColor[c] ? c : "other";
}
```

```js
// vessel class selection
const classInput = Inputs.select(
  ["fishing", "cargo", "passenger", "carrier", "seismic_vessel", "other"],
  {label: "Vessel class", value: "fishing"}
);
const selectedClass = (stepIndex >= 0)
  ? tourSteps[stepIndex].vesselClass
  : Generators.input(classInput);

display(classInput);
```


```js
const showLines = view(
  Inputs.toggle({label: "Show lines to eventual destination on hover?", value: true})
);
```

```js
const tourSteps = [
  {
    label: "Fishing · Argentine Shelf",
    vesselClass: "fishing",
    center: [-60, -45], k: 5,
    text: [
      "International regulations generally forbid fishing within 200 miles of the coast. This region is called an Economic Exclusion Zone (EEZ)",
      "'Mile 201' is then a common location for ships to go dark",
      "","",
      "The high seas edge of Argentina's economic zone is a major hotspot. Squid jiggers go dark to mask fishing at this 200 mile boundary."
    ],
    source: "https://zenodo.org/records/4893397",
    sourceLabel: "Oceana (2021)"
  },
  {
    label: "Fishing · Galápagos & Peru",
    vesselClass: "fishing",
    center: [-88, -8], k: 4,
    text: [
      "The same distant water squid fleet works the Pacific side, skirting the EEZ around Ecuador's and down past Peru."
    ],
    source: "https://maritime-executive.com/editorials/south-america-plans-regional-response-to-china-s-squid-fleet",
    sourceLabel: "Maritime Executive (2021)"
  },
  {
    label: "Cargo · Africa–Middle East corridor",
    vesselClass: "cargo",
    center: [50, 12], k: 2.5,
    text: [
      "Cargo and tanker gaps tell a different story: sanctions evasion.",
      "A near continuous line of dark fleet disabling traces the East African coast, the Red Sea and the Gulf.",
      "These ships are likely tankers going dark to move sanctioned oil, then reappearing downstream."
    ],
    source: "https://www.nber.org/system/files/working_papers/w33486/w33486.pdf",
    sourceLabel: "NBER (2025)"
  }
];
```
```js
// tour step
const stepIndex = Mutable(0);
const setStep = (i) => stepIndex.value = i;
```

```js
const i = stepIndex;

let panel;

if (i < 0) {
  panel = html`<div>
    Tour complete. Explore the map freely using the dropdown.
  </div>`;
} else {
  const step = tourSteps[i];
  const isLast = i >= tourSteps.length - 1;

  panel = html`<div>
    <strong>${step.label}</strong> (${i + 1} / ${tourSteps.length})
    ${step.text.map(line => html`<p>${line}</p>`)}
    <a href=${step.source} target="_blank">Source: ${step.sourceLabel}</a>
    <br>
    <button onclick=${() => setStep(isLast ? -1 : i + 1)}>
      ${isLast ? "Explore the map" : "Next hotspot"}
    </button>
  </div>`;
}

display(panel);
```



```js
// Filter for a valid position/dropdown selection only
const filtered = gaps.filter(
  d => bucket(d.vessel_class) === selectedClass
       && d.off_lat != null && d.off_lon != null
);
```


```js
// tooltips
const tooltip = d3.create("div")
    .style("position", "fixed")
    .style("pointer-events", "none")      // so it never blocks the mouse
    .style("background", "rgba(10,14,26,0.95)")
    .style("border", "1px solid #555")
    .style("border-radius", "4px")
    .style("padding", "6px 8px")
    .style("font", "12px sans-serif")
    .style("color", "#eee")
    .style("opacity", 0)                  // hidden initially
    .style("z-index", 1000);

display(tooltip.node());
```








```js
// Build the SVG
const svg = d3.create("svg")
    .attr("viewBox", [0, 0, width, height])
    .attr("width", width)
    .attr("style", "background: #0a0e1a; max-width: 100%; height: auto;");



//Zoom/pan group
const g = svg.append("g");
// map
g.append("path")
    .attr("d", path({type: "Sphere"}))
    .attr("fill", "#0a1a3a")
    .attr("stroke", "#000000");

g.append("path")
    .attr("d", path(landGeo))
    .attr("fill", "#525252")
    .attr("stroke", "#00030a")
    .attr("stroke-width", 0.5);



const points = filtered.map(d => {
  const off = projection([d.off_lon, d.off_lat]);
  const on  = (d.on_lon != null && d.on_lat != null)
              ? projection([d.on_lon, d.on_lat])
              : null;
  return {data: d, off, on};
}).filter(p => p.off);




//lines
const lineLayer = g.append("g")
    .attr("fill", "none")
    .attr("stroke", "#ffffff")
    .attr("stroke-width", 0.5)
    .attr("stroke-opacity", 0.8);
let currentTransform = d3.zoomIdentity;

// dots
g.append("g")
  .selectAll("circle")
  .data(points)
  .join("circle")
     .attr("transform", d => `translate(${d.off[0]},${d.off[1]})`)
    .attr("r", 1.5)
    .attr("fill", d => classColor[bucket(d.data.vessel_class)])
    .attr("fill-opacity", 0.5)
    .on("mouseenter", (event, d) => {
      const e = d.data;
      tooltip
        .style("opacity", 1)
        .html(`
          ${e.vessel_name ?? "Unknown vessel"}
          Class: ${e.vessel_class ?? "—"}
          Flag: ${e.vessel_flag ?? "—"}
          Dark for: ${e.gap_hours ? Math.round(e.gap_hours) + " h" : "—"}
        `);
    })
    .on("mousemove", (event) => {
      tooltip
        .style("left", (event.clientX + 12) + "px")
        .style("top", (event.clientY + 12) + "px");
    })
    .on("mouseleave", () => {
      tooltip.style("opacity", 0);
    });

const zoom = d3.zoom()
    .scaleExtent([1, 40])                 // min 1x max 40x
    .translateExtent([[0, 0], [width, height]])  // dont pan off the map
    .on("zoom", (event) => {
      currentTransform = event.transform;
      g.attr("transform", currentTransform);
    });

svg.call(zoom);

const NEAR_RADIUS = 5;   // pixels away from a dot before drawing a line

svg.on("mousemove", (event) => {
  if (!showLines) return; 
  const [mx, my] = d3.pointer(event);
  const [gx, gy] = currentTransform.invert([mx, my]);
  const near = points.filter(p => {
    if (!p.on) return false;             
    const dx = p.off[0] - gx;
    const dy = p.off[1] - gy;
    return dx * dx + dy * dy < NEAR_RADIUS * NEAR_RADIUS;
  });

  lineLayer.selectAll("line")
    .data(near, p => p.data.id)
    .join("line")
      .attr("x1", p => p.off[0])
      .attr("y1", p => p.off[1])
      .attr("x2", p => p.on[0])
      .attr("y2", p => p.on[1]);
});

svg.on("mouseleave", () => lineLayer.selectAll("line").remove());

if (stepIndex >= 0) { //actually step through tour
  const step = tourSteps[stepIndex];
  const [cx, cy] = projection(step.center);  
  const t = d3.zoomIdentity
      .translate(width / 2, height / 2)       
      .scale(step.k)                          
      .translate(-cx, -cy);                    
  svg.transition().duration(1200).call(zoom.transform, t);
}

display(svg.node());
```

---

[View the source code on GitHub](https://github.com/Cart3rharris42/going-dark)

---

## Sources

Data from [Global Fishing Watch](https://globalfishingwatch.org), AIS-disabling ("gap") events dataset. 

context drawn from:

- Welch, H. et al. (2022). "Hot spots of unseen fishing vessels." *Science Advances.* https://www.science.org/doi/10.1126/sciadv.abq2109
- Oceana (2021). "Now You See Me, Now You Don't: Vanishing Vessels Along Argentina's Waters." https://zenodo.org/records/4893397
- Park, J. et al. (2020). "Illuminating dark fishing fleets in North Korea." *Science Advances*, via Global Fishing Watch. https://globalfishingwatch.org/success-story/illuminating-dark-fishing-in-north-korea/
- Maritime Executive (2021). "South America Plans Regional Response to China's Squid Fleet." https://maritime-executive.com/editorials/south-america-plans-regional-response-to-china-s-squid-fleet
- National Bureau of Economic Research (2025). "The (Un)Intended Consequences of Oil Sanctions and Dark Shipping." https://www.nber.org/system/files/working_papers/w33486/w33486.pdf
- Global Fishing Watch. "Going Dark: When Vessels Turn Off AIS Broadcasts." https://globalfishingwatch.org/data/going-dark-when-vessels-turn-off-ais-broadcasts/