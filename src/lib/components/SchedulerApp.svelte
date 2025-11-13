<script lang="ts">
  import { createEventDispatcher } from 'svelte';
  import SchedulerHeader from './SchedulerHeader.svelte';
  import CostOverview from './CostOverview.svelte';
  import StaffRoster from './StaffRoster.svelte';
  import ScheduleConfigs from './ScheduleConfigs.svelte';
  import ScheduleDetail from './ScheduleDetail.svelte';

  export let regularRate: number;
  export let overtimeRate: number;
  export let requiredStaff: number;
  export let startDate: Date;
  export let saveStatus: string;
  export let lastSaved: Date | null;
  export let staff: any[];
  export let schedules: any[];
  export let dates: any[];
  export let days: string[];
  export let costAnalysis: any;
  export let editingOpen: Record<number, boolean>;
  export let collapsedSchedules: Record<number, boolean>;
  export let newStaffName: string;
  export let newScheduleName: string;
  export let fileInput: HTMLInputElement;
  export let generateSchedule: (schedule: any, staffId: number) => number[];
  export let generateScheduleForAssignment: (schedule: any, assignment: any) => number[];
  export let calculateScheduleCosts: (schedule: any) => any;

  const dispatch = createEventDispatcher();

  function toInputValue(d: Date) {
    const y = d.getFullYear();
    const m = String(d.getMonth() + 1).padStart(2, '0');
    const dd = String(d.getDate()).padStart(2, '0');
    return `${y}-${m}-${dd}`;
  }
</script>

<div class="min-h-screen bg-gradient-to-br from-slate-50 to-slate-100 p-6">
  <div class="max-w-7xl mx-auto">
    <SchedulerHeader
      {regularRate}
      {overtimeRate}
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

    <StaffRoster
      {staff}
      bind:newStaffName
      on:addStaff
      on:deleteStaff
      on:updateStaffName
    />

    <ScheduleConfigs
      {schedules}
      {editingOpen}
      bind:newScheduleName
      on:addSchedule
      on:deleteSchedule
      on:updateSchedule
      on:toggleEditing
    />

    {#each schedules as schedule (schedule.id + '-' + (schedule.lastEdited || 0) + '-' + startDate.getTime())}
      <ScheduleDetail
        {schedule}
        {staff}
        {dates}
        {days}
        {requiredStaff}
        {regularRate}
        {overtimeRate}
        isEditing={editingOpen[schedule.id]}
        isCollapsed={collapsedSchedules[schedule.id]}
        {generateSchedule}
        {generateScheduleForAssignment}
        {calculateScheduleCosts}
        on:toggleCollapse
        on:toggleEditing
        on:addSimStaff
        on:deleteSimStaff
        on:updateSimStaff
        on:updateAssignment
        on:toggleDayOff
        on:toggleSimDayOff
        on:toggleScheduleDay
        on:toggleScheduleDayForSim
        on:resetManualEdits
        on:reorderList
      />
    {/each}

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