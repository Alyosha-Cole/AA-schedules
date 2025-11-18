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

  const dispatch = createEventDispatcher();

  // Get effective time options for this position
  function getEffectiveTimeOptions(posId: number) {
    const settings = schedule.advancedSettings?.[posId];
    if (!settings) return [];
    
    if (settings.inheritFrom !== null && schedule.advancedSettings[settings.inheritFrom]) {
      return schedule.advancedSettings[settings.inheritFrom].timeOptions;
    }
    
    return settings.timeOptions;
  }

  $: timeOptions = getEffectiveTimeOptions(positionId);
  $: sortedTimeOptions = timeOptions.slice().sort((a, b) => a.order - b.order);

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
    // Check if there's an override
    if (staffOverrides[dayIndex]) {
      return staffOverrides[dayIndex];
    }
    
    // Fall back to determining from the staff's actual schedule
    const assignment = schedule.positionAssignments?.[positionId]?.[staffId];
    if (!assignment) return 'OFF';
    
    // Get the actual schedule pattern for this staff
    const staffSchedule = assignment.manualSchedule || [];
    const isWorking = staffSchedule[dayIndex];
    
    if (isWorking) {
      // Working - determine the shift time
      if (schedule.type === '12-hour') {
        return '8:30a-8:30p';
      } else {
        return assignment.shift === 'AM' ? '7a-5p' : '1p-11p';
      }
    } else {
      // Not working
      return 'OFF';
    }
  }

  function handleDayChange(dayIndex: number, value: string) {
    if (value === '') {
      // Remove override
      delete staffOverrides[dayIndex];
    } else {
      // Set override
      staffOverrides[dayIndex] = value;
    }
    
    // Trigger reactivity
    schedule.staffScheduleOverrides[positionId][staffId] = { ...staffOverrides };
    dispatch('scheduleChanged');
  }

  // Get the default schedule label for display purposes
  function getDefaultScheduleLabel(dayIndex: number): string {
    // This would be based on the staff's assignment
    // For now, we'll show a placeholder
    const assignment = schedule.positionAssignments?.[positionId]?.[staffId];
    if (!assignment) return 'Default';
    
    if (schedule.type === '12-hour') {
      return '8:30a-8:30p / OFF';
    } else {
      return assignment.shift === 'AM' ? '7a-5p / OFF' : '1p-11p / OFF';
    }
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

      <div class="space-y-3">
        {#each Array(14) as _, dayIndex}
          {@const dateInfo = dates[dayIndex]}
          {@const currentValue = getDayValue(dayIndex)}
          
          <div class="flex items-center gap-4 p-3 border border-slate-300 rounded-lg bg-white hover:border-slate-400 transition-colors">
            <div class="w-40">
              <div class="flex items-center gap-2">
                <span class="font-semibold text-slate-800">{dateInfo.dayName}</span>
                <span class="text-sm text-slate-600">{dateInfo.date}</span>
              </div>
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

      <!-- Info Box -->
      <div class="mt-6 p-4 bg-amber-50 border border-amber-200 rounded-lg">
        <h4 class="text-sm font-semibold text-amber-900 mb-2">Note:</h4>
        <ul class="text-xs text-amber-800 space-y-1">
          <li>• The dropdown shows the current schedule for each day</li>
          <li>• Selecting a different option will override the default for that day</li>
          <li>• Changes made here will immediately appear in the schedule table</li>
          <li>• If staff is reassigned to a different shift/team, these overrides will be cleared</li>
        </ul>
      </div>
    </div>

    <!-- Footer -->
    <div class="flex items-center justify-end gap-3 p-6 border-t border-slate-200">
      <button
        on:click={() => dispatch('close')}
        class="px-6 py-2 bg-slate-200 text-slate-800 rounded-lg hover:bg-slate-300 transition-colors font-medium"
      >
        Cancel
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