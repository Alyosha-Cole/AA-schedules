<script lang="ts">
  import { ClipboardCopy, Check } from 'lucide-svelte';

  export let reports: Array<{
    residentId: number;
    firstName: string;
    lastName: string;
    attributes: Record<string, string>;
    unit: string;
    timestamp: string;
    notes: Record<string, string>;
  }> = [];
  
  export let residentSettings: Record<string, {
    inheritFrom: string | null;
    attributes: Array<{
      id: string;
      name: string;
      type: 'text' | 'cycle' | 'select';
      required: boolean;
      options?: Array<{ value: string; label: string; color: string }>;
      order: number;
      showInHeader: boolean;
    }>;
  }> = {};

  export let noteSettings: {
    categories: Array<{
      id: string;
      name: string;
      color: string;
      order: number;
      checkboxes: Array<{ id: string; label: string }>;
    }>;
  } = { categories: [] };

  let selectedUnit = 'All Units';
  let selectedResident = 'All Residents';
  let weekOffset = 0;
  let copySuccess = false;

  // Safely get unique units from reports
  function getUnits(reportList: typeof reports): string[] {
    if (!reportList || reportList.length === 0) return ['All Units'];
    const uniqueUnits = [...new Set(reportList.map(r => r.unit).filter(Boolean))];
    return ['All Units', ...uniqueUnits];
  }

  // Safely get unique residents from reports
  function getResidentOptions(reportList: typeof reports): string[] {
    if (!reportList || reportList.length === 0) return ['All Residents'];
    const uniqueResidents = [...new Set(reportList.map(r => {
      if (!r.firstName || !r.lastName) return null;
      return `${r.firstName} ${r.lastName.charAt(0)}.`;
    }).filter(Boolean))] as string[];
    return ['All Residents', ...uniqueResidents];
  }

  $: units = getUnits(reports);
  $: residentOptions = getResidentOptions(reports);
  $: sortedCategories = [...(noteSettings?.categories || [])].sort((a, b) => a.order - b.order);

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

  let weekDates = getWeekDates(0);
  $: weekDates = getWeekDates(weekOffset);

  // Filter reports safely
  function filterReports(
    reportList: typeof reports, 
    unit: string, 
    resident: string, 
    dates: { start: Date; end: Date }
  ) {
    if (!reportList || reportList.length === 0) return [];
    
    return reportList.filter(report => {
      if (!report || !report.timestamp) return false;
      
      try {
        const reportDate = new Date(report.timestamp);
        if (isNaN(reportDate.getTime())) return false;
        
        const matchesUnit = unit === 'All Units' || report.unit === unit;
        const residentDisplay = `${report.firstName || ''} ${(report.lastName || '').charAt(0)}.`;
        const matchesResident = resident === 'All Residents' || residentDisplay === resident;
        const matchesWeek = reportDate >= dates.start && reportDate <= dates.end;
        
        return matchesUnit && matchesResident && matchesWeek;
      } catch {
        return false;
      }
    });
  }

  $: filteredReports = filterReports(reports, selectedUnit, selectedResident, weekDates);

  // Get effective attributes for a unit
  function getEffectiveAttributes(unit: string) {
    if (!residentSettings || !unit) return [];
    const settings = residentSettings[unit];
    if (!settings) return [];
    if (settings.inheritFrom && residentSettings[settings.inheritFrom]) {
      return residentSettings[settings.inheritFrom].attributes || [];
    }
    return settings.attributes || [];
  }

  function getOptionDisplay(unit: string, attrId: string, value: string) {
    if (!unit || !attrId) return { label: value || '', color: '#6b7280' };
    const attrs = getEffectiveAttributes(unit);
    const attr = attrs.find(a => a.id === attrId);
    if (!attr?.options) return { label: value || '', color: '#6b7280' };
    const option = attr.options.find(o => o.value === value);
    return option || { label: value || '', color: '#6b7280' };
  }

  function getCategoryById(catId: string) {
    return noteSettings?.categories?.find(c => c.id === catId);
  }

  function formatResidentHeader(report: typeof reports[0]): string {
    if (!report) return '';
    const attrs = getEffectiveAttributes(report.unit);
    const sortedAttrs = [...attrs].sort((a, b) => (a.order || 0) - (b.order || 0));
    
    let header = `${report.firstName || ''} ${(report.lastName || '').charAt(0)}. (${report.unit || ''})`;
    
    for (const attr of sortedAttrs.filter(a => a.showInHeader)) {
      const value = report.attributes?.[attr.id];
      if (value) {
        if (attr.options) {
          const option = attr.options.find(o => o.value === value);
          header += `(${option?.label || value})`;
        } else {
          header += `(${value})`;
        }
      }
    }
    
    return header;
  }

  function copyToClipboard() {
    let text = `SHIFT REPORTS\n`;
    text += `Week of ${weekDates.start.toLocaleDateString()} - ${weekDates.end.toLocaleDateString()}\n\n`;

    filteredReports.forEach(report => {
      text += `=================\n`;
      text += `${formatResidentHeader(report)}\n`;
      text += `Date: ${formatDateOnly(report.timestamp)}\n\n`;
      
      // Add each category's notes
      for (const category of sortedCategories) {
        const noteContent = report.notes?.[category.id];
        if (noteContent && noteContent.trim()) {
          text += `${category.name.toUpperCase()}:\n${noteContent}\n\n`;
        }
      }
    });

    navigator.clipboard.writeText(text);
    copySuccess = true;
    setTimeout(() => copySuccess = false, 3000);
  }

  function formatDate(date: Date, options: Intl.DateTimeFormatOptions) {
    try {
      return date.toLocaleDateString('en-US', options);
    } catch {
      return '';
    }
  }

  function formatDateOnly(timestamp: string) {
    try {
      return new Date(timestamp).toLocaleDateString('en-US', {
        weekday: 'short',
        month: 'short',
        day: 'numeric',
        year: 'numeric'
      });
    } catch {
      return timestamp;
    }
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
      {#each filteredReports as report (report.timestamp + '-' + report.residentId)}
        {@const attrs = getEffectiveAttributes(report.unit)}
        {@const sortedAttrs = [...attrs].sort((a, b) => (a.order || 0) - (b.order || 0))}
        
        <div class="report-entry">
          <div class="report-header">
            <div class="resident-display">
              <span class="resident-name">{report.firstName} {(report.lastName || '').charAt(0)}.</span>
              <span class="resident-meta">
                <span class="meta-badge unit-badge">{report.unit}</span>
                {#each sortedAttrs.filter(a => a.showInHeader) as attr (attr.id)}
                  {@const value = report.attributes?.[attr.id] || ''}
                  {#if value}
                    {#if attr.options}
                      {@const display = getOptionDisplay(report.unit, attr.id, value)}
                      <span 
                        class="meta-badge"
                        style="background-color: {display.color}20; color: {display.color}"
                      >
                        {display.label}
                      </span>
                    {:else}
                      <span class="meta-badge text-badge">{value}</span>
                    {/if}
                  {/if}
                {/each}
              </span>
            </div>
            <span class="report-date">{formatDateOnly(report.timestamp)}</span>
          </div>
          
          <div class="report-content">
            {#each sortedCategories as category (category.id)}
              {@const noteContent = report.notes?.[category.id]}
              {#if noteContent && noteContent.trim()}
                <div class="report-field">
                  <div class="report-field-label" style="color: {category.color}">
                    <span class="label-bar" style="background-color: {category.color}"></span>
                    {category.name}
                  </div>
                  <div class="report-field-value">{noteContent}</div>
                </div>
              {/if}
            {/each}
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
  }

  .week-nav-btn:disabled {
    background: #e1e4e8;
    cursor: not-allowed;
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

  .text-badge {
    background: #e0e7ff;
    color: #3730a3;
  }

  .report-date {
    font-size: 0.875rem;
    color: #6b7280;
    font-weight: 500;
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

  .report-field-label .label-bar {
    width: 3px;
    height: 12px;
    border-radius: 2px;
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