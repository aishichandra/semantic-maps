<script>
    import { onMount } from 'svelte';
    import { scaleLinear, scaleOrdinal } from 'd3-scale';
    import { max, min } from 'd3-array';
    import DetailCard from './DetailCard.svelte';
    import { schemeCategory10 } from 'd3-scale-chromatic';
    
    export let data = [];
    export let domainColumn = "";
    export let opacity = 1;
    export let selectedValues = new Set();
    export let searchQuery = ""; 

    let annotations = [
        { x: 500, y: 400, radius: 30, label: "Border Belt Independent and North Carolina Coastal Federation have both published about GenX — an industrial chemical", label_x: 100, label_y: -90 },
        { x: 720, y: 170, radius: 50, label: "Only Carolina Peacemaker seems to be publishing about personal health", label_x: -70, label_y: 80 }
    ];

    let canvas;
    let ctx;
    
    let containerWidth = 800;
    let containerHeight = 600;
    
    const margin = { top: 20, right: 20, bottom: 20, left: 20 };
    const radius = 6;
    
    let hoveredData = null;
    let lastHoveredData = null;
    let zoomScale = 1;
    let zoomCenter = { x: 0, y: 0 };
    
    $: innerWidth = containerWidth - margin.left - margin.right;
    $: innerHeight = containerHeight - margin.top - margin.bottom;
    
    $: xScale = scaleLinear()
      .domain([min(data, d => d.x), max(data, d => d.x)])
      .range([0, innerWidth]);
    
    $: yScale = scaleLinear()
      .domain([min(data, d => d.y), max(data, d => d.y)])
      .range([innerHeight, 0]);
    
    const colorScale = scaleOrdinal()
      .domain([...new Set(data.map(d => d[domainColumn]))])
      .range(schemeCategory10);
    
    function draw() {
      ctx.clearRect(0, 0, containerWidth, containerHeight);
      ctx.save();
      ctx.translate(zoomCenter.x, zoomCenter.y);
      ctx.scale(zoomScale, zoomScale);
      ctx.translate(-zoomCenter.x, -zoomCenter.y);

      // Draw data points
      data.forEach(d => {
        const isHighlighted = searchQuery && d.title && d.title.toLowerCase().includes(searchQuery.toLowerCase());
        const isSelected = selectedValues.has(d[domainColumn]);

        ctx.beginPath();
        ctx.arc(margin.left + xScale(d.x), margin.top + yScale(d.y), radius, 0, Math.PI * 2);
        ctx.fillStyle = colorScale(d[domainColumn]);
        ctx.globalAlpha = isHighlighted || isSelected ? 1 : opacity * 0.4;
        ctx.fill();
      });

      // Draw annotations
      annotations.forEach(annotation => {
        ctx.globalAlpha = 1;

        // Draw annotation circle
        ctx.beginPath();
        ctx.arc(annotation.x, annotation.y, annotation.radius, 0, Math.PI * 2);
        ctx.strokeStyle = "red";
        ctx.lineWidth = 2;
        ctx.setLineDash([5, 5]);
        ctx.stroke();

        // Calculate the starting point of the line on the circle's outline
        const labelX = annotation.x + annotation.label_x;
        const labelY = annotation.y + annotation.label_y;
        const angle = Math.atan2(labelY - annotation.y, labelX - annotation.x); // Angle between circle center and label
        const startX = annotation.x + annotation.radius * Math.cos(angle); // X-coordinate on the circle's outline
        const startY = annotation.y + annotation.radius * Math.sin(angle); // Y-coordinate on the circle's outline

        // Draw line connecting the circle outline to the label
        ctx.setLineDash([]); // Solid line for the connector
        ctx.beginPath();
        ctx.moveTo(startX, startY); // Start at the circle's outline
        ctx.lineTo(labelX, labelY); // Draw to the label position
        ctx.strokeStyle = "red";
        ctx.lineWidth = 1;
        ctx.stroke();

        // Draw label with wrapping
        if (annotation.label) {
          const maxWidth = 150; // Maximum width for the label
          const lineHeight = 12; // Line height for wrapped text

          ctx.font = "11px Arial";
          ctx.fillStyle = "red";

          // Split the label into multiple lines if necessary
          const words = annotation.label.split(" ");
          let line = "";
          let y = labelY;

          words.forEach((word, index) => {
            const testLine = line + word + " ";
            const testWidth = ctx.measureText(testLine).width;

            if (testWidth > maxWidth && index > 0) {
              ctx.fillText(line, labelX, y);
              line = word + " ";
              y += lineHeight;
            } else {
              line = testLine;
            }
          });

          // Draw the remaining text
          ctx.fillText(line, labelX, y);
        }
      });

      ctx.setLineDash([]); // Reset line dash

      if (hoveredData) {
        ctx.beginPath();
        ctx.arc(margin.left + xScale(hoveredData.x), margin.top + yScale(hoveredData.y), radius, 0, Math.PI * 2);
        ctx.fillStyle = colorScale(hoveredData[domainColumn]);
        ctx.globalAlpha = 1;
        ctx.fill();
        ctx.strokeStyle = 'black';
        ctx.lineWidth = 2;
        ctx.stroke();
      }

      ctx.restore();
    }
    
    function handleMouseMove(event) {
      const rect = canvas.getBoundingClientRect();
      const mouseX = event.clientX - rect.left - margin.left;
      const mouseY = event.clientY - rect.top - margin.top;
    
      const adjustedX = (mouseX - zoomCenter.x) / zoomScale + zoomCenter.x;
      const adjustedY = (mouseY - zoomCenter.y) / zoomScale + zoomCenter.y;
    
      const foundData = data.find(d => {
        const dx = xScale(d.x) - adjustedX;
        const dy = yScale(d.y) - adjustedY;
        return Math.sqrt(dx * dx + dy * dy) < radius + 3;
      });
    
      if (foundData) {
        hoveredData = foundData;
        lastHoveredData = foundData;
      } else {
        hoveredData = lastHoveredData;
      }
    
      draw();
    }
    
    function handleMouseLeave() {
      hoveredData = lastHoveredData;
      draw();
    }
    
    onMount(() => {
      ctx = canvas.getContext('2d');
      draw();
    });
    
    $: if (ctx && data.length) draw();
</script>

<div class="chart-container">
    <canvas
      bind:this={canvas}
      width={containerWidth}
      height={containerHeight}
      on:mousemove={handleMouseMove}
      on:mouseleave={handleMouseLeave}
    ></canvas>
    <DetailCard {hoveredData} {data} {domainColumn} {colorScale} />
</div>

<style>
    .chart-container {
      position: relative;
      display: flex;
      justify-content: center;
      align-items: center;
      flex-direction: column;
      width: 100%;
      height: 100%;
    }
  
    canvas {
      cursor: crosshair;
      border-radius: 8px;
      background-color: white;
      max-width: 100%;
      height: auto;
    }

    .filter-panel {
      display: flex;
      flex-direction: column;
      gap: 1.5rem; /* Increase spacing between elements */
      padding: 1.5rem; /* Add more padding for better readability */
      border-radius: 10px; /* Rounded corners */
      background-color: #f9f9f9; /* Softer background color */
      box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1); /* Add a subtle shadow */
      width: 300px; /* Slightly wider panel for better layout */
    }

    .filter-panel label {
      font-weight: bold;
      font-size: 1rem; /* Increase font size for labels */
      color: #333; /* Darker text for better contrast */
    }

    .filter-panel input,
    .filter-panel select {
      padding: 0.5rem;
      border: 1px solid #ddd;
      border-radius: 5px;
      font-size: 0.9rem; /* Consistent font size for inputs */
      width: 100%;
      box-sizing: border-box;
    }

    .filter-panel input:focus,
    .filter-panel select:focus {
      outline: none;
      border-color: #4c8bf5;
      box-shadow: 0 0 4px rgba(76, 139, 245, 0.5);
    }

    .filter-panel .multi-select {
      height: auto;
      max-height: 150px; /* Limit height for better readability */
      overflow-y: auto; /* Add scroll for long lists */
    }

    .filter-panel .section {
      border-top: 1px solid #ddd; /* Add a separator between sections */
      padding-top: 1rem;
      margin-top: 1rem;
    }

    .filter-panel .section:first-child {
      border-top: none; /* Remove separator for the first section */
      padding-top: 0;
      margin-top: 0;
    }
</style>