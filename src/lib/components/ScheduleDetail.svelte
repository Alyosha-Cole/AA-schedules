<script lang="ts">
  import { ChevronDown, ChevronRight, Trash2, Plus } from 'lucide-svelte';
  import { createEventDispatcher } from 'svelte';

  export let schedule: any;
  export let staffPositions: any[];
  export let dates: any[];
  export let days: string[];
  export let isEditing: boolean;
  export let isCollapsed: boolean;
  export let generateSchedule: (schedule: any, positionId: number, staffId: number) => number[];
  export let generateScheduleForAssignment: (schedule: any, assignment: any) => number[];
  export let calculateScheduleCosts: (schedule: any) => any;
  export let getDaysOffPattern: (daysOff: number[]) => string;

  const dispatch = createEventDispatcher();

  $: costs = calculateScheduleCosts(schedule);

  const daysOffOptions = [
    { value: 'sun-mon', label: 'Sun-Mon' },
    { value: 'mon-tue', label: 'Mon-Tue' },
    { value: 'tue-wed', label: 'Tue-Wed' },
    { value: 'wed-thu', label: 'Wed-Thu' },
    { value: 'thu-fri', label: 'Thu-Fri' },
    { value: 'fri-sat', label: 'Fri-Sat' },
    { value: 'sat-sun', label: 'Sat-Sun' }
  ];

  let dragItem: { scheduleId: number; positionId: number; id: string | number } | null = null;
  let dragOverIndex: number | null = null;
  let activePositionId: number | null = null;

  $: if (staffPositions.length > 0 && activePositionId === null) {
    activePositionId = staffPositions[0].id;
  }

  $: activePosition = staffPositions.find(p => p.id === activePositionId);
  $: activeDetails = costs.positionDetails[activePositionId] || [];
  $: activeCoverage = costs.positionCoverage[activePositionId] || { dailyCoverage: [], amCoverage: [], pmCoverage: [] };
  $: activeRequired = schedule.requiredCounts[activePositionId] || { day12: 0, am: 0, pm: 0 };

  function onDragStartRow(scheduleId: number, positionId: number, id: string | number, e: DragEvent) {
    dragItem = { scheduleId, positionId, id };
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

  function onDropRow(scheduleId: number, positionId: number, id: string | number, e: DragEvent) {
    e.preventDefault();
    dragOverIndex = null;
    if (dragItem && dragItem.scheduleId === scheduleId && dragItem.positionId === positionId && dragItem.id !== id) {
      dispatch('reorderList', { scheduleId, positionId, fromId: dragItem.id, toId: id });
    }
    dragItem = null;
  }

  function onDragEndRow(e: DragEvent) {
    dragOverIndex = null;
    dragItem = null;
  }

  function getMinCoverage(positionId: number) {
    const coverage = costs.positionCoverage[positionId];
    if (!coverage) return 0;
    
    if (schedule.type === '12-hour') {
      return Math.min(...coverage.dailyCoverage);
    } else {
      return Math.min(Math.min(...coverage.amCoverage), Math.min(...coverage.pmCoverage));
    }
  }

  function getRequiredCount(positionId: number) {
    const required = schedule.requiredCounts[positionId];
    if (!required) return 0;
    
    if (schedule.type === '12-hour') {
      return required.day12;
    } else {
      return Math.max(required.am, required.pm);
    }
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
        <div class="text-sm text-slate-600 flex gap-4">
          {#each staffPositions as position}
            {@const assigned = Object.keys(schedule.positionAssignments[position.id] || {}).length}
            <span>
              <span 
                class="inline-block w-2 h-2 rounded-full mr-1"
                style="background-color: {position.color}"
              ></span>
              {position.name}: {assigned}
            </span>
          {/each}
        </div>
      </div>
    </div>

    <div class="flex items-center gap-4">
      <div class="flex flex-col gap-2">
        {#if activePosition}
          {#if schedule.type === '12-hour'}
            <div class="flex items-center gap-2">
              <span 
                class="inline-block w-3 h-3 rounded-full"
                style="background-color: {activePosition.color}"
              ></span>
              <label class="text-xs text-slate-600">Required {activePosition.name}:</label>
              <input
                type="number"
                min="0"
                class="w-16 px-2 py-1 border border-slate-300 rounded text-sm"
                value={activeRequired.day12}
                on:input={(e) => dispatch('updateRequiredCount', { 
                  scheduleId: schedule.id, 
                  positionId: activePositionId, 
                  field: 'day12', 
                  value: parseInt((e.currentTarget as HTMLInputElement).value) || 0 
                })}
              />
            </div>
          {:else}
            <div class="flex items-center gap-2">
              <span 
                class="inline-block w-3 h-3 rounded-full"
                style="background-color: {activePosition.color}"
              ></span>
              <label class="text-xs text-slate-600">Required AM {activePosition.name}:</label>
              <input
                type="number"
                min="0"
                class="w-16 px-2 py-1 border border-slate-300 rounded text-sm"
                value={activeRequired.am}
                on:input={(e) => dispatch('updateRequiredCount', { 
                  scheduleId: schedule.id, 
                  positionId: activePositionId, 
                  field: 'am', 
                  value: parseInt((e.currentTarget as HTMLInputElement).value) || 0 
                })}
              />
            </div>
            <div class="flex items-center gap-2">
              <span 
                class="inline-block w-3 h-3 rounded-full"
                style="background-color: {activePosition.color}"
              ></span>
              <label class="text-xs text-slate-600">Required PM {activePosition.name}:</label>
              <input
                type="number"
                min="0"
                class="w-16 px-2 py-1 border border-slate-300 rounded text-sm"
                value={activeRequired.pm}
                on:input={(e) => dispatch('updateRequiredCount', { 
                  scheduleId: schedule.id, 
                  positionId: activePositionId, 
                  field: 'pm', 
                  value: parseInt((e.currentTarget as HTMLInputElement).value) || 0 
                })}
              />
            </div>
          {/if}
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
          {#each staffPositions as position}
            {@const minCov = getMinCoverage(position.id)}
            {@const reqCov = getRequiredCount(position.id)}
            <div class="flex items-center gap-2 mb-1">
              <span 
                class="inline-block w-2 h-2 rounded-full"
                style="background-color: {position.color}"
              ></span>
              <div class={`text-lg font-bold ${minCov >= reqCov ? 'text-emerald-600' : 'text-red-600'}`}>
                {minCov}/{reqCov}
              </div>
            </div>
          {/each}
          <div class="text-xs text-slate-600">Min Coverage</div>
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

    <!-- Position Selector -->
    <div class="mb-4 flex gap-2">
      {#each staffPositions as position (position.id)}
        <button
          class="px-4 py-2 rounded font-medium transition-colors flex items-center gap-2"
          style="background-color: {activePositionId === position.id ? position.color : '#e2e8f0'}; color: {activePositionId === position.id ? 'white' : '#334155'}"
          on:click={() => activePositionId = position.id}
        >
          <span 
            class="inline-block w-3 h-3 rounded-full"
            style="background-color: {activePositionId === position.id ? 'white' : position.color}"
          ></span>
          {position.name}
        </button>
      {/each}
      
      <!-- Add SIM buttons for active position -->
      {#if activePosition}
        <div class="ml-auto flex gap-2">
          {#if schedule.type === '12-hour'}
            <button
              class="px-3 py-2 bg-amber-600 text-white rounded hover:bg-amber-700 text-sm flex items-center gap-1"
              on:click={() => dispatch('addSimStaff', { scheduleId: schedule.id, positionId: activePositionId, kind: 'A' })}
            >
              <Plus class="w-4 h-4" aria-hidden="true" />
              Add SIM A
            </button>
            <button
              class="px-3 py-2 bg-amber-600 text-white rounded hover:bg-amber-700 text-sm flex items-center gap-1"
              on:click={() => dispatch('addSimStaff', { scheduleId: schedule.id, positionId: activePositionId, kind: 'B' })}
            >
              <Plus class="w-4 h-4" aria-hidden="true" />
              Add SIM B
            </button>
          {:else}
            <button
              class="px-3 py-2 bg-amber-600 text-white rounded hover:bg-amber-700 text-sm flex items-center gap-1"
              on:click={() => dispatch('addSimStaff', { scheduleId: schedule.id, positionId: activePositionId, kind: 'AM' })}
            >
              <Plus class="w-4 h-4" aria-hidden="true" />
              Add SIM AM
            </button>
            <button
              class="px-3 py-2 bg-amber-600 text-white rounded hover:bg-amber-700 text-sm flex items-center gap-1"
              on:click={() => dispatch('addSimStaff', { scheduleId: schedule.id, positionId: activePositionId, kind: 'PM' })}
            >
              <Plus class="w-4 h-4" aria-hidden="true" />
              Add SIM PM
            </button>
          {/if}
        </div>
      {/if}
    </div>

    <div class="overflow-x-auto">
      <table class="w-full border-separate border-spacing-0 text-sm">
        <thead>
          <tr>
            <th class="border border-slate-300 px-3 py-2 text-left font-semibold sticky left-0 bg-slate-100 z-50 top-[var(--sticky-top)]">#</th>
            <th class="border border-slate-300 px-3 py-2 text-left font-semibold sticky left-[3.5rem] bg-slate-100 z-50 top-[var(--sticky-top)]">
              {activePosition?.name || 'Staff'}
            </th>
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
          {#each activeDetails as detail, idx (detail.id)}
            {@const shiftLabel = schedule.type === '12-hour'
              ? '8:30a-8:30p'
              : (detail.shift === 'AM' ? '7a-5p' : '1p-11p')}
            {@const isSim = String(detail.id).startsWith('sim-')}
            {@const assignment = !isSim 
              ? schedule.positionAssignments[activePositionId]?.[detail.id] 
              : (schedule.positionSimStaff[activePositionId] || []).find(s => s.id === detail.id)}
            {@const isDragging = dragItem?.id === detail.id}
            {@const isDropTarget = dragOverIndex === idx}
            
            <tr 
              class={`hover:bg-slate-50 ${detail.simulated ? 'bg-amber-50' : ''} cursor-grab active:cursor-grabbing ${isDragging ? 'dragging' : ''} ${isDropTarget ? 'drag-over' : ''}`}
              style="background-color: {!detail.simulated && activePosition ? `${activePosition.color}10` : ''}"
              draggable="true"
              on:dragstart={(e) => onDragStartRow(schedule.id, activePositionId, detail.id, e)}
              on:dragover={(e) => onDragOverRow(idx, e)}
              on:dragleave={onDragLeaveRow}
              on:drop={(e) => onDropRow(schedule.id, activePositionId, detail.id, e)}
              on:dragend={onDragEndRow}
            >
              <td 
                class="border border-slate-300 px-3 py-2 font-medium sticky left-0 bg-white w-14 text-right z-20"
                style="color: {activePosition?.color}"
              >
                {idx + 1}.
              </td>
              
              <td class="border border-slate-300 px-3 py-2 sticky left-[3.5rem] bg-white z-20">
                <div class="flex items-center justify-between gap-2 mb-1">
                  <div class="flex items-center gap-2 flex-1 min-w-0">
                    <span class="font-medium truncate">{detail.name}</span>
                    {#if detail.simulated}
                      <span class="text-xs bg-amber-600 text-white px-1.5 py-0.5 rounded flex-shrink-0">SIM</span>
                      <button
                        on:click={() => dispatch('deleteSimStaff', { scheduleId: schedule.id, positionId: activePositionId, simId: detail.id })}
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
                    class="compact-select w-16 py-0.5 border rounded text-xs focus:ring-1"
                    style="border-color: {activePosition?.color}; focus:ring-color: {activePosition?.color}"
                    value={assignment?.team || 'A'}
                    on:change={(e) => {
                      const val = (e.currentTarget as HTMLSelectElement).value;
                      if (isSim) {
                        dispatch('updateSimStaff', { scheduleId: schedule.id, positionId: activePositionId, simId: detail.id, field: 'team', value: val });
                      } else {
                        dispatch('updateAssignment', { scheduleId: schedule.id, positionId: activePositionId, staffId: detail.id, field: 'team', value: val });
                      }
                    }}
                  >
                    <option value="A">A</option>
                    <option value="B">B</option>
                  </select>
                {:else}
                  <div class="flex gap-1">
                    <select
                      class="compact-select w-12 py-0.5 border rounded text-xs focus:ring-1"
                      style="border-color: {activePosition?.color}"
                      value={assignment?.shift || 'AM'}
                      on:change={(e) => {
                        const val = (e.currentTarget as HTMLSelectElement).value;
                        if (isSim) {
                          dispatch('updateSimStaff', { scheduleId: schedule.id, positionId: activePositionId, simId: detail.id, field: 'shift', value: val });
                        } else {
                          dispatch('updateAssignment', { scheduleId: schedule.id, positionId: activePositionId, staffId: detail.id, field: 'shift', value: val });
                        }
                      }}
                    >
                      <option value="AM">AM</option>
                      <option value="PM">PM</option>
                    </select>
                    <select
                      class="compact-select w-20 py-0.5 border rounded text-xs focus:ring-1"
                      style="border-color: {activePosition?.color}"
                      value={getDaysOffPattern(assignment?.daysOff || [6, 0])}
                      on:change={(e) => {
                        const val = (e.currentTarget as HTMLSelectElement).value;
                        if (isSim) {
                          dispatch('updateSimDaysOffPattern', { scheduleId: schedule.id, positionId: activePositionId, simId: detail.id, pattern: val });
                        } else {
                          dispatch('updateDaysOffPattern', { scheduleId: schedule.id, positionId: activePositionId, staffId: detail.id, pattern: val });
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
                      dispatch('toggleScheduleDayForSim', { scheduleId: schedule.id, positionId: activePositionId, simId: detail.id, dayIndex: dIdx });
                    } else {
                      dispatch('toggleScheduleDay', { scheduleId: schedule.id, positionId: activePositionId, staffId: detail.id, dayIndex: dIdx });
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
              <td 
                class="border border-slate-300 px-3 py-2 text-center font-semibold"
                style="color: {activePosition?.color}"
              >
                ${detail.cost.toLocaleString()}
                {#if detail.overtimeHours > 0 && activePosition}
                  <div class="text-xs text-slate-600">
                    (${(detail.regularHours * activePosition.rate).toLocaleString()} + ${(detail.overtimeHours * activePosition.overtimeRate).toLocaleString()})
                  </div>
                {/if}
              </td>
            </tr>
          {/each}

          <!-- Coverage rows -->
          {#if schedule.type === '12-hour'}
            <tr class="font-bold" style="background-color: {activePosition?.color}20">
              <td class="border border-slate-300 px-3 py-2 sticky left-0 z-10" style="background-color: {activePosition?.color}20"></td>
              <td class="border border-slate-300 px-3 py-2 sticky left-[3.5rem] z-10" style="background-color: {activePosition?.color}20">
                {activePosition?.name} Coverage
              </td>
              {#each activeCoverage.dailyCoverage as c}
                <td class={`border border-slate-300 px-2 py-2 text-center ${c >= activeRequired.day12 ? 'bg-green-200 text-green-800' : 'bg-red-200 text-red-800'}`}>
                  {c}/{activeRequired.day12}
                </td>
              {/each}
              <td class="border border-slate-300 px-3 py-2 text-center" style="background-color: {activePosition?.color}20" colspan="2"></td>
            </tr>
          {:else}
            <tr class="font-bold" style="background-color: {activePosition?.color}20">
              <td class="border border-slate-300 px-3 py-2 sticky left-0 z-10" style="background-color: {activePosition?.color}20"></td>
              <td class="border border-slate-300 px-3 py-2 sticky left-[3.5rem] z-10" style="background-color: {activePosition?.color}20">
                AM {activePosition?.name} Coverage
              </td>
              {#each activeCoverage.amCoverage as c}
                <td class={`border border-slate-300 px-2 py-2 text-center ${c >= activeRequired.am ? 'bg-green-200 text-green-800' : 'bg-red-200 text-red-800'}`}>
                  {c}/{activeRequired.am}
                </td>
              {/each}
              <td class="border border-slate-300 px-3 py-2 text-center" style="background-color: {activePosition?.color}20" colspan="2" rowspan="2"></td>
            </tr>
            <tr class="font-bold" style="background-color: {activePosition?.color}20">
              <td class="border border-slate-300 px-3 py-2 sticky left-0 z-10" style="background-color: {activePosition?.color}20"></td>
              <td class="border border-slate-300 px-3 py-2 sticky left-[3.5rem] z-10" style="background-color: {activePosition?.color}20">
                PM {activePosition?.name} Coverage
              </td>
              {#each activeCoverage.pmCoverage as c}
                <td class={`border border-slate-300 px-2 py-2 text-center ${c >= activeRequired.pm ? 'bg-green-200 text-green-800' : 'bg-red-200 text-red-800'}`}>
                  {c}/{activeRequired.pm}
                </td>
              {/each}
            </tr>
          {/if}
        </tbody>
      </table>
    </div>
  {/if}
</div>