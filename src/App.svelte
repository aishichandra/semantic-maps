<script>
  import { onMount } from 'svelte';
  import Papa from 'papaparse';
  import Scatterplot from './components/Scatterplot.svelte';

  let data = [], columns = [], domainColumn = "org", uniqueValues = [];
  let selectedValues = new Set(); 
  let opacity = 1, startDate = null, endDate = null;
  let filteredData = [], allDates = [];
  let startDateIndex = 0;
  let endDateIndex = 0;
  let isPlaying = false;
  let playInterval = null;
  let searchQuery = "";

  $: filteredData = data.filter(d =>
    (!startDate || d.date >= startDate) &&
    (!endDate || d.date <= endDate) &&
    (searchQuery.trim() === "" || (d.title && d.title.toLowerCase().includes(searchQuery.toLowerCase())))
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
  }

  function handleSelectionChange(event) {
    selectedValues = new Set([...event.target.selectedOptions].map(o => o.value));
  }

  function handleDateChange(e, type) {
    const newDate = e.target.value ? new Date(e.target.value) : null;
    if (type === 'start') startDate = newDate;
    else endDate = newDate;
  }

  function formatDateInput(date) {
    return date ? date.toISOString().split('T')[0] : '';
  }

  function handleFileUpload(event) {
    const file = event.target.files[0];
    if (file) {
      const reader = new FileReader();
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
  }

  function togglePlayPause() {
    if (isPlaying) {
      clearInterval(playInterval);
      isPlaying = false;
    } else {
      isPlaying = true;
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
  }
</script>

<!-- App Layout -->
<div class="container">
  <div class="title-section">
    <h1 class="title">Interactive Semantic Maps</h1>
    <p class="subtitle">The default dataset shows the semantic map of three North Carolina organizations</p>
  </div>

  <div class="content">
    <!-- Filters Panel -->
    <div class="filter-panel">

      <div class="nerd-box">
        <details open>
          <summary>What is a Semantic Map?</summary>
          <div class="nerd-box-content">
            <p>
              A <strong>semantic map</strong> visualizes text data by meaning—closer dots are more similar.
            </p>
            <ul>
              <li>📁 Upload a CSV with <code>x</code>, <code>y</code>, and <code>date</code> columns</li>
              <li>🎨 Color by a metadata column</li>
              <li>🔍 Search by keyword or filter by category</li>
              <li>⏱ Animate or drag date range to explore changes</li>
            </ul>
          </div>
        </details>
      </div>

      <label>📁 Upload CSV:</label>
      <input type="file" accept=".csv" on:change={handleFileUpload} />

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

      <label>🔍 Search Title:</label>
      <input type="text" placeholder="Search..." on:input={handleSearch} />

      <label>💡 Adjust Opacity:</label>
      <input type="range" min="0.01" max="1" step="0.1" bind:value={opacity} />

      <div class="date-controls">
        <label>📅 Date Range:</label>
        <input type="date" value={formatDateInput(startDate)} on:change={(e) => handleDateChange(e, 'start')} />
        <input type="date" value={formatDateInput(endDate)} on:change={(e) => handleDateChange(e, 'end')} />

        <div class="date-range-slider" style="--start-percent: {startPercent}%; --end-percent: {endPercent}%;">
          <input type="range" min="0" max={allDates.length - 1} bind:value={startDateIndex}
            on:input={() => {
              if (startDateIndex > endDateIndex) startDateIndex = endDateIndex;
              startDate = allDates[startDateIndex];
            }}
            class="slider start-slider" />
          <input type="range" min="0" max={allDates.length - 1} bind:value={endDateIndex}
            on:input={() => {
              if (endDateIndex < startDateIndex) endDateIndex = startDateIndex;
              endDate = allDates[endDateIndex];
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

    <!-- Scatterplot Area -->
    <div class="scatterplot-container">
      {#if filteredData.length}
        <Scatterplot 
          data={filteredData}
          {domainColumn}
          {selectedValues}
          {opacity}
          {searchQuery}
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
  }

  .content {
    display: flex;
    gap: 1.5rem;
  }

  .filter-panel {
    background: #f9fafb;
    padding: 1rem;
    border-radius: 10px;
    width: 300px; /* Fixed width for the panel */
    display: flex;
    flex-direction: column;
    gap: 1rem;
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.06);
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
    width: 100%; /* Make inputs and selects take full width of the panel */
    box-sizing: border-box; /* Ensure padding doesn't affect width */
  }

  .multi-select {
    max-height: 150px;
    overflow-y: auto;
    width: 100%; 
  }

  .filter-panel input:focus,
  .filter-panel select:focus {
    outline: none;
    border-color: #4c8bf5;
    box-shadow: 0 0 4px rgba(76, 139, 245, 0.5);
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
    /* border-left: 4px solid #4c8bf5; */
    padding: 0.75rem;
    border-radius: 8px;
    font-size: 0.85rem;
    line-height: 1.5;
  }

  .nerd-box summary {
    font-weight: bold;
    cursor: pointer;
    list-style: none;
  }

  .nerd-box-content ul {
    padding-left: 1.2rem;
  }

  .nerd-box code {
    background: #e1e1e1;
    border-radius: 3px;
    padding: 2px 4px;
    font-family: monospace;
  }
</style>
