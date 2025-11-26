<script lang="ts">
  import { Trash2, Plus } from 'lucide-svelte';
  import { createEventDispatcher } from 'svelte';

  export let unit: string;
  export let residents: Array<{
    id: number;
    firstName: string;
    lastName: string;
    county: string;
    level: number;
  }>;

  const dispatch = createEventDispatcher();

  let newFirstName = '';
  let newLastName = '';
  let newCounty = '';

  function addResident() {
    if (!newFirstName.trim() || !newLastName.trim()) return;
    
    dispatch('addResident', {
      unit,
      firstName: newFirstName.trim(),
      lastName: newLastName.trim(),
      county: newCounty.trim() || 'Unknown'
    });
    
    newFirstName = '';
    newLastName = '';
    newCounty = '';
  }

  function deleteResident(id: number) {
    dispatch('deleteResident', { unit, id });
  }

  function cycleLevel(id: number, currentLevel: number) {
    const newLevel = (currentLevel + 1) % 4;
    dispatch('updateLevel', { unit, id, level: newLevel });
  }

  function handleKeydown(e: KeyboardEvent) {
    if (e.key === 'Enter') {
      addResident();
    }
  }
</script>

<div class="resident-roster">
  <div class="roster-header">
    <h3>{unit} Residents</h3>
    <span class="resident-count">{residents.length} resident{residents.length !== 1 ? 's' : ''}</span>
  </div>

  <!-- Add Resident Form -->
  <div class="add-resident-form">
    <input
      type="text"
      bind:value={newFirstName}
      on:keydown={handleKeydown}
      placeholder="First Name"
      class="input-field"
    />
    <input
      type="text"
      bind:value={newLastName}
      on:keydown={handleKeydown}
      placeholder="Last Name"
      class="input-field"
    />
    <input
      type="text"
      bind:value={newCounty}
      on:keydown={handleKeydown}
      placeholder="County"
      class="input-field county-input"
    />
    <button class="add-btn" on:click={addResident}>
      <Plus class="w-4 h-4" />
      Add
    </button>
  </div>

  <!-- Resident List -->
  {#if residents.length === 0}
    <div class="empty-list">
      No residents added to {unit} yet.
    </div>
  {:else}
    <div class="resident-list">
      {#each residents as resident (resident.id)}
        <div class="resident-item">
          <div class="resident-info">
            <span class="resident-name">
              {resident.firstName} {resident.lastName.charAt(0)}.
            </span>
            <span class="resident-county">({resident.county})</span>
          </div>
          
          <div class="resident-actions">
            <button 
              class="level-btn level-{resident.level}"
              on:click={() => cycleLevel(resident.id, resident.level)}
              title="Click to change level"
            >
              Lvl {resident.level}
            </button>
            
            <button 
              class="delete-btn"
              on:click={() => deleteResident(resident.id)}
              title="Delete resident"
            >
              <Trash2 class="w-4 h-4" />
            </button>
          </div>
        </div>
      {/each}
    </div>
  {/if}
</div>

<style>
  .resident-roster {
    background: white;
    border-radius: 12px;
    padding: 1.5rem;
    box-shadow: 0 2px 8px rgba(44, 95, 124, 0.08);
    margin-bottom: 1.5rem;
  }

  .roster-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 1rem;
    padding-bottom: 0.75rem;
    border-bottom: 2px solid #e1e4e8;
  }

  .roster-header h3 {
    font-size: 1.25rem;
    font-weight: 700;
    color: #1a3f52;
    margin: 0;
  }

  .resident-count {
    font-size: 0.875rem;
    color: #6b7280;
    background: #f3f4f6;
    padding: 0.25rem 0.75rem;
    border-radius: 9999px;
  }

  .add-resident-form {
    display: flex;
    gap: 0.5rem;
    margin-bottom: 1rem;
    flex-wrap: wrap;
  }

  .input-field {
    flex: 1;
    min-width: 120px;
    padding: 0.625rem 0.875rem;
    border: 2px solid #e1e4e8;
    border-radius: 8px;
    font-size: 0.875rem;
    transition: all 0.2s ease;
  }

  .input-field:focus {
    outline: none;
    border-color: #2c5f7c;
    box-shadow: 0 0 0 3px rgba(44, 95, 124, 0.1);
  }

  .county-input {
    max-width: 150px;
  }

  .add-btn {
    display: flex;
    align-items: center;
    gap: 0.375rem;
    padding: 0.625rem 1rem;
    background: #5a9e7d;
    color: white;
    border: none;
    border-radius: 8px;
    font-weight: 600;
    font-size: 0.875rem;
    cursor: pointer;
    transition: all 0.2s ease;
  }

  .add-btn:hover {
    background: #4a8d6a;
    transform: translateY(-1px);
  }

  .empty-list {
    text-align: center;
    padding: 2rem;
    color: #6b7280;
    font-style: italic;
    background: #f8f9fa;
    border-radius: 8px;
  }

  .resident-list {
    display: flex;
    flex-direction: column;
    gap: 0.5rem;
  }

  .resident-item {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 0.75rem 1rem;
    background: #f8f9fa;
    border-radius: 8px;
    transition: all 0.2s ease;
  }

  .resident-item:hover {
    background: #f1f5f9;
  }

  .resident-info {
    display: flex;
    align-items: center;
    gap: 0.5rem;
  }

  .resident-name {
    font-weight: 600;
    color: #1a3f52;
  }

  .resident-county {
    font-size: 0.875rem;
    color: #6b7280;
  }

  .resident-actions {
    display: flex;
    align-items: center;
    gap: 0.5rem;
  }

  .level-btn {
    padding: 0.375rem 0.75rem;
    border: none;
    border-radius: 6px;
    font-weight: 600;
    font-size: 0.75rem;
    cursor: pointer;
    transition: all 0.2s ease;
    min-width: 50px;
  }

  .level-btn:hover {
    transform: scale(1.05);
  }

  .level-btn.level-0 {
    background: #fee2e2;
    color: #991b1b;
  }

  .level-btn.level-1 {
    background: #fef3c7;
    color: #92400e;
  }

  .level-btn.level-2 {
    background: #d1fae5;
    color: #065f46;
  }

  .level-btn.level-3 {
    background: #dbeafe;
    color: #1e40af;
  }

  .delete-btn {
    display: flex;
    align-items: center;
    justify-content: center;
    padding: 0.375rem;
    background: transparent;
    color: #dc2626;
    border: none;
    border-radius: 6px;
    cursor: pointer;
    transition: all 0.2s ease;
  }

  .delete-btn:hover {
    background: #fee2e2;
  }
</style>