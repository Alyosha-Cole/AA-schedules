<script lang="ts">
  import { GripVertical, Plus, Trash2, Pencil, X, Copy, ClipboardPaste, GripHorizontal } from 'lucide-svelte';

  export let dates: any[];
  export let onPrint: () => void;

  // Column definition with IDs for drag-and-drop
  let columns = [
    { id: 'weekdayTime', name: 'Weekday Time', type: 'time', group: 'weekday' },
    { id: 'mon', name: 'Monday', type: 'day', dayIndex: 0, group: 'weekday' },
    { id: 'tue', name: 'Tuesday', type: 'day', dayIndex: 1, group: 'weekday' },
    { id: 'wed', name: 'Wednesday', type: 'day', dayIndex: 2, group: 'weekday' },
    { id: 'thu', name: 'Thursday', type: 'day', dayIndex: 3, group: 'weekday' },
    { id: 'fri', name: 'Friday', type: 'day', dayIndex: 4, group: 'weekday' },
    { id: 'weekendTime', name: 'Weekend Time', type: 'time', group: 'weekend' },
    { id: 'sat', name: 'Saturday', type: 'day', dayIndex: 5, group: 'weekend' },
    { id: 'sun', name: 'Sunday', type: 'day', dayIndex: 6, group: 'weekend' },
  ];

  let timeSlots = [
    { id: 1, weekdayTime: '7:00-7:15 AM', weekendTime: '8:00-8:25 AM', weekdays: 'Wake up-Hygiene', weekend: 'Wake up-Hygiene' },
    { id: 2, weekdayTime: '7:15-7:30 AM', weekendTime: '8:30-9:00 AM', weekdays: 'Room Clean', weekend: 'Breakfast/Med pass' },
    { id: 3, weekdayTime: '7:30-8:00 AM', weekendTime: '9:10-9:50 AM', weekdays: 'Morning Check-in', weekend: 'Staff led group' },
    { id: 4, weekdayTime: '8:00-8:25 AM', weekendTime: '10:00-11:00 AM', weekdays: 'Med pass', weekend: 'Rec Time' },
    { id: 5, weekdayTime: '8:30-8:50 AM', weekendTime: '11:30-11:50 AM', weekdays: 'Goal of the day', weekend: 'Lunch' },
    { id: 6, weekdayTime: '9:00-12:00 PM', weekendTime: '12:00-12:50 PM', weekdays: 'School', weekend: 'Major Clean-up' },
    { id: 7, weekdayTime: '11:00-1:00 PM', weekendTime: '1:00-1:50 PM', weekdays: 'Lunch Periods', weekend: 'Writing Skills' },
    { id: 8, weekdayTime: '1:00-2:30 PM', weekendTime: '2:00-2:50 PM', weekdays: 'School', weekend: 'TV Program' },
    { id: 9, weekdayTime: '2:30-3:20 PM', weekendTime: '3:00-4:00 PM', weekdays: 'Rec-Time', weekend: 'Rec-Time' },
    { id: 10, weekdayTime: '3:20-3:50 PM', weekendTime: '4:10-4:30 PM', weekdays: 'Unstructured Time', weekend: 'Unstructured time' },
    { id: 11, weekdayTime: '4:00-4:20 PM', weekendTime: '4:30-4:50 PM', weekdays: 'Dinner', weekend: 'Dinner' },
    { id: 12, weekdayTime: '4:30-5:30 PM', weekendTime: '5:30-6:00 PM', weekdays: 'Group', weekend: 'Unstructured time' },
    { id: 13, weekdayTime: '5:30-6:30 PM', weekendTime: '6:00-7:00 PM', weekdays: 'Showers', weekend: 'Showers' },
    { id: 14, weekdayTime: '6:30-7:45 PM', weekendTime: '7:00-7:45 PM', weekdays: 'PM Check-in/Med pass', weekend: 'PM Check-in/Med pass' },
    { id: 15, weekdayTime: '8:00 PM', weekendTime: '8:00 PM', weekdays: 'Bed Time', weekend: 'Bed Time' },
  ];

  let scheduleData: Record<number, Record<number, string>> = {};
  let dragItem: number | null = null;
  let dragOverIndex: number | null = null;
  let colDragItem: string | null = null;
  let colDragOverId: string | null = null;
  let showColumnModal = false;
  let editingColumn: typeof columns[0] | null = null;
  let editColumnName = '';
  let columnClipboard: { columnId: string; data: Record<number, string>; timeData?: string[] } | null = null;
  let nextId = 16;

  function initializeSchedule() {
    timeSlots.forEach(slot => {
      scheduleData[slot.id] = {};
      for (let i = 0; i < 7; i++) {
        scheduleData[slot.id][i] = i < 5 ? slot.weekdays : slot.weekend;
      }
    });
  }

  initializeSchedule();

  // === ROW DRAG AND DROP ===
  function onDragStart(slotId: number, e: DragEvent) {
    dragItem = slotId;
    if (e.dataTransfer) {
      e.dataTransfer.effectAllowed = 'move';
      e.dataTransfer.setData('text/plain', `row-${slotId}`);
    }
  }

  function onDragOver(idx: number, e: DragEvent) {
    e.preventDefault();
    if (e.dataTransfer) e.dataTransfer.dropEffect = 'move';
    if (dragItem !== null) dragOverIndex = idx;
  }

  function onDragLeave() { dragOverIndex = null; }

  function onDrop(targetSlotId: number, e: DragEvent) {
    e.preventDefault();
    dragOverIndex = null;
    if (dragItem !== null && dragItem !== targetSlotId) {
      const fromIndex = timeSlots.findIndex(s => s.id === dragItem);
      const toIndex = timeSlots.findIndex(s => s.id === targetSlotId);
      if (fromIndex !== -1 && toIndex !== -1) {
        const newSlots = [...timeSlots];
        const [movedSlot] = newSlots.splice(fromIndex, 1);
        newSlots.splice(toIndex, 0, movedSlot);
        timeSlots = newSlots;
      }
    }
    dragItem = null;
  }

  function onDragEnd() { dragOverIndex = null; dragItem = null; }

  // === COLUMN DRAG AND DROP ===
  function onColDragStart(colId: string, e: DragEvent) {
    colDragItem = colId;
    if (e.dataTransfer) {
      e.dataTransfer.effectAllowed = 'move';
      e.dataTransfer.setData('text/plain', `col-${colId}`);
    }
  }

  function onColDragOver(colId: string, e: DragEvent) {
    e.preventDefault();
    if (e.dataTransfer) e.dataTransfer.dropEffect = 'move';
    if (colDragItem !== null) colDragOverId = colId;
  }

  function onColDragLeave() { colDragOverId = null; }

  function onColDrop(targetColId: string, e: DragEvent) {
    e.preventDefault();
    colDragOverId = null;
    if (colDragItem !== null && colDragItem !== targetColId) {
      const fromIndex = columns.findIndex(c => c.id === colDragItem);
      const toIndex = columns.findIndex(c => c.id === targetColId);
      if (fromIndex !== -1 && toIndex !== -1) {
        const newCols = [...columns];
        const [movedCol] = newCols.splice(fromIndex, 1);
        newCols.splice(toIndex, 0, movedCol);
        columns = newCols;
      }
    }
    colDragItem = null;
  }

  function onColDragEnd() { colDragOverId = null; colDragItem = null; }

  // === COLUMN EDIT MODAL ===
  function openColumnModal(col: typeof columns[0]) {
    editingColumn = col;
    editColumnName = col.name;
    showColumnModal = true;
  }

  function closeColumnModal() {
    showColumnModal = false;
    editingColumn = null;
    editColumnName = '';
  }

  function saveColumnName() {
    if (editingColumn && editColumnName.trim()) {
      const idx = columns.findIndex(c => c.id === editingColumn!.id);
      if (idx !== -1) {
        columns[idx] = { ...columns[idx], name: editColumnName.trim() };
        columns = columns;
      }
    }
    closeColumnModal();
  }

  function deleteColumn() {
    if (!editingColumn) return;
    if (editingColumn.type === 'time') { alert('Cannot delete time columns'); return; }
    const dayColumns = columns.filter(c => c.type === 'day');
    if (dayColumns.length <= 1) { alert('Cannot delete the last day column'); return; }
    columns = columns.filter(c => c.id !== editingColumn!.id);
    closeColumnModal();
  }

  function copyColumn() {
    if (!editingColumn) return;
    if (editingColumn.type === 'time') {
      const isWeekdayTime = editingColumn.id === 'weekdayTime';
      const timeData = timeSlots.map(slot => isWeekdayTime ? slot.weekdayTime : slot.weekendTime);
      columnClipboard = { columnId: editingColumn.id, data: {}, timeData };
    } else {
      const dayIndex = editingColumn.dayIndex!;
      const data: Record<number, string> = {};
      timeSlots.forEach(slot => { data[slot.id] = scheduleData[slot.id]?.[dayIndex] || ''; });
      columnClipboard = { columnId: editingColumn.id, data };
    }
    alert(`Column "${editingColumn.name}" copied to clipboard!`);
  }

  function pasteColumn() {
    if (!editingColumn || !columnClipboard) { alert('No column data in clipboard. Copy a column first.'); return; }
    if (editingColumn.type === 'time' && columnClipboard.timeData) {
      const isWeekdayTime = editingColumn.id === 'weekdayTime';
      timeSlots = timeSlots.map((slot, idx) => {
        const newSlot = { ...slot };
        if (isWeekdayTime) newSlot.weekdayTime = columnClipboard!.timeData![idx] || '';
        else newSlot.weekendTime = columnClipboard!.timeData![idx] || '';
        return newSlot;
      });
    } else if (editingColumn.type === 'day') {
      const dayIndex = editingColumn.dayIndex!;
      timeSlots.forEach(slot => {
        if (scheduleData[slot.id]) scheduleData[slot.id][dayIndex] = columnClipboard!.data[slot.id] || '';
      });
      scheduleData = scheduleData;
    }
    alert(`Pasted data into column "${editingColumn.name}"!`);
  }

  function addRow() {
    const newSlot = { id: nextId++, weekdayTime: '', weekendTime: '', weekdays: '', weekend: '' };
    timeSlots = [...timeSlots, newSlot];
    scheduleData[newSlot.id] = {};
    for (let i = 0; i < 7; i++) scheduleData[newSlot.id][i] = '';
  }

  function deleteRow(slotId: number) {
    if (timeSlots.length <= 1) { alert('Cannot delete the last row'); return; }
    timeSlots = timeSlots.filter(s => s.id !== slotId);
    delete scheduleData[slotId];
  }

  function addColumn() {
    const newId = `custom_${Date.now()}`;
    const newDayIndex = Math.max(...columns.filter(c => c.type === 'day').map(c => c.dayIndex ?? -1)) + 1;
    const newCol = { id: newId, name: `Day ${newDayIndex + 1}`, type: 'day' as const, dayIndex: newDayIndex, group: 'custom' as const };
    columns = [...columns, newCol];
    timeSlots.forEach(slot => { if (scheduleData[slot.id]) scheduleData[slot.id][newDayIndex] = ''; });
  }

  function getCellValue(slot: typeof timeSlots[0], col: typeof columns[0]): string {
    if (col.id === 'weekdayTime') return slot.weekdayTime;
    if (col.id === 'weekendTime') return slot.weekendTime;
    if (col.type === 'day' && col.dayIndex !== undefined) return scheduleData[slot.id]?.[col.dayIndex] || '';
    return '';
  }

  function setCellValue(slot: typeof timeSlots[0], col: typeof columns[0], value: string) {
    if (col.id === 'weekdayTime') {
      const idx = timeSlots.findIndex(s => s.id === slot.id);
      if (idx !== -1) timeSlots[idx] = { ...timeSlots[idx], weekdayTime: value };
    } else if (col.id === 'weekendTime') {
      const idx = timeSlots.findIndex(s => s.id === slot.id);
      if (idx !== -1) timeSlots[idx] = { ...timeSlots[idx], weekendTime: value };
    } else if (col.type === 'day' && col.dayIndex !== undefined) {
      if (scheduleData[slot.id]) scheduleData[slot.id][col.dayIndex] = value;
    }
  }
</script>

<style>
  .editable-cell { transition: background-color 0.2s; }
  .editable-cell:hover { background-color: rgba(59, 130, 246, 0.05); }
  .editable-cell textarea { width: 100%; min-height: 45px; resize: vertical; }
  .drag-row { transition: all 0.2s; cursor: move; }
  .drag-row.dragging { opacity: 0.5; background-color: rgba(59, 130, 246, 0.1); }
  .drag-row.drag-over { border-top: 3px solid #3b82f6; }
  .drag-col { transition: all 0.2s; position: relative; }
  .drag-col.col-dragging { opacity: 0.5; background-color: rgba(59, 130, 246, 0.3) !important; }
  .drag-col.col-drag-over { box-shadow: inset 3px 0 0 #3b82f6; }
  .col-header-content { display: flex; flex-direction: column; align-items: center; gap: 2px; }
  .col-drag-handle { cursor: grab; opacity: 0.6; transition: opacity 0.2s; }
  .col-drag-handle:hover { opacity: 1; }
  .col-edit-btn { opacity: 0; transition: opacity 0.2s; }
  .drag-col:hover .col-edit-btn { opacity: 1; }
  .modal-overlay { position: fixed; inset: 0; background: rgba(0, 0, 0, 0.5); display: flex; align-items: center; justify-content: center; z-index: 1000; }
  .modal-content { background: white; border-radius: 12px; padding: 24px; min-width: 350px; max-width: 90vw; box-shadow: 0 25px 50px -12px rgba(0, 0, 0, 0.25); }
  @media print { .drag-row { cursor: default; } .col-drag-handle, .col-edit-btn { display: none !important; } }
</style>

<div class="overflow-x-auto">
  <table class="master-schedule-table w-full border-collapse text-[10px]">
    <thead>
      <tr>
        <th class="border-2 border-slate-900 bg-slate-700 text-white px-1 py-1 w-[30px] print:hidden"></th>
        {#each columns as col (col.id)}
          <th class="border-2 border-slate-900 bg-slate-700 text-white px-1 py-1 text-center font-bold drag-col {col.type === 'time' ? 'w-[85px] text-left' : ''} {colDragItem === col.id ? 'col-dragging' : ''} {colDragOverId === col.id ? 'col-drag-over' : ''}"
            draggable="true" on:dragstart={(e) => onColDragStart(col.id, e)} on:dragover={(e) => onColDragOver(col.id, e)} on:dragleave={onColDragLeave} on:drop={(e) => onColDrop(col.id, e)} on:dragend={onColDragEnd}>
            <div class="col-header-content">
              <div class="col-drag-handle print:hidden"><GripHorizontal class="w-3 h-3 text-slate-300" /></div>
              <div class="font-bold text-[9px]">{col.name}</div>
              <button class="col-edit-btn print:hidden p-0.5 rounded hover:bg-slate-600 transition-colors" on:click|stopPropagation={() => openColumnModal(col)} title="Edit column"><Pencil class="w-3 h-3 text-slate-300" /></button>
            </div>
          </th>
        {/each}
        <th class="border-2 border-slate-900 bg-slate-700 text-white px-1 py-1 w-[30px] print:hidden"></th>
      </tr>
    </thead>
    <tbody>
      {#each timeSlots as slot, idx (slot.id)}
        <tr draggable="true" on:dragstart={(e) => onDragStart(slot.id, e)} on:dragover={(e) => onDragOver(idx, e)} on:dragleave={onDragLeave} on:drop={(e) => onDrop(slot.id, e)} on:dragend={onDragEnd} class="drag-row {dragOverIndex === idx ? 'drag-over' : ''} {dragItem === slot.id ? 'dragging' : ''}">
          <td class="border-2 border-slate-900 bg-slate-50 px-1 py-1 text-center cursor-move print:hidden"><GripVertical class="w-4 h-4 text-slate-400 inline-block" /></td>
          {#each columns as col (col.id)}
            {#if col.type === 'time'}
              <td class="border-2 border-slate-900 bg-slate-100 px-1 py-1 align-top">
                <input type="text" value={getCellValue(slot, col)} on:input={(e) => setCellValue(slot, col, e.currentTarget.value)} class="w-full px-1 py-1 text-[8px] font-semibold border border-slate-300 rounded focus:ring-1 focus:ring-blue-500 focus:border-transparent print:border-0" placeholder="Time..." />
              </td>
            {:else}
              <td class="editable-cell border-2 border-slate-900 px-1 py-1 align-top">
                <textarea value={getCellValue(slot, col)} on:input={(e) => setCellValue(slot, col, e.currentTarget.value)} class="w-full min-h-[45px] px-1 py-1 text-[10px] border border-slate-300 rounded focus:ring-1 focus:ring-blue-500 focus:border-transparent resize-y print:border-0 print:p-0 print:min-h-0" />
              </td>
            {/if}
          {/each}
          <td class="border-2 border-slate-900 bg-slate-50 px-1 py-1 text-center print:hidden">
            <button on:click={() => deleteRow(slot.id)} class="text-red-600 hover:text-red-800 hover:bg-red-50 p-1 rounded transition-colors" title="Delete row"><Trash2 class="w-4 h-4" /></button>
          </td>
        </tr>
      {/each}
    </tbody>
  </table>
  
  <div class="mt-4 flex gap-2 print:hidden">
    <button on:click={addRow} class="px-4 py-2 bg-blue-600 text-white rounded-lg hover:bg-blue-700 flex items-center gap-2 font-medium text-sm transition-colors"><Plus class="w-4 h-4" />Add Row</button>
    <button on:click={addColumn} class="px-4 py-2 bg-indigo-600 text-white rounded-lg hover:bg-indigo-700 flex items-center gap-2 font-medium text-sm transition-colors"><Plus class="w-4 h-4" />Add Column</button>
  </div>

  {#if columnClipboard}
    <div class="mt-2 text-sm text-slate-600 print:hidden">
      <span class="inline-flex items-center gap-1 bg-green-100 text-green-800 px-2 py-1 rounded"><Copy class="w-3 h-3" />Clipboard: "{columns.find(c => c.id === columnClipboard?.columnId)?.name || 'Column'}" data ready to paste</span>
    </div>
  {/if}
</div>

{#if showColumnModal && editingColumn}
  <div class="modal-overlay" on:click={closeColumnModal} on:keydown={(e) => e.key === 'Escape' && closeColumnModal()}>
    <div class="modal-content" on:click|stopPropagation>
      <div class="flex items-center justify-between mb-4">
        <h3 class="text-lg font-bold text-slate-800">Edit Column</h3>
        <button on:click={closeColumnModal} class="p-1 rounded hover:bg-slate-100 transition-colors"><X class="w-5 h-5 text-slate-500" /></button>
      </div>
      <div class="mb-4">
        <label class="block text-sm font-medium text-slate-700 mb-1">Column Name</label>
        <input type="text" bind:value={editColumnName} class="w-full px-3 py-2 border border-slate-300 rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-transparent" placeholder="Enter column name..." />
      </div>
      <div class="flex flex-col gap-2">
        <button on:click={saveColumnName} class="w-full px-4 py-2 bg-blue-600 text-white rounded-lg hover:bg-blue-700 flex items-center justify-center gap-2 font-medium transition-colors">Save Name</button>
        <div class="flex gap-2">
          <button on:click={copyColumn} class="flex-1 px-4 py-2 bg-slate-100 text-slate-700 rounded-lg hover:bg-slate-200 flex items-center justify-center gap-2 font-medium transition-colors"><Copy class="w-4 h-4" />Copy Column</button>
          <button on:click={pasteColumn} class="flex-1 px-4 py-2 bg-slate-100 text-slate-700 rounded-lg hover:bg-slate-200 flex items-center justify-center gap-2 font-medium transition-colors {!columnClipboard ? 'opacity-50' : ''}" disabled={!columnClipboard}><ClipboardPaste class="w-4 h-4" />Paste Into</button>
        </div>
        {#if editingColumn.type === 'day'}
          <button on:click={deleteColumn} class="w-full px-4 py-2 bg-red-100 text-red-700 rounded-lg hover:bg-red-200 flex items-center justify-center gap-2 font-medium transition-colors"><Trash2 class="w-4 h-4" />Delete Column</button>
        {/if}
      </div>
      <p class="mt-4 text-xs text-slate-500">Tip: Drag columns by the grip handle to reorder them. Copy a column's data and paste it into another column.</p>
    </div>
  </div>
{/if}