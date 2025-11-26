<script lang="ts">
  import { ChevronDown, ChevronRight } from 'lucide-svelte';
  import NotesList from './NotesList.svelte';
  import NotesForm from './NotesForm.svelte';
  import ViewNotes from './ViewNotes.svelte';

  // State
  let currentView: 'entry' | 'view' = 'entry';
  let showResidentManagement = true;

  // Residents per unit
  let residents: Record<string, Array<{
    id: number;
    firstName: string;
    lastName: string;
    county: string;
    level: number;
  }>> = {
    'Unit 1': [],
    'Unit 2': [],
    'Unit 3': []
  };

  // Reports
  let reports: Array<{
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
  }> = [];

  // ID counters per unit
  let idCounters: Record<string, number> = {
    'Unit 1': 0,
    'Unit 2': 0,
    'Unit 3': 0
  };

  function addResident(event: CustomEvent<{ unit: string; firstName: string; lastName: string; county: string }>) {
    const { unit, firstName, lastName, county } = event.detail;
    idCounters[unit] = (idCounters[unit] || 0) + 1;
    
    residents[unit] = [
      ...residents[unit],
      {
        id: idCounters[unit],
        firstName,
        lastName,
        county,
        level: 0
      }
    ];
    
    // Trigger reactivity
    residents = residents;
  }

  function deleteResident(event: CustomEvent<{ unit: string; id: number }>) {
    const { unit, id } = event.detail;
    residents[unit] = residents[unit].filter(r => r.id !== id);
    residents = residents;
  }

  function updateLevel(event: CustomEvent<{ unit: string; id: number; level: number }>) {
    const { unit, id, level } = event.detail;
    residents[unit] = residents[unit].map(r => 
      r.id === id ? { ...r, level } : r
    );
    residents = residents;
  }

  function handleSubmitReports(event: CustomEvent<typeof reports>) {
    reports = [...reports, ...event.detail];
  }
</script>

<div class="notes-container">
  <!-- Header -->
  <div class="header">
    <h1>Shift Report System</h1>
    <p>Document resident activities and behaviors for probation reporting</p>
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
            {#each Object.keys(residents) as unit}
              <NotesList
                {unit}
                residents={residents[unit]}
                on:addResident={addResident}
                on:deleteResident={deleteResident}
                on:updateLevel={updateLevel}
              />
            {/each}
          </div>
        </div>
      {/if}
    </div>

    <!-- Shift Report Entry Form -->
    <NotesForm 
      {residents}
      on:submit={handleSubmitReports}
    />
  {/if}

  <!-- View Reports -->
  {#if currentView === 'view'}
    <ViewNotes {reports} />
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

  .header h1 {
    font-size: 2rem;
    font-weight: 700;
    margin-bottom: 0.5rem;
    position: relative;
  }

  .header p {
    opacity: 0.9;
    font-size: 1rem;
    position: relative;
    margin: 0;
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