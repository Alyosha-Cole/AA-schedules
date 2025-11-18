<script lang="ts">
  import { Calendar, Download, Upload, Save, Lock, Trash2, ChevronDown, ChevronRight } from 'lucide-svelte';
  import { createEventDispatcher } from 'svelte';

  export let staffPositions: any[];
  export let startDate: Date;
  export let saveStatus: string;
  export let lastSaved: Date | null;
  export let toInputValue: (d: Date) => string;

  const dispatch = createEventDispatcher();
  
  let isCollapsed = false;
</script>

<div class="bg-white rounded-lg shadow-lg p-6 mb-6">
  <div class="flex items-start justify-between mb-4">
    <div class="flex-1">
      <div class="flex items-center gap-2">
        <button
          on:click={() => isCollapsed = !isCollapsed}
          class="p-2 hover:bg-slate-100 rounded transition-colors"
          aria-expanded={!isCollapsed}
          aria-label={isCollapsed ? 'Expand header' : 'Collapse header'}
        >
          {#if isCollapsed}
            <ChevronRight class="w-6 h-6 text-slate-600" aria-hidden="true" />
          {:else}
            <ChevronDown class="w-6 h-6 text-slate-600" aria-hidden="true" />
          {/if}
        </button>
        <h1 class="text-3xl font-bold text-slate-800 flex items-center gap-2">
          <Calendar class="text-blue-600" aria-hidden="true" />
          Juvenile Facility Staffing Scheduler
        </h1>
      </div>
      {#if !isCollapsed}
        <p class="text-slate-600 ml-14">Compare Different Schedule Configurations</p>
      {/if}
    </div>
    
    <div class="flex flex-col gap-2 items-end">
      <div class="flex gap-2">
        <button
          on:click={() => dispatch('logout')}
          class="px-4 py-2 bg-slate-600 text-white rounded-lg hover:bg-slate-700 flex items-center gap-2 font-medium text-sm"
          title="Logout"
        >
          <Lock class="w-4 h-4" aria-hidden="true" />
          Logout
        </button>
        
        <button
          on:click={() => dispatch('export')}
          class="px-4 py-2 bg-green-600 text-white rounded-lg hover:bg-green-700 flex items-center gap-2 font-medium text-sm"
          title="Export schedule to JSON file"
        >
          <Download class="w-4 h-4" aria-hidden="true" />
          Export JSON
        </button>
        
        <button
          on:click={() => dispatch('import')}
          class="px-4 py-2 bg-blue-600 text-white rounded-lg hover:bg-blue-700 flex items-center gap-2 font-medium text-sm"
          title="Import schedule from JSON file"
        >
          <Upload class="w-4 h-4" aria-hidden="true" />
          Import JSON
        </button>
        
        <button
          on:click={() => dispatch('clearAll')}
          class="px-4 py-2 bg-red-600 text-white rounded-lg hover:bg-red-700 flex items-center gap-2 font-medium text-sm"
          title="Clear all data and reset to defaults"
        >
          <Trash2 class="w-4 h-4" aria-hidden="true" />
          Clear All
        </button>
      </div>
      
      {#if saveStatus}
        <div class="flex items-center gap-2 text-sm text-green-600">
          <Save class="w-4 h-4" aria-hidden="true" />
          <span>{saveStatus}</span>
        </div>
      {:else if lastSaved}
        <div class="text-xs text-slate-500">
          Last saved: {lastSaved.toLocaleTimeString()}
        </div>
      {/if}
    </div>
  </div>
  
  {#if !isCollapsed}
    <div class="mt-2 text-sm text-slate-600 flex gap-4 flex-wrap">
      {#each staffPositions as position, idx}
        <span>
          <span 
            class="inline-block w-3 h-3 rounded-full mr-1"
            style="background-color: {position.color}"
          ></span>
          <strong>{position.name}:</strong> ${position.rate}/hr (OT: ${position.overtimeRate}/hr)
        </span>
        {#if idx < staffPositions.length - 1}
          <span aria-hidden="true">•</span>
        {/if}
      {/each}
    </div>

    <div class="mt-4 flex flex-wrap items-center gap-2">
      <label class="text-sm text-slate-700">2-Week Start:</label>
      <input
        type="date"
        class="px-3 py-2 border border-slate-300 rounded focus:ring-2 focus:ring-blue-500 focus:border-transparent"
        on:change={(e) => dispatch('setStartDate', e)}
        value={toInputValue(startDate)}
        aria-label="Two week start date"
      />
      <button
        class="px-3 py-2 bg-slate-100 rounded border border-slate-300"
        on:click={() => dispatch('shiftPeriod', -14)}
        aria-label="Previous two weeks"
      >
        −14d
      </button>
      <button
        class="px-3 py-2 bg-slate-100 rounded border border-slate-300"
        on:click={() => dispatch('shiftPeriod', +14)}
        aria-label="Next two weeks"
      >
        +14d
      </button>
    </div>
    
    <div class="mt-4 p-3 bg-blue-50 border border-blue-200 rounded-lg">
      <div class="flex items-start gap-2 text-sm text-blue-800">
        <Save class="w-5 h-5 flex-shrink-0 mt-0.5" aria-hidden="true" />
        <div>
          <div class="font-semibold mb-1">💾 Auto-Save Enabled</div>
          <div>Your changes are automatically saved to your browser. Use <strong>Export JSON</strong> to save a copy you can share with others, and <strong>Import JSON</strong> to load saved schedules.</div>
        </div>
      </div>
    </div>
  {/if}
</div>