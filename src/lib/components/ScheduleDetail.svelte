<script lang="ts">
  import { ChevronDown, ChevronRight, Trash2, Plus } from 'lucide-svelte';
  import { createEventDispatcher } from 'svelte';

  export let schedule: any;
  export let staff: any[];
  export let dates: any[];
  export let days: string[];
  export let requiredStaff: number;
  export let regularRate: number;
  export let overtimeRate: number;
  export let isEditing: boolean;
  export let isCollapsed: boolean;
  export let generateSchedule: (schedule: any, staffId: number) => number[];
  export let generateScheduleForAssignment: (schedule: any, assignment: any) => number[];
  export let calculateScheduleCosts: (schedule: any) => any;
  export let getDaysOffPattern: (daysOff: number[]) => string;

  const dispatch = createEventDispatcher();

  $: costs = calculateScheduleCosts(schedule);
  $: minCoverage = schedule.type === '12-hour'
    ? Math.min(...costs.dailyCoverage)
    : Math.min(Math.min(...costs.amCoverage), Math.min(...costs.pmCoverage));
  $: assignedStaff = Object.keys(schedule.assignments).length;
  $: minStaffNeeded = requiredStaff * 2;

  const daysOffOptions = [
    { value: 'sun-mon', label: 'Sun-Mon' },
    { value: 'mon-tue', label: 'Mon-Tue' },
    { value: 'tue-wed', label: 'Tue-Wed' },
    { value: 'wed-thu', label: 'Wed-Thu' },
    { value: 'thu-fri', label: 'Thu-Fri' },
    { value: 'fri-sat', label: 'Fri-Sat' },
    { value: 'sat-sun', label: 'Sat-Sun' }
  ];

  let dragItem: { scheduleId: number; id: string | number } | null = null;
  let dragOverIndex: number | null = null;

  function onDragStartRow(scheduleId: number, id: string | number, e: DragEvent) {
    dragItem = { scheduleId, id };
    e.dataTransfer!.effectAllowed = 'move';
    e.dataTransfer!.setData('text/plain', String(id));
  }

  function onDragOverRow(idx: number, e: DragEvent) {
    e.preventDefault();
    e.dataTransfer!.dropEffect = 'move';
    dragOverIndex = idx;
  }

  function onDragLeaveRow() {
    dragOverIndex = null;
  }

  function onDropRow(scheduleId: number, id: string | number, e: DragEvent) {
    e.preventDefault();
    dragOverIndex = null;
    if (dragItem && dragItem.scheduleId === scheduleId && dragItem.id !== id) {
      dispatch('reorderList', { scheduleId, fromId: dragItem.id, toId: id });
    }
    dragItem = null;
  }

  function onDragEndRow(e: DragEvent) {
    dragOverIndex = null;
    dragItem = null;
  }
</script>

<style>
  .compact-select {
    appearance: none;
    -webkit-appearance: none;
    -moz-appearance: none;
    background-image: none;
    padding-right: 0.25rem;
    padding-left: 0.25rem;
  }
  
  .drag-over {
    border-top: 3px solid #3b82f6;
  }
  
  .dragging {
    opacity: 0.5;
  }
</style>

<div class="bg-white rounded-lg shadow-lg p-6 mb-6">
  <div class="flex items-center justify-between mb-4 pb-4 border-b-2 border-slate-200">
    <div class="flex items-center gap-3">
      <button
        on:click={() => dispatch('toggleCollapse', schedule.id)}
        class="p-2 hover:bg-slate-100 rounded transition-colors"
        aria-expanded={!isCollapsed}
        aria-label={isCollapsed ? 'Expand schedule' : 'Collapse schedule'}
      >
        {#if isCollapsed}
          <ChevronRight class="w-6 h-6 text-slate-600" aria-hidden="true" />
        {:else}
          <ChevronDown class="w-6 h-6 text-slate-600" aria-hidden="true" />
        {/if}
      </button>
      
      <div class="flex flex-col gap-1">
        <h2 class="text-2xl font-bold text-slate-800">{schedule.name}</h2>
        <div class="text-sm text-slate-600">
          Assigned (real): {assignedStaff} / {minStaffNeeded} needed
        </div>
      </div>
    </div>

    <div class="flex items-center gap-4">
      <div class="flex gap-2">
        {#if schedule.type === '12-hour'}
          <button
            class="px-3 py-2 bg-amber-600 text-white rounded hover:bg-amber-700 text-sm flex items-center gap-1"
            on:click={() => dispatch('addSimStaff', { scheduleId: schedule.id, kind: 'A' })}
          >
            <Plus class="w-4 h-4" aria-hidden="true" />
            Add SIM Team A
          </button>
          <button
            class="px-3 py-2 bg-amber-600 text-white rounded hover:bg-amber-700 text-sm flex items-center gap-1"
            on:click={() => dispatch('addSimStaff', { scheduleId: schedule.id, kind: 'B' })}
          >
            <Plus class="w-4 h-4" aria-hidden="true" />
            Add SIM Team B
          </button>
        {:else}
          <button
            class="px-3 py-2 bg-amber-600 text-white rounded hover:bg-amber-700 text-sm flex items-center gap-1"
            on:click={() => dispatch('addSimStaff', { scheduleId: schedule.id, kind: 'AM' })}
          >
            <Plus class="w-4 h-4" aria-hidden="true" />
            Add SIM AM
          </button>
          <button
            class="px-3 py-2 bg-amber-600 text-white rounded hover:bg-amber-700 text-sm flex items-center gap-1"
            on:click={() => dispatch('addSimStaff', { scheduleId: schedule.id, kind: 'PM' })}
          >
            <Plus class="w-4 h-4" aria-hidden="true" />
            Add SIM PM
          </button>
        {/if}
      </div>

      <div class="flex gap-4">
        <div class="text-right">
          <div class="text-3xl font-bold text-green-600">${costs.totalCost.toLocaleString()}</div>
          <div class="text-sm text-slate-600">Total 2-Week Cost</div>
          {#if costs.gapOTHours > 0}
            <div class="text-xs text-orange-600 mt-1">(includes ${costs.gapOTCost.toLocaleString()} gap OT)</div>
          {/if}
        </div>
        <div class="text-right">
          <div class="text-3xl font-bold text-blue-600">{costs.totalHours}</div>
          <div class="text-sm text-slate-600">Scheduled Hours</div>
        </div>
        <div class="text-right">
          <div class={`text-3xl font-bold ${minCoverage >= requiredStaff ? 'text-emerald-600' : 'text-red-600'}`}>
            {minCoverage}/{requiredStaff}
          </div>
          <div class="text-sm text-slate-600">
            {schedule.type === '12-hour' ? 'Min Coverage (any day)' : 'Min AM/PM Coverage'}
          </div>
        </div>
      </div>
    </div>
  </div>

  {#if !isCollapsed}
    <div class="mb-3 flex items-center justify-between">
      <div class="p-3 bg-blue-50 border border-blue-200 rounded-lg flex-1 mr-2">
        <div class="flex items-center gap-2 text-sm text-blue-800">
          <span class="font-semibold">💡 Tip:</span>
          <span>Click any cell to toggle ON/OFF. Use dropdowns to change assignments. Drag rows to reorder.</span>
        </div>
      </div>
      <button
        on:click={() => dispatch('resetManualEdits', schedule.id)}
        class="px-4 py-3 bg-orange-500 text-white rounded-lg hover:bg-orange-600 transition-colors text-sm font-medium whitespace-nowrap"
      >
        Reset All Edits
      </button>
    </div>

    <div class="overflow-x-auto">
      <table class="w-full border-separate border-spacing-0 text-sm">
        <thead>
          <tr>
            <th class="border border-slate-300 px-3 py-2 text-left font-semibold sticky left-0 bg-slate-100 z-50 top-[var(--sticky-top)]">#</th>
            <th class="border border-slate-300 px-3 py-2 text-left font-semibold sticky left-[3.5rem] bg-slate-100 z-50 top-[var(--sticky-top)]">Staff</th>
            {#each dates as d}
              <th class="border border-slate-300 px-2 py-2 text-center min-w-[56px] sticky top-[var(--sticky-top)] bg-slate-100 z-40">
                <div class="text-xs font-semibold">{d.date}</div>
                <div class="text-xs text-slate-600">{d.dayName}</div>
              </th>
            {/each}
            <th class="border border-slate-300 px-3 py-2 text-center font-semibold sticky top-[var(--sticky-top)] bg-slate-100 z-40">Hours</th>
            <th class="border border-slate-300 px-3 py-2 text-center font-semibold sticky top-[var(--sticky-top)] bg-slate-100 z-40">Cost</th>
          </tr>
        </thead>
        <tbody>
          {#each costs.staffDetails as detail, idx (detail.id)}
            {@const shiftLabel = schedule.type === '12-hour'
              ? '8:30a-8:30p'
              : (detail.shift === 'AM' ? '7a-5p' : '1p-11p')}
            {@const isSim = String(detail.id).startsWith('sim-')}
            {@const assignment = !isSim ? schedule.assignments[detail.id] : schedule.simStaff.find(s => s.id === detail.id)}
            {@const isDragging = dragItem?.id === detail.id}
            {@const isDropTarget = dragOverIndex === idx}
            
            <tr 
              class={`hover:bg-slate-50 ${detail.simulated ? 'bg-amber-50' : ''} cursor-grab active:cursor-grabbing ${isDragging ? 'dragging' : ''} ${isDropTarget ? 'drag-over' : ''}`}
              draggable="true"
              on:dragstart={(e) => onDragStartRow(schedule.id, detail.id, e)}
              on:dragover={(e) => onDragOverRow(idx, e)}
              on:dragleave={onDragLeaveRow}
              on:drop={(e) => onDropRow(schedule.id, detail.id, e)}
              on:dragend={onDragEndRow}
            >
              <td class="border border-slate-300 px-3 py-2 font-medium sticky left-0 bg-white w-14 text-right text-slate-500 z-20">
                {idx + 1}.
              </td>
              
              <td class="border border-slate-300 px-3 py-2 sticky left-[3.5rem] bg-white z-20">
                <div class="flex items-center justify-between gap-2 mb-1">
                  <div class="flex items-center gap-2 flex-1 min-w-0">
                    <span class="font-medium truncate">{detail.name}</span>
                    {#if detail.simulated}
                      <span class="text-xs bg-amber-600 text-white px-1.5 py-0.5 rounded flex-shrink-0">SIM</span>
                      <button
                        on:click={() => dispatch('deleteSimStaff', { scheduleId: schedule.id, simId: detail.id })}
                        class="p-0.5 text-red-600 hover:bg-red-50 rounded flex-shrink-0"
                        title="Delete simulated staff"
                      >
                        <Trash2 class="w-3 h-3" aria-hidden="true" />
                      </button>
                    {/if}
                    {#if detail.isEdited}
                      <span class="text-xs bg-yellow-400 text-yellow-900 px-1.5 py-0.5 rounded flex-shrink-0">EDITED</span>
                    {/if}
                  </div>
                </div>
                
                {#if schedule.type === '12-hour'}
                  <select
                    class="compact-select w-16 py-0.5 border border-slate-300 rounded text-xs focus:ring-1 focus:ring-blue-500"
                    value={assignment?.team || 'A'}
                    on:change={(e) => {
                      const val = (e.currentTarget as HTMLSelectElement).value;
                      if (isSim) {
                        dispatch('updateSimStaff', { scheduleId: schedule.id, simId: detail.id, field: 'team', value: val });
                      } else {
                        dispatch('updateAssignment', { scheduleId: schedule.id, staffId: detail.id, field: 'team', value: val });
                      }
                    }}
                  >
                    <option value="A">A</option>
                    <option value="B">B</option>
                  </select>
                {:else}
                  <div class="flex gap-1">
                    <select
                      class="compact-select w-12 py-0.5 border border-slate-300 rounded text-xs focus:ring-1 focus:ring-blue-500"
                      value={assignment?.shift || 'AM'}
                      on:change={(e) => {
                        const val = (e.currentTarget as HTMLSelectElement).value;
                        if (isSim) {
                          dispatch('updateSimStaff', { scheduleId: schedule.id, simId: detail.id, field: 'shift', value: val });
                        } else {
                          dispatch('updateAssignment', { scheduleId: schedule.id, staffId: detail.id, field: 'shift', value: val });
                        }
                      }}
                    >
                      <option value="AM">AM</option>
                      <option value="PM">PM</option>
                    </select>
                    <select
                      class="compact-select w-20 py-0.5 border border-slate-300 rounded text-xs focus:ring-1 focus:ring-blue-500"
                      value={getDaysOffPattern(assignment?.daysOff || [6, 0])}
                      on:change={(e) => {
                        const val = (e.currentTarget as HTMLSelectElement).value;
                        if (isSim) {
                          dispatch('updateSimDaysOffPattern', { scheduleId: schedule.id, simId: detail.id, pattern: val });
                        } else {
                          dispatch('updateDaysOffPattern', { scheduleId: schedule.id, staffId: detail.id, pattern: val });
                        }
                      }}
                    >
                      {#each daysOffOptions as option}
                        <option value={option.value}>{option.label}</option>
                      {/each}
                    </select>
                  </div>
                {/if}
              </td>

              {#each detail.schedule as working, dIdx}
                <td
                  class={`border border-slate-300 px-1 py-2 text-center cursor-pointer transition-colors ${
                    working ? 'bg-green-100 hover:bg-green-200' : 'bg-red-50 hover:bg-red-100'
                  }`}
                  title={`Click to toggle ${dates[dIdx].dayName} (${working ? 'ON' : 'OFF'})`}
                  on:click={() => {
                    if (String(detail.id).startsWith('sim-')) {
                      dispatch('toggleScheduleDayForSim', { scheduleId: schedule.id, simId: detail.id, dayIndex: dIdx });
                    } else {
                      dispatch('toggleScheduleDay', { scheduleId: schedule.id, staffId: detail.id, dayIndex: dIdx });
                    }
                  }}
                >
                  {#if working}
                    <div class="text-xs font-semibold text-green-800">{shiftLabel}</div>
                  {:else}
                    <div class="text-xs text-red-600">OFF</div>
                  {/if}
                </td>
              {/each}

              <td class="border border-slate-300 px-3 py-2 text-center font-semibold">
                {detail.hours}
                {#if detail.overtimeHours > 0}
                  <div class="text-xs text-orange-600">
                    ({detail.regularHours} reg + {detail.overtimeHours} OT)
                  </div>
                {/if}
              </td>
              <td class="border border-slate-300 px-3 py-2 text-center font-semibold text-green-700">
                ${detail.cost.toLocaleString()}
                {#if detail.overtimeHours > 0}
                  <div class="text-xs text-slate-600">
                    (${(detail.regularHours * regularRate).toLocaleString()} + ${(detail.overtimeHours * overtimeRate).toLocaleString()})
                  </div>
                {/if}
              </td>
            </tr>
          {/each}

          {#if schedule.type === '12-hour'}
            <tr class="bg-slate-100 font-bold">
              <td class="border border-slate-300 px-3 py-2 sticky left-0 bg-slate-100 z-10"></td>
              <td class="border border-slate-300 px-3 py-2 sticky left-[3.5rem] bg-slate-100 z-10">Daily Coverage</td>
              {#each costs.dailyCoverage as c}
                <td class={`border border-slate-300 px-2 py-2 text-center ${c >= requiredStaff ? 'bg-green-200 text-green-800' : 'bg-red-200 text-red-800'}`}>
                  {c}/{requiredStaff}
                </td>
              {/each}
              <td class="border border-slate-300 px-3 py-2 text-center bg-slate-100">{costs.totalHours}</td>
              <td class="border border-slate-300 px-3 py-2 text-center text-green-700 bg-slate-100">
                ${costs.totalCost.toLocaleString()}
              </td>
            </tr>
          {:else}
            <tr class="bg-blue-50 font-bold">
              <td class="border border-slate-300 px-3 py-2 sticky left-0 bg-blue-50 z-10"></td>
              <td class="border border-slate-300 px-3 py-2 sticky left-[3.5rem] bg-blue-50 z-10">AM Coverage</td>
              {#each costs.amCoverage as c}
                <td class={`border border-slate-300 px-2 py-2 text-center ${c >= requiredStaff ? 'bg-green-200 text-green-800' : 'bg-red-200 text-red-800'}`}>
                  {c}/{requiredStaff}
                </td>
              {/each}
              <td class="border border-slate-300 px-3 py-2 text-center bg-blue-50" rowspan="2">
                {costs.totalHours}
                {#if costs.gapOTHours > 0}
                  <div class="text-xs text-orange-600">+{costs.gapOTHours} gap</div>
                {/if}
              </td>
              <td class="border border-slate-300 px-3 py-2 text-center text-green-700 bg-blue-50" rowspan="2">
                ${costs.totalCost.toLocaleString()}
              </td>
            </tr>
            <tr class="bg-purple-50 font-bold">
              <td class="border border-slate-300 px-3 py-2 sticky left-0 bg-purple-50 z-10"></td>
              <td class="border border-slate-300 px-3 py-2 sticky left-[3.5rem] bg-purple-50 z-10">PM Coverage</td>
              {#each costs.pmCoverage as c}
                <td class={`border border-slate-300 px-2 py-2 text-center ${c >= requiredStaff ? 'bg-green-200 text-green-800' : 'bg-red-200 text-red-800'}`}>
                  {c}/{requiredStaff}
                </td>
              {/each}
            </tr>
          {/if}
        </tbody>
      </table>
    </div>
  {/if}
</div>