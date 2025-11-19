<script lang="ts">
  import { createEventDispatcher } from 'svelte';
  import { X } from 'lucide-svelte';

  export let schedule: any;
  export let positionId: number;
  export let staffId: number;
  export let staffName: string;
  export let positionName: string;
  export let days: string[];
  export let dates: any[]; // Array of date objects with dayName and date fields
  export let generateSchedule: (schedule: any, positionId: number, staffId: number) => number[];

  const dispatch = createEventDispatcher();

  // Get effective time options for this position
  function getEffectiveTimeOptions(posId: number) {
    if (!schedule.advancedSettings || !schedule.advancedSettings[posId]) {
      // Return defaults when no advanced settings exist
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
    
    const settings = schedule.advancedSettings[posId];
    
    // Check if this position inherits from another
    if (settings.inheritFrom !== null && schedule.advancedSettings[settings.inheritFrom]) {
      return schedule.advancedSettings[settings.inheritFrom].timeOptions || [];
    }
    
    return settings.timeOptions || [];
  }

  $: timeOptions = getEffectiveTimeOptions(positionId);
  $: sortedTimeOptions = timeOptions.slice().sort((a, b) => a.order - b.order);

  // Generate the actual schedule for this staff member
  $: actualSchedule = generateSchedule(schedule, positionId, staffId);

  // Initialize staff schedule overrides if not present
  $: {
    if (!schedule.staffScheduleOverrides) {
      schedule.staffScheduleOverrides = {};
    }
    if (!schedule.staffScheduleOverrides[positionId]) {
      schedule.staffScheduleOverrides[positionId] = {};
    }
    if (!schedule.staffScheduleOverrides[positionId][staffId]) {
      schedule.staffScheduleOverrides[positionId][staffId] = {};
    }
  }

  $: staffOverrides = schedule.staffScheduleOverrides[positionId][staffId];

  // Get the display value for a specific day - should show what's actually in the schedule
  function getDayValue(dayIndex: number): string {
    // Check if there's an override first
    if (staffOverrides[dayIndex]) {
      return staffOverrides[dayIndex];
    }
    
    // Use the generated schedule to determine if they're working
    const isWorking = actualSchedule[dayIndex] === 1;
    
    if (isWorking) {
      // Working - determine the shift time
      const assignment = schedule.positionAssignments?.[positionId]?.[staffId];
      if (schedule.type === '12-hour') {
        return '8:30a-8:30p';
      } else {
        return assignment?.shift === 'AM' ? '7a-5p' : '1p-11p';
      }
    } else {
      // Not working
      return 'OFF';
    }
  }

  function handleDayChange(dayIndex: number, value: string) {
    // Determine what the default value would be without override
    const isWorking = actualSchedule[dayIndex] === 1;
    const assignment = schedule.positionAssignments?.[positionId]?.[staffId];
    let defaultValue = 'OFF';
    
    if (isWorking) {
      if (schedule.type === '12-hour') {
        defaultValue = '8:30a-8:30p';
      } else {
        defaultValue = assignment?.shift === 'AM' ? '7a-5p' : '1p-11p';
      }
    }
    
    // If the new value matches the default, remove the override
    if (value === defaultValue) {
      delete staffOverrides[dayIndex];
    } else {
      // Set override
      staffOverrides[dayIndex] = value;
    }
    
    // Trigger reactivity
    schedule.staffScheduleOverrides[positionId][staffId] = { ...staffOverrides };
    schedule = schedule;
    dispatch('scheduleChanged');
  }
</script>

<div class="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center z-50 p-4">
  <div class="bg-white rounded-lg shadow-xl max-w-2xl w-full max-h-[90vh] flex flex-col">
    <!-- Header -->
    <div class="flex items-center justify-between p-6 border-b border-slate-200">
      <div>
        <h2 class="text-2xl font-bold text-slate-800">{staffName}</h2>
        <p class="text-sm text-slate-600 mt-1">{positionName} - Individual Schedule</p>
      </div>
      <button
        on:click={() => dispatch('close')}
        class="p-2 hover:bg-slate-100 rounded-lg transition-colors"
        aria-label="Close"
      >
        <X class="w-6 h-6" />
      </button>
    </div>

    <!-- Content -->
    <div class="flex-1 overflow-y-auto p-6">
      <div class="mb-4">
        <p class="text-sm text-slate-600 mb-4">
          Set custom schedules for each day. Changes will override the default schedule pattern for this staff member.
        </p>
      </div>

      {#if sortedTimeOptions.length === 0}
        <div class="p-4 bg-red-50 border border-red-200 rounded-lg mb-4">
          <p class="text-sm text-red-800">No time options available. Please configure advanced settings for this position first.</p>
        </div>
      {:else}
        <div class="space-y-3">
          {#each Array(14) as _, dayIndex}
            {@const dateInfo = dates[dayIndex]}
            {@const currentValue = getDayValue(dayIndex)}
            {@const isOverridden = !!staffOverrides[dayIndex]}
            
            <div class="flex items-center gap-4 p-3 border rounded-lg bg-white transition-colors {isOverridden ? 'border-blue-400 bg-blue-50' : 'border-slate-300 hover:border-slate-400'}">
              <div class="w-40">
                <div class="flex items-center gap-2">
                  <span class="font-semibold text-slate-800">{dateInfo.dayName}</span>
                  <span class="text-sm text-slate-600">{dateInfo.date}</span>
                </div>
                {#if isOverridden}
                  <span class="text-xs text-blue-600 font-medium">Custom</span>
                {/if}
              </div>
              
              <select
                value={currentValue}
                on:change={(e) => handleDayChange(dayIndex, e.currentTarget.value)}
                class="flex-1 px-3 py-2 border border-slate-300 rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-transparent"
              >
                {#each sortedTimeOptions as option (option.id)}
                  <option value={option.label}>{option.label}</option>
                {/each}
              </select>
            </div>
          {/each}
        </div>
      {/if}

      <!-- Info Box -->
      <div class="mt-6 p-4 bg-amber-50 border border-amber-200 rounded-lg">
        <h4 class="text-sm font-semibold text-amber-900 mb-2">How it works:</h4>
        <ul class="text-xs text-amber-800 space-y-1">
          <li>• The dropdown shows what's currently scheduled for each day</li>
          <li>• Days with custom overrides are highlighted in blue</li>
          <li>• Selecting a different option will override the default pattern for that day</li>
          <li>• Selecting the default value will remove any custom override</li>
          <li>• Changes appear immediately in the schedule table</li>
        </ul>
      </div>
    </div>

    <!-- Footer -->
    <div class="flex items-center justify-end gap-3 p-6 border-t border-slate-200">
      <button
        on:click={() => dispatch('close')}
        class="px-6 py-2 bg-slate-200 text-slate-800 rounded-lg hover:bg-slate-300 transition-colors font-medium"
      >
        Close
      </button>
      <button
        on:click={() => { dispatch('save'); dispatch('close'); }}
        class="px-6 py-2 bg-blue-600 text-white rounded-lg hover:bg-blue-700 transition-colors font-medium"
      >
        Save Changes
      </button>
    </div>
  </div>
</div>