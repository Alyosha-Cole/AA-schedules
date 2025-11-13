<script lang="ts">
  import { ChevronDown, ChevronRight, Edit2, Trash2 } from 'lucide-svelte';
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

  const dispatch = createEventDispatcher();

  $: costs = calculateScheduleCosts(schedule);
  $: minCoverage = schedule.type === '12-hour'
    ? Math.min(...costs.dailyCoverage)
    : Math.min(Math.min(...costs.amCoverage), Math.min(...costs.pmCoverage));
  $: assignedStaff = Object.keys(schedule.assignments).length;
  $: minStaffNeeded = requiredStaff * 2;

  let dragItem: { scheduleId: number; id: string | number } | null = null;

  function onDragStartRow(scheduleId: number, id: string | number, e: DragEvent) {
    dragItem = { scheduleId, id };
    (e.currentTarget as HTMLElement).classList.add('opacity-60');
    e.dataTransfer?.setData('text/plain', String(id));
    e.dataTransfer?.setDragImage(new Image(), 0, 0);
  }

  function onDragOverRow(e: DragEvent) {
    e.preventDefault();
  }

  function onDropRow(scheduleId: number, id: string | number, e: DragEvent) {
    e.preventDefault();
    if (dragItem && dragItem.scheduleId === scheduleId) {
      dispatch('reorderList', { scheduleId, fromId: dragItem.id, toId: id });
    }
  }

  function onDragEndRow(e: DragEvent) {
    (e.currentTarget as HTMLElement).classList.remove('opacity-60');
    dragItem = null;
  }
</script>

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

    <div class="flex items-center gap-2">
      <button
        on:click={() => dispatch('toggleEditing', schedule.id)}
        class="px-4 py-2 bg-blue-500 text-white rounded hover:bg-blue-600 flex items-center gap-2"
        aria-pressed={isEditing}
        aria-controls={`assign-panel-${schedule.id}`}
      >
        <Edit2 class="w-4 h-4" aria-hidden="true" />
        {isEditing ? 'Done' : 'Assign'}
      </button>

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
    {#if isEditing}
      <div id={`assign-panel-${schedule.id}`} class="mb-4 p-4 bg-slate-50 rounded-lg" tabindex="-1">
        <h3 class="font-bold mb-3">Assign & Order (drag to reorder)</h3>

        <div class="mb-3 flex gap-2">
          {#if schedule.type === '12-hour'}
            <button
              class="px-3 py-1 bg-amber-600 text-white rounded"
              on:click={() => dispatch('addSimStaff', { scheduleId: schedule.id, kind: 'A' })}
            >
              + [SIM] Team A
            </button>
            <button
              class="px-3 py-1 bg-amber-600 text-white rounded"
              on:click={() => dispatch('addSimStaff', { scheduleId: schedule.id, kind: 'B' })}
            >
              + [SIM] Team B
            </button>
          {:else}
            <button
              class="px-3 py-1 bg-amber-600 text-white rounded"
              on:click={() => dispatch('addSimStaff', { scheduleId: schedule.id, kind: 'AM' })}
            >
              + [SIM] AM
            </button>
            <button
              class="px-3 py-1 bg-amber-600 text-white rounded"
              on:click={() => dispatch('addSimStaff', { scheduleId: schedule.id, kind: 'PM' })}
            >
              + [SIM] PM
            </button>
          {/if}
        </div>

        <div class="grid grid-cols-1 md:grid-cols-2 gap-3">
          {#each schedule.displayOrder as rowId, idx (rowId)}
            {@const isSim = String(rowId).startsWith('sim-')}
            {@const simObj = isSim ? schedule.simStaff.find(s => s.id === rowId) : null}
            {@const realObj = !isSim ? staff.find(p => p.id === rowId) : null}
            {@const a = !isSim
              ? (schedule.assignments[rowId] || (schedule.type === '12-hour' ? { team: 'A' } : { shift: 'AM', daysOff: [6,0] }))
              : (simObj || {})}

            <div
              class={`flex items-start gap-3 border rounded-lg p-3 bg-white ${isSim ? 'border-amber-300' : 'border-slate-200'} cursor-grab`}
              draggable="true"
              on:dragstart={(e) => onDragStartRow(schedule.id, rowId, e)}
              on:dragover={onDragOverRow}
              on:drop={(e) => onDropRow(schedule.id, rowId, e)}
              on:dragend={onDragEndRow}
            >
              <span class="w-6 text-right text-slate-500 text-sm select-none">{idx + 1}.</span>

              <div class="flex-1">
                <div class="flex items-center gap-2 mb-2">
                  {#if isSim}
                    <span class="text-xs bg-amber-600 text-white px-2 py-0.5 rounded">SIM</span>
                    <input
                      class="px-2 py-1 border rounded w-full"
                      value={simObj.name}
                      on:input={(e) => dispatch('updateSimStaff', { scheduleId: schedule.id, simId: simObj.id, field: 'name', value: (e.currentTarget as HTMLInputElement).value })}
                      aria-label={`Name for ${simObj.id}`}
                    />
                    <button
                      class="p-1 text-red-600 hover:bg-red-50 rounded"
                      on:click={() => dispatch('deleteSimStaff', { scheduleId: schedule.id, simId: simObj.id })}
                      aria-label={`Delete ${simObj.name}`}
                    >
                      <Trash2 class="w-4 h-4" aria-hidden="true" />
                    </button>
                  {:else}
                    <div class="font-medium">{realObj?.name ?? '(Unknown)'}</div>
                  {/if}
                </div>

                {#if schedule.type === '12-hour'}
                  <div class="flex items-center gap-2">
                    <label class="text-sm" for={`team-${schedule.id}-${rowId}`}>Team:</label>
                    <select
                      id={`team-${schedule.id}-${rowId}`}
                      class="px-3 py-1 border rounded focus:ring-2 focus:ring-blue-500"
                      value={a.team || 'A'}
                      on:change={(e) => {
                        const val = (e.currentTarget as HTMLSelectElement).value;
                        if (isSim) {
                          dispatch('updateSimStaff', { scheduleId: schedule.id, simId: simObj.id, field: 'team', value: val });
                        } else {
                          dispatch('updateAssignment', { scheduleId: schedule.id, staffId: rowId, field: 'team', value: val });
                        }
                      }}
                    >
                      <option value="A">A</option>
                      <option value="B">B</option>
                    </select>
                  </div>
                {:else}
                  <div class="flex items-center gap-2 mb-2">
                    <label class="text-sm" for={`shift-${schedule.id}-${rowId}`}>Shift:</label>
                    <select
                      id={`shift-${schedule.id}-${rowId}`}
                      class="px-3 py-1 border rounded focus:ring-2 focus:ring-blue-500"
                      value={a.shift || 'AM'}
                      on:change={(e) => {
                        const val = (e.currentTarget as HTMLSelectElement).value;
                        if (isSim) {
                          dispatch('updateSimStaff', { scheduleId: schedule.id, simId: simObj.id, field: 'shift', value: val });
                        } else {
                          dispatch('updateAssignment', { scheduleId: schedule.id, staffId: rowId, field: 'shift', value: val });
                        }
                      }}
                    >
                      <option value="AM">AM (7a-5p)</option>
                      <option value="PM">PM (1p-11p)</option>
                    </select>
                  </div>

                  <div class="flex items-center gap-2" role="group" aria-label="Days off">
                    <span class="text-sm">Days Off:</span>
                    {#each days as d, di}
                      <button
                        class={`px-2 py-1 rounded text-xs font-medium transition-colors ${
                          (a.daysOff || []).includes(di)
                            ? 'bg-red-500 text-white'
                            : 'bg-slate-200 text-slate-700 hover:bg-slate-300'
                        }`}
                        title={`${d} - ${(a.daysOff || []).includes(di) ? 'Day Off' : 'Working'}`}
                        on:click={() => {
                          if (isSim) {
                            dispatch('toggleSimDayOff', { scheduleId: schedule.id, simId: simObj.id, dayIndex: di });
                          } else {
                            dispatch('toggleDayOff', { scheduleId: schedule.id, staffId: rowId, dayIndex: di });
                          }
                        }}
                        aria-pressed={(a.daysOff || []).includes(di)}
                      >
                        {d[0]}
                      </button>
                    {/each}
                  </div>
                {/if}
              </div>
            </div>
          {/each}
        </div>
      </div>
    {/if}

    <div class="mb-3 flex items-center justify-between">
      <div class="p-3 bg-blue-50 border border-blue-200 rounded-lg flex-1 mr-2">
        <div class="flex items-center gap-2 text-sm text-blue-800">
          <span class="font-semibold">💡 Tip:</span>
          <span>Click any cell to toggle ON/OFF (works for both real and simulated rows).</span>
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
            <tr class={`hover:bg-slate-50 ${detail.simulated ? 'bg-amber-50' : ''}`}>
              <td class="border border-slate-300 px-3 py-2 font-medium sticky left-0 bg-white w-14 text-right text-slate-500 z-20">
                {idx + 1}.
              </td>
              <td class="border border-slate-300 px-3 py-2 font-medium sticky left-[3.5rem] bg-white z-20">
                {detail.name}
                {#if detail.simulated}
                  <span class="ml-2 text-xs bg-amber-600 text-white px-2 py-0.5 rounded">SIM</span>
                {/if}
                {#if schedule.type === '12-hour' && detail.team}
                  <div class="text-xs text-slate-500">Team {detail.team}</div>
                {/if}
                {#if schedule.type === '10-hour' && detail.shift}
                  <div class="text-xs text-slate-500">{detail.shift} Shift</div>
                {/if}
                {#if detail.isEdited}
                  <span class="ml-2 text-xs bg-yellow-400 text-yellow-900 px-2 py-0.5 rounded">EDITED</span>
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


