<script>
  import { onMount } from "svelte";
  import { select } from "d3-selection";
  import { scaleTime, scaleLinear } from "d3-scale";
  import { axisBottom, axisLeft } from "d3-axis";
  import { line, curveBasis } from "d3-shape";
  import { timeParse, timeFormat } from "d3-time-format";
  import { extent } from "d3-array";
  import { format } from "d3-format";

  let width = 800;
  let height = 400;
  let margin = { top: 20, right: 30, bottom: 40, left: 70 };
  let allData = [];
  let data = [];
  let selectedArtist = "Coi Leray";

  const parseDate = timeParse("%Y-%m-%d");
  const formatDate = timeFormat("%m/%d");

  onMount(async () => {
    try {
      const response = await fetch("/CLEANED_featured_Spotify_artist_info.csv");
      const text = await response.text();
      allData = parseCSV(text);
      data = filterData(selectedArtist);
      console.log("Processed Data:", data);
      createChart();
    } catch (error) {
      console.error("Error loading or processing data:", error);
    }
  });

  function parseCSV(text) {
    const rows = text.split("\n").slice(1);
    return rows
      .map((row) => {
        const columns = row.split(",");
        if (columns.length > 3) {
          const name = columns[2].trim();
          const dateStr = columns[0].trim();
          let listeners = columns[3].trim();

          listeners = +listeners.replace(/[^0-9]/g, "");

          const date = parseDate(dateStr);

          return {
            name: name.toLowerCase(),
            date: date,
            listeners: listeners,
          };
        }
        return null;
      })
      .filter((d) => d !== null);
  }

  function filterData(artist) {
    return allData.filter((d) => d.name === artist.toLowerCase());
  }

  function createChart() {
    const svg = select("#chart")
      .attr("width", width)
      .attr("height", height)
      .html("")
      .append("g")
      .attr("transform", `translate(${margin.left},${margin.top})`);

    const x = scaleTime()
      .domain(extent(data, (d) => d.date))
      .range([0, width - margin.left - margin.right]);

    const y = scaleLinear()
      .domain([
        Math.floor(Math.min(...data.map((d) => d.listeners)) / 1e6) * 1e6,
        Math.ceil(Math.max(...data.map((d) => d.listeners)) / 1e6) * 1e6,
      ])
      .range([height - margin.top - margin.bottom, 0]);

    svg.append("g").call(axisLeft(y).tickFormat(format("~s")));

    svg
      .append("g")
      .attr("transform", `translate(0,${height - margin.top - margin.bottom})`)
      .call(axisBottom(x).tickFormat(formatDate));

    svg.selectAll(".domain").style("stroke", "white");
    svg.selectAll(".tick line").style("stroke", "white");

    svg
      .append("rect")
      .attr("x", 0)
      .attr("y", 0)
      .attr("height", height - margin.top - margin.bottom)
      .attr("width", width - margin.left - margin.right)
      .style("stroke", "white")
      .style("fill", "none")
      .style("stroke-width", "2px");

    const lineGenerator = line()
      .x((d) => x(d.date))
      .y((d) => y(d.listeners))
      .curve(curveBasis);

    svg
      .append("path")
      .datum(data)
      .attr("fill", "none")
      .attr("stroke", "limegreen")
      .attr("stroke-width", 1.5)
      .attr("d", lineGenerator);

    const horizontalLine = svg
      .append("line")
      .attr("class", "hover-line")
      .attr("x1", 0)
      .attr("x2", width - margin.left - margin.right)
      .attr("y1", 0)
      .attr("y2", 0)
      .attr("stroke", "white")
      .attr("stroke-width", 1)
      .attr("opacity", 0);

    const textLabel = svg
      .append("text")
      .attr("class", "hover-text")
      .attr("x", width - margin.left - margin.right - 10)
      .attr("y", 0)
      .attr("dy", "-0.5em")
      .attr("text-anchor", "end")
      .attr("fill", "white")
      .attr("opacity", 0);

    svg
      .selectAll(".dot")
      .data(data)
      .enter()
      .append("circle")
      .attr("class", "dot")
      .attr("cx", function (d) {
        return x(d.date);
      })
      .attr("cy", function (d) {
        return y(d.listeners);
      })
      .attr("r", 5)
      .style("fill", "white")
      .on("mouseover", function (event, d) {
        select(this).attr("r", 7);
        horizontalLine
          .attr("y1", y(d.listeners))
          .attr("y2", y(d.listeners))
          .attr("opacity", 1);
        textLabel
          .attr("y", y(d.listeners))
          .attr("x", x(d.date) + 10)
          .attr("opacity", 1)
          .text(format("~s")(d.listeners));
      })
      .on("mouseout", function (d) {
        select(this).attr("r", 5);
        horizontalLine.attr("opacity", 0);
        textLabel.attr("opacity", 0);
      });

    svg
      .append("text")
      .attr("transform", "rotate(-90)")
      .attr("y", 0 - margin.left)
      .attr("x", 0 - height / 2 + 35)
      .attr("dy", "1em")
      .style("text-anchor", "middle")
      .style("fill", "white")
      .text("Monthly Listeners (In Millions)");

    svg
      .append("text")
      .attr(
        "transform",
        `translate(${(width - margin.left - margin.right) / 2} ,${height - margin.top + margin.bottom - 10})`
      )
      .style("text-anchor", "middle")
      .style("fill", "white")
      .text("Date");
  }

  function handleSelection(event) {
    selectedArtist = event.target.value;
    data = filterData(selectedArtist);
    createChart();
  }
</script>

<h1 style="text-align: center; color: white;">
  Monthly Listeners for Industry Plants
</h1>
<div style="display: flex; justify-content: space-between;">
  <svg id="chart"></svg>
  <div
    style="display: flex; align-items: center; justify-content: center; flex-grow: 1;"
  >
    <select id="options" on:change={handleSelection}>
      <option value="Coi Leray">Coi Leray</option>
      <option value="Ice Spice">Ice Spice</option>
      <option value="The Kid Laroi">The Kid Laroi</option>
      <option value="Dominic Fike">Dominic Fike</option>
      <option value="PinkPantheress">Pink Pantheress</option>
    </select>
  </div>
</div>

<style>
  svg {
    display: block;
    margin: auto;
  }

  #options {
    margin-left: 20px;
    font-size: 16px;
  }

  .hover-line {
    stroke: white;
    stroke-width: 1;
    opacity: 0;
  }

  .hover-text {
    fill: rgb(255, 255, 255);
    opacity: 0;
  }
</style>
