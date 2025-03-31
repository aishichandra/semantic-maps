<script>
  import { onMount } from 'svelte';
  import Papa from 'papaparse';
  import Scatterplot from './components/Scatterplot.svelte';

  let data = [], columns = [], domainColumn = "", uniqueValues = [];
  let selectedValues = new Set(); 
  let opacity = 1, startDate = null, endDate = null;
  let filteredData = [], allDates = [];

  $: filteredData = data.filter(d => (!startDate || d.date >= startDate) && (!endDate || d.date <= endDate));

  onMount(async () => {
    const response = await fetch('/semantic-maps/data.csv');
    const csvText = await response.text();
    parseCSV(csvText);
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
      
      <label>Date Scrubber:</label>
      <input 
        type="range" 
        min="0" 
        max={allDates.length - 1} 
        value={allDates.findIndex(d => d >= endDate) }
        on:input={(e) => {
          const index = parseInt(e.target.value);
          endDate = allDates[index] || allDates[allDates.length - 1];
        }}
      />
      <div class="current-date">{formatDateInput(endDate)}</div>
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
</style>
