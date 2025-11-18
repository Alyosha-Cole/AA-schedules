<script lang="ts">
  import { Plus, Trash2, ChevronDown, ChevronRight } from 'lucide-svelte';
  import { createEventDispatcher } from 'svelte';

  export let schedules: any[];
  export let editingOpen: Record<number, boolean>;
  export let newScheduleName: string;

  const dispatch = createEventDispatcher();
  
  let isCollapsed = false;

  function handleAdd() {
    dispatch('addSchedule');
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
        aria-label={isCollapsed ? 'Expand schedule configurations' : 'Collapse schedule configurations'}
      >
        {#if isCollapsed}
          <ChevronRight class="w-6 h-6 text-slate-600" aria-hidden="true" />
        {:else}
          <ChevronDown class="w-6 h-6 text-slate-600" aria-hidden="true" />
        {/if}
      </button>
      <h2 class="text-xl font-bold text-slate-800">Schedule Configurations</h2>
    </div>
  </div>

  {#if !isCollapsed}
    <div class="flex gap-2 mb-4">
      <input
        type="text"
        placeholder="Enter schedule name"
        aria-label="Schedule name"
        bind:value={newScheduleName}
        on:keydown={handleKeydown}
        class="flex-1 px-4 py-2 border border-slate-300 rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-transparent"
      />
      <button
        on:click={handleAdd}
        class="px-6 py-2 bg-green-600 text-white rounded-lg hover:bg-green-700 flex items-center gap-2 font-medium"
        aria-label="Add schedule"
      >
        <Plus class="w-5 h-5" aria-hidden="true" />
        Add Schedule
      </button>
    </div>

    <div class="space-y-2 mb-6">
      {#each schedules as schedule (schedule.id)}
        <div class="flex items-center gap-4 border border-slate-200 rounded-lg p-3 bg-white">
          <input
            type="text"
            value={schedule.name}
            on:input={(e) => dispatch('updateSchedule', { id: schedule.id, field: 'name', value: (e.currentTarget as HTMLInputElement).value })}
            class="flex-1 px-3 py-2 border border-slate-300 rounded font-medium focus:ring-2 focus:ring-blue-500 focus:border-transparent"
            aria-label={`Schedule name for ${schedule.name}`}
          />
          <select
            value={schedule.type}
            on:change={(e) => dispatch('updateSchedule', { id: schedule.id, field: 'type', value: (e.currentTarget as HTMLSelectElement).value })}
            class="px-3 py-2 border border-slate-300 rounded focus:ring-2 focus:ring-blue-500 focus:border-transparent"
            aria-label="Shift length"
          >
            <option value="12-hour">12-Hour Shifts</option>
            <option value="10-hour">10-Hour Shifts</option>
          </select>
          {#if schedule.type === '12-hour'}
            <select
              value={schedule.rotation}
              on:change={(e) => dispatch('updateSchedule', { id: schedule.id, field: 'rotation', value: (e.currentTarget as HTMLSelectElement).value })}
              class="px-3 py-2 border border-slate-300 rounded focus:ring-2 focus:ring-blue-500 focus:border-transparent"
              aria-label="12-hour rotation"
            >
              <option value="2-2-3">2-2-3 Rotation</option>
              <option value="4-3">4-3 Rotation (B: Sun–Tue, A: Wed–Fri; Sat alternates)</option>
            </select>
          {/if}
          <button
            on:click={() => dispatch('deleteSchedule', schedule.id)}
            class="p-2 text-red-600 hover:bg-red-50 rounded transition-colors"
            aria-label={`Delete ${schedule.name}`}
          >
            <Trash2 class="w-5 h-5" aria-hidden="true" />
          </button>
        </div>
      {/each}
    </div>


  {/if}
</div>