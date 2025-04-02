<script>
  import { onMount } from 'svelte';
  import Papa from 'papaparse';
  import Scatterplot from './components/Scatterplot.svelte';

  let data = [], columns = [], domainColumn = "org", uniqueValues = [];
  let selectedValues = new Set(); 
  let opacity = 0.5, startDate = null, endDate = null;
  let filteredData = [], allDates = [];
  let startDateIndex = 0;
  let endDateIndex = 0;
  let isPlaying = false;
  let playInterval = null;
  let searchQuery = "";
  let showAnnotations = true; // New state variable for annotation visibility

  $: filteredData = data.filter(d =>
    (!startDate || d.date >= startDate) &&
    (!endDate || d.date <= endDate)
  );

  $: startPercent = allDates.length > 1 ? (startDateIndex / (allDates.length - 1)) * 100 : 0;
  $: endPercent = allDates.length > 1 ? (endDateIndex / (allDates.length - 1)) * 100 : 100;

  onMount(async () => {
    const response = await fetch('/semantic-maps/data.csv');
    const csvText = await response.text();
    parseCSV(csvText);

    return () => {
      if (playInterval) clearInterval(playInterval);
    };
  });

  function parseCSV(csvText) {
    const result = Papa.parse(csvText, { header: true });
    data = result.data.filter(d => d.x && d.y && d.date)
      .map(d => ({ ...d, x: +d.x, y: +d.y, date: new Date(d.date) }));

    columns = result.meta.fields || [];
    allDates = [...new Set(data.map(d => d.date.getTime()))]
      .sort((a, b) => a - b)
      .map(t => new Date(t));

    startDate = allDates[0];
    endDate = allDates[allDates.length - 1];
    startDateIndex = 0;
    endDateIndex = allDates.length - 1;

    if (domainColumn) {
      uniqueValues = [...new Set(data.map(d => d[domainColumn]).filter(Boolean))];
    }
  }

  function handleDomainChange(event) {
    domainColumn = event.target.value;
    uniqueValues = domainColumn ? [...new Set(data.map(d => d[domainColumn]).filter(Boolean))] : [];
    selectedValues = new Set();
    showAnnotations = false;
  }

  function handleSelectionChange(event) {
    selectedValues = new Set([...event.target.selectedOptions].map(o => o.value));
    showAnnotations = false;
  }

  function handleDateChange(e, type) {
    const newDate = e.target.value ? new Date(e.target.value) : null;
    if (type === 'start') startDate = newDate;
    else endDate = newDate;
    showAnnotations = false;
  }

  function formatDateInput(date) {
    return date ? date.toISOString().split('T')[0] : '';
  }

  function handleFileUpload(event) {
    const file = event.target.files[0];
    if (file) {
      const reader = new FileReader();
      showAnnotations = false;
      reader.onload = (e) => parseCSV(e.target.result);
      reader.readAsText(file);
    }
  }

  function shiftDateRange(days) {
    const rangeDuration = endDateIndex - startDateIndex;
    let newStartIndex = startDateIndex + days;
    let newEndIndex = endDateIndex + days;

    if (newStartIndex < 0) {
      newStartIndex = 0;
      newEndIndex = rangeDuration;
    }

    if (newEndIndex >= allDates.length) {
      newEndIndex = allDates.length - 1;
      newStartIndex = Math.max(0, newEndIndex - rangeDuration);
    }

    startDateIndex = newStartIndex;
    endDateIndex = newEndIndex;
    startDate = allDates[startDateIndex] || allDates[0];
    endDate = allDates[endDateIndex] || allDates[allDates.length - 1];
    showAnnotations = false;
  }

  function togglePlayPause() {
    if (isPlaying) {
      clearInterval(playInterval);
      isPlaying = false;
    } else {
      isPlaying = true;
      showAnnotations = false;
      playInterval = setInterval(() => {
        if (endDateIndex >= allDates.length - 1) {
          clearInterval(playInterval);
          isPlaying = false;
          return;
        }
        shiftDateRange(1);
      }, 500);
    }
  }

  function handleSearch(event) {
    searchQuery = event.target.value;
    showAnnotations = false;
  }
  
  function handleOpacityChange() {
    showAnnotations = false;
  }
</script>

<!-- App Layout -->
<div class="container">
  <div class="title-section">
    <h1 class="title">North Carolina Semantic Map [DEMO]</h1>
    <p class="subtitle">This semantic map contains data from three organizations in North Carolina. <a href = "https://peacemakeronline.com/">Carolina Peacemaker</a> and <a href = "https://borderbelt.org">Border Belt</a> are local news organizations and <a href= "https://www.nccoast.org/about-us/">NC Coastal Federation</a> is a nonprofit focused on coastal restoration.</p>
  </div>

  <div class="content">
    <!-- Filters Panel -->
    <div class="filter-panel">

      <div class="nerd-box">
        <details open>
          <summary>➕ What is a Semantic Map?</summary>
          <div class="nerd-box-content">
            <p>
              Semantic maps are tools that allow users to visually explore how different topics and ideas are connected. 
              By analyzing the relationships between articles, these maps can reveal patterns and clusters in the data.
            </p>
            <p>
              Each dot in this semantic map represents an article. When the dots are closer together, the articles are similar in meaning.
            </p>
            <p>
              Newsrooms can use semantic maps to:
            </p>
            <ul>
              <li>Audit their coverage by identifying themes that may be underrepresented or overlooked.</li>
              <li>Track how their editorial focus evolves over time.</li>
              <li>Discover which topics are being covered by other outlets, offering opportunities to adjust their editorial focus or collaborate.</li>
            </ul>
          </div>
        </details>
      </div>

      <label>📁 Upload CSV:</label>
      <input type="file" accept=".csv" on:change={handleFileUpload} />
      <label>🔍 Search Title:</label>
      <input type="text" placeholder="Search..." on:input={handleSearch} />

      {#if columns.length}
        <label>🎨 Color by Column:</label>
        <select on:change={handleDomainChange} bind:value={domainColumn}>
          <option value="" disabled>Select column</option>
          {#each columns as column}
            <option value={column}>{column}</option>
          {/each}
        </select>
      {/if}

      {#if uniqueValues.length}
        <label>✨ Highlight Values:</label>
        <select multiple size="5" class="multi-select" on:change={handleSelectionChange}>
          {#each uniqueValues as value}
            <option value={value}>{value}</option>
          {/each}
        </select>
      {/if}

      <label>💡 Adjust Opacity:</label>
      <input type="range" min="0.01" max="1" step="0.1" bind:value={opacity} on:input={handleOpacityChange} />

      <div class="date-controls">
        <label>📅 Date Range:</label>
        <input type="date" value={formatDateInput(startDate)} on:change={(e) => handleDateChange(e, 'start')} />
        <input type="date" value={formatDateInput(endDate)} on:change={(e) => handleDateChange(e, 'end')} />

        <div class="date-range-slider" style="--start-percent: {startPercent}%; --end-percent: {endPercent}%;">
          <input type="range" min="0" max={allDates.length - 1} bind:value={startDateIndex}
            on:input={() => {
              if (startDateIndex > endDateIndex) startDateIndex = endDateIndex;
              startDate = allDates[startDateIndex];
              showAnnotations = false;
            }}
            class="slider start-slider" />
          <input type="range" min="0" max={allDates.length - 1} bind:value={endDateIndex}
            on:input={() => {
              if (endDateIndex < startDateIndex) endDateIndex = startDateIndex;
              endDate = allDates[endDateIndex];
              showAnnotations = false;
            }}
            class="slider end-slider" />
        </div>

        <div class="date-range-labels">
          <span>{formatDateInput(startDate)}</span>
          <span>{formatDateInput(endDate)}</span>
        </div>

        <div class="date-navigation">
          <button on:click={() => shiftDateRange(-7)}>-1W</button>
          <button on:click={() => shiftDateRange(-1)}>-1D</button>
          <button class="play-button" on:click={togglePlayPause}>
            {#if isPlaying}
              ❚❚
            {:else}
              ▶
            {/if}
          </button>
          <button on:click={() => shiftDateRange(1)}>+1D</button>
          <button on:click={() => shiftDateRange(7)}>+1W</button>
        </div>
      </div>
    </div>

    <div class="scatterplot-container">
      {#if filteredData.length}
        <Scatterplot 
          data={filteredData} 
          {domainColumn} 
          {selectedValues} 
          {opacity} 
          {searchQuery}
          {showAnnotations}
        />
      {:else}
        <p>Loading data...</p>
      {/if}
    </div>
  </div>
</div>

<style>
  .container {
    padding: 1rem;
    display: flex;
    flex-direction: column;
    gap: 1rem;
    font-family: 'Inter', sans-serif;
  }

  .title-section {
    text-align: center;
  }

  .title {
    font-size: 2rem;
    font-weight: 700;
    margin-bottom: 0.5rem;
  }

  .subtitle {
    font-size: 1.1rem;
    color: #666;
    width: 60%;
    margin: 0 auto;
    line-height: 1.5;
  }

  .content {
    display: flex;
    gap: 1.5rem;
    align-items: flex-start; /* Align items at the top */
  }

  .filter-panel {
    background: #f9fafb;
    /* border-right: #969696 1px solid; */
    padding: 1.5rem;
    width: 300px; 
    display: flex;
    flex-direction: column;
    gap: 1rem;
    max-height: 100vh; 
    position: sticky; 
    overflow: auto;
    top: 1rem; 
  }

  .filter-panel label {
    font-weight: 600;
    font-size: 0.9rem;
  }

  .filter-panel input,
  .filter-panel select {
    padding: 0.5rem;
    font-size: 0.9rem;
    border: 1px solid #ccc;
    border-radius: 5px;
    width: 100%; 
    box-sizing: border-box; 
  }

  .multi-select {
    min-height: 150px;
    overflow-y: auto;
    width: 100%; 
  }

  .filter-panel input:focus,
  .filter-panel select:focus {
    outline: none;
    border-color: #4c8bf5;
  }

  .scatterplot-container {
    flex-grow: 1;
    display: flex;
    justify-content: center;
    align-items: center;
    background: #fff;
    border-radius: 10px;
    min-height: 500px;
    padding: 1rem;
    /* box-shadow: inset 0 0 10px rgba(0, 0, 0, 0.05); */
  }

  .date-controls label {
    display: block; 
    margin-bottom: 0.5rem; 
    font-weight: 600; 
  }

  .date-controls input[type="date"] {
    margin-bottom: 0.5rem;
  }

  .date-range-slider {
    height: 30px;
    position: relative;
  }

  .slider {
    position: absolute;
    width: 100%;
    pointer-events: none;
    background: linear-gradient(
      to right,
      #ccc 0%,
      #ccc var(--start-percent),
      #4c8bf5 var(--start-percent),
      #4c8bf5 var(--end-percent),
      #ccc var(--end-percent),
      #ccc 100%
    );
    appearance: none;
    height: 6px;
    border-radius: 3px;
    outline: none;
  }

  .slider::-webkit-slider-thumb {
    pointer-events: auto;
    -webkit-appearance: none;
    width: 18px;
    height: 18px;
    background: white;
    border: 2px solid #4c8bf5;
    border-radius: 50%;
    cursor: pointer;
  }

  .date-range-labels {
    display: flex;
    justify-content: space-between;
    font-size: 0.85rem;
    color: #444;
  }

  .date-navigation {
    display: flex;
    gap: 0.5rem;
    justify-content: space-between;
  }

  .date-navigation button {
    flex: 1;
    padding: 0.4rem;
    font-size: 0.85rem;
    background: #f0f0f0;
    border: 1px solid #ccc;
    border-radius: 4px;
    cursor: pointer;
  }

  .play-button {
    background: #4c8bf5;
    color: white;
    font-weight: bold;
  }

  .nerd-box {
    background: #eef5ff;
    border: 1px solid #4c8bf5;
    padding: 1rem;
    border-radius: 8px;
    font-size: 0.9rem;
    line-height: 1.6;
    margin-bottom: 1rem;
  }

  .nerd-box summary {
    /* toggle */
    
    font-weight: bold;
    cursor: pointer;
    list-style: none;
    margin-bottom: 0.5rem;
  }

  .nerd-box-content p {
    margin-bottom: 1rem;
    color: #444;
  }

  .nerd-box-content ul {
    list-style-type: disc; /* Add bullet points */
    padding-left: 1.5rem; /* Indent the list */
    margin-bottom: 1rem; /* Add spacing below the list */
  }

  .nerd-box-content li {
    margin-bottom: 0.5rem; /* Add spacing between list items */
    color: #444;
  }
</style>
