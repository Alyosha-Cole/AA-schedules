<script lang="ts">
  import { Check, Calendar } from 'lucide-svelte';
  import { createEventDispatcher } from 'svelte';

  export let residents: Record<string, Array<{
    id: number;
    firstName: string;
    lastName: string;
    attributes: Record<string, string>;
  }>>;
  
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
  }>;

  export let noteSettings: {
    categories: Array<{
      id: string;
      name: string;
      color: string;
      order: number;
      checkboxes: Array<{ id: string; label: string }>;
    }>;
  };

  const dispatch = createEventDispatcher();

  let activeUnit = Object.keys(residents)[0] || 'Unit 1';
  let formData: Record<string, Record<string, string>> = {};
  let showSuccess = false;
  
  // Date selection - defaults to today
  let selectedDate = new Date().toISOString().split('T')[0];

  $: unitResidents = residents[activeUnit] || [];
  $: sortedCategories = [...(noteSettings?.categories || [])].sort((a, b) => a.order - b.order);
  
  // Get effective attributes for the active unit
  $: effectiveAttributes = (() => {
    const settings = residentSettings[activeUnit];
    if (!settings) return [];
    if (settings.inheritFrom && residentSettings[settings.inheritFrom]) {
      return residentSettings[settings.inheritFrom].attributes || [];
    }
    return settings.attributes || [];
  })();
  
  $: sortedAttributes = [...effectiveAttributes].sort((a, b) => a.order - b.order);

  function getOptionDisplay(attrId: string, value: string) {
    const attr = effectiveAttributes.find(a => a.id === attrId);
    if (!attr?.options) return { label: value, color: '#6b7280' };
    const option = attr.options.find(o => o.value === value);
    return option || { label: value, color: '#6b7280' };
  }

  function handleTextChange(residentId: number, categoryId: string, value: string) {
    const key = `${activeUnit}-${residentId}`;
    formData = {
      ...formData,
      [key]: {
        ...formData[key],
        [categoryId]: value
      }
    };
  }

  function handleCheckboxToggle(residentId: number, categoryId: string, text: string) {
    const key = `${activeUnit}-${residentId}`;
    const currentText = formData[key]?.[categoryId] || '';
    const newText = currentText.includes(text)
      ? currentText.replace(text + '. ', '')
      : currentText + text + '. ';

    handleTextChange(residentId, categoryId, newText);
  }

  function isChecked(residentId: number, categoryId: string, text: string): boolean {
    const key = `${activeUnit}-${residentId}`;
    return formData[key]?.[categoryId]?.includes(text) || false;
  }

  function getFormValue(residentId: number, categoryId: string): string {
    const key = `${activeUnit}-${residentId}`;
    return formData[key]?.[categoryId] || '';
  }

  function handleSubmit() {
    const reportsToSubmit: any[] = [];
    // Use selected date with current time
    const dateObj = new Date(selectedDate + 'T' + new Date().toTimeString().split(' ')[0]);
    const timestamp = dateObj.toISOString();

    Object.entries(formData).forEach(([key, data]) => {
      // Check if any category has content
      const hasContent = Object.values(data).some(v => v && v.trim());
      
      if (hasContent) {
        const [unit, residentIdStr] = key.split('-');
        const residentId = parseInt(residentIdStr);
        const resident = residents[unit]?.find(r => r.id === residentId);
        
        if (resident) {
          reportsToSubmit.push({
            residentId,
            firstName: resident.firstName,
            lastName: resident.lastName,
            attributes: { ...resident.attributes },
            unit,
            timestamp,
            notes: { ...data } // Store all category data
          });
        }
      }
    });

    if (reportsToSubmit.length > 0) {
      dispatch('submit', reportsToSubmit);
      formData = {};
      showSuccess = true;
      setTimeout(() => showSuccess = false, 3000);
    }
  }

  function formatDate(date: Date, options: Intl.DateTimeFormatOptions) {
    return date.toLocaleDateString('en-US', options);
  }
</script>

<div class="notes-form">
  {#if showSuccess}
    <div class="success-message">
      <Check class="w-5 h-5" />
      Shift reports submitted successfully!
    </div>
  {/if}

  <!-- Date Selection -->
  <div class="date-selection">
    <div class="date-picker-wrapper">
      <Calendar class="w-5 h-5 text-slate-500" />
      <label class="date-label">Report Date:</label>
      <input
        type="date"
        bind:value={selectedDate}
        class="date-input"
      />
      {#if selectedDate !== new Date().toISOString().split('T')[0]}
        <span class="date-warning">⚠️ Not today's date</span>
      {/if}
    </div>
  </div>

  <div class="unit-tabs">
    {#each Object.keys(residents) as unit}
      <button
        class="unit-tab {activeUnit === unit ? 'active' : ''}"
        on:click={() => activeUnit = unit}
      >
        {unit}
        <span class="tab-count">({residents[unit]?.length || 0})</span>
      </button>
    {/each}
  </div>

  {#if sortedCategories.length === 0}
    <div class="empty-state">
      <p>No note categories configured.</p>
      <p class="empty-hint">Click the Settings button above to add categories like "Positive Behaviors", "General Notes", etc.</p>
    </div>
  {:else if unitResidents.length === 0}
    <div class="empty-state">
      <p>No residents in {activeUnit}.</p>
      <p class="empty-hint">Add residents in the "Manage Residents" section above.</p>
    </div>
  {:else}
    <div class="residents-grid">
      {#each unitResidents as resident (resident.id)}
        <div class="resident-card">
          <div class="resident-header">
            <div class="resident-title">
              <h2 class="resident-name">{resident.firstName} {resident.lastName.charAt(0)}.</h2>
              <div class="resident-badges">
                {#each sortedAttributes.filter(a => a.showInHeader) as attr}
                  {@const value = resident.attributes[attr.id] || ''}
                  {#if (attr.type === 'cycle' || attr.type === 'select') && attr.options}
                    {@const display = getOptionDisplay(attr.id, value)}
                    <span 
                      class="badge"
                      style="background-color: {display.color}20; color: {display.color}"
                    >
                      {display.label}
                    </span>
                  {:else if value}
                    <span class="badge text-badge">{value}</span>
                  {/if}
                {/each}
              </div>
            </div>
            <span class="timestamp">
              {formatDate(new Date(selectedDate), { weekday: 'short', month: 'short', day: 'numeric' })}
            </span>
          </div>

          <!-- Dynamic Categories -->
          {#each sortedCategories as category (category.id)}
            <div class="report-section">
              <div class="section-label" style="color: {category.color}">
                <span class="label-bar" style="background-color: {category.color}"></span>
                {category.name}
              </div>
              <textarea
                class="text-area"
                value={getFormValue(resident.id, category.id)}
                on:input={(e) => handleTextChange(resident.id, category.id, e.currentTarget.value)}
                placeholder="Enter notes for {category.name.toLowerCase()}..."
              ></textarea>
              
              {#if category.checkboxes.length > 0}
                <div class="checkboxes">
                  {#each category.checkboxes as checkbox (checkbox.id)}
                    <label 
                      class="checkbox-item {isChecked(resident.id, category.id, checkbox.label) ? 'checked' : ''}"
                      style="--check-color: {category.color}"
                    >
                      <input
                        type="checkbox"
                        checked={isChecked(resident.id, category.id, checkbox.label)}
                        on:change={() => handleCheckboxToggle(resident.id, category.id, checkbox.label)}
                      />
                      <span>{checkbox.label}</span>
                    </label>
                  {/each}
                </div>
              {/if}
            </div>
          {/each}
        </div>
      {/each}
    </div>

    <button class="submit-btn" on:click={handleSubmit}>
      Submit Shift Reports for {formatDate(new Date(selectedDate), { month: 'short', day: 'numeric', year: 'numeric' })}
    </button>
  {/if}
</div>

<style>
  .notes-form {
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

  .date-selection {
    background: white;
    padding: 1rem 1.5rem;
    border-radius: 12px;
    margin-bottom: 1.5rem;
    box-shadow: 0 2px 8px rgba(44, 95, 124, 0.08);
  }

  .date-picker-wrapper {
    display: flex;
    align-items: center;
    gap: 0.75rem;
    flex-wrap: wrap;
  }

  .date-label {
    font-weight: 600;
    color: #1a3f52;
  }

  .date-input {
    padding: 0.5rem 1rem;
    border: 2px solid #e1e4e8;
    border-radius: 8px;
    font-size: 1rem;
    font-weight: 500;
    color: #1a3f52;
    cursor: pointer;
    transition: all 0.2s ease;
  }

  .date-input:focus {
    outline: none;
    border-color: #2c5f7c;
    box-shadow: 0 0 0 3px rgba(44, 95, 124, 0.1);
  }

  .date-warning {
    font-size: 0.875rem;
    color: #d97706;
    font-weight: 500;
  }

  .unit-tabs {
    display: flex;
    gap: 0.5rem;
    margin-bottom: 2rem;
    background: white;
    padding: 0.5rem;
    border-radius: 12px;
    box-shadow: 0 2px 8px rgba(44, 95, 124, 0.08);
  }

  .unit-tab {
    flex: 1;
    padding: 0.875rem 1.5rem;
    border: none;
    background: transparent;
    color: #6b7280;
    font-size: 0.95rem;
    font-weight: 600;
    border-radius: 8px;
    cursor: pointer;
    transition: all 0.3s ease;
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 0.5rem;
  }

  .unit-tab:hover {
    background: #f8f9fa;
    color: #1e2936;
  }

  .unit-tab.active {
    background: #2c5f7c;
    color: white;
    box-shadow: 0 2px 8px rgba(44, 95, 124, 0.2);
  }

  .tab-count {
    font-size: 0.8rem;
    opacity: 0.8;
  }

  .empty-state {
    text-align: center;
    padding: 3rem;
    background: white;
    border-radius: 12px;
    color: #6b7280;
  }

  .empty-state p {
    margin: 0.5rem 0;
  }

  .empty-hint {
    font-size: 0.875rem;
    font-style: italic;
  }

  .residents-grid {
    display: grid;
    gap: 1.5rem;
  }

  .resident-card {
    background: white;
    border-radius: 16px;
    padding: 2rem;
    box-shadow: 0 4px 16px rgba(44, 95, 124, 0.08);
    transition: all 0.3s ease;
    border: 1px solid #e1e4e8;
  }

  .resident-card:hover {
    box-shadow: 0 8px 24px rgba(44, 95, 124, 0.12);
    transform: translateY(-2px);
  }

  .resident-header {
    display: flex;
    justify-content: space-between;
    align-items: flex-start;
    margin-bottom: 1.5rem;
    padding-bottom: 1rem;
    border-bottom: 2px solid #e1e4e8;
  }

  .resident-title {
    display: flex;
    flex-direction: column;
    gap: 0.5rem;
  }

  .resident-name {
    font-size: 1.5rem;
    font-weight: 700;
    color: #1a3f52;
    margin: 0;
  }

  .resident-badges {
    display: flex;
    gap: 0.5rem;
    flex-wrap: wrap;
  }

  .badge {
    padding: 0.25rem 0.5rem;
    border-radius: 4px;
    font-size: 0.75rem;
    font-weight: 600;
  }

  .text-badge {
    background: #e0e7ff;
    color: #3730a3;
  }

  .timestamp {
    color: #6b7280;
    font-size: 0.875rem;
  }

  .report-section {
    margin-bottom: 1.5rem;
  }

  .section-label {
    font-weight: 600;
    font-size: 0.875rem;
    text-transform: uppercase;
    letter-spacing: 0.5px;
    margin-bottom: 0.75rem;
    display: flex;
    align-items: center;
    gap: 0.5rem;
  }

  .section-label .label-bar {
    width: 4px;
    height: 16px;
    border-radius: 2px;
  }

  .text-area {
    width: 100%;
    min-height: 100px;
    padding: 1rem;
    border: 2px solid #e1e4e8;
    border-radius: 10px;
    font-family: inherit;
    font-size: 0.95rem;
    resize: vertical;
    transition: all 0.3s ease;
    background: #f8f9fa;
  }

  .text-area:focus {
    outline: none;
    border-color: #2c5f7c;
    background: white;
    box-shadow: 0 0 0 3px rgba(44, 95, 124, 0.1);
  }

  .checkboxes {
    display: flex;
    flex-wrap: wrap;
    gap: 0.75rem;
    margin-top: 0.75rem;
  }

  .checkbox-item {
    display: flex;
    align-items: center;
    gap: 0.5rem;
    padding: 0.5rem 0.875rem;
    background: #f8f9fa;
    border: 2px solid #e1e4e8;
    border-radius: 8px;
    cursor: pointer;
    transition: all 0.2s ease;
    font-size: 0.875rem;
  }

  .checkbox-item:hover {
    background: white;
    border-color: var(--check-color, #3d7a9e);
  }

  .checkbox-item input[type="checkbox"] {
    cursor: pointer;
    width: 18px;
    height: 18px;
    accent-color: var(--check-color, #2c5f7c);
  }

  .checkbox-item.checked {
    background: var(--check-color, #2c5f7c);
    border-color: var(--check-color, #2c5f7c);
    color: white;
  }

  .submit-btn {
    width: 100%;
    padding: 1rem 2rem;
    background: linear-gradient(135deg, #5a9e7d 0%, #4a8d6a 100%);
    color: white;
    border: none;
    border-radius: 12px;
    font-size: 1rem;
    font-weight: 600;
    cursor: pointer;
    transition: all 0.3s ease;
    margin-top: 2rem;
    box-shadow: 0 4px 12px rgba(90, 158, 125, 0.3);
  }

  .submit-btn:hover {
    transform: translateY(-2px);
    box-shadow: 0 6px 20px rgba(90, 158, 125, 0.4);
  }
</style>