<script lang="ts">
  import { createEventDispatcher } from 'svelte';
  import { X, Plus, Trash2, GripVertical } from 'lucide-svelte';

  export let schedule: any;
  export let staffPositions: any[];

  const dispatch = createEventDispatcher();

  let selectedPositionId: number | null = null;
  let draggedItem: { positionId: number; optionId: string } | null = null;
  let dragOverId: string | null = null;

  // Initialize selectedPositionId to first position
  $: if (staffPositions.length > 0 && selectedPositionId === null) {
    selectedPositionId = staffPositions[0].id;
  }

  $: selectedPosition = staffPositions.find(p => p.id === selectedPositionId);
  $: isFirstPosition = selectedPositionId === staffPositions[0]?.id;

  // Get default time options based on schedule type
  function getDefaultTimeOptions(scheduleType: string): Array<any> {
    const baseOptions = [
      { id: 'off', label: 'OFF', includeInCycle: true, order: 0, isFixed: true, color: '#ef4444' }, // red
      { id: 'pto', label: 'PTO', includeInCycle: false, order: 1, isFixed: true, color: '#f59e0b' } // amber
    ];

    if (scheduleType === '12-hour') {
      return [
        ...baseOptions,
        { id: '830a-830p', label: '8:30a-8:30p', includeInCycle: true, order: 2, isFixed: false, color: '#10b981' } // green
      ];
    } else { // 10-hour
      return [
        ...baseOptions,
        { id: '7a-5p', label: '7a-5p', includeInCycle: true, order: 2, isFixed: false, color: '#10b981' }, // green
        { id: '1p-11p', label: '1p-11p', includeInCycle: true, order: 3, isFixed: false, color: '#3b82f6' } // blue
      ];
    }
  }

  // Initialize advanced settings if not present
  $: {
    if (!schedule.advancedSettings) {
      schedule.advancedSettings = {};
    }
    
    // Initialize each position if not present
    for (const position of staffPositions) {
      if (!schedule.advancedSettings[position.id]) {
        schedule.advancedSettings[position.id] = {
          inheritFrom: null,
          timeOptions: getDefaultTimeOptions(schedule.type)
        };
      }
    }
  }

  $: currentSettings = selectedPositionId ? schedule.advancedSettings[selectedPositionId] : null;
  $: availablePositions = staffPositions.filter(p => p.id !== selectedPositionId);

  // Get the effective time options (either own or inherited)
  function getEffectiveTimeOptions(positionId: number) {
    const settings = schedule.advancedSettings[positionId];
    if (!settings) return [];
    
    if (settings.inheritFrom !== null && schedule.advancedSettings[settings.inheritFrom]) {
      return schedule.advancedSettings[settings.inheritFrom].timeOptions;
    }
    
    return settings.timeOptions;
  }

  $: effectiveTimeOptions = selectedPositionId ? getEffectiveTimeOptions(selectedPositionId) : [];
  $: isInheriting = currentSettings?.inheritFrom !== null;

  function handleInheritChange(e: Event) {
    const value = (e.target as HTMLSelectElement).value;
    if (currentSettings) {
      currentSettings.inheritFrom = value === 'none' ? null : parseInt(value);
      dispatch('settingsChanged');
    }
  }

  function addTimeOption() {
    if (!currentSettings || isInheriting) return;
    
    // Generate a random color for new options
    const colors = ['#3b82f6', '#10b981', '#f59e0b', '#ef4444', '#8b5cf6', '#ec4899', '#14b8a6', '#f97316'];
    const randomColor = colors[Math.floor(Math.random() * colors.length)];
    
    const newOption = {
      id: `custom-${Date.now()}`,
      label: '',
      includeInCycle: true,
      order: currentSettings.timeOptions.length,
      isFixed: false,
      color: randomColor
    };
    
    currentSettings.timeOptions = [...currentSettings.timeOptions, newOption];
    dispatch('settingsChanged');
  }

  function deleteTimeOption(optionId: string) {
    if (!currentSettings || isInheriting) return;
    
    currentSettings.timeOptions = currentSettings.timeOptions.filter(opt => opt.id !== optionId);
    // Reorder
    currentSettings.timeOptions.forEach((opt, idx) => opt.order = idx);
    dispatch('settingsChanged');
  }

  function updateOptionLabel(optionId: string, newLabel: string) {
    if (!currentSettings || isInheriting) return;
    
    const option = currentSettings.timeOptions.find(opt => opt.id === optionId);
    if (option) {
      option.label = newLabel;
      // Reset inherit if any change is made
      currentSettings.inheritFrom = null;
      dispatch('settingsChanged');
    }
  }

  function updateOptionColor(optionId: string, newColor: string) {
    if (!currentSettings || isInheriting) return;
    
    const option = currentSettings.timeOptions.find(opt => opt.id === optionId);
    if (option) {
      option.color = newColor;
      // Reset inherit if any change is made
      currentSettings.inheritFrom = null;
      dispatch('settingsChanged');
    }
  }

  function toggleIncludeInCycle(optionId: string) {
    if (!currentSettings || isInheriting) return;
    
    const option = currentSettings.timeOptions.find(opt => opt.id === optionId);
    if (option) {
      option.includeInCycle = !option.includeInCycle;
      // Reset inherit if any change is made
      currentSettings.inheritFrom = null;
      dispatch('settingsChanged');
    }
  }

  // Drag and drop handlers
  function onDragStart(positionId: number, optionId: string, e: DragEvent) {
    if (isInheriting) return;
    draggedItem = { positionId, optionId };
    e.dataTransfer!.effectAllowed = 'move';
  }

  function onDragOver(optionId: string, e: DragEvent) {
    if (isInheriting) return;
    e.preventDefault();
    e.dataTransfer!.dropEffect = 'move';
    dragOverId = optionId;
  }

  function onDragLeave() {
    dragOverId = null;
  }

  function onDrop(targetOptionId: string, e: DragEvent) {
    e.preventDefault();
    dragOverId = null;
    
    if (!draggedItem || !currentSettings || isInheriting) return;
    if (draggedItem.positionId !== selectedPositionId) return;
    if (draggedItem.optionId === targetOptionId) return;
    
    const fromIndex = currentSettings.timeOptions.findIndex(opt => opt.id === draggedItem.optionId);
    const toIndex = currentSettings.timeOptions.findIndex(opt => opt.id === targetOptionId);
    
    if (fromIndex === -1 || toIndex === -1) return;
    
    const newOptions = [...currentSettings.timeOptions];
    const [movedItem] = newOptions.splice(fromIndex, 1);
    newOptions.splice(toIndex, 0, movedItem);
    
    // Update order
    newOptions.forEach((opt, idx) => opt.order = idx);
    currentSettings.timeOptions = newOptions;
    
    // Reset inherit
    currentSettings.inheritFrom = null;
    dispatch('settingsChanged');
    draggedItem = null;
  }

  function onDragEnd() {
    dragOverId = null;
    draggedItem = null;
  }
</script>

<div class="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center z-50 p-4">
  <div class="bg-white rounded-lg shadow-xl max-w-4xl w-full max-h-[90vh] flex flex-col">
    <!-- Header -->
    <div class="flex items-center justify-between p-6 border-b border-slate-200">
      <h2 class="text-2xl font-bold text-slate-800">Advanced Schedule Settings</h2>
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
      <!-- Position Selector -->
      <div class="mb-6">
        <label class="block text-sm font-semibold text-slate-700 mb-2">
          Select Position
        </label>
        <select
          bind:value={selectedPositionId}
          class="w-full px-4 py-2 border border-slate-300 rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-transparent"
        >
          {#each staffPositions as position (position.id)}
            <option value={position.id}>{position.name}</option>
          {/each}
        </select>
      </div>

      {#if selectedPosition && currentSettings}
        <!-- Inherit From Dropdown -->
        {#if !isFirstPosition}
          <div class="mb-6 p-4 bg-slate-50 rounded-lg">
            <div class="flex items-center gap-3">
              <label class="text-sm font-semibold text-slate-700 whitespace-nowrap">
                Schedule same as:
              </label>
              <select
                value={currentSettings.inheritFrom === null ? 'none' : currentSettings.inheritFrom}
                on:change={handleInheritChange}
                class="flex-1 px-3 py-2 border border-slate-300 rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-transparent"
              >
                <option value="none">None (Custom)</option>
                {#each availablePositions as position (position.id)}
                  <option value={position.id}>{position.name}</option>
                {/each}
              </select>
            </div>
            {#if isInheriting}
              <p class="mt-2 text-xs text-slate-600">
                This position inherits all settings from the selected position. Any changes will switch to custom settings.
              </p>
            {/if}
          </div>
        {/if}

        <!-- Time Options -->
        <div class="mb-4">
          <div class="flex items-center justify-between mb-3">
            <h3 class="text-lg font-semibold text-slate-800">Time Options</h3>
            {#if !isInheriting}
              <button
                on:click={addTimeOption}
                class="flex items-center gap-2 px-3 py-1.5 bg-green-600 text-white rounded-lg hover:bg-green-700 transition-colors text-sm"
              >
                <Plus class="w-4 h-4" />
                Add Option
              </button>
            {/if}
          </div>

          <div class="space-y-2">
            {#each effectiveTimeOptions.sort((a, b) => a.order - b.order) as option (option.id)}
              <div
                class="flex items-center gap-3 p-3 border border-slate-300 rounded-lg bg-white {dragOverId === option.id ? 'border-blue-500 bg-blue-50' : ''}"
                draggable={!isInheriting}
                on:dragstart={(e) => onDragStart(selectedPositionId!, option.id, e)}
                on:dragover={(e) => onDragOver(option.id, e)}
                on:dragleave={onDragLeave}
                on:drop={(e) => onDrop(option.id, e)}
                on:dragend={onDragEnd}
              >
                <!-- Drag Handle -->
                {#if !isInheriting}
                  <div class="cursor-move text-slate-400 hover:text-slate-600">
                    <GripVertical class="w-5 h-5" />
                  </div>
                {/if}

                <!-- Color Picker -->
                <input
                  type="color"
                  value={option.color || '#3b82f6'}
                  on:input={(e) => updateOptionColor(option.id, e.currentTarget.value)}
                  disabled={isInheriting}
                  class="w-10 h-10 rounded border border-slate-300 cursor-pointer disabled:cursor-not-allowed"
                  title="Choose color"
                />

                <!-- Time Label Input -->
                <input
                  type="text"
                  value={option.label}
                  on:input={(e) => updateOptionLabel(option.id, e.currentTarget.value)}
                  disabled={isInheriting || option.isFixed}
                  placeholder="Enter time (e.g., 7a-5p)"
                  class="flex-1 px-3 py-2 border border-slate-300 rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-transparent disabled:bg-slate-100 disabled:cursor-not-allowed"
                />

                <!-- Include in Cycle Checkbox -->
                <label class="flex items-center gap-2 cursor-pointer">
                  <input
                    type="checkbox"
                    checked={option.includeInCycle}
                    on:change={() => toggleIncludeInCycle(option.id)}
                    disabled={isInheriting}
                    class="w-4 h-4 text-blue-600 rounded focus:ring-2 focus:ring-blue-500 disabled:cursor-not-allowed"
                  />
                  <span class="text-sm text-slate-700 whitespace-nowrap">In Cycle</span>
                </label>

                <!-- Delete Button -->
                {#if !option.isFixed && !isInheriting}
                  <button
                    on:click={() => deleteTimeOption(option.id)}
                    class="p-2 text-red-600 hover:bg-red-50 rounded-lg transition-colors"
                    aria-label="Delete option"
                  >
                    <Trash2 class="w-4 h-4" />
                  </button>
                {:else if option.isFixed}
                  <div class="w-10"></div>
                {/if}
              </div>
            {/each}
          </div>

          {#if !isInheriting && effectiveTimeOptions.length === 0}
            <p class="text-sm text-slate-500 text-center py-4">
              No time options defined. Click "Add Option" to create one.
            </p>
          {/if}
        </div>

        <!-- Info Box -->
        <div class="mt-6 p-4 bg-blue-50 border border-blue-200 rounded-lg">
          <h4 class="text-sm font-semibold text-blue-900 mb-2">How it works:</h4>
          <ul class="text-xs text-blue-800 space-y-1">
            <li>• <strong>In Cycle</strong>: Options with this checked will appear when clicking on schedule blocks to cycle through</li>
            <li>• <strong>Not In Cycle</strong>: Options will only appear in individual staff schedule dropdowns</li>
            <li>• <strong>Drag to reorder</strong>: The order determines both cycle order and dropdown order</li>
            <li>• <strong>OFF and PTO</strong>: These are fixed options and cannot be deleted</li>
          </ul>
        </div>
      {/if}
    </div>

    <!-- Footer -->
    <div class="flex items-center justify-end gap-3 p-6 border-t border-slate-200">
      <button
        on:click={() => dispatch('close')}
        class="px-6 py-2 bg-blue-600 text-white rounded-lg hover:bg-blue-700 transition-colors font-medium"
      >
        Done
      </button>
    </div>
  </div>
</div>

<style>
  /* Prevent text selection while dragging */
  :global(.dragging) {
    opacity: 0.5;
  }
</style>