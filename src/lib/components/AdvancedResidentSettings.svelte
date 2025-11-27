<script lang="ts">
  import { createEventDispatcher, onMount } from 'svelte';
  import { X, Plus, Trash2, GripVertical, Settings, Users, FileText, Building2 } from 'lucide-svelte';

  export let units: string[];
  export let residentSettings: Record<string, {
    inheritFrom: string | null;
    attributes: Array<{
      id: string;
      name: string;
      type: 'text' | 'cycle' | 'select';
      required: boolean;
      options?: Array<{ value: string; label: string; color: string }>;
      order: number;
      showInHeader: boolean;
    }>;
  }>;
  
  export let noteSettings: {
    categories: Array<{
      id: string;
      name: string;
      color: string;
      order: number;
      checkboxes: Array<{ id: string; label: string }>;
    }>;
  };

  const dispatch = createEventDispatcher();

  // Tabs
  let activeTab: 'units' | 'residents' | 'notes' = 'units';

  // Professional color palette
  const COLOR_PALETTE = [
    '#16a34a', // green
    '#2563eb', // blue
    '#d97706', // amber/orange
    '#dc2626', // red
    '#9333ea', // purple
    '#0891b2', // cyan
    '#4f46e5', // indigo
    '#c026d3', // fuchsia
    '#84cc16', // lime
    '#f59e0b', // yellow
  ];

  let selectedUnit: string | null = null;
  let draggedItem: { unitId: string; attrId: string } | null = null;
  let dragOverId: string | null = null;
  let colorIndex = 0;

  // For category drag
  let draggedCategory: string | null = null;
  let dragOverCategoryId: string | null = null;

  // For unit drag
  let draggedUnitIndex: number | null = null;
  let dragOverUnitIndex: number | null = null;

  // For editing
  let editingAttribute: string | null = null;
  let newOptionLabel = '';
  let newCheckboxLabel = '';
  let newUnitName = '';
  let editingUnitIndex: number | null = null;
  let editingUnitName = '';

  // Get default attributes
  function getDefaultAttributes() {
    return [
      {
        id: 'county',
        name: 'County',
        type: 'text' as const,
        required: true,
        order: 0,
        showInHeader: true
      },
      {
        id: 'level',
        name: 'Level',
        type: 'cycle' as const,
        required: true,
        options: [
          { value: '0', label: 'Lvl 0', color: '#dc2626' },
          { value: '1', label: 'Lvl 1', color: '#f59e0b' },
          { value: '2', label: 'Lvl 2', color: '#16a34a' },
          { value: '3', label: 'Lvl 3', color: '#2563eb' }
        ],
        order: 1,
        showInHeader: true
      }
    ];
  }

  onMount(() => {
    initializeSettings();
    if (units.length > 0 && selectedUnit === null) {
      selectedUnit = units[0];
    }
  });

  function initializeSettings() {
    let needsSave = false;

    for (const unit of units) {
      if (!residentSettings[unit]) {
        residentSettings[unit] = {
          inheritFrom: null,
          attributes: getDefaultAttributes()
        };
        needsSave = true;
      }
    }

    if (needsSave) {
      dispatch('residentSettingsChanged', residentSettings);
    }
  }

  $: currentSettings = selectedUnit ? residentSettings[selectedUnit] : null;
  $: availableUnits = units.filter(u => u !== selectedUnit);
  $: isInheriting = currentSettings?.inheritFrom !== null;

  $: effectiveAttributes = (() => {
    if (!currentSettings) return [];
    if (isInheriting && residentSettings[currentSettings.inheritFrom!]) {
      return residentSettings[currentSettings.inheritFrom!].attributes || [];
    }
    return currentSettings.attributes || [];
  })();

  $: sortedAttributes = [...effectiveAttributes].sort((a, b) => a.order - b.order);
  $: isFirstUnit = selectedUnit === units[0];
  
  $: sortedCategories = [...(noteSettings?.categories || [])].sort((a, b) => a.order - b.order);

  function saveResidentSettings() {
    residentSettings = { ...residentSettings };
    dispatch('residentSettingsChanged', residentSettings);
  }

  function saveNoteSettings() {
    noteSettings = { ...noteSettings };
    dispatch('noteSettingsChanged', noteSettings);
  }

  function saveUnits() {
    dispatch('unitsChanged', units);
  }

  function handleInheritChange(e: Event) {
    const value = (e.target as HTMLSelectElement).value;
    if (currentSettings && selectedUnit) {
      const newInheritFrom = value === 'none' ? null : value;
      residentSettings[selectedUnit] = {
        ...currentSettings,
        inheritFrom: newInheritFrom
      };
      saveResidentSettings();
    }
  }

  // =====================
  // Unit Management Functions
  // =====================

  function addUnit() {
    if (!newUnitName.trim()) return;
    
    const unitName = newUnitName.trim();
    if (units.includes(unitName)) {
      alert('A unit with this name already exists');
      return;
    }
    
    units = [...units, unitName];
    
    // Initialize settings for new unit
    residentSettings[unitName] = {
      inheritFrom: null,
      attributes: getDefaultAttributes()
    };
    
    newUnitName = '';
    saveUnits();
    saveResidentSettings();
  }

  function deleteUnit(index: number) {
    if (units.length <= 1) {
      alert('You must have at least one unit');
      return;
    }
    
    const unitName = units[index];
    if (!confirm(`Delete "${unitName}"? This will remove all residents and settings for this unit.`)) {
      return;
    }
    
    // Remove from units array
    units = units.filter((_, i) => i !== index);
    
    // Remove settings
    delete residentSettings[unitName];
    
    // Update any units that inherited from this one
    for (const key of Object.keys(residentSettings)) {
      if (residentSettings[key].inheritFrom === unitName) {
        residentSettings[key].inheritFrom = null;
      }
    }
    
    // Update selected unit if needed
    if (selectedUnit === unitName) {
      selectedUnit = units[0] || null;
    }
    
    saveUnits();
    saveResidentSettings();
  }

  function startEditingUnit(index: number) {
    editingUnitIndex = index;
    editingUnitName = units[index];
  }

  function saveUnitName(index: number) {
    const newName = editingUnitName.trim();
    const oldName = units[index];
    
    if (!newName) {
      editingUnitIndex = null;
      return;
    }
    
    if (newName !== oldName && units.includes(newName)) {
      alert('A unit with this name already exists');
      return;
    }
    
    if (newName !== oldName) {
      // Update units array
      units = units.map((u, i) => i === index ? newName : u);
      
      // Transfer settings to new name
      residentSettings[newName] = residentSettings[oldName];
      delete residentSettings[oldName];
      
      // Update any units that inherited from this one
      for (const key of Object.keys(residentSettings)) {
        if (residentSettings[key].inheritFrom === oldName) {
          residentSettings[key].inheritFrom = newName;
        }
      }
      
      // Update selected unit if needed
      if (selectedUnit === oldName) {
        selectedUnit = newName;
      }
      
      saveUnits();
      saveResidentSettings();
    }
    
    editingUnitIndex = null;
  }

  function onUnitDragStart(index: number, e: DragEvent) {
    draggedUnitIndex = index;
    e.dataTransfer!.effectAllowed = 'move';
  }

  function onUnitDragOver(index: number, e: DragEvent) {
    e.preventDefault();
    e.dataTransfer!.dropEffect = 'move';
    dragOverUnitIndex = index;
  }

  function onUnitDragLeave() {
    dragOverUnitIndex = null;
  }

  function onUnitDrop(targetIndex: number, e: DragEvent) {
    e.preventDefault();
    dragOverUnitIndex = null;
    
    if (draggedUnitIndex === null || draggedUnitIndex === targetIndex) {
      draggedUnitIndex = null;
      return;
    }
    
    const reordered = [...units];
    const [movedItem] = reordered.splice(draggedUnitIndex, 1);
    reordered.splice(targetIndex, 0, movedItem);
    
    units = reordered;
    saveUnits();
    draggedUnitIndex = null;
  }

  function onUnitDragEnd() {
    dragOverUnitIndex = null;
    draggedUnitIndex = null;
  }

  // =====================
  // Resident Attribute Functions
  // =====================

  function addAttribute() {
    if (!currentSettings || isInheriting || !selectedUnit) return;
    
    const newAttr = {
      id: `custom-${Date.now()}`,
      name: '',
      type: 'text' as const,
      required: false,
      order: currentSettings.attributes.length,
      showInHeader: true
    };
    
    residentSettings[selectedUnit] = {
      ...currentSettings,
      attributes: [...currentSettings.attributes, newAttr]
    };
    
    saveResidentSettings();
  }

  function deleteAttribute(attrId: string) {
    if (!currentSettings || isInheriting || !selectedUnit) return;
    
    const filteredAttrs = currentSettings.attributes.filter(a => a.id !== attrId);
    filteredAttrs.forEach((attr, idx) => attr.order = idx);
    
    residentSettings[selectedUnit] = {
      ...currentSettings,
      attributes: filteredAttrs
    };
    
    saveResidentSettings();
  }

  function updateAttributeName(attrId: string, newName: string) {
    if (!currentSettings || isInheriting || !selectedUnit) return;
    
    const updatedAttrs = currentSettings.attributes.map(attr =>
      attr.id === attrId ? { ...attr, name: newName } : attr
    );
    
    residentSettings[selectedUnit] = {
      ...currentSettings,
      attributes: updatedAttrs,
      inheritFrom: null
    };
    
    saveResidentSettings();
  }

  function updateAttributeType(attrId: string, newType: 'text' | 'cycle' | 'select') {
    if (!currentSettings || isInheriting || !selectedUnit) return;
    
    const updatedAttrs = currentSettings.attributes.map(attr => {
      if (attr.id !== attrId) return attr;
      
      const updated = { ...attr, type: newType };
      
      if ((newType === 'cycle' || newType === 'select') && !attr.options) {
        updated.options = [
          { value: 'option1', label: 'Option 1', color: COLOR_PALETTE[0] }
        ];
      }
      
      return updated;
    });
    
    residentSettings[selectedUnit] = {
      ...currentSettings,
      attributes: updatedAttrs,
      inheritFrom: null
    };
    
    saveResidentSettings();
  }

  function toggleRequired(attrId: string) {
    if (!currentSettings || isInheriting || !selectedUnit) return;
    
    const updatedAttrs = currentSettings.attributes.map(attr =>
      attr.id === attrId ? { ...attr, required: !attr.required } : attr
    );
    
    residentSettings[selectedUnit] = {
      ...currentSettings,
      attributes: updatedAttrs,
      inheritFrom: null
    };
    
    saveResidentSettings();
  }

  function toggleShowInHeader(attrId: string) {
    if (!currentSettings || isInheriting || !selectedUnit) return;
    
    const updatedAttrs = currentSettings.attributes.map(attr =>
      attr.id === attrId ? { ...attr, showInHeader: !attr.showInHeader } : attr
    );
    
    residentSettings[selectedUnit] = {
      ...currentSettings,
      attributes: updatedAttrs,
      inheritFrom: null
    };
    
    saveResidentSettings();
  }

  function addOption(attrId: string) {
    if (!currentSettings || isInheriting || !selectedUnit) return;
    
    const nextColor = COLOR_PALETTE[colorIndex % COLOR_PALETTE.length];
    colorIndex++;
    
    const updatedAttrs = currentSettings.attributes.map(attr => {
      if (attr.id !== attrId) return attr;
      
      const newOption = {
        value: newOptionLabel || `option-${Date.now()}`,
        label: newOptionLabel || 'New Option',
        color: nextColor
      };
      
      return {
        ...attr,
        options: [...(attr.options || []), newOption]
      };
    });
    
    residentSettings[selectedUnit] = {
      ...currentSettings,
      attributes: updatedAttrs,
      inheritFrom: null
    };
    
    newOptionLabel = '';
    saveResidentSettings();
  }

  function deleteOption(attrId: string, optionValue: string) {
    if (!currentSettings || isInheriting || !selectedUnit) return;
    
    const updatedAttrs = currentSettings.attributes.map(attr => {
      if (attr.id !== attrId) return attr;
      
      return {
        ...attr,
        options: (attr.options || []).filter(o => o.value !== optionValue)
      };
    });
    
    residentSettings[selectedUnit] = {
      ...currentSettings,
      attributes: updatedAttrs,
      inheritFrom: null
    };
    
    saveResidentSettings();
  }

  function updateOptionColor(attrId: string, optionValue: string, newColor: string) {
    if (!currentSettings || isInheriting || !selectedUnit) return;
    
    const updatedAttrs = currentSettings.attributes.map(attr => {
      if (attr.id !== attrId) return attr;
      
      return {
        ...attr,
        options: (attr.options || []).map(o =>
          o.value === optionValue ? { ...o, color: newColor } : o
        )
      };
    });
    
    residentSettings[selectedUnit] = {
      ...currentSettings,
      attributes: updatedAttrs,
      inheritFrom: null
    };
    
    saveResidentSettings();
  }

  function updateOptionLabel(attrId: string, optionValue: string, newLabel: string) {
    if (!currentSettings || isInheriting || !selectedUnit) return;
    
    const updatedAttrs = currentSettings.attributes.map(attr => {
      if (attr.id !== attrId) return attr;
      
      return {
        ...attr,
        options: (attr.options || []).map(o =>
          o.value === optionValue ? { ...o, label: newLabel } : o
        )
      };
    });
    
    residentSettings[selectedUnit] = {
      ...currentSettings,
      attributes: updatedAttrs,
      inheritFrom: null
    };
    
    saveResidentSettings();
  }

  // Attribute drag and drop
  function onDragStart(unitId: string, attrId: string, e: DragEvent) {
    draggedItem = { unitId, attrId };
    e.dataTransfer!.effectAllowed = 'move';
  }

  function onDragOver(attrId: string, e: DragEvent) {
    e.preventDefault();
    e.dataTransfer!.dropEffect = 'move';
    dragOverId = attrId;
  }

  function onDragLeave() {
    dragOverId = null;
  }

  function onDrop(targetAttrId: string, e: DragEvent) {
    e.preventDefault();
    dragOverId = null;
    
    if (!draggedItem || !currentSettings || isInheriting || !selectedUnit) {
      draggedItem = null;
      return;
    }
    
    if (draggedItem.unitId !== selectedUnit || draggedItem.attrId === targetAttrId) {
      draggedItem = null;
      return;
    }
    
    const fromIndex = currentSettings.attributes.findIndex(a => a.id === draggedItem!.attrId);
    const toIndex = currentSettings.attributes.findIndex(a => a.id === targetAttrId);
    
    if (fromIndex === -1 || toIndex === -1) {
      draggedItem = null;
      return;
    }
    
    const reorderedAttrs = [...currentSettings.attributes];
    const [movedItem] = reorderedAttrs.splice(fromIndex, 1);
    reorderedAttrs.splice(toIndex, 0, movedItem);
    reorderedAttrs.forEach((attr, idx) => attr.order = idx);
    
    residentSettings[selectedUnit] = {
      ...currentSettings,
      attributes: reorderedAttrs,
      inheritFrom: null
    };
    
    saveResidentSettings();
    draggedItem = null;
  }

  function onDragEnd() {
    dragOverId = null;
    draggedItem = null;
  }

  // =====================
  // Note Category Functions
  // =====================

  function addCategory() {
    const nextColor = COLOR_PALETTE[noteSettings.categories.length % COLOR_PALETTE.length];
    
    const newCategory = {
      id: `cat-${Date.now()}`,
      name: '',
      color: nextColor,
      order: noteSettings.categories.length,
      checkboxes: []
    };
    
    noteSettings.categories = [...noteSettings.categories, newCategory];
    saveNoteSettings();
  }

  function deleteCategory(catId: string) {
    noteSettings.categories = noteSettings.categories.filter(c => c.id !== catId);
    noteSettings.categories.forEach((cat, idx) => cat.order = idx);
    saveNoteSettings();
  }

  function updateCategoryName(catId: string, newName: string) {
    noteSettings.categories = noteSettings.categories.map(cat =>
      cat.id === catId ? { ...cat, name: newName } : cat
    );
    saveNoteSettings();
  }

  function updateCategoryColor(catId: string, newColor: string) {
    noteSettings.categories = noteSettings.categories.map(cat =>
      cat.id === catId ? { ...cat, color: newColor } : cat
    );
    saveNoteSettings();
  }

  function addCheckbox(catId: string) {
    if (!newCheckboxLabel.trim()) return;
    
    noteSettings.categories = noteSettings.categories.map(cat => {
      if (cat.id !== catId) return cat;
      
      return {
        ...cat,
        checkboxes: [...cat.checkboxes, { id: `cb-${Date.now()}`, label: newCheckboxLabel.trim() }]
      };
    });
    
    newCheckboxLabel = '';
    saveNoteSettings();
  }

  function deleteCheckbox(catId: string, cbId: string) {
    noteSettings.categories = noteSettings.categories.map(cat => {
      if (cat.id !== catId) return cat;
      
      return {
        ...cat,
        checkboxes: cat.checkboxes.filter(cb => cb.id !== cbId)
      };
    });
    saveNoteSettings();
  }

  function updateCheckboxLabel(catId: string, cbId: string, newLabel: string) {
    noteSettings.categories = noteSettings.categories.map(cat => {
      if (cat.id !== catId) return cat;
      
      return {
        ...cat,
        checkboxes: cat.checkboxes.map(cb =>
          cb.id === cbId ? { ...cb, label: newLabel } : cb
        )
      };
    });
    saveNoteSettings();
  }

  // Category drag and drop
  function onCategoryDragStart(catId: string, e: DragEvent) {
    draggedCategory = catId;
    e.dataTransfer!.effectAllowed = 'move';
  }

  function onCategoryDragOver(catId: string, e: DragEvent) {
    e.preventDefault();
    e.dataTransfer!.dropEffect = 'move';
    dragOverCategoryId = catId;
  }

  function onCategoryDragLeave() {
    dragOverCategoryId = null;
  }

  function onCategoryDrop(targetCatId: string, e: DragEvent) {
    e.preventDefault();
    dragOverCategoryId = null;
    
    if (!draggedCategory || draggedCategory === targetCatId) {
      draggedCategory = null;
      return;
    }
    
    const fromIndex = noteSettings.categories.findIndex(c => c.id === draggedCategory);
    const toIndex = noteSettings.categories.findIndex(c => c.id === targetCatId);
    
    if (fromIndex === -1 || toIndex === -1) {
      draggedCategory = null;
      return;
    }
    
    const reordered = [...noteSettings.categories];
    const [movedItem] = reordered.splice(fromIndex, 1);
    reordered.splice(toIndex, 0, movedItem);
    reordered.forEach((cat, idx) => cat.order = idx);
    
    noteSettings.categories = reordered;
    saveNoteSettings();
    draggedCategory = null;
  }

  function onCategoryDragEnd() {
    dragOverCategoryId = null;
    draggedCategory = null;
  }
</script>

<div class="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center z-50 p-4">
  <div class="bg-white rounded-lg shadow-xl max-w-5xl w-full max-h-[90vh] flex flex-col">
    <!-- Header -->
    <div class="flex items-center justify-between p-6 border-b border-slate-200">
      <div class="flex items-center gap-3">
        <Settings class="w-6 h-6 text-slate-600" />
        <h2 class="text-2xl font-bold text-slate-800">Settings</h2>
      </div>
      <button
        on:click={() => dispatch('close')}
        class="p-2 hover:bg-slate-100 rounded-lg transition-colors"
        aria-label="Close"
      >
        <X class="w-6 h-6" />
      </button>
    </div>

    <!-- Tabs -->
    <div class="flex border-b border-slate-200">
      <button
        class="flex items-center gap-2 px-6 py-3 font-medium transition-colors {activeTab === 'units' ? 'text-blue-600 border-b-2 border-blue-600' : 'text-slate-600 hover:text-slate-800'}"
        on:click={() => activeTab = 'units'}
      >
        <Building2 class="w-4 h-4" />
        Units
      </button>
      <button
        class="flex items-center gap-2 px-6 py-3 font-medium transition-colors {activeTab === 'residents' ? 'text-blue-600 border-b-2 border-blue-600' : 'text-slate-600 hover:text-slate-800'}"
        on:click={() => activeTab = 'residents'}
      >
        <Users class="w-4 h-4" />
        Resident Attributes
      </button>
      <button
        class="flex items-center gap-2 px-6 py-3 font-medium transition-colors {activeTab === 'notes' ? 'text-blue-600 border-b-2 border-blue-600' : 'text-slate-600 hover:text-slate-800'}"
        on:click={() => activeTab = 'notes'}
      >
        <FileText class="w-4 h-4" />
        Note Categories
      </button>
    </div>

    <!-- Content -->
    <div class="flex-1 overflow-y-auto p-6">
      
      <!-- UNITS TAB -->
      {#if activeTab === 'units'}
        <div class="mb-4">
          <div class="flex items-center justify-between mb-3">
            <h3 class="text-lg font-semibold text-slate-800">Manage Units</h3>
          </div>

          <p class="text-sm text-slate-600 mb-4">
            Add, rename, reorder, or remove units. Drag to reorder.
          </p>

          <!-- Add Unit -->
          <div class="flex items-center gap-2 mb-6 p-4 bg-slate-50 rounded-lg">
            <input
              type="text"
              bind:value={newUnitName}
              placeholder="New unit name (e.g., Unit 4, East Wing)"
              on:keydown={(e) => e.key === 'Enter' && addUnit()}
              class="flex-1 px-3 py-2 border border-slate-300 rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-transparent"
            />
            <button
              on:click={addUnit}
              class="flex items-center gap-2 px-4 py-2 bg-green-600 text-white rounded-lg hover:bg-green-700 transition-colors"
            >
              <Plus class="w-4 h-4" />
              Add Unit
            </button>
          </div>

          <!-- Unit List -->
          <div class="space-y-2">
            {#each units as unit, index (unit)}
              {@const isDragging = draggedUnitIndex === index}
              <div
                class="flex items-center gap-3 p-3 bg-white border border-slate-300 rounded-lg transition-all {dragOverUnitIndex === index ? 'border-blue-500 bg-blue-50' : ''} {isDragging ? 'opacity-50' : ''}"
                draggable={true}
                on:dragstart={(e) => onUnitDragStart(index, e)}
                on:dragover={(e) => onUnitDragOver(index, e)}
                on:dragleave={onUnitDragLeave}
                on:drop={(e) => onUnitDrop(index, e)}
                on:dragend={onUnitDragEnd}
              >
                <div class="cursor-move text-slate-400 hover:text-slate-600">
                  <GripVertical class="w-5 h-5" />
                </div>

                {#if editingUnitIndex === index}
                  <input
                    type="text"
                    bind:value={editingUnitName}
                    on:blur={() => saveUnitName(index)}
                    on:keydown={(e) => {
                      if (e.key === 'Enter') saveUnitName(index);
                      if (e.key === 'Escape') editingUnitIndex = null;
                    }}
                    class="flex-1 px-3 py-2 border border-blue-500 rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-transparent"
                    autofocus
                  />
                {:else}
                  <div 
                    class="flex-1 px-3 py-2 font-medium text-slate-800 cursor-pointer hover:bg-slate-50 rounded"
                    on:dblclick={() => startEditingUnit(index)}
                  >
                    {unit}
                  </div>
                {/if}

                <span class="text-xs text-slate-500">
                  Double-click to rename
                </span>

                <button
                  on:click={() => deleteUnit(index)}
                  class="p-2 text-red-600 hover:bg-red-50 rounded-lg transition-colors"
                  title="Delete unit"
                >
                  <Trash2 class="w-4 h-4" />
                </button>
              </div>
            {/each}
          </div>

          {#if units.length === 0}
            <div class="text-center py-8 text-slate-500">
              No units defined. Add at least one unit to get started.
            </div>
          {/if}

          <!-- Info Box -->
          <div class="mt-6 p-4 bg-blue-50 border border-blue-200 rounded-lg">
            <h4 class="text-sm font-semibold text-blue-900 mb-2">Tips:</h4>
            <ul class="text-xs text-blue-800 space-y-1">
              <li>• Double-click a unit name to rename it</li>
              <li>• Drag units to reorder them</li>
              <li>• Deleting a unit removes all its residents</li>
              <li>• Unit order affects how they appear in tabs</li>
            </ul>
          </div>
        </div>
      {/if}

      <!-- RESIDENT ATTRIBUTES TAB -->
      {#if activeTab === 'residents'}
        <!-- Unit Selector -->
        <div class="mb-6">
          <label class="block text-sm font-semibold text-slate-700 mb-2">
            Select Unit
          </label>
          <select
            bind:value={selectedUnit}
            class="w-full px-4 py-2 border border-slate-300 rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-transparent"
          >
            {#each units as unit}
              <option value={unit}>{unit}</option>
            {/each}
          </select>
        </div>

        {#if currentSettings}
          <!-- Inherit From Dropdown -->
          {#if !isFirstUnit}
            <div class="mb-6 p-4 bg-slate-50 rounded-lg">
              <div class="flex items-center gap-3">
                <label class="text-sm font-semibold text-slate-700 whitespace-nowrap">
                  Copy attributes from:
                </label>
                <select
                  value={currentSettings.inheritFrom === null ? 'none' : currentSettings.inheritFrom}
                  on:change={handleInheritChange}
                  class="flex-1 px-3 py-2 border border-slate-300 rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-transparent"
                >
                  <option value="none">None (Custom)</option>
                  {#each availableUnits as unit}
                    <option value={unit}>{unit}</option>
                  {/each}
                </select>
              </div>
              {#if isInheriting}
                <p class="mt-2 text-xs text-slate-600">
                  This unit inherits all attribute settings from the selected unit.
                </p>
              {/if}
            </div>
          {/if}

          <!-- Attributes -->
          <div class="mb-4">
            <div class="flex items-center justify-between mb-3">
              <h3 class="text-lg font-semibold text-slate-800">Resident Attributes</h3>
              {#if !isInheriting}
                <button
                  on:click={addAttribute}
                  class="flex items-center gap-2 px-3 py-1.5 bg-green-600 text-white rounded-lg hover:bg-green-700 transition-colors text-sm"
                >
                  <Plus class="w-4 h-4" />
                  Add Attribute
                </button>
              {/if}
            </div>

            <div class="space-y-4">
              {#each sortedAttributes as attr (attr.id)}
                {@const isDragging = draggedItem?.attrId === attr.id}
                <div
                  class="border border-slate-300 rounded-lg bg-white transition-all {dragOverId === attr.id ? 'border-blue-500 bg-blue-50' : ''} {isDragging ? 'opacity-50' : ''}"
                  draggable={!isInheriting}
                  on:dragstart={(e) => onDragStart(selectedUnit, attr.id, e)}
                  on:dragover={(e) => onDragOver(attr.id, e)}
                  on:dragleave={onDragLeave}
                  on:drop={(e) => onDrop(attr.id, e)}
                  on:dragend={onDragEnd}
                >
                  <div class="flex items-center gap-3 p-3">
                    {#if !isInheriting}
                      <div class="cursor-move text-slate-400 hover:text-slate-600">
                        <GripVertical class="w-5 h-5" />
                      </div>
                    {/if}

                    <input
                      type="text"
                      value={attr.name}
                      on:input={(e) => updateAttributeName(attr.id, e.currentTarget.value)}
                      disabled={isInheriting}
                      placeholder="Attribute name"
                      class="flex-1 px-3 py-2 border border-slate-300 rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-transparent disabled:bg-slate-100 disabled:cursor-not-allowed"
                    />

                    <select
                      value={attr.type}
                      on:change={(e) => updateAttributeType(attr.id, e.currentTarget.value)}
                      disabled={isInheriting}
                      class="px-3 py-2 border border-slate-300 rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-transparent disabled:bg-slate-100 disabled:cursor-not-allowed"
                    >
                      <option value="text">Text Input</option>
                      <option value="cycle">Cycle (Click)</option>
                      <option value="select">Dropdown</option>
                    </select>

                    <label class="flex items-center gap-2 cursor-pointer whitespace-nowrap">
                      <input
                        type="checkbox"
                        checked={attr.required}
                        on:change={() => toggleRequired(attr.id)}
                        disabled={isInheriting}
                        class="w-4 h-4 text-blue-600 rounded"
                      />
                      <span class="text-sm text-slate-700">Required</span>
                    </label>

                    <label class="flex items-center gap-2 cursor-pointer whitespace-nowrap">
                      <input
                        type="checkbox"
                        checked={attr.showInHeader}
                        on:change={() => toggleShowInHeader(attr.id)}
                        disabled={isInheriting}
                        class="w-4 h-4 text-blue-600 rounded"
                      />
                      <span class="text-sm text-slate-700">Badge</span>
                    </label>

                    {#if !isInheriting}
                      <button
                        on:click={() => deleteAttribute(attr.id)}
                        class="p-2 text-red-600 hover:bg-red-50 rounded-lg transition-colors"
                      >
                        <Trash2 class="w-4 h-4" />
                      </button>
                    {/if}
                  </div>

                  {#if (attr.type === 'cycle' || attr.type === 'select') && attr.options}
                    <div class="border-t border-slate-200 p-3 bg-slate-50">
                      <div class="space-y-2 mb-3">
                        {#each attr.options as option}
                          <div class="flex items-center gap-2 p-2 bg-white rounded border border-slate-200">
                            <input
                              type="color"
                              value={option.color}
                              on:input={(e) => updateOptionColor(attr.id, option.value, e.currentTarget.value)}
                              disabled={isInheriting}
                              class="w-8 h-8 rounded border border-slate-300 cursor-pointer"
                            />
                            <input
                              type="text"
                              value={option.label}
                              on:input={(e) => updateOptionLabel(attr.id, option.value, e.currentTarget.value)}
                              disabled={isInheriting}
                              class="flex-1 px-2 py-1 border border-slate-300 rounded text-sm"
                            />
                            <span 
                              class="px-2 py-1 rounded text-xs font-semibold"
                              style="background-color: {option.color}20; color: {option.color}"
                            >
                              {option.label}
                            </span>
                            {#if !isInheriting && attr.options.length > 1}
                              <button
                                on:click={() => deleteOption(attr.id, option.value)}
                                class="p-1 text-red-500 hover:bg-red-50 rounded"
                              >
                                <Trash2 class="w-3 h-3" />
                              </button>
                            {/if}
                          </div>
                        {/each}
                      </div>

                      {#if !isInheriting}
                        <div class="flex items-center gap-2">
                          <input
                            type="text"
                            bind:value={newOptionLabel}
                            placeholder="New option label"
                            class="flex-1 px-2 py-1.5 border border-slate-300 rounded text-sm"
                          />
                          <button
                            on:click={() => addOption(attr.id)}
                            class="px-3 py-1.5 bg-blue-600 text-white rounded hover:bg-blue-700 text-sm"
                          >
                            Add
                          </button>
                        </div>
                      {/if}
                    </div>
                  {/if}
                </div>
              {/each}
            </div>
          </div>
        {/if}
      {/if}

      <!-- NOTE CATEGORIES TAB -->
      {#if activeTab === 'notes'}
        <div class="mb-4">
          <div class="flex items-center justify-between mb-3">
            <h3 class="text-lg font-semibold text-slate-800">Note Categories</h3>
            <button
              on:click={addCategory}
              class="flex items-center gap-2 px-3 py-1.5 bg-green-600 text-white rounded-lg hover:bg-green-700 transition-colors text-sm"
            >
              <Plus class="w-4 h-4" />
              Add Category
            </button>
          </div>

          <p class="text-sm text-slate-600 mb-4">
            Define the sections that appear on shift reports (e.g., Positive Behaviors, General Notes, etc.)
          </p>

          <div class="space-y-4">
            {#each sortedCategories as cat (cat.id)}
              {@const isDragging = draggedCategory === cat.id}
              <div
                class="border border-slate-300 rounded-lg bg-white transition-all {dragOverCategoryId === cat.id ? 'border-blue-500 bg-blue-50' : ''} {isDragging ? 'opacity-50' : ''}"
                draggable={true}
                on:dragstart={(e) => onCategoryDragStart(cat.id, e)}
                on:dragover={(e) => onCategoryDragOver(cat.id, e)}
                on:dragleave={onCategoryDragLeave}
                on:drop={(e) => onCategoryDrop(cat.id, e)}
                on:dragend={onCategoryDragEnd}
              >
                <!-- Category Header -->
                <div class="flex items-center gap-3 p-3 border-b border-slate-200">
                  <div class="cursor-move text-slate-400 hover:text-slate-600">
                    <GripVertical class="w-5 h-5" />
                  </div>

                  <input
                    type="color"
                    value={cat.color}
                    on:input={(e) => updateCategoryColor(cat.id, e.currentTarget.value)}
                    class="w-10 h-10 rounded border border-slate-300 cursor-pointer"
                  />

                  <input
                    type="text"
                    value={cat.name}
                    on:input={(e) => updateCategoryName(cat.id, e.currentTarget.value)}
                    placeholder="Category name (e.g., Positive Behaviors)"
                    class="flex-1 px-3 py-2 border border-slate-300 rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-transparent"
                  />

                  <span 
                    class="px-3 py-1 rounded text-sm font-semibold"
                    style="background-color: {cat.color}20; color: {cat.color}"
                  >
                    Preview
                  </span>

                  <button
                    on:click={() => deleteCategory(cat.id)}
                    class="p-2 text-red-600 hover:bg-red-50 rounded-lg transition-colors"
                  >
                    <Trash2 class="w-4 h-4" />
                  </button>
                </div>

                <!-- Checkboxes -->
                <div class="p-3 bg-slate-50">
                  <div class="text-sm font-medium text-slate-600 mb-2">Quick-select checkboxes:</div>
                  
                  <div class="space-y-2 mb-3">
                    {#each cat.checkboxes as cb (cb.id)}
                      <div class="flex items-center gap-2 p-2 bg-white rounded border border-slate-200">
                        <input
                          type="text"
                          value={cb.label}
                          on:input={(e) => updateCheckboxLabel(cat.id, cb.id, e.currentTarget.value)}
                          class="flex-1 px-2 py-1 border border-slate-300 rounded text-sm"
                        />
                        <button
                          on:click={() => deleteCheckbox(cat.id, cb.id)}
                          class="p-1 text-red-500 hover:bg-red-50 rounded"
                        >
                          <Trash2 class="w-3 h-3" />
                        </button>
                      </div>
                    {/each}
                  </div>

                  <div class="flex items-center gap-2">
                    <input
                      type="text"
                      bind:value={newCheckboxLabel}
                      placeholder="New checkbox label"
                      on:keydown={(e) => e.key === 'Enter' && addCheckbox(cat.id)}
                      class="flex-1 px-2 py-1.5 border border-slate-300 rounded text-sm"
                    />
                    <button
                      on:click={() => addCheckbox(cat.id)}
                      class="px-3 py-1.5 bg-blue-600 text-white rounded hover:bg-blue-700 text-sm"
                    >
                      Add
                    </button>
                  </div>
                </div>
              </div>
            {/each}
          </div>

          {#if sortedCategories.length === 0}
            <div class="text-center py-8 text-slate-500">
              No categories defined. Click "Add Category" to create one.
            </div>
          {/if}

          <!-- Info Box -->
          <div class="mt-6 p-4 bg-blue-50 border border-blue-200 rounded-lg">
            <h4 class="text-sm font-semibold text-blue-900 mb-2">How it works:</h4>
            <ul class="text-xs text-blue-800 space-y-1">
              <li>• Each category becomes a section in the shift report form</li>
              <li>• Checkboxes appear as quick-select options that auto-fill text</li>
              <li>• Drag categories to reorder them on the form</li>
              <li>• Colors are used to visually distinguish sections</li>
            </ul>
          </div>
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