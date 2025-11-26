<script lang="ts">
  import { ClipboardCopy, Check } from 'lucide-svelte';

  export let reports: Array<{
    residentId: number;
    firstName: string;
    lastName: string;
    county: string;
    level: number;
    unit: string;
    timestamp: string;
    positive?: string;
    general?: string;
    negative?: string;
  }>;

  let selectedUnit = 'All Units';
  let selectedResident = 'All Residents';
  let weekOffset = 0;
  let copySuccess = false;

  $: units = ['All Units', ...new Set(reports.map(r => r.unit))];
  $: residentOptions = ['All Residents', ...new Set(reports.map(r => `${r.firstName} ${r.lastName.charAt(0)}.`))];

  function getWeekDates(offset: number) {
    const now = new Date();
    const currentDay = now.getDay();
    const diff = now.getDate() - currentDay + (currentDay === 0 ? -6 : 1);
    const monday = new Date(now);
    monday.setDate(diff + (offset * 7));
    monday.setHours(0, 0, 0, 0);
    
    const sunday = new Date(monday);
    sunday.setDate(sunday.getDate() + 6);
    sunday.setHours(23, 59, 59, 999);
    
    return { start: monday, end: sunday };
  }

  $: weekDates = getWeekDates(weekOffset);

  $: filteredReports = reports.filter(report => {
    const reportDate = new Date(report.timestamp);
    const matchesUnit = selectedUnit === 'All Units' || report.unit === selectedUnit;
    const residentDisplay = `${report.firstName} ${report.lastName.charAt(0)}.`;
    const matchesResident = selectedResident === 'All Residents' || residentDisplay === selectedResident;
    const matchesWeek = reportDate >= weekDates.start && reportDate <= weekDates.end;
    
    return matchesUnit && matchesResident && matchesWeek;
  });

  function formatResidentHeader(report: typeof reports[0]): string {
    return `${report.firstName} ${report.lastName.charAt(0)}. (${report.unit})(${report.county})(Lvl ${report.level})`;
  }

  function copyToClipboard() {
    let text = `SHIFT REPORTS\n`;
    text += `Week of ${weekDates.start.toLocaleDateString()} - ${weekDates.end.toLocaleDateString()}\n\n`;

    filteredReports.forEach(report => {
      text += `=================\n`;
      text += `${formatResidentHeader(report)}\n`;
      text += `Date: ${new Date(report.timestamp).toLocaleString()}\n\n`;
      
      if (report.positive) {
        text += `POSITIVE BEHAVIORS:\n${report.positive}\n\n`;
      }
      if (report.general) {
        text += `GENERAL NOTES:\n${report.general}\n\n`;
      }
      if (report.negative) {
        text += `BEHAVIORAL CONCERNS:\n${report.negative}\n\n`;
      }
    });

    navigator.clipboard.writeText(text);
    copySuccess = true;
    setTimeout(() => copySuccess = false, 3000);
  }

  function formatDate(date: Date, options: Intl.DateTimeFormatOptions) {
    return date.toLocaleDateString('en-US', options);
  }

  function formatTimestamp(timestamp: string) {
    return new Date(timestamp).toLocaleString();
  }
</script>

<div class="view-notes">
  {#if copySuccess}
    <div class="success-message">
      <Check class="w-5 h-5" />
      Reports copied to clipboard successfully!
    </div>
  {/if}

  <div class="filters">
    <div class="filters-grid">
      <div class="filter-group">
        <label class="filter-label">Unit</label>
        <select 
          class="filter-select"
          bind:value={selectedUnit}
        >
          {#each units as unit}
            <option value={unit}>{unit}</option>
          {/each}
        </select>
      </div>

      <div class="filter-group">
        <label class="filter-label">Resident</label>
        <select 
          class="filter-select"
          bind:value={selectedResident}
        >
          {#each residentOptions as resident}
            <option value={resident}>{resident}</option>
          {/each}
        </select>
      </div>
    </div>
  </div>

  <div class="week-navigation">
    <button 
      class="week-nav-btn"
      on:click={() => weekOffset -= 1}
    >
      ← Previous Week
    </button>
    <div class="current-week">
      {formatDate(weekDates.start, { month: 'short', day: 'numeric' })} - {formatDate(weekDates.end, { month: 'short', day: 'numeric', year: 'numeric' })}
    </div>
    <button 
      class="week-nav-btn"
      on:click={() => weekOffset += 1}
      disabled={weekOffset >= 0}
    >
      Next Week →
    </button>
  </div>

  {#if filteredReports.length > 0}
    <button class="copy-btn" on:click={copyToClipboard}>
      <ClipboardCopy class="w-5 h-5" />
      Copy All Reports for This Week
    </button>
  {/if}

  <div class="reports-container">
    {#if filteredReports.length === 0}
      <div class="no-reports">
        No reports found for the selected criteria.
      </div>
    {:else}
      {#each filteredReports as report, idx}
        <div class="report-entry">
          <div class="report-header">
            <div class="resident-display">
              <span class="resident-name">{report.firstName} {report.lastName.charAt(0)}.</span>
              <span class="resident-meta">
                <span class="meta-badge unit-badge">{report.unit}</span>
                <span class="meta-badge county-badge">{report.county}</span>
                <span class="meta-badge level-badge level-{report.level}">Lvl {report.level}</span>
              </span>
            </div>
            <span class="report-timestamp">{formatTimestamp(report.timestamp)}</span>
          </div>
          
          <div class="report-content">
            {#if report.positive}
              <div class="report-field">
                <div class="report-field-label positive">Positive Behaviors</div>
                <div class="report-field-value">{report.positive}</div>
              </div>
            {/if}
            {#if report.general}
              <div class="report-field">
                <div class="report-field-label general">General Notes</div>
                <div class="report-field-value">{report.general}</div>
              </div>
            {/if}
            {#if report.negative}
              <div class="report-field">
                <div class="report-field-label negative">Behavioral Concerns</div>
                <div class="report-field-value">{report.negative}</div>
              </div>
            {/if}
          </div>
        </div>
      {/each}
    {/if}
  </div>
</div>

<style>
  .view-notes {
    margin-top: 2rem;
  }

  .success-message {
    background: #5a9e7d;
    color: white;
    padding: 1rem 1.5rem;
    border-radius: 10px;
    margin-bottom: 1.5rem;
    font-weight: 500;
    display: flex;
    align-items: center;
    gap: 0.5rem;
    animation: slideDown 0.3s ease;
  }

  @keyframes slideDown {
    from {
      opacity: 0;
      transform: translateY(-10px);
    }
    to {
      opacity: 1;
      transform: translateY(0);
    }
  }

  .filters {
    background: white;
    padding: 2rem;
    border-radius: 16px;
    margin-bottom: 2rem;
    box-shadow: 0 4px 16px rgba(44, 95, 124, 0.08);
  }

  .filters-grid {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
    gap: 1.5rem;
  }

  .filter-group {
    display: flex;
    flex-direction: column;
    gap: 0.5rem;
  }

  .filter-label {
    font-weight: 600;
    font-size: 0.875rem;
    color: #1a3f52;
    text-transform: uppercase;
    letter-spacing: 0.5px;
  }

  .filter-select {
    padding: 0.75rem 1rem;
    border: 2px solid #e1e4e8;
    border-radius: 10px;
    font-family: inherit;
    font-size: 0.95rem;
    background: #f8f9fa;
    cursor: pointer;
    transition: all 0.3s ease;
  }

  .filter-select:focus {
    outline: none;
    border-color: #2c5f7c;
    background: white;
  }

  .week-navigation {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 2rem;
    background: white;
    padding: 1.5rem;
    border-radius: 12px;
    box-shadow: 0 2px 8px rgba(44, 95, 124, 0.08);
  }

  .week-nav-btn {
    padding: 0.75rem 1.5rem;
    background: #2c5f7c;
    color: white;
    border: none;
    border-radius: 8px;
    font-weight: 600;
    cursor: pointer;
    transition: all 0.3s ease;
  }

  .week-nav-btn:hover:not(:disabled) {
    background: #3d7a9e;
    transform: translateX(4px);
  }

  .week-nav-btn:first-child:hover:not(:disabled) {
    transform: translateX(-4px);
  }

  .week-nav-btn:disabled {
    background: #e1e4e8;
    cursor: not-allowed;
    transform: none;
  }

  .current-week {
    font-size: 1.25rem;
    font-weight: 600;
    color: #1a3f52;
  }

  .copy-btn {
    padding: 0.75rem 1.5rem;
    background: #e8925c;
    color: white;
    border: none;
    border-radius: 8px;
    font-weight: 600;
    cursor: pointer;
    transition: all 0.3s ease;
    margin-bottom: 2rem;
    display: flex;
    align-items: center;
    gap: 0.5rem;
  }

  .copy-btn:hover {
    background: #d67e49;
    transform: translateY(-2px);
    box-shadow: 0 4px 12px rgba(232, 146, 92, 0.3);
  }

  .reports-container {
    background: white;
    border-radius: 16px;
    padding: 2rem;
    box-shadow: 0 4px 16px rgba(44, 95, 124, 0.08);
  }

  .report-entry {
    border-left: 4px solid #2c5f7c;
    padding: 1.5rem;
    margin-bottom: 1.5rem;
    background: #f8f9fa;
    border-radius: 0 12px 12px 0;
    transition: all 0.3s ease;
  }

  .report-entry:hover {
    background: white;
    box-shadow: 0 2px 8px rgba(44, 95, 124, 0.08);
  }

  .report-entry:last-child {
    margin-bottom: 0;
  }

  .report-header {
    display: flex;
    justify-content: space-between;
    align-items: flex-start;
    margin-bottom: 1rem;
    flex-wrap: wrap;
    gap: 0.5rem;
  }

  .resident-display {
    display: flex;
    flex-direction: column;
    gap: 0.5rem;
  }

  .resident-name {
    font-size: 1.25rem;
    font-weight: 700;
    color: #1a3f52;
  }

  .resident-meta {
    display: flex;
    gap: 0.375rem;
    flex-wrap: wrap;
  }

  .meta-badge {
    padding: 0.2rem 0.5rem;
    border-radius: 4px;
    font-size: 0.7rem;
    font-weight: 600;
  }

  .unit-badge {
    background: #f3f4f6;
    color: #374151;
  }

  .county-badge {
    background: #e0e7ff;
    color: #3730a3;
  }

  .level-badge {
    min-width: 40px;
    text-align: center;
  }

  .level-badge.level-0 {
    background: #fee2e2;
    color: #991b1b;
  }

  .level-badge.level-1 {
    background: #fef3c7;
    color: #92400e;
  }

  .level-badge.level-2 {
    background: #d1fae5;
    color: #065f46;
  }

  .level-badge.level-3 {
    background: #dbeafe;
    color: #1e40af;
  }

  .report-timestamp {
    font-size: 0.875rem;
    color: #6b7280;
  }

  .report-content {
    display: grid;
    gap: 1rem;
  }

  .report-field {
    display: grid;
    gap: 0.375rem;
  }

  .report-field-label {
    font-size: 0.75rem;
    font-weight: 600;
    text-transform: uppercase;
    letter-spacing: 0.5px;
    display: flex;
    align-items: center;
    gap: 0.375rem;
  }

  .report-field-label::before {
    content: '';
    width: 3px;
    height: 12px;
    border-radius: 2px;
  }

  .report-field-label.positive {
    color: #065f46;
  }

  .report-field-label.positive::before {
    background: #5a9e7d;
  }

  .report-field-label.general {
    color: #1a3f52;
  }

  .report-field-label.general::before {
    background: #2c5f7c;
  }

  .report-field-label.negative {
    color: #92400e;
  }

  .report-field-label.negative::before {
    background: #d97847;
  }

  .report-field-value {
    color: #1e2936;
    line-height: 1.6;
    padding-left: 0.75rem;
  }

  .no-reports {
    text-align: center;
    padding: 3rem;
    color: #6b7280;
    font-style: italic;
  }
</style>