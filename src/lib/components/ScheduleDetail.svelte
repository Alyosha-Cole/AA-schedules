<script lang="ts">
  import { ChevronDown, ChevronRight, Trash2, Plus, Printer, Settings } from 'lucide-svelte';
  import { createEventDispatcher } from 'svelte';
  import AdvancedScheduleSettings from './AdvancedScheduleSettings.svelte';
  import IndividualStaffSchedule from './IndividualStaffSchedule.svelte';

  export let schedule: any;
  export let schedules: any[] = [];
  export let activeScheduleId: number | null = null;
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

  // State for popups
  let showAdvancedSettings = false;
  let showIndividualSchedule = false;
  let selectedStaff: { positionId: number; staffId: number; name: string; positionName: string } | null = null;

  $: if (staffPositions.length > 0 && activePositionId === null) {
    activePositionId = staffPositions[0].id;
  }

  $: activePosition = staffPositions.find(p => p.id === activePositionId);
  $: activeDetails = costs.positionDetails[activePositionId] || [];
  $: renderKey = activePositionId + '-' + JSON.stringify(schedule.staffScheduleOverrides || {});  // Force re-render
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

  function printSchedule() {
    // Mark this schedule for printing
    const scheduleEl = document.querySelector(`[data-schedule-id="${schedule.id}"]`);
    if (scheduleEl) {
      scheduleEl.classList.add('print-target');
      window.print();
      // Remove the class after printing
      setTimeout(() => scheduleEl.classList.remove('print-target'), 100);
    }
  }

  function openIndividualStaffSchedule(positionId: number, staffId: number, staffName: string, positionName: string) {
    selectedStaff = { positionId, staffId, name: staffName, positionName };
    showIndividualSchedule = true;
  }

  // Get effective time options for a position (handles inheritance)
  // ULTRA DEFENSIVE - Returns completely independent copies with color isolation
  function getEffectiveTimeOptions(positionId: number) {
    if (!schedule.advancedSettings || !schedule.advancedSettings[positionId]) {
      // Return defaults
      const defaults = [
        { id: 'off', label: 'OFF', includeInCycle: true, order: 0, isFixed: true, color: '#dc2626' },
        { id: 'pto', label: 'PTO', includeInCycle: false, order: 1, isFixed: true, color: '#f59e0b' }
      ];
      
      if (schedule.type === '12-hour') {
        defaults.push({ id: '830a-830p', label: '8:30a-8:30p', includeInCycle: true, order: 2, isFixed: false, color: '#16a34a' });
      } else {
        defaults.push({ id: '7a-5p', label: '7a-5p', includeInCycle: true, order: 2, isFixed: false, color: '#16a34a' });
        defaults.push({ id: '1p-11p', label: '1p-11p', includeInCycle: true, order: 3, isFixed: false, color: '#2563eb' });
      }
      
      return defaults;
    }
    
    const settings = schedule.advancedSettings[positionId];
    
    // Get source options
    let sourceOptions;
    if (settings.inheritFrom !== null && schedule.advancedSettings[settings.inheritFrom]) {
      sourceOptions = schedule.advancedSettings[settings.inheritFrom].timeOptions || [];
    } else {
      sourceOptions = settings.timeOptions || [];
    }
    
    // DEEP COPY with explicit color copying to prevent ANY reference sharing
    return sourceOptions.map(opt => ({
      id: opt.id,
      label: opt.label,
      includeInCycle: opt.includeInCycle,
      order: opt.order,
      isFixed: opt.isFixed,
      color: String(opt.color) // Force string copy of color
    }));
  }

  // Get the cycl able time options (those with includeInCycle = true)
  function getCyclableOptions(positionId: number) {
    const options = getEffectiveTimeOptions(positionId);
    return options.filter(opt => opt.includeInCycle).sort((a, b) => a.order - b.order);
  }

  // Get current time option label for a staff member on a specific day
  function getCurrentTimeLabel(positionId: number, staffId: number, dayIndex: number, working: boolean): string {
    // Check for individual staff override first
    const override = schedule.staffScheduleOverrides?.[positionId]?.[staffId]?.[dayIndex];
    if (override) return override;
    
    // Fall back to default based on working status
    if (working) {
      const assignment = schedule.positionAssignments?.[positionId]?.[staffId];
      if (schedule.type === '12-hour') {
        return '8:30a-8:30p';
      } else {
        return assignment?.shift === 'AM' ? '7a-5p' : '1p-11p';
      }
    }
    
    return 'OFF';
  }

  // Get current time option object (with color) for a staff member on a specific day
  // Returns fresh object EVERY time with explicit color string
  function getCurrentTimeOption(positionId: number, staffId: number, dayIndex: number, working: boolean) {
    const label = getCurrentTimeLabel(positionId, staffId, dayIndex, working);
    const timeOptions = getEffectiveTimeOptions(positionId);
    const option = timeOptions.find(opt => opt.label === label);
    
    if (option) {
      // Return completely fresh object with forced color string
      return {
        id: option.id,
        label: option.label,
        includeInCycle: option.includeInCycle,
        order: option.order,
        isFixed: option.isFixed,
        color: String(option.color) // Force new string
      };
    }
    
    // Fallback
    const fallbackColor = label === 'OFF' ? '#dc2626' : (label === 'PTO' ? '#f59e0b' : '#16a34a');
    return {
      id: 'fallback',
      label,
      color: fallbackColor,
      includeInCycle: true,
      order: 999,
      isFixed: false
    };
  }

  // Cycle to next time option - AGGRESSIVE REACTIVITY VERSION
  function cycleTimeOption(positionId: number, staffId: number, dayIndex: number) {
    const cyclableOptions = getCyclableOptions(positionId);
    
    console.log('=== CYCLE START ===', { positionId, staffId, dayIndex, cyclableCount: cyclableOptions.length });
    
    if (cyclableOptions.length === 0) {
      console.warn('No cyclable options available');
      return;
    }
    
    // Get staff's current schedule
    const assignment = schedule.positionAssignments?.[positionId]?.[staffId];
    if (!assignment) {
      console.warn('No assignment found');
      return;
    }
    
    const staffSchedule = assignment.manualSchedule || [];
    const isWorking = staffSchedule[dayIndex];
    
    // Get current label
    const currentLabel = getCurrentTimeLabel(positionId, staffId, dayIndex, isWorking);
    
    // Find in cyclable options
    let currentIndexInCycle = cyclableOptions.findIndex(opt => opt.label === currentLabel);
    
    // Calculate next
    let nextIndexInCycle = currentIndexInCycle === -1 ? 0 : (currentIndexInCycle + 1) % cyclableOptions.length;
    const nextOption = cyclableOptions[nextIndexInCycle];
    
    console.log('Cycling:', {
      currentLabel,
      currentIndexInCycle,
      nextIndexInCycle,
      nextLabel: nextOption.label,
      cyclableLabels: cyclableOptions.map(o => o.label)
    });
    
    // Initialize deeply if needed
    if (!schedule.staffScheduleOverrides) {
      schedule.staffScheduleOverrides = {};
    }
    if (!schedule.staffScheduleOverrides[positionId]) {
      schedule.staffScheduleOverrides[positionId] = {};
    }
    if (!schedule.staffScheduleOverrides[positionId][staffId]) {
      schedule.staffScheduleOverrides[positionId][staffId] = {};
    }
    
    // Set the override
    schedule.staffScheduleOverrides[positionId][staffId][dayIndex] = nextOption.label;
    
    console.log('Set override:', nextOption.label, 'at position', positionId, 'staff', staffId, 'day', dayIndex);
    
    // SUPER AGGRESSIVE REACTIVITY - Create entirely new object
    schedule = {
      ...schedule,
      staffScheduleOverrides: {
        ...schedule.staffScheduleOverrides,
        [positionId]: {
          ...schedule.staffScheduleOverrides[positionId],
          [staffId]: {
            ...schedule.staffScheduleOverrides[positionId][staffId]
          }
        }
      }
    };
    
    console.log('=== CYCLE END ===', 'New schedule created');
    
    // Dispatch with the schedule
    dispatch('scheduleUpdated', schedule);
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

  /* Prevent text selection when clicking schedule cells */
  .schedule-cell {
    user-select: none;
    -webkit-user-select: none;
    -moz-user-select: none;
    -ms-user-select: none;
  }

  @media print {
    /* Hide ALL schedules by default */
    :global([data-schedule-id]) {
      display: none !important;
    }
    
    /* Show ONLY the schedule being printed */
    :global([data-schedule-id].print-target) {
      display: block !important;
    }

    /* Hide interactive elements and executive info */
    :global(button),
    :global(input[type="number"]),
    :global(select),
    .text-sm.text-slate-600.flex.gap-4,
    .flex.items-center.gap-4:has(input) {
      display: none !important;
    }

    /* Hide the regular table and show print table */
    .overflow-x-auto {
      display: none !important;
    }

    .hidden.print\\:block {
      display: block !important;
    }

    /* Portrait orientation for Week 1/Week 2 layout */
    @page {
      margin: 0.5in;
      size: portrait;
    }

    :global(body) {
      print-color-adjust: exact;
      -webkit-print-color-adjust: exact;
    }

    /* Compact container */
    .bg-white {
      padding: 0.25rem !important;
      margin: 0 !important;
      box-shadow: none !important;
    }

    /* Compact header */
    .flex.items-center.justify-between.mb-4 {
      margin-bottom: 0.25rem !important;
      padding-bottom: 0.25rem !important;
      border-bottom: 1px solid #cbd5e1 !important;
    }

    h2 {
      font-size: 14pt !important;
      margin: 0 0 0.5rem 0 !important;
      text-align: center !important;
    }

    /* Hide the top-left title, keep only centered title */
    .flex.items-center.justify-between.mb-4 h2 {
      display: none !important;
    }

    /* Week headers */
    h3 {
      font-size: 11pt !important;
      margin: 0.5rem 0 0.25rem 0 !important;
      page-break-after: avoid !important;
    }

    /* Page breaks */
    .print-page-break-after {
      page-break-after: always !important;
    }

    .print-page-break-before {
      page-break-before: always !important;
    }

    /* Hide position selector and print header */
    .mb-4.flex.gap-2,
    .hidden.print\\:block.mb-2 {
      display: none !important;
    }

    /* Print table styles */
    table {
      width: 100% !important;
      border-collapse: collapse !important;
      font-size: 7pt !important;
      page-break-inside: auto !important;
      border: 2px solid #1e293b !important;
    }

    thead {
      display: table-header-group !important;
    }

    tbody tr {
      page-break-inside: avoid !important;
    }

    th, td {
      print-color-adjust: exact !important;
      -webkit-print-color-adjust: exact !important;
      padding: 2px 6px !important;
      border: 1px solid #1e293b !important;
    }

    /* Make name column narrower - about 1/4 size */
    th:first-child,
    td:first-child {
      width: 25% !important;
      max-width: 25% !important;
    }

    /* Position header rows */
    tbody tr[style*="background-color"] td {
      font-weight: bold !important;
      font-size: 8pt !important;
    }

    /* Ensure colors print */
    .bg-green-100 {
      background-color: #dcfce7 !important;
    }

    .bg-slate-200 {
      background-color: #e2e8f0 !important;
    }
  }
</style>

<!-- Schedule Toggler Section - Shows which schedule you're viewing -->
{#if schedules.length > 1}
  <div class="bg-white rounded-xl shadow-lg p-6 mb-6 print:hidden">
    <h3 class="text-lg font-semibold text-slate-800 mb-4">Currently Viewing:</h3>
    <div class="flex flex-wrap gap-2">
      {#each schedules as sched (sched.id)}
        <button
          on:click={() => dispatch('setActiveSchedule', sched.id)}
          class="px-4 py-2 rounded-lg font-medium transition-all duration-200 {activeScheduleId === sched.id
            ? 'bg-blue-600 text-white shadow-md'
            : 'bg-slate-100 text-slate-700 hover:bg-slate-200'}"
        >
          {sched.name}
        </button>
      {/each}
    </div>
  </div>
{/if}

<div class="bg-white rounded-lg shadow-lg p-6 mb-6" data-schedule-id="{schedule.id}">
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
      <button
        on:click={printSchedule}
        class="p-2 hover:bg-blue-50 rounded transition-colors print:hidden"
        title="Print this schedule"
        aria-label="Print schedule"
      >
        <Printer class="w-6 h-6 text-blue-600" aria-hidden="true" />
      </button>
      <button
        on:click={() => showAdvancedSettings = true}
        class="p-2 hover:bg-purple-50 rounded transition-colors print:hidden"
        title="Advanced Schedule Settings"
        aria-label="Advanced schedule settings"
      >
        <Settings class="w-6 h-6 text-purple-600" aria-hidden="true" />
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
    <!-- Position Selector -->
    <div class="mb-4 flex gap-2 print:hidden">
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
          
          <button
            on:click={() => dispatch('resetManualEdits', schedule.id)}
            class="px-3 py-2 bg-orange-500 text-white rounded hover:bg-orange-600 transition-colors text-sm font-medium"
          >
            Reset All Edits
          </button>
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
            <th class="border border-slate-300 px-3 py-2 text-center font-semibold sticky top-[var(--sticky-top)] bg-slate-100 z-40 w-24">Hours</th>
            <th class="border border-slate-300 px-3 py-2 text-center font-semibold sticky top-[var(--sticky-top)] bg-slate-100 z-40 w-32">Cost</th>
          </tr>
        </thead>
        <tbody>
          {#each activeDetails as detail, idx (renderKey + "-" + detail.id)}
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
              class={`hover:bg-slate-50 ${detail.simulated ? 'bg-amber-50' : ''} ${isDragging ? 'dragging' : ''} ${isDropTarget ? 'drag-over' : ''}`}
              style="background-color: {!detail.simulated && activePosition ? `${activePosition.color}10` : ''}"
              on:dragover={(e) => onDragOverRow(idx, e)}
              on:dragleave={onDragLeaveRow}
              on:drop={(e) => onDropRow(schedule.id, activePositionId, detail.id, e)}
              on:dragend={onDragEndRow}
            >
              <td 
                class="border border-slate-300 px-3 py-2 font-medium sticky left-0 bg-white w-14 text-right z-20 cursor-grab active:cursor-grabbing"
                style="color: {activePosition?.color}"
                draggable="true"
                on:dragstart={(e) => onDragStartRow(schedule.id, activePositionId, detail.id, e)}
              >
                {idx + 1}.
              </td>
              
              <td class="border border-slate-300 px-3 py-2 sticky left-[3.5rem] bg-white z-20">
                {#if detail.simulated}
                  <div class="flex items-center gap-2 mb-1">
                    <span class="text-xs bg-amber-600 text-white px-1.5 py-0.5 rounded flex-shrink-0">SIM</span>
                    <button
                      on:click={() => dispatch('deleteSimStaff', { scheduleId: schedule.id, positionId: activePositionId, simId: detail.id })}
                      class="p-0.5 text-red-600 hover:bg-red-50 rounded flex-shrink-0"
                      title="Delete simulated staff"
                    >
                      <Trash2 class="w-3 h-3" aria-hidden="true" />
                    </button>
                  </div>
                  
                  {#if schedule.type === '12-hour'}
                    <select
                      class="compact-select w-16 py-0.5 border rounded text-xs focus:ring-1"
                      style="border-color: {activePosition?.color}; focus:ring-color: {activePosition?.color}"
                      value={assignment?.team || 'A'}
                      on:change={(e) => {
                        const val = (e.currentTarget as HTMLSelectElement).value;
                        dispatch('updateSimStaff', { scheduleId: schedule.id, positionId: activePositionId, simId: detail.id, field: 'team', value: val });
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
                          dispatch('updateSimStaff', { scheduleId: schedule.id, positionId: activePositionId, simId: detail.id, field: 'shift', value: val });
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
                          dispatch('updateSimDaysOffPattern', { scheduleId: schedule.id, positionId: activePositionId, simId: detail.id, pattern: val });
                        }}
                      >
                        {#each daysOffOptions as option}
                          <option value={option.value}>{option.label}</option>
                        {/each}
                      </select>
                    </div>
                  {/if}
                {:else}
                  <div class="flex items-center justify-between gap-2 mb-1">
                    <div class="flex items-center gap-2 flex-1 min-w-0">
                      <button
                        on:click={() => openIndividualStaffSchedule(activePositionId, detail.id, detail.name, activePosition?.name || '')}
                        class="font-medium truncate hover:text-blue-600 hover:underline cursor-pointer text-left"
                      >
                        {detail.name}
                      </button>
                    </div>
                  </div>
                  
                  {#if schedule.type === '12-hour'}
                    <select
                      class="compact-select w-16 py-0.5 border rounded text-xs focus:ring-1"
                      style="border-color: {activePosition?.color}; focus:ring-color: {activePosition?.color}"
                      value={assignment?.team || 'A'}
                      on:change={(e) => {
                        const val = (e.currentTarget as HTMLSelectElement).value;
                        dispatch('updateAssignment', { scheduleId: schedule.id, positionId: activePositionId, staffId: detail.id, field: 'team', value: val });
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
                          dispatch('updateAssignment', { scheduleId: schedule.id, positionId: activePositionId, staffId: detail.id, field: 'shift', value: val });
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
                          dispatch('updateDaysOffPattern', { scheduleId: schedule.id, positionId: activePositionId, staffId: detail.id, pattern: val });
                        }}
                      >
                        {#each daysOffOptions as option}
                          <option value={option.value}>{option.label}</option>
                        {/each}
                      </select>
                    </div>
                  {/if}
                {/if}
              </td>

              {#each detail.schedule as working, dIdx (activePositionId + '-' + detail.id + '-' + dIdx)}
                {@const timeOption = getCurrentTimeOption(activePositionId, detail.id, dIdx, working)}
                {@const currentLabel = timeOption?.label || 'OFF'}
                {@const cellColor = timeOption?.color || '#dc2626'}
                <td
                  class={`schedule-cell border border-slate-300 px-1 py-2 text-center cursor-pointer transition-colors`}
                  style="background-color: {cellColor}20; border-color: {cellColor}80;"
                  title={`Click to cycle ${dates[dIdx].dayName} (${currentLabel})`}
                  on:click={() => {
                    if (String(detail.id).startsWith('sim-')) {
                      dispatch('toggleScheduleDayForSim', { scheduleId: schedule.id, positionId: activePositionId, simId: detail.id, dayIndex: dIdx });
                    } else {
                      // Use new cycling logic
                      cycleTimeOption(activePositionId, detail.id, dIdx);
                    }
                  }}
                >
                  <div class="text-xs font-semibold text-slate-800">{currentLabel}</div>
                </td>
              {/each}

              <td class="border border-slate-300 px-3 py-2 text-center font-semibold w-24">
                {detail.hours}
                {#if detail.overtimeHours > 0}
                  <div class="text-xs text-orange-600">
                    ({detail.regularHours} reg + {detail.overtimeHours} OT)
                  </div>
                {/if}
              </td>
              <td 
                class="border border-slate-300 px-3 py-2 text-center font-semibold w-32"
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

  <!-- PRINT-ONLY TABLES: Week 1 and Week 2 as separate pages -->
  <div class="hidden print:block">
    <!-- Schedule Title (only on first page) -->
    <h2 class="text-center font-bold text-lg mb-3">{schedule.name}</h2>
    
    <!-- Week 1 Table -->
    <div class="print-page-break-after">
      <h3 class="text-center font-bold text-base mb-2">Week 1</h3>
      <table class="w-full border-collapse text-xs mb-4">
        <thead>
          <tr>
            <th class="border border-slate-900 px-2 py-1 bg-slate-200 font-bold text-left">Name</th>
            {#each dates.slice(0, 7) as d}
              <th class="border border-slate-900 px-1 py-1 bg-slate-200 text-xs">
                <div>{d.dayName}</div>
                <div class="text-[6pt]">{d.date}</div>
              </th>
            {/each}
          </tr>
        </thead>
        <tbody>
          {#each staffPositions as position (position.id)}
            {@const positionDetails = costs.positionDetails[position.id] || []}
            
            <!-- Position header row -->
            <tr style="background-color: {position.color}40">
              <td class="border border-slate-900 px-2 py-1 font-bold text-sm" style="color: {position.color}">
                {position.name}
              </td>
              <td class="border border-slate-900" colspan="7"></td>
            </tr>
            
            <!-- Staff rows for this position -->
            {#each positionDetails as detail}
              {@const sched = detail.schedule}
              {@const assignment = !detail.simulated 
                ? schedule.positionAssignments[position.id]?.[detail.id]
                : (schedule.positionSimStaff[position.id] || []).find(s => s.id === detail.id)}
              {@const teamLabel = assignment?.team || assignment?.shift || ''}
              {@const shiftTime = schedule.type === '12-hour'
                ? '8:30a-8:30p'
                : (assignment?.shift === 'AM' ? '7a-5p' : '1p-11p')}
              
              <tr>
                <td class="border border-slate-900 px-2 py-1 text-left">
                  <span class="font-medium">{detail.name}</span>
                  {#if teamLabel}
                    <span class="text-[6pt] ml-1">({teamLabel})</span>
                  {/if}
                </td>
                
                <!-- Week 1 days (0-6) -->
                {#each sched.slice(0, 7) as working}
                  <td class="border border-slate-900 px-1 py-1 text-center text-[6pt] {working ? 'bg-green-100' : ''}">
                    {working ? shiftTime : 'OFF'}
                  </td>
                {/each}
              </tr>
            {/each}
          {/each}
        </tbody>
      </table>
    </div>

    <!-- Week 2 Table (new page) -->
    <div class="print-page-break-before">
      <h3 class="text-center font-bold text-base mb-2">Week 2</h3>
      <table class="w-full border-collapse text-xs">
        <thead>
          <tr>
            <th class="border border-slate-900 px-2 py-1 bg-slate-200 font-bold text-left">Name</th>
            {#each dates.slice(7, 14) as d}
              <th class="border border-slate-900 px-1 py-1 bg-slate-200 text-xs">
                <div>{d.dayName}</div>
                <div class="text-[6pt]">{d.date}</div>
              </th>
            {/each}
          </tr>
        </thead>
        <tbody>
          {#each staffPositions as position (position.id)}
            {@const positionDetails = costs.positionDetails[position.id] || []}
            
            <!-- Position header row -->
            <tr style="background-color: {position.color}40">
              <td class="border border-slate-900 px-2 py-1 font-bold text-sm" style="color: {position.color}">
                {position.name}
              </td>
              <td class="border border-slate-900" colspan="7"></td>
            </tr>
            
            <!-- Staff rows for this position -->
            {#each positionDetails as detail}
              {@const sched = detail.schedule}
              {@const assignment = !detail.simulated 
                ? schedule.positionAssignments[position.id]?.[detail.id]
                : (schedule.positionSimStaff[position.id] || []).find(s => s.id === detail.id)}
              {@const teamLabel = assignment?.team || assignment?.shift || ''}
              {@const shiftTime = schedule.type === '12-hour'
                ? '8:30a-8:30p'
                : (assignment?.shift === 'AM' ? '7a-5p' : '1p-11p')}
              
              <tr>
                <td class="border border-slate-900 px-2 py-1 text-left">
                  <span class="font-medium">{detail.name}</span>
                  {#if teamLabel}
                    <span class="text-[6pt] ml-1">({teamLabel})</span>
                  {/if}
                </td>
                
                <!-- Week 2 days (7-13) -->
                {#each sched.slice(7, 14) as working}
                  <td class="border border-slate-900 px-1 py-1 text-center text-[6pt] {working ? 'bg-green-100' : ''}">
                    {working ? shiftTime : 'OFF'}
                  </td>
                {/each}
              </tr>
            {/each}
          {/each}
        </tbody>
      </table>
    </div>
  </div>

  <!-- Advanced Settings Popup -->
  {#if showAdvancedSettings}
    <AdvancedScheduleSettings
      {schedule}
      {staffPositions}
      on:close={() => showAdvancedSettings = false}
      on:settingsChanged={() => {
        schedule = schedule;
        dispatch('scheduleUpdated', schedule);
      }}
    />
  {/if}

  <!-- Individual Staff Schedule Popup -->
  {#if showIndividualSchedule && selectedStaff}
    <IndividualStaffSchedule
      {schedule}
      {days}
      {dates}
      positionId={selectedStaff.positionId}
      staffId={selectedStaff.staffId}
      staffName={selectedStaff.name}
      positionName={selectedStaff.positionName}
      on:close={() => {
        showIndividualSchedule = false;
        selectedStaff = null;
      }}
      on:scheduleChanged={() => {
        schedule = schedule;
        dispatch('scheduleUpdated', schedule);
      }}
      on:save={() => {
        dispatch('scheduleUpdated', schedule);
      }}
    />
  {/if}
</div>