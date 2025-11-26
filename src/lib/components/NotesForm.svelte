<script lang="ts">
  import { Check } from 'lucide-svelte';
  import { createEventDispatcher } from 'svelte';

  export let residents: Record<string, Array<{
    id: number;
    firstName: string;
    lastName: string;
    county: string;
    level: number;
  }>>;

  const dispatch = createEventDispatcher();

  const POSITIVE_BEHAVIORS = [
    'Participated in group activities',
    'Showed respect to staff',
    'Helped another resident',
    'Completed assigned tasks',
    'Displayed positive attitude'
  ];

  const GENERAL_NOTES = [
    'Ate all meals',
    'Attended classes',
    'Participated in recreation',
    'Took medications',
    'Had visitor'
  ];

  const NEGATIVE_BEHAVIORS = [
    'Refused to follow directions',
    'Verbal altercation with peer',
    'Disruptive in class',
    'Property damage',
    'Left designated area without permission'
  ];

  let activeUnit = 'Unit 1';
  let formData: Record<string, { positive?: string; general?: string; negative?: string }> = {};
  let showSuccess = false;

  $: unitResidents = residents[activeUnit] || [];

  function handleTextChange(residentId: number, section: string, value: string) {
    const key = `${activeUnit}-${residentId}`;
    formData = {
      ...formData,
      [key]: {
        ...formData[key],
        [section]: value
      }
    };
  }

  function handleCheckboxToggle(residentId: number, section: string, text: string) {
    const key = `${activeUnit}-${residentId}`;
    const currentText = formData[key]?.[section as keyof typeof formData[string]] || '';
    const newText = currentText.includes(text)
      ? currentText.replace(text + '. ', '')
      : currentText + text + '. ';

    handleTextChange(residentId, section, newText);
  }

  function isChecked(residentId: number, section: string, text: string): boolean {
    const key = `${activeUnit}-${residentId}`;
    return formData[key]?.[section as keyof typeof formData[string]]?.includes(text) || false;
  }

  function getFormValue(residentId: number, section: string): string {
    const key = `${activeUnit}-${residentId}`;
    return formData[key]?.[section as keyof typeof formData[string]] || '';
  }

  function handleSubmit() {
    const reportsToSubmit: any[] = [];
    const timestamp = new Date().toISOString();

    Object.entries(formData).forEach(([key, data]) => {
      if (data.positive || data.general || data.negative) {
        const [unit, residentIdStr] = key.split('-');
        const residentId = parseInt(residentIdStr);
        const resident = residents[unit]?.find(r => r.id === residentId);
        
        if (resident) {
          reportsToSubmit.push({
            residentId,
            firstName: resident.firstName,
            lastName: resident.lastName,
            county: resident.county,
            level: resident.level,
            unit,
            timestamp,
            ...data
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

  {#if unitResidents.length === 0}
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
                <span class="badge county-badge">{resident.county}</span>
                <span class="badge level-badge level-{resident.level}">Lvl {resident.level}</span>
              </div>
            </div>
            <span class="timestamp">
              {formatDate(new Date(), { weekday: 'short', month: 'short', day: 'numeric' })}
            </span>
          </div>

          <!-- Positive Behaviors -->
          <div class="report-section">
            <div class="section-label positive">
              <span class="label-bar"></span>
              Positive Behaviors
            </div>
            <textarea
              class="text-area"
              value={getFormValue(resident.id, 'positive')}
              on:input={(e) => handleTextChange(resident.id, 'positive', e.currentTarget.value)}
              placeholder="Document positive behaviors and accomplishments..."
            ></textarea>
            <div class="checkboxes">
              {#each POSITIVE_BEHAVIORS as behavior}
                <label class="checkbox-item {isChecked(resident.id, 'positive', behavior) ? 'checked' : ''}">
                  <input
                    type="checkbox"
                    checked={isChecked(resident.id, 'positive', behavior)}
                    on:change={() => handleCheckboxToggle(resident.id, 'positive', behavior)}
                  />
                  <span>{behavior}</span>
                </label>
              {/each}
            </div>
          </div>

          <!-- General Notes -->
          <div class="report-section">
            <div class="section-label general">
              <span class="label-bar"></span>
              General Notes
            </div>
            <textarea
              class="text-area"
              value={getFormValue(resident.id, 'general')}
              on:input={(e) => handleTextChange(resident.id, 'general', e.currentTarget.value)}
              placeholder="Daily activities, meals, visitors, etc..."
            ></textarea>
            <div class="checkboxes">
              {#each GENERAL_NOTES as note}
                <label class="checkbox-item {isChecked(resident.id, 'general', note) ? 'checked' : ''}">
                  <input
                    type="checkbox"
                    checked={isChecked(resident.id, 'general', note)}
                    on:change={() => handleCheckboxToggle(resident.id, 'general', note)}
                  />
                  <span>{note}</span>
                </label>
              {/each}
            </div>
          </div>

          <!-- Negative Behaviors -->
          <div class="report-section">
            <div class="section-label negative">
              <span class="label-bar"></span>
              Behavioral Concerns
            </div>
            <textarea
              class="text-area"
              value={getFormValue(resident.id, 'negative')}
              on:input={(e) => handleTextChange(resident.id, 'negative', e.currentTarget.value)}
              placeholder="Document any behavioral issues or concerns..."
            ></textarea>
            <div class="checkboxes">
              {#each NEGATIVE_BEHAVIORS as behavior}
                <label class="checkbox-item {isChecked(resident.id, 'negative', behavior) ? 'checked' : ''}">
                  <input
                    type="checkbox"
                    checked={isChecked(resident.id, 'negative', behavior)}
                    on:change={() => handleCheckboxToggle(resident.id, 'negative', behavior)}
                  />
                  <span>{behavior}</span>
                </label>
              {/each}
            </div>
          </div>
        </div>
      {/each}
    </div>

    <button class="submit-btn" on:click={handleSubmit}>
      Submit Shift Reports
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
  }

  .badge {
    padding: 0.25rem 0.5rem;
    border-radius: 4px;
    font-size: 0.75rem;
    font-weight: 600;
  }

  .county-badge {
    background: #e0e7ff;
    color: #3730a3;
  }

  .level-badge {
    min-width: 45px;
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
    color: #1a3f52;
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

  .section-label.positive .label-bar {
    background: #5a9e7d;
  }

  .section-label.general .label-bar {
    background: #2c5f7c;
  }

  .section-label.negative .label-bar {
    background: #d97847;
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
    border-color: #3d7a9e;
  }

  .checkbox-item input[type="checkbox"] {
    cursor: pointer;
    width: 18px;
    height: 18px;
    accent-color: #2c5f7c;
  }

  .checkbox-item.checked {
    background: #2c5f7c;
    border-color: #2c5f7c;
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