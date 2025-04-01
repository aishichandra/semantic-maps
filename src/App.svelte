<script>
  import { onMount } from 'svelte';
  import Papa from 'papaparse';
  import Scatterplot from './components/Scatterplot.svelte';

  let data = [], columns = [], domainColumn = "", uniqueValues = [];
  let selectedValues = new Set(); 
  let opacity = 1, startDate = null, endDate = null;
  let filteredData = [], allDates = [];
  let startDateIndex = 0;
  let endDateIndex = 0;
  let isPlaying = false;
  let playInterval = null;

  $: filteredData = data.filter(d => (!startDate || d.date >= startDate) && (!endDate || d.date <= endDate));

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
    if (type === 'start') {
      startDate = newDate;
    } else {
      endDate = newDate;
    }
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
      playInterval = null;
    } else {
      isPlaying = true;
      playInterval = setInterval(() => {
        if (endDateIndex >= allDates.length - 1) {
          clearInterval(playInterval);
          isPlaying = false;
          playInterval = null;
          return;
        }
        shiftDateRange(1);
      }, 500);
    }
  }
</script>

<div class="container">
  <div class="filter-panel">
    <label>Upload CSV File:</label>
    <input type="file" accept=".csv" on:change={handleFileUpload} />

    {#if columns.length}
      <label>Select a column for coloring:</label>
      <select on:change={handleDomainChange} bind:value={domainColumn}>
        <option value="" disabled selected>Select column</option>
        {#each columns as column}
          <option value={column}>{column}</option>
        {/each}
      </select>
    {/if}

    {#if uniqueValues.length}
      <label>Select values to highlight:</label>
      <select multiple size="5" class="multi-select" on:change={handleSelectionChange}>
        {#each uniqueValues as value}
          <option value={value}>{value}</option>
        {/each}
      </select>
    {/if}

    <label>Adjust Opacity:</label>
    <input type="range" min="0.01" max="1" step="0.1" bind:value={opacity} />

    <div class="date-controls">
      <label>Start Date:</label>
      <input type="date" value={formatDateInput(startDate)} on:change={(e) => handleDateChange(e, 'start')} />
      
      <label>End Date:</label>
      <input type="date" value={formatDateInput(endDate)} on:change={(e) => handleDateChange(e, 'end')} />
      
      <label>Date Range Slider:</label>
      <div class="date-range-slider" style="--start-percent: {startPercent}%; --end-percent: {endPercent}%;">
        <input 
          type="range" 
          min="0" 
          max={allDates.length - 1} 
          bind:value={startDateIndex}
          on:input={() => {
            if (startDateIndex > endDateIndex) startDateIndex = endDateIndex;
            startDate = allDates[startDateIndex] || allDates[0];
          }}
          class="slider start-slider"
        />
        <input 
          type="range" 
          min="0" 
          max={allDates.length - 1} 
          bind:value={endDateIndex}
          on:input={() => {
            if (endDateIndex < startDateIndex) endDateIndex = startDateIndex;
            endDate = allDates[endDateIndex] || allDates[allDates.length - 1];
          }}
          class="slider end-slider"
        />
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
            <span class="pause-icon">❚❚</span>
          {:else}
            <span class="play-icon">▶</span>
          {/if}
        </button>
        <button on:click={() => shiftDateRange(1)}>+1D</button>
        <button on:click={() => shiftDateRange(7)}>+1W</button>
      </div>
    </div>
  </div>

  <div class="scatterplot-container">
    {#if filteredData.length}
      <Scatterplot data={filteredData} {domainColumn} {selectedValues} {opacity} />
    {:else}
      <p>Loading data...</p>
    {/if}
  </div>
</div>

<style>
  .container {
    display: flex;
    gap: 1rem;
    padding: 1rem;
  }
  .filter-panel {
    display: flex;
    flex-direction: column;
    gap: 1rem;
    padding: 1rem;
    border-radius: 10px;
    background-color: #fafafa;
    width: 280px;
  }
  .filter-panel label {
    font-weight: bold;
  }
  .multi-select option {
    padding: 6px;
  }
  .scatterplot-container {
    flex-grow: 1;
    display: flex;
    justify-content: center;
    align-items: center;
  }
  .current-date {
    font-size: 0.9rem;
    margin-top: 5px;
  }
  .date-controls {
    display: flex;
    flex-direction: column;
    gap: 0.5rem;
  }
  .date-range-slider {
    position: relative;
    height: 30px;
    margin: 10px 0;
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
    -webkit-appearance: none;
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
  .slider::-moz-range-thumb {
    pointer-events: auto;
    width: 18px;
    height: 18px;
    background: white;
    border: 2px solid #4c8bf5;
    border-radius: 50%;
    cursor: pointer;
  }
  .start-slider::-webkit-slider-thumb {
    z-index: 3;
  }
  .end-slider::-webkit-slider-thumb {
    z-index: 3;
  }
  .date-range-labels {
    display: flex;
    justify-content: space-between;
    margin-top: 5px;
    font-size: 0.9rem;
  }
  .date-navigation {
    display: flex;
    justify-content: space-between;
    margin-top: 10px;
    gap: 5px;
  }
  .date-navigation button {
    flex: 1;
    padding: 8px 0;
    border: 1px solid #ddd;
    background-color: #f5f5f5;
    border-radius: 4px;
    cursor: pointer;
    font-weight: bold;
    transition: background-color 0.2s;
  }
  .date-navigation button:hover {
    background-color: #e0e0e0;
  }
  .play-button {
    background-color: #4c8bf5 !important;
    color: white;
  }
  .play-button:hover {
    background-color: #3a78e0 !important;
  }
  .play-icon, .pause-icon {
    display: inline-block;
    font-size: 14px;
  }
  .pause-icon {
    letter-spacing: -2px;
  }
</style>
