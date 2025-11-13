<script lang="ts">
  import { Plus, Trash2 } from 'lucide-svelte';
  import { createEventDispatcher } from 'svelte';

  export let staff: any[];
  export let newStaffName: string;

  const dispatch = createEventDispatcher();

  function handleAdd() {
    dispatch('addStaff');
  }

  function handleKeydown(e: KeyboardEvent) {
    if (e.key === 'Enter') handleAdd();
  }
</script>

<div class="bg-white rounded-lg shadow-lg p-6 mb-6">
  <div class="flex items-center justify-between mb-4 gap-2">
    <h2 class="text-xl font-bold text-slate-800">Staff Roster</h2>
    <div class="text-sm text-slate-500">Order per schedule is set inside "Assign".</div>
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
      class="px-6 py-2 bg-blue-600 text-white rounded-lg hover:bg-blue-700 flex items-center gap-2 font-medium"
      aria-label="Add staff"
    >
      <Plus class="w-5 h-5" aria-hidden="true" />
      Add Staff
    </button>
  </div>

  <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 2xl:grid-cols-4 gap-2">
    {#each staff as s, index (s.id)}
      <div class="flex items-center gap-2 border border-slate-200 rounded-lg p-2 bg-white">
        <span class="w-6 text-right text-slate-500 text-sm">{index + 1}.</span>
        <input
          type="text"
          value={s.name}
          on:input={(e) => dispatch('updateStaffName', { id: s.id, name: (e.currentTarget as HTMLInputElement).value })}
          class="flex-1 px-2 py-1 border border-slate-300 rounded text-sm focus:ring-2 focus:ring-blue-500 focus:border-transparent"
          aria-label={`Name for ${s.name}`}
        />
        <button
          on:click={() => dispatch('deleteStaff', s.id)}
          class="p-1 text-red-600 hover:bg-red-50 rounded transition-colors"
          aria-label={`Delete ${s.name}`}
        >
          <Trash2 class="w-4 h-4" aria-hidden="true" />
        </button>
      </div>
    {/each}
  </div>
</div>