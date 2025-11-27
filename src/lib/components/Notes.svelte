<script lang="ts">
  import { ChevronDown, ChevronRight, Settings, ChevronLeft, Plus } from 'lucide-svelte';
  import NotesList from './NotesList.svelte';
  import NotesForm from './NotesForm.svelte';
  import ViewNotes from './ViewNotes.svelte';
  import AdvancedResidentSettings from './AdvancedResidentSettings.svelte';

  // State
  let currentView: 'entry' | 'view' = 'entry';
  let showResidentManagement = true;
  let showAdvancedSettings = false;

  // Unit names - now dynamic
  let unitNames = ['Unit 1', 'Unit 2', 'Unit 3'];

  // Default attribute settings
  function getDefaultAttributes() {
    return [
      {
        id: 'county',
        name: 'County',
        type: 'text' as const,
        required: true,
        order: 0,
        showInHeader: true
      },
      {
        id: 'level',
        name: 'Level',
        type: 'cycle' as const,
        required: true,
        options: [
          { value: '0', label: 'Lvl 0', color: '#dc2626' },
          { value: '1', label: 'Lvl 1', color: '#f59e0b' },
          { value: '2', label: 'Lvl 2', color: '#16a34a' },
          { value: '3', label: 'Lvl 3', color: '#2563eb' }
        ],
        order: 1,
        showInHeader: true
      }
    ];
  }

  // Default note categories
  function getDefaultNoteCategories() {
    return [
      {
        id: 'positive',
        name: 'Positive Behaviors',
        color: '#16a34a',
        order: 0,
        checkboxes: [
          { id: 'cb1', label: 'Participated in group activities' },
          { id: 'cb2', label: 'Showed respect to staff' },
          { id: 'cb3', label: 'Helped another resident' },
          { id: 'cb4', label: 'Completed assigned tasks' },
          { id: 'cb5', label: 'Displayed positive attitude' }
        ]
      },
      {
        id: 'general',
        name: 'General Notes',
        color: '#2563eb',
        order: 1,
        checkboxes: [
          { id: 'cb6', label: 'Ate all meals' },
          { id: 'cb7', label: 'Attended classes' },
          { id: 'cb8', label: 'Participated in recreation' },
          { id: 'cb9', label: 'Took medications' },
          { id: 'cb10', label: 'Had visitor' }
        ]
      },
      {
        id: 'negative',
        name: 'Behavioral Concerns',
        color: '#d97706',
        order: 2,
        checkboxes: [
          { id: 'cb11', label: 'Refused to follow directions' },
          { id: 'cb12', label: 'Verbal altercation with peer' },
          { id: 'cb13', label: 'Disruptive in class' },
          { id: 'cb14', label: 'Property damage' },
          { id: 'cb15', label: 'Left designated area without permission' }
        ]
      }
    ];
  }

  // Resident settings per unit
  let residentSettings: Record<string, {
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

  // Initialize settings for each unit
  function initializeUnitSettings() {
    for (const unit of unitNames) {
      if (!residentSettings[unit]) {
        residentSettings[unit] = {
          inheritFrom: null,
          attributes: getDefaultAttributes()
        };
      }
    }
    // Initialize residents too
    for (const unit of unitNames) {
      if (!residents[unit]) {
        residents[unit] = [];
      }
      if (!idCounters[unit]) {
        idCounters[unit] = 0;
      }
    }
  }

  // Note settings (categories and checkboxes)
  let noteSettings: {
    categories: Array<{
      id: string;
      name: string;
      color: string;
      order: number;
      checkboxes: Array<{ id: string; label: string }>;
    }>;
  } = {
    categories: getDefaultNoteCategories()
  };

  // Residents per unit
  let residents: Record<string, Array<{
    id: number;
    firstName: string;
    lastName: string;
    attributes: Record<string, string>;
  }>> = {};

  // Reports
  let reports: Array<{
    residentId: number;
    firstName: string;
    lastName: string;
    attributes: Record<string, string>;
    unit: string;
    timestamp: string;
    notes: Record<string, string>;
  }> = [];

  // ID counters per unit
  let idCounters: Record<string, number> = {};

  // Initialize on load
  initializeUnitSettings();

  // Get effective attributes for a unit
  function getEffectiveAttributes(unit: string) {
    const settings = residentSettings[unit];
    if (!settings) return [];
    if (settings.inheritFrom && residentSettings[settings.inheritFrom]) {
      return residentSettings[settings.inheritFrom].attributes || [];
    }
    return settings.attributes || [];
  }

  function addResident(event: CustomEvent<{ unit: string; firstName: string; lastName: string; attributes: Record<string, string> }>) {
    const { unit, firstName, lastName, attributes } = event.detail;
    idCounters[unit] = (idCounters[unit] || 0) + 1;
    
    residents[unit] = [
      ...residents[unit],
      {
        id: idCounters[unit],
        firstName,
        lastName,
        attributes
      }
    ];
    
    residents = residents;
  }

  function deleteResident(event: CustomEvent<{ unit: string; id: number }>) {
    const { unit, id } = event.detail;
    residents[unit] = residents[unit].filter(r => r.id !== id);
    residents = residents;
  }

  function updateAttribute(event: CustomEvent<{ unit: string; id: number; attrId: string; value: string }>) {
    const { unit, id, attrId, value } = event.detail;
    residents[unit] = residents[unit].map(r => 
      r.id === id 
        ? { ...r, attributes: { ...r.attributes, [attrId]: value } }
        : r
    );
    residents = residents;
  }

  function handleSubmitReports(event: CustomEvent<typeof reports>) {
    reports = [...reports, ...event.detail];
  }

  function handleResidentSettingsChanged(event: CustomEvent<typeof residentSettings>) {
    residentSettings = event.detail;
  }

  function handleNoteSettingsChanged(event: CustomEvent<typeof noteSettings>) {
    noteSettings = event.detail;
  }

  function handleUnitsChanged(event: CustomEvent<string[]>) {
    const oldUnits = unitNames;
    const newUnits = event.detail;
    
    unitNames = newUnits;
    
    // Clean up residents for deleted units
    for (const unit of oldUnits) {
      if (!newUnits.includes(unit)) {
        delete residents[unit];
        delete idCounters[unit];
      }
    }
    
    // Initialize new units
    initializeUnitSettings();
    
    // Force reactivity
    residents = { ...residents };
    residentSettings = { ...residentSettings };
  }
</script>

<div class="notes-container">
  <!-- Header -->
  <div class="header">
    <div class="header-content">
      <div>
        <h1>Shift Report System</h1>
        <p>Document resident activities and behaviors for probation reporting</p>
      </div>
      <button 
        class="settings-btn"
        on:click={() => showAdvancedSettings = true}
        title="Configure resident attributes and note categories"
      >
        <Settings class="w-5 h-5" />
        Settings
      </button>
    </div>
  </div>

  <!-- Navigation -->
  <div class="nav-buttons">
    <button 
      class="nav-btn {currentView === 'entry' ? 'active' : ''}"
      on:click={() => currentView = 'entry'}
    >
      📝 Create Shift Reports
    </button>
    <button 
      class="nav-btn {currentView === 'view' ? 'active' : ''}"
      on:click={() => currentView = 'view'}
    >
      📊 View Reports
    </button>
  </div>

  <!-- Entry View -->
  {#if currentView === 'entry'}
    <!-- Resident Management Section (Collapsible) -->
    <div class="management-section">
      <button 
        class="management-toggle"
        on:click={() => showResidentManagement = !showResidentManagement}
      >
        {#if showResidentManagement}
          <ChevronDown class="w-5 h-5" />
        {:else}
          <ChevronRight class="w-5 h-5" />
        {/if}
        <span>Manage Residents</span>
        <span class="resident-total">
          ({Object.values(residents).reduce((sum, arr) => sum + arr.length, 0)} total)
        </span>
      </button>
      
      {#if showResidentManagement}
        <div class="management-content">
          <div class="units-grid">
            {#each unitNames as unit}
              <NotesList
                {unit}
                residents={residents[unit]}
                attributeSettings={getEffectiveAttributes(unit)}
                on:addResident={addResident}
                on:deleteResident={deleteResident}
                on:updateAttribute={updateAttribute}
              />
            {/each}
          </div>
        </div>
      {/if}
    </div>

    <!-- Shift Report Entry Form -->
    <NotesForm 
      {residents}
      {residentSettings}
      {noteSettings}
      on:submit={handleSubmitReports}
    />
  {/if}

  <!-- View Reports -->
  {#if currentView === 'view'}
    <ViewNotes {reports} {residentSettings} {noteSettings} />
  {/if}

  <!-- Advanced Settings Modal -->
  {#if showAdvancedSettings}
    <AdvancedResidentSettings
      units={unitNames}
      {residentSettings}
      {noteSettings}
      on:close={() => showAdvancedSettings = false}
      on:residentSettingsChanged={handleResidentSettingsChanged}
      on:noteSettingsChanged={handleNoteSettingsChanged}
      on:unitsChanged={handleUnitsChanged}
    />
  {/if}
</div>

<style>
  .notes-container {
    max-width: 1400px;
    margin: 0 auto;
  }

  .header {
    background: linear-gradient(135deg, #2c5f7c 0%, #3d7a9e 100%);
    color: white;
    padding: 2rem 2.5rem;
    border-radius: 16px;
    margin-bottom: 2rem;
    box-shadow: 0 8px 24px rgba(44, 95, 124, 0.15);
    position: relative;
    overflow: hidden;
  }

  .header::before {
    content: '';
    position: absolute;
    top: 0;
    right: 0;
    width: 300px;
    height: 300px;
    background: radial-gradient(circle, rgba(255,255,255,0.1) 0%, transparent 70%);
    border-radius: 50%;
    transform: translate(30%, -30%);
  }

  .header-content {
    display: flex;
    justify-content: space-between;
    align-items: flex-start;
    position: relative;
  }

  .header h1 {
    font-size: 2rem;
    font-weight: 700;
    margin-bottom: 0.5rem;
  }

  .header p {
    opacity: 0.9;
    font-size: 1rem;
    margin: 0;
  }

  .settings-btn {
    display: flex;
    align-items: center;
    gap: 0.5rem;
    padding: 0.75rem 1.25rem;
    background: rgba(255, 255, 255, 0.15);
    color: white;
    border: 1px solid rgba(255, 255, 255, 0.3);
    border-radius: 8px;
    font-weight: 600;
    cursor: pointer;
    transition: all 0.2s ease;
  }

  .settings-btn:hover {
    background: rgba(255, 255, 255, 0.25);
    transform: translateY(-1px);
  }

  .nav-buttons {
    display: flex;
    gap: 1rem;
    margin-bottom: 2rem;
  }

  .nav-btn {
    flex: 1;
    padding: 1rem 2rem;
    border: 2px solid #2c5f7c;
    background: white;
    color: #2c5f7c;
    font-size: 1rem;
    font-weight: 600;
    border-radius: 12px;
    cursor: pointer;
    transition: all 0.3s ease;
  }

  .nav-btn:hover {
    transform: translateY(-2px);
    box-shadow: 0 4px 12px rgba(44, 95, 124, 0.15);
    background: #2c5f7c;
    color: white;
  }

  .nav-btn.active {
    background: #2c5f7c;
    color: white;
  }

  .management-section {
    background: white;
    border-radius: 16px;
    margin-bottom: 2rem;
    box-shadow: 0 4px 16px rgba(44, 95, 124, 0.08);
    overflow: hidden;
  }

  .management-toggle {
    width: 100%;
    padding: 1.25rem 1.5rem;
    background: linear-gradient(135deg, #f8fafc 0%, #f1f5f9 100%);
    border: none;
    cursor: pointer;
    display: flex;
    align-items: center;
    gap: 0.75rem;
    font-size: 1.1rem;
    font-weight: 600;
    color: #1a3f52;
    transition: all 0.2s ease;
  }

  .management-toggle:hover {
    background: linear-gradient(135deg, #f1f5f9 0%, #e2e8f0 100%);
  }

  .resident-total {
    font-size: 0.9rem;
    font-weight: 500;
    color: #6b7280;
  }

  .management-content {
    padding: 1.5rem;
    border-top: 1px solid #e2e8f0;
  }

  .units-grid {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(350px, 1fr));
    gap: 1.5rem;
  }
</style>