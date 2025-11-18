<script lang="ts">
  import { Plus, Trash2, ChevronDown, ChevronRight } from 'lucide-svelte';
  import { createEventDispatcher } from 'svelte';

  export let staffPositions: any[];
  export let newPositionName: string;

  const dispatch = createEventDispatcher();
  
  let isCollapsed = false;

  function handleAdd() {
    dispatch('addPosition');
  }

  function handleKeydown(e: KeyboardEvent) {
    if (e.key === 'Enter') handleAdd();
  }
</script>

<div class="bg-white rounded-lg shadow-lg p-6 mb-6">
  <div class="flex items-center justify-between mb-4">
    <div class="flex items-center gap-3">
      <button
        on:click={() => isCollapsed = !isCollapsed}
        class="p-2 hover:bg-slate-100 rounded transition-colors"
        aria-expanded={!isCollapsed}
        aria-label={isCollapsed ? 'Expand staff position types' : 'Collapse staff position types'}
      >
        {#if isCollapsed}
          <ChevronRight class="w-6 h-6 text-slate-600" aria-hidden="true" />
        {:else}
          <ChevronDown class="w-6 h-6 text-slate-600" aria-hidden="true" />
        {/if}
      </button>
      <h2 class="text-xl font-bold text-slate-800">Staff Position Types</h2>
    </div>
  </div>

  {#if !isCollapsed}
    <div class="flex gap-2 mb-4">
      <input
        type="text"
        placeholder="Enter position type name (e.g., Floor Staff, Supervisors, Night Staff)"
        aria-label="Position type name"
        bind:value={newPositionName}
        on:keydown={handleKeydown}
        class="flex-1 px-4 py-2 border border-slate-300 rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-transparent"
      />
      <button
        on:click={handleAdd}
        class="px-6 py-2 bg-green-600 text-white rounded-lg hover:bg-green-700 flex items-center gap-2 font-medium"
        aria-label="Add position type"
      >
        <Plus class="w-5 h-5" aria-hidden="true" />
        Add Position Type
      </button>
    </div>

    <div class="space-y-2">
      {#each staffPositions as position (position.id)}
        <div class="flex items-center gap-4 border border-slate-200 rounded-lg p-3 bg-slate-50">
          <input
            type="text"
            value={position.name}
            on:input={(e) => dispatch('updatePosition', { id: position.id, field: 'name', value: (e.currentTarget as HTMLInputElement).value })}
            class="flex-1 px-3 py-2 border border-slate-300 rounded font-medium focus:ring-2 focus:ring-blue-500 focus:border-transparent"
            aria-label={`Position name for ${position.name}`}
          />
          
          <div class="flex items-center gap-2">
            <label class="text-sm text-slate-600">Rate:</label>
            <span class="text-sm">$</span>
            <input
              type="number"
              min="0"
              step="0.5"
              value={position.rate}
              on:input={(e) => dispatch('updatePosition', { id: position.id, field: 'rate', value: parseFloat((e.currentTarget as HTMLInputElement).value) || 0 })}
              class="w-20 px-2 py-1 border border-slate-300 rounded text-sm focus:ring-2 focus:ring-blue-500"
              aria-label="Hourly rate"
            />
            <span class="text-sm">/hr</span>
          </div>

          <div class="flex items-center gap-2">
            <label class="text-sm text-slate-600">OT Rate:</label>
            <span class="text-sm">$</span>
            <input
              type="number"
              min="0"
              step="0.5"
              value={position.overtimeRate}
              on:input={(e) => dispatch('updatePosition', { id: position.id, field: 'overtimeRate', value: parseFloat((e.currentTarget as HTMLInputElement).value) || 0 })}
              class="w-20 px-2 py-1 border border-slate-300 rounded text-sm focus:ring-2 focus:ring-blue-500"
              aria-label="Overtime rate"
            />
            <span class="text-sm">/hr</span>
          </div>

          <div class="flex items-center gap-2">
            <label class="text-sm text-slate-600">Color:</label>
            <input
              type="color"
              value={position.color}
              on:input={(e) => dispatch('updatePosition', { id: position.id, field: 'color', value: (e.currentTarget as HTMLInputElement).value })}
              class="w-12 h-8 border border-slate-300 rounded cursor-pointer"
              aria-label="Position color"
            />
          </div>

          <div class="text-sm text-slate-600">
            {position.people.length} staff
          </div>

          <button
            on:click={() => dispatch('deletePosition', position.id)}
            class="p-2 text-red-600 hover:bg-red-50 rounded transition-colors"
            aria-label={`Delete ${position.name}`}
            disabled={staffPositions.length <= 1}
          >
            <Trash2 class="w-5 h-5" aria-hidden="true" />
          </button>
        </div>
      {/each}
    </div>

    <div class="mt-4 p-3 bg-amber-50 border border-amber-200 rounded-lg">
      <div class="text-sm text-amber-800">
        <span class="font-semibold">💡 Note:</span> Each position type will have its own roster below. You can add staff to each position and set different rates for different roles (e.g., Floor Staff at $25/hr, Supervisors at $30/hr).
      </div>
    </div>
  {/if}
</div>