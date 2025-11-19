<script lang="ts">
  import { createEventDispatcher } from 'svelte';
  import SchedulerHeader from './SchedulerHeader.svelte';
  import CostOverview from './CostOverview.svelte';
  import StaffConfigs from './StaffConfigs.svelte';
  import PositionRoster from './PositionRoster.svelte';
  import ScheduleConfigs from './ScheduleConfigs.svelte';
  import ScheduleDetail from './ScheduleDetail.svelte';

  export let startDate: Date;
  export let saveStatus: string;
  export let lastSaved: Date | null;
  export let staffPositions: any[];
  export let schedules: any[];
  export let dates: any[];
  export let days: string[];
  export let costAnalysis: any;
  export let editingOpen: Record<number, boolean>;
  export let collapsedSchedules: Record<number, boolean>;
  export let newPositionName: string;
  export let newStaffNames: Record<number, string>;
  export let newScheduleName: string;
  export let fileInput: HTMLInputElement;
  export let generateSchedule: (schedule: any, positionId: number, staffId: number) => number[];
  export let generateScheduleForAssignment: (schedule: any, assignment: any) => number[];
  export let calculateScheduleCosts: (schedule: any) => any;
  export let getDaysOffPattern: (daysOff: number[]) => string;

  const dispatch = createEventDispatcher();

  // Track the active schedule tab
  let activeScheduleId: number | null = null;

  // Set the first schedule as active when schedules change
  $: if (schedules.length > 0 && (activeScheduleId === null || !schedules.find(s => s.id === activeScheduleId))) {
    activeScheduleId = schedules[0].id;
  }

  // Get the active schedule
  $: activeSchedule = schedules.find(s => s.id === activeScheduleId);

  function toInputValue(d: Date) {
    const y = d.getFullYear();
    const m = String(d.getMonth() + 1).padStart(2, '0');
    const dd = String(d.getDate()).padStart(2, '0');
    return `${y}-${m}-${dd}`;
  }
  
  function handleSetActiveSchedule(event: CustomEvent<number>) {
    activeScheduleId = event.detail;
  }
</script>

<div class="min-h-screen bg-gradient-to-br from-slate-50 to-slate-100 p-6 print:p-0 print:bg-white">
  <div class="max-w-7xl mx-auto">
    <div class="print:hidden">
    <SchedulerHeader
      {staffPositions}
      {startDate}
      {saveStatus}
      {lastSaved}
      on:logout
      on:export
      on:import
      on:clearAll
      on:shiftPeriod
      on:setStartDate
      {toInputValue}
    />

    <CostOverview
      {schedules}
      {costAnalysis}
    />

    <StaffConfigs
      {staffPositions}
      bind:newPositionName
      on:addPosition
      on:deletePosition
      on:updatePosition
    />

{#each staffPositions as position (position.id)}
  <PositionRoster
    {position}
    bind:newStaffName={newStaffNames[position.id]}
    on:addStaff={() => dispatch('addStaffToPosition', position.id)}
    on:deleteStaff={(e) => dispatch('deleteStaffFromPosition', e.detail)}
    on:updateStaff={(e) => dispatch('updateStaffInPosition', e.detail)}
  />
{/each}

    <ScheduleConfigs
      {schedules}
      {editingOpen}
      bind:newScheduleName
      on:addSchedule
      on:deleteSchedule
      on:updateSchedule
      on:toggleEditing
    />
    </div>

    <!-- Active Schedule Detail -->
    {#if activeSchedule}
        <ScheduleDetail
          schedule={activeSchedule}
          {schedules}
          {activeScheduleId}
          {staffPositions}
          {dates}
          {days}
          isEditing={editingOpen[activeSchedule.id]}
          isCollapsed={collapsedSchedules[activeSchedule.id]}
          {generateSchedule}
          {generateScheduleForAssignment}
          {calculateScheduleCosts}
          {getDaysOffPattern}
          on:toggleCollapse
          on:toggleEditing
          on:addSimStaff
          on:deleteSimStaff
          on:updateSimStaff
          on:updateAssignment
          on:toggleScheduleDay
          on:toggleScheduleDayForSim
          on:resetManualEdits
          on:reorderList
          on:updateDaysOffPattern
          on:updateSimDaysOffPattern
          on:updateRequiredCount
          on:updateSchedule
          on:scheduleUpdated
          on:setActiveSchedule={handleSetActiveSchedule}
        />
    {/if}

    <input
      type="file"
      accept=".json"
      bind:this={fileInput}
      on:change={(e) => dispatch('importFile', e)}
      class="hidden"
      aria-label="Import JSON file"
    />
  </div>
</div>