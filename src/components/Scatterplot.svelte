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
    export let showAnnotations = true;

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

      // Draw annotations only if showAnnotations is true
      if (showAnnotations) {
        annotations.forEach(annotation => {
          ctx.globalAlpha = 1;

          // Draw annotation circle
          ctx.beginPath();
          ctx.arc(annotation.x, annotation.y, annotation.radius, 0, Math.PI * 2);
          ctx.strokeStyle = "red";
          ctx.lineWidth = 2;
          ctx.setLineDash([5, 5]);
          ctx.stroke();

          // Set text properties for measuring
          const maxWidth = 150; // Maximum width for the label
          const lineHeight = 12; // Line height for wrapped text
          ctx.font = "11px Arial";
          
          // Calculate label bounding box
          const labelX = annotation.x + annotation.label_x;
          const labelY = annotation.y + annotation.label_y;
          
          // Determine text bounds by measuring and wrapping text
          let textWidth = 0;
          let textHeight = 0;
          let lines = [];
          let currentLine = "";
          
          // Process text wrapping to calculate height and width
          if (annotation.label) {
            const words = annotation.label.split(" ");
            let line = "";
            
            words.forEach((word, index) => {
              const testLine = line + word + " ";
              const testWidth = ctx.measureText(testLine).width;
              
              if (testWidth > maxWidth && index > 0) {
                lines.push(line);
                textWidth = Math.max(textWidth, ctx.measureText(line).width);
                line = word + " ";
              } else {
                line = testLine;
              }
            });
            
            lines.push(line);
            textWidth = Math.max(textWidth, ctx.measureText(line).width);
            textHeight = lineHeight * lines.length;
          }
          
          // Label rectangle bounds
          const rectX = labelX;
          const rectY = labelY - lineHeight; // Offset to account for text baseline
          const rectWidth = textWidth;
          const rectHeight = textHeight;
          
          // Find closest point on the rectangle to the circle center
          // First determine which side of the rectangle is closest to the circle center
          let closestX, closestY;
          
          // Calculate x-coordinate of closest point
          if (annotation.x < rectX) {
            closestX = rectX;
          } else if (annotation.x > rectX + rectWidth) {
            closestX = rectX + rectWidth;
          } else {
            closestX = annotation.x;
          }
          
          // Calculate y-coordinate of closest point
          if (annotation.y < rectY) {
            closestY = rectY;
          } else if (annotation.y > rectY + rectHeight) {
            closestY = rectY + rectHeight;
          } else {
            closestY = annotation.y;
          }
          
          // Calculate the starting point of the line on the circle's outline
          const angle = Math.atan2(closestY - annotation.y, closestX - annotation.x);
          const startX = annotation.x + annotation.radius * Math.cos(angle);
          const startY = annotation.y + annotation.radius * Math.sin(angle);

          // Draw line connecting the circle outline to the closest point on the rectangle
          ctx.setLineDash([]); // Solid line for the connector
          ctx.beginPath();
          ctx.moveTo(startX, startY);
          ctx.lineTo(closestX, closestY);
          ctx.strokeStyle = "red";
          ctx.lineWidth = 1;
          ctx.stroke();

          // Draw label with wrapping
          if (annotation.label) {
            ctx.font = "11px Arial";
            ctx.fillStyle = "red";

            // Draw each line of text
            lines.forEach((line, index) => {
              ctx.fillText(line, labelX, labelY + index * lineHeight);
            });
          }
        });
      }

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
      
      // Calculate scaling ratio between internal canvas dimensions and displayed dimensions
      const scaleX = containerWidth / rect.width;
      const scaleY = containerHeight / rect.height;
      
      // Adjust mouse coordinates based on the scaling ratio
      const mouseX = (event.clientX - rect.left) * scaleX - margin.left;
      const mouseY = (event.clientY - rect.top) * scaleY - margin.top;
    
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
    
    $: if (ctx && data.length) {
        draw();
    }
    
    $: if (ctx) {
        opacity, selectedValues, searchQuery, showAnnotations, domainColumn; // Watch these props
        if (data.length) draw(); // Redraw when any of these change
    }
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