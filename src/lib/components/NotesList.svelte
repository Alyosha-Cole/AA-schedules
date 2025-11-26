<script lang="ts">
  import { Trash2, Plus } from 'lucide-svelte';
  import { createEventDispatcher } from 'svelte';

  export let unit: string;
  export let residents: Array<{
    id: number;
    firstName: string;
    lastName: string;
    attributes: Record<string, string>;
  }>;
  export let attributeSettings: Array<{
    id: string;
    name: string;
    type: 'text' | 'cycle' | 'select';
    required: boolean;
    options?: Array<{ value: string; label: string; color: string }>;
    order: number;
    showInHeader: boolean;
  }>;

  const dispatch = createEventDispatcher();

  let newFirstName = '';
  let newLastName = '';
  let newAttributes: Record<string, string> = {};

  // Initialize new attributes when settings change
  $: {
    const current = { ...newAttributes };
    for (const attr of attributeSettings) {
      if (!(attr.id in current)) {
        if (attr.type === 'cycle' || attr.type === 'select') {
          current[attr.id] = attr.options?.[0]?.value || '';
        } else {
          current[attr.id] = '';
        }
      }
    }
    newAttributes = current;
  }

  function addResident() {
    if (!newFirstName.trim() || !newLastName.trim()) return;
    
    // Check required fields
    for (const attr of attributeSettings) {
      if (attr.required && !newAttributes[attr.id]?.trim()) {
        alert(`${attr.name} is required`);
        return;
      }
    }
    
    dispatch('addResident', {
      unit,
      firstName: newFirstName.trim(),
      lastName: newLastName.trim(),
      attributes: { ...newAttributes }
    });
    
    newFirstName = '';
    newLastName = '';
    // Reset to defaults
    const resetAttrs: Record<string, string> = {};
    for (const attr of attributeSettings) {
      if (attr.type === 'cycle' || attr.type === 'select') {
        resetAttrs[attr.id] = attr.options?.[0]?.value || '';
      } else {
        resetAttrs[attr.id] = '';
      }
    }
    newAttributes = resetAttrs;
  }

  function deleteResident(id: number) {
    dispatch('deleteResident', { unit, id });
  }

  function cycleAttribute(residentId: number, attrId: string, currentValue: string) {
    const attr = attributeSettings.find(a => a.id === attrId);
    if (!attr || attr.type !== 'cycle' || !attr.options) return;
    
    const currentIndex = attr.options.findIndex(o => o.value === currentValue);
    const nextIndex = (currentIndex + 1) % attr.options.length;
    const newValue = attr.options[nextIndex].value;
    
    dispatch('updateAttribute', { unit, id: residentId, attrId, value: newValue });
  }

  function getOptionDisplay(attrId: string, value: string) {
    const attr = attributeSettings.find(a => a.id === attrId);
    if (!attr?.options) return { label: value, color: '#6b7280' };
    const option = attr.options.find(o => o.value === value);
    return option || { label: value, color: '#6b7280' };
  }

  function handleKeydown(e: KeyboardEvent) {
    if (e.key === 'Enter') {
      addResident();
    }
  }

  $: sortedAttributes = [...attributeSettings].sort((a, b) => a.order - b.order);
</script>

<div class="resident-roster">
  <div class="roster-header">
    <h3>{unit} Residents</h3>
    <span class="resident-count">{residents.length} resident{residents.length !== 1 ? 's' : ''}</span>
  </div>

  <!-- Add Resident Form -->
  <div class="add-resident-form">
    <div class="form-row">
      <input
        type="text"
        bind:value={newFirstName}
        on:keydown={handleKeydown}
        placeholder="First Name *"
        class="input-field"
      />
      <input
        type="text"
        bind:value={newLastName}
        on:keydown={handleKeydown}
        placeholder="Last Name *"
        class="input-field"
      />
    </div>
    
    <div class="form-row attributes-row">
      {#each sortedAttributes as attr}
        {#if attr.type === 'text'}
          <input
            type="text"
            bind:value={newAttributes[attr.id]}
            on:keydown={handleKeydown}
            placeholder="{attr.name}{attr.required ? ' *' : ''}"
            class="input-field attr-input"
          />
        {:else if attr.type === 'select' || attr.type === 'cycle'}
          <select
            bind:value={newAttributes[attr.id]}
            class="input-field attr-input"
          >
            {#each attr.options || [] as option}
              <option value={option.value}>{option.label}</option>
            {/each}
          </select>
        {/if}
      {/each}
      
      <button class="add-btn" on:click={addResident}>
        <Plus class="w-4 h-4" />
        Add
      </button>
    </div>
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
            
            <!-- Attribute badges -->
            <div class="attribute-badges">
              {#each sortedAttributes.filter(a => a.showInHeader) as attr}
                {@const value = resident.attributes[attr.id] || ''}
                {#if attr.type === 'cycle' && attr.options}
                  {@const display = getOptionDisplay(attr.id, value)}
                  <button 
                    class="attr-badge clickable"
                    style="background-color: {display.color}20; color: {display.color}; border-color: {display.color}40"
                    on:click={() => cycleAttribute(resident.id, attr.id, value)}
                    title="Click to change {attr.name}"
                  >
                    {display.label}
                  </button>
                {:else if attr.type === 'select' && attr.options}
                  {@const display = getOptionDisplay(attr.id, value)}
                  <span 
                    class="attr-badge"
                    style="background-color: {display.color}20; color: {display.color}"
                  >
                    {display.label}
                  </span>
                {:else if value}
                  <span class="attr-badge text-badge">
                    {value}
                  </span>
                {/if}
              {/each}
            </div>
          </div>
          
          <div class="resident-actions">
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
    flex-direction: column;
    gap: 0.5rem;
    margin-bottom: 1rem;
  }

  .form-row {
    display: flex;
    gap: 0.5rem;
    flex-wrap: wrap;
  }

  .attributes-row {
    align-items: center;
  }

  .input-field {
    flex: 1;
    min-width: 100px;
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

  .attr-input {
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
    white-space: nowrap;
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
    gap: 0.75rem;
    flex-wrap: wrap;
  }

  .resident-name {
    font-weight: 600;
    color: #1a3f52;
  }

  .attribute-badges {
    display: flex;
    gap: 0.375rem;
    flex-wrap: wrap;
  }

  .attr-badge {
    padding: 0.25rem 0.5rem;
    border-radius: 4px;
    font-size: 0.7rem;
    font-weight: 600;
  }

  .attr-badge.clickable {
    cursor: pointer;
    border: 1px solid;
    transition: all 0.2s ease;
  }

  .attr-badge.clickable:hover {
    transform: scale(1.05);
    box-shadow: 0 2px 4px rgba(0,0,0,0.1);
  }

  .text-badge {
    background: #e0e7ff;
    color: #3730a3;
  }

  .resident-actions {
    display: flex;
    align-items: center;
    gap: 0.5rem;
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