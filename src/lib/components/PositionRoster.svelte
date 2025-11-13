<script lang="ts">
  import { Plus, Trash2 } from 'lucide-svelte';
  import { createEventDispatcher } from 'svelte';

  export let position: any;
  export let newStaffName: string;

  const dispatch = createEventDispatcher();

  function handleAdd() {
    dispatch('addStaff');
  }

  function handleKeydown(e: KeyboardEvent) {
    if (e.key === 'Enter') handleAdd();
  }

  function getColorClasses(color: string) {
    // Convert hex to tailwind-like classes for borders and backgrounds
    return {
      border: `border-[${color}]`,
      bg: `bg-[${color}]`,
      bgLight: `bg-[${color}]/10`,
      text: `text-[${color}]`
    };
  }
</script>

<div class="bg-white rounded-lg shadow-lg p-6 mb-6">
  <div class="flex items-center justify-between mb-4 gap-2">
    <div class="flex items-center gap-3">
      <div 
        class="w-4 h-4 rounded"
        style="background-color: {position.color}"
      ></div>
      <h2 class="text-xl font-bold text-slate-800">{position.name} Roster</h2>
    </div>
    <div class="text-sm text-slate-500">
      ${position.rate}/hr (OT: ${position.overtimeRate}/hr)
    </div>
  </div>

  <div class="flex gap-2 mb-4">
    <input
      type="text"
      placeholder="Enter staff name"
      aria-label="Staff name"
      bind:value={newStaffName}
      on:keydown={handleKeydown}
      class="flex-1 px-4 py-2 border border-slate-300 rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-transparent"
    />
    <button
      on:click={handleAdd}
      class="px-6 py-2 text-white rounded-lg hover:opacity-90 flex items-center gap-2 font-medium"
      style="background-color: {position.color}"
      aria-label="Add staff"
    >
      <Plus class="w-5 h-5" aria-hidden="true" />
      Add to {position.name}
    </button>
  </div>

  <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 2xl:grid-cols-4 gap-2">
    {#each position.people as person, index (person.id)}
      <div 
        class="flex items-center gap-2 border-2 rounded-lg p-2"
        style="border-color: {position.color}; background-color: {position.color}10"
      >
        <span 
          class="w-6 text-right text-sm font-semibold"
          style="color: {position.color}"
        >
          {index + 1}.
        </span>
        <input
          type="text"
          value={person.name}
          on:input={(e) => dispatch('updateStaff', { positionId: position.id, staffId: person.id, name: (e.currentTarget as HTMLInputElement).value })}
          class="flex-1 px-2 py-1 border border-slate-300 rounded text-sm focus:ring-2 focus:ring-blue-500 focus:border-transparent"
          aria-label={`Name for ${person.name}`}
        />
        <button
          on:click={() => dispatch('deleteStaff', { positionId: position.id, staffId: person.id })}
          class="p-1 text-red-600 hover:bg-red-50 rounded transition-colors"
          aria-label={`Delete ${person.name}`}
        >
          <Trash2 class="w-4 h-4" aria-hidden="true" />
        </button>
      </div>
    {/each}
  </div>
</div>