<script lang="ts">
  import { createEventDispatcher } from 'svelte';
  import SchedulerHeader from './SchedulerHeader.svelte';
  import CostOverview from './CostOverview.svelte';
  import StaffConfigs from './StaffConfigs.svelte';
  import PositionRoster from './PositionRoster.svelte';
  import ScheduleConfigs from './ScheduleConfigs.svelte';
  import ScheduleDetail from './ScheduleDetail.svelte';
  import MasterSchedule from './MasterSchedule.svelte';
  import PrintableSheets from './PrintableSheets.svelte';
  import { Calendar, Users, FileText } from 'lucide-svelte';

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

  // Track the active main tab (schedules, master-schedule, or printable-sheets)
  let activeTab: 'schedules' | 'master-schedule' | 'printable-sheets' = 'schedules';

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

      <!-- Tab Navigation -->
      <div class="bg-white rounded-lg shadow-lg mb-6">
        <div class="flex border-b border-slate-200">
          <button
            on:click={() => activeTab = 'schedules'}
            class="flex items-center gap-2 px-6 py-4 font-medium transition-colors relative {activeTab === 'schedules' ? 'text-blue-600 border-b-2 border-blue-600' : 'text-slate-600 hover:text-slate-900 hover:bg-slate-50'}"
          >
            <Calendar class="w-5 h-5" />
            Staff Schedules
          </button>
          <button
            on:click={() => activeTab = 'master-schedule'}
            class="flex items-center gap-2 px-6 py-4 font-medium transition-colors relative {activeTab === 'master-schedule' ? 'text-blue-600 border-b-2 border-blue-600' : 'text-slate-600 hover:text-slate-900 hover:bg-slate-50'}"
          >
            <Users class="w-5 h-5" />
            Master Schedule
          </button>
          <button
            on:click={() => activeTab = 'printable-sheets'}
            class="flex items-center gap-2 px-6 py-4 font-medium transition-colors relative {activeTab === 'printable-sheets' ? 'text-blue-600 border-b-2 border-blue-600' : 'text-slate-600 hover:text-slate-900 hover:bg-slate-50'}"
          >
            <FileText class="w-5 h-5" />
            Printable Sheets
          </button>
        </div>
      </div>

      {#if activeTab === 'schedules'}
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
      {/if}
    </div>

    <!-- Active Schedule Detail (for Staff Schedules tab) -->
    {#if activeTab === 'schedules' && activeSchedule}
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

    <!-- Master Schedule View -->
    {#if activeTab === 'master-schedule'}
      <MasterSchedule
        {dates}
        {days}
        {startDate}
      />
    {/if}

    <!-- Printable Sheets View -->
    {#if activeTab === 'printable-sheets'}
      <PrintableSheets />
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