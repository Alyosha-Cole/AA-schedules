<script lang="ts">
  import { GripVertical, Plus, Trash2, Pencil, X, Copy, ClipboardPaste, GripHorizontal, Download, Upload } from 'lucide-svelte';
  import { createEventDispatcher } from 'svelte';

  const dispatch = createEventDispatcher();

  // Props - schedule data comes from parent
  export let columns: Array<{
    id: string;
    name: string;
    type: 'time' | 'day';
    dayIndex?: number;
    group?: string;
  }>;
  
  export let timeSlots: Array<{
    id: number;
    weekdayTime: string;
    weekendTime: string;
    weekdays: string;
    weekend: string;
  }>;
  
  export let scheduleData: Record<number, Record<number, string>>;
  
  export let unitName: string = 'Schedule';
  export let columnClipboard: { columnId: string; data: Record<number, string>; timeData?: string[] } | null = null;

  // Row Drag state
  let dragItem: number | null = null;
  let dragOverIndex: number | null = null;

  // Column Drag state
  let colDragItem: string | null = null;
  let colDragOverId: string | null = null;

  // Column Edit Modal state
  let showColumnModal = false;
  let editingColumn: typeof columns[0] | null = null;
  let editColumnName = '';

  // File input for import
  let fileInput: HTMLInputElement;

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
    if (dragItem !== null) {
      dragOverIndex = idx;
    }
  }

  function onDragLeave() {
    dragOverIndex = null;
  }

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
        dispatch('updateTimeSlots', newSlots);
      }
    }
    dragItem = null;
  }

  function onDragEnd() {
    dragOverIndex = null;
    dragItem = null;
  }

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
    if (colDragItem !== null) {
      colDragOverId = colId;
    }
  }

  function onColDragLeave() {
    colDragOverId = null;
  }

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
        dispatch('updateColumns', newCols);
      }
    }
    colDragItem = null;
  }

  function onColDragEnd() {
    colDragOverId = null;
    colDragItem = null;
  }

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
        const newCols = [...columns];
        newCols[idx] = { ...newCols[idx], name: editColumnName.trim() };
        dispatch('updateColumns', newCols);
      }
    }
    closeColumnModal();
  }

  function deleteColumn() {
    if (!editingColumn) return;
    
    if (editingColumn.type === 'time') {
      alert('Cannot delete time columns');
      return;
    }
    
    const dayColumns = columns.filter(c => c.type === 'day');
    if (dayColumns.length <= 1) {
      alert('Cannot delete the last day column');
      return;
    }
    
    dispatch('updateColumns', columns.filter(c => c.id !== editingColumn!.id));
    closeColumnModal();
  }

  function copyColumn() {
    if (!editingColumn) return;
    
    if (editingColumn.type === 'time') {
      const isWeekdayTime = editingColumn.id === 'weekdayTime';
      const timeData = timeSlots.map(slot => isWeekdayTime ? slot.weekdayTime : slot.weekendTime);
      dispatch('updateClipboard', {
        columnId: editingColumn.id,
        data: {},
        timeData
      });
    } else {
      const dayIndex = editingColumn.dayIndex!;
      const data: Record<number, string> = {};
      timeSlots.forEach(slot => {
        data[slot.id] = scheduleData[slot.id]?.[dayIndex] || '';
      });
      dispatch('updateClipboard', {
        columnId: editingColumn.id,
        data
      });
    }
    
    alert(`Column "${editingColumn.name}" copied to clipboard!`);
  }

  function pasteColumn() {
    if (!editingColumn || !columnClipboard) {
      alert('No column data in clipboard. Copy a column first.');
      return;
    }
    
    if (editingColumn.type === 'time' && columnClipboard.timeData) {
      const isWeekdayTime = editingColumn.id === 'weekdayTime';
      const newSlots = timeSlots.map((slot, idx) => {
        const newSlot = { ...slot };
        if (isWeekdayTime) {
          newSlot.weekdayTime = columnClipboard!.timeData![idx] || '';
        } else {
          newSlot.weekendTime = columnClipboard!.timeData![idx] || '';
        }
        return newSlot;
      });
      dispatch('updateTimeSlots', newSlots);
    } else if (editingColumn.type === 'day') {
      const dayIndex = editingColumn.dayIndex!;
      const newScheduleData = { ...scheduleData };
      timeSlots.forEach(slot => {
        if (newScheduleData[slot.id]) {
          newScheduleData[slot.id] = { ...newScheduleData[slot.id], [dayIndex]: columnClipboard!.data[slot.id] || '' };
        }
      });
      dispatch('updateScheduleData', newScheduleData);
    }
    
    alert(`Pasted data into column "${editingColumn.name}"!`);
  }

  // Add new row
  function addRow() {
    dispatch('addRow');
  }

  // Delete row
  function deleteRow(slotId: number) {
    if (timeSlots.length <= 1) {
      alert('Cannot delete the last row');
      return;
    }
    dispatch('deleteRow', slotId);
  }

  // Add new column
  function addColumn() {
    dispatch('addColumn');
  }

  // Get cell value based on column type
  function getCellValue(slot: typeof timeSlots[0], col: typeof columns[0]): string {
    if (col.id === 'weekdayTime') return slot.weekdayTime;
    if (col.id === 'weekendTime') return slot.weekendTime;
    if (col.type === 'day' && col.dayIndex !== undefined) {
      return scheduleData[slot.id]?.[col.dayIndex] || '';
    }
    return '';
  }

  function setCellValue(slot: typeof timeSlots[0], col: typeof columns[0], value: string) {
    if (col.id === 'weekdayTime') {
      const newSlots = timeSlots.map(s => s.id === slot.id ? { ...s, weekdayTime: value } : s);
      dispatch('updateTimeSlots', newSlots);
    } else if (col.id === 'weekendTime') {
      const newSlots = timeSlots.map(s => s.id === slot.id ? { ...s, weekendTime: value } : s);
      dispatch('updateTimeSlots', newSlots);
    } else if (col.type === 'day' && col.dayIndex !== undefined) {
      const newScheduleData = { ...scheduleData };
      if (newScheduleData[slot.id]) {
        newScheduleData[slot.id] = { ...newScheduleData[slot.id], [col.dayIndex]: value };
      }
      dispatch('updateScheduleData', newScheduleData);
    }
  }

  // Export to Excel with formatting
  async function exportToExcel() {
    try {
      // Dynamically import SheetJS
      const XLSX = await import('https://cdn.sheetjs.com/xlsx-0.20.0/package/xlsx.mjs');
      
      // Prepare data
      const headers = columns.map(c => c.name);
      const data: any[][] = [headers];
      
      timeSlots.forEach(slot => {
        const row = columns.map(col => {
          if (col.id === 'weekdayTime') return slot.weekdayTime || '';
          if (col.id === 'weekendTime') return slot.weekendTime || '';
          if (col.type === 'day' && col.dayIndex !== undefined) {
            return scheduleData[slot.id]?.[col.dayIndex] || '';
          }
          return '';
        });
        data.push(row);
      });

      // Create workbook and worksheet
      const wb = XLSX.utils.book_new();
      const ws = XLSX.utils.aoa_to_sheet(data);

      // Set column widths
      const colWidths = columns.map(col => {
        if (col.type === 'time') {
          return { wch: 18 }; // Width for time columns
        } else {
          return { wch: 25 }; // Width for day columns
        }
      });
      ws['!cols'] = colWidths;

      // Set row heights to accommodate wrapped text
      const rowHeights = data.map((_, idx) => {
        if (idx === 0) return { hpt: 30 }; // Header row height
        return { hpt: 60 }; // Data row height
      });
      ws['!rows'] = rowHeights;

      // Apply styling to all cells
      const range = XLSX.utils.decode_range(ws['!ref'] || 'A1');
      
      for (let R = range.s.r; R <= range.e.r; ++R) {
        for (let C = range.s.c; C <= range.e.c; ++C) {
          const cellAddress = XLSX.utils.encode_cell({ r: R, c: C });
          if (!ws[cellAddress]) continue;
          
          // Initialize cell style
          ws[cellAddress].s = {
            alignment: {
              vertical: 'top',
              horizontal: 'left',
              wrapText: true
            },
            border: {
              top: { style: 'thin', color: { rgb: '000000' } },
              bottom: { style: 'thin', color: { rgb: '000000' } },
              left: { style: 'thin', color: { rgb: '000000' } },
              right: { style: 'thin', color: { rgb: '000000' } }
            },
            font: {
              name: 'Arial',
              sz: 11
            }
          };

          // Header row styling
          if (R === 0) {
            ws[cellAddress].s.fill = {
              fgColor: { rgb: '334155' } // Slate-700 color
            };
            ws[cellAddress].s.font = {
              name: 'Arial',
              sz: 11,
              bold: true,
              color: { rgb: 'FFFFFF' }
            };
            ws[cellAddress].s.alignment = {
              vertical: 'center',
              horizontal: 'center',
              wrapText: true
            };
          }
          
          // Time column styling (light gray background)
          const col = columns[C];
          if (col && col.type === 'time' && R > 0) {
            ws[cellAddress].s.fill = {
              fgColor: { rgb: 'F1F5F9' } // Slate-100 color
            };
            ws[cellAddress].s.font = {
              name: 'Arial',
              sz: 10,
              bold: true
            };
          }
        }
      }

      // Add worksheet to workbook
      XLSX.utils.book_append_sheet(wb, ws, 'Schedule');

      // Generate Excel file and download
      XLSX.writeFile(wb, `${unitName.replace(/\s+/g, '_')}_Schedule_${new Date().toISOString().split('T')[0]}.xlsx`);
      
    } catch (error) {
      console.error('Export failed:', error);
      alert('Failed to export Excel file. Please try again.');
    }
  }

  // Parse CSV line handling quoted fields with commas
  function parseCSVLine(line: string): string[] {
    const result: string[] = [];
    let current = '';
    let inQuotes = false;
    
    for (let i = 0; i < line.length; i++) {
      const char = line[i];
      const nextChar = line[i + 1];
      
      if (char === '"') {
        if (inQuotes && nextChar === '"') {
          current += '"';
          i++; // Skip next quote
        } else {
          inQuotes = !inQuotes;
        }
      } else if (char === ',' && !inQuotes) {
        result.push(current);
        current = '';
      } else {
        current += char;
      }
    }
    result.push(current);
    
    return result;
  }

  // Import from CSV
  async function importFromExcel(event: Event) {
    const input = event.target as HTMLInputElement;
    const file = input.files?.[0];
    if (!file) return;

    const fileExtension = file.name.split('.').pop()?.toLowerCase();

    if (fileExtension === 'xlsx') {
      // Handle Excel file
      try {
        const XLSX = await import('https://cdn.sheetjs.com/xlsx-0.20.0/package/xlsx.mjs');
        
        const arrayBuffer = await file.arrayBuffer();
        const workbook = XLSX.read(arrayBuffer, { type: 'array' });
        const firstSheet = workbook.Sheets[workbook.SheetNames[0]];
        const data = XLSX.utils.sheet_to_json(firstSheet, { header: 1 }) as any[][];
        
        if (data.length < 2) {
          alert('Invalid file format - no data found');
          return;
        }

        // First row is headers
        const headerCells = data[0].map(cell => String(cell || ''));
        
        // Update column names from header
        const updatedColumns = columns.map((col, index) => {
          if (headerCells[index]) {
            return { ...col, name: headerCells[index] };
          }
          return col;
        });

        // Remaining rows are data (convert all cells to strings)
        const dataLines = data.slice(1)
          .filter(row => row && row.length > 0)
          .map(row => row.map(cell => String(cell || '')));
        
        if (dataLines.length === 0) {
          alert('No data found in file');
          return;
        }

        // Send parsed data along with updated column structure
        dispatch('importData', { 
          dataLines, 
          columns: updatedColumns 
        });
        
        alert('Schedule imported successfully!');
      } catch (error) {
        console.error('Import failed:', error);
        alert('Failed to import Excel file. Please try again.');
      }
    } else {
      // Handle CSV file (original logic)
      const reader = new FileReader();
      reader.onload = (e) => {
        const text = e.target?.result as string;
        const lines = text.split('\n');
        
        if (lines.length < 2) {
          alert('Invalid file format');
          return;
        }

        // Parse the header row to get column names
        const headerLine = lines[0];
        const headerCells = parseCSVLine(headerLine);
        
        // Update column names from header
        const updatedColumns = columns.map((col, index) => {
          if (headerCells[index]) {
            return { ...col, name: headerCells[index] };
          }
          return col;
        });

        // Parse all data lines (skip header)
        const dataLines = lines
          .slice(1)
          .filter(line => line.trim())
          .map(line => parseCSVLine(line));
        
        if (dataLines.length === 0) {
          alert('No data found in file');
          return;
        }

        // Send parsed data along with updated column structure
        dispatch('importData', { 
          dataLines, 
          columns: updatedColumns 
        });
        
        alert('Schedule imported successfully!');
      };
      
      reader.readAsText(file);
    }
    
    input.value = '';
  }
</script>

<style>
  .editable-cell {
    transition: background-color 0.2s;
  }

  .editable-cell:hover {
    background-color: rgba(59, 130, 246, 0.05);
  }

  .editable-cell textarea {
    width: 100%;
    min-height: 45px;
    resize: vertical;
  }

  .drag-row {
    transition: all 0.2s;
    cursor: move;
  }

  .drag-row.dragging {
    opacity: 0.5;
    background-color: rgba(59, 130, 246, 0.1);
  }

  .drag-row.drag-over {
    border-top: 3px solid #3b82f6;
  }

  .drag-col {
    transition: all 0.2s;
    position: relative;
  }

  .drag-col.col-dragging {
    opacity: 0.5;
    background-color: rgba(59, 130, 246, 0.3) !important;
  }

  .drag-col.col-drag-over {
    box-shadow: inset 3px 0 0 #3b82f6;
  }

  .col-header-content {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 2px;
  }

  .col-drag-handle {
    cursor: grab;
    opacity: 0.6;
    transition: opacity 0.2s;
  }

  .col-drag-handle:hover {
    opacity: 1;
  }

  .col-edit-btn {
    opacity: 0;
    transition: opacity 0.2s;
  }

  .drag-col:hover .col-edit-btn {
    opacity: 1;
  }

  .modal-overlay {
    position: fixed;
    inset: 0;
    background: rgba(0, 0, 0, 0.5);
    display: flex;
    align-items: center;
    justify-content: center;
    z-index: 1000;
  }

  .modal-content {
    background: white;
    border-radius: 12px;
    padding: 24px;
    min-width: 350px;
    max-width: 90vw;
    box-shadow: 0 25px 50px -12px rgba(0, 0, 0, 0.25);
  }

  @media print {
    .drag-row {
      cursor: default;
    }
    .col-drag-handle,
    .col-edit-btn {
      display: none !important;
    }
  }
</style>

<div class="overflow-x-auto">
  <table class="master-schedule-table w-full border-collapse text-[10px]">
    <thead>
      <tr>
        <!-- Row drag handle column -->
        <th class="border-2 border-slate-900 bg-slate-700 text-white px-1 py-1 w-[30px] print:hidden"></th>
        
        <!-- Dynamic columns -->
        {#each columns as col (col.id)}
          <th 
            class="border-2 border-slate-900 bg-slate-700 text-white px-1 py-1 text-center font-bold drag-col 
                   {col.type === 'time' ? 'w-[85px] text-left' : ''} 
                   {colDragItem === col.id ? 'col-dragging' : ''} 
                   {colDragOverId === col.id ? 'col-drag-over' : ''}"
            draggable="true"
            on:dragstart={(e) => onColDragStart(col.id, e)}
            on:dragover={(e) => onColDragOver(col.id, e)}
            on:dragleave={onColDragLeave}
            on:drop={(e) => onColDrop(col.id, e)}
            on:dragend={onColDragEnd}
          >
            <div class="col-header-content">
              <div class="col-drag-handle print:hidden">
                <GripHorizontal class="w-3 h-3 text-slate-300" />
              </div>
              <div class="font-bold text-[9px]">{col.name}</div>
              <button 
                class="col-edit-btn print:hidden p-0.5 rounded hover:bg-slate-600 transition-colors"
                on:click|stopPropagation={() => openColumnModal(col)}
                title="Edit column"
              >
                <Pencil class="w-3 h-3 text-slate-300" />
              </button>
            </div>
          </th>
        {/each}
        
        <!-- Delete row column -->
        <th class="border-2 border-slate-900 bg-slate-700 text-white px-1 py-1 w-[30px] print:hidden"></th>
      </tr>
    </thead>
    <tbody>
      {#each timeSlots as slot, idx (slot.id)}
        <tr
          draggable="true"
          on:dragstart={(e) => onDragStart(slot.id, e)}
          on:dragover={(e) => onDragOver(idx, e)}
          on:dragleave={onDragLeave}
          on:drop={(e) => onDrop(slot.id, e)}
          on:dragend={onDragEnd}
          class="drag-row {dragOverIndex === idx ? 'drag-over' : ''} {dragItem === slot.id ? 'dragging' : ''}"
        >
          <!-- Drag Handle -->
          <td class="border-2 border-slate-900 bg-slate-50 px-1 py-1 text-center cursor-move print:hidden">
            <GripVertical class="w-4 h-4 text-slate-400 inline-block" />
          </td>
          
          <!-- Dynamic columns based on current order -->
          {#each columns as col (col.id)}
            {#if col.type === 'time'}
              <td class="border-2 border-slate-900 bg-slate-100 px-1 py-1 align-top">
                <input
                  type="text"
                  value={getCellValue(slot, col)}
                  on:input={(e) => setCellValue(slot, col, e.currentTarget.value)}
                  class="w-full px-1 py-1 text-[8px] font-semibold border border-slate-300 rounded focus:ring-1 focus:ring-blue-500 focus:border-transparent print:border-0"
                  placeholder="Time..."
                />
              </td>
            {:else}
              <td class="editable-cell border-2 border-slate-900 px-1 py-1 align-top">
                <textarea
                  value={getCellValue(slot, col)}
                  on:input={(e) => setCellValue(slot, col, e.currentTarget.value)}
                  class="w-full min-h-[45px] px-1 py-1 text-[10px] border border-slate-300 rounded focus:ring-1 focus:ring-blue-500 focus:border-transparent resize-y print:border-0 print:p-0 print:min-h-0"
                />
              </td>
            {/if}
          {/each}
          
          <!-- Delete Button -->
          <td class="border-2 border-slate-900 bg-slate-50 px-1 py-1 text-center print:hidden">
            <button
              on:click={() => deleteRow(slot.id)}
              class="text-red-600 hover:text-red-800 hover:bg-red-50 p-1 rounded transition-colors"
              title="Delete row"
            >
              <Trash2 class="w-4 h-4" />
            </button>
          </td>
        </tr>
      {/each}
    </tbody>
  </table>
  
  <!-- Add Row & Column Buttons -->
  <div class="mt-4 flex gap-2 print:hidden">
    <button
      on:click={addRow}
      class="px-4 py-2 bg-blue-600 text-white rounded-lg hover:bg-blue-700 flex items-center gap-2 font-medium text-sm transition-colors"
    >
      <Plus class="w-4 h-4" />
      Add Row
    </button>
    <button
      on:click={addColumn}
      class="px-4 py-2 bg-indigo-600 text-white rounded-lg hover:bg-indigo-700 flex items-center gap-2 font-medium text-sm transition-colors"
    >
      <Plus class="w-4 h-4" />
      Add Column
    </button>
  </div>

  <!-- Export/Import Buttons -->
  <div class="mt-4 flex gap-2 print:hidden">
    <button
      on:click={exportToExcel}
      class="px-4 py-2 bg-green-600 text-white rounded-lg hover:bg-green-700 flex items-center gap-2 font-medium text-sm transition-colors"
    >
      <Download class="w-4 h-4" />
      Export to Excel
    </button>
    
    <label class="px-4 py-2 bg-purple-600 text-white rounded-lg hover:bg-purple-700 flex items-center gap-2 font-medium text-sm transition-colors cursor-pointer">
      <Upload class="w-4 h-4" />
      Import from Excel
      <input
        type="file"
        accept=".csv,.xlsx"
        bind:this={fileInput}
        on:change={importFromExcel}
        class="hidden"
      />
    </label>
  </div>

  <!-- Clipboard indicator -->
  {#if columnClipboard}
    <div class="mt-2 text-sm text-slate-600 print:hidden">
      <span class="inline-flex items-center gap-1 bg-green-100 text-green-800 px-2 py-1 rounded">
        <Copy class="w-3 h-3" />
        Clipboard: Column data ready to paste
      </span>
    </div>
  {/if}
</div>

<!-- Column Edit Modal -->
{#if showColumnModal && editingColumn}
  <div 
    class="modal-overlay" 
    on:click={closeColumnModal} 
    on:keydown={(e) => e.key === 'Escape' && closeColumnModal()}
    role="dialog"
    aria-modal="true"
    aria-labelledby="column-modal-title"
  >
    <div class="modal-content" on:click|stopPropagation role="document">
      <div class="flex items-center justify-between mb-4">
        <h3 id="column-modal-title" class="text-lg font-bold text-slate-800">Edit Column</h3>
        <button 
          on:click={closeColumnModal}
          class="p-1 rounded hover:bg-slate-100 transition-colors"
          aria-label="Close modal"
        >
          <X class="w-5 h-5 text-slate-500" />
        </button>
      </div>
      
      <!-- Column Name -->
      <div class="mb-4">
        <label for="column-name-input" class="block text-sm font-medium text-slate-700 mb-1">Column Name</label>
        <input
          id="column-name-input"
          type="text"
          bind:value={editColumnName}
          class="w-full px-3 py-2 border border-slate-300 rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-transparent"
          placeholder="Enter column name..."
        />
      </div>
      
      <!-- Action Buttons -->
      <div class="flex flex-col gap-2">
        <button
          on:click={saveColumnName}
          class="w-full px-4 py-2 bg-blue-600 text-white rounded-lg hover:bg-blue-700 flex items-center justify-center gap-2 font-medium transition-colors"
        >
          Save Name
        </button>
        
        <div class="flex gap-2">
          <button
            on:click={copyColumn}
            class="flex-1 px-4 py-2 bg-slate-100 text-slate-700 rounded-lg hover:bg-slate-200 flex items-center justify-center gap-2 font-medium transition-colors"
          >
            <Copy class="w-4 h-4" />
            Copy Column
          </button>
          
          <button
            on:click={pasteColumn}
            class="flex-1 px-4 py-2 bg-slate-100 text-slate-700 rounded-lg hover:bg-slate-200 flex items-center justify-center gap-2 font-medium transition-colors {!columnClipboard ? 'opacity-50' : ''}"
            disabled={!columnClipboard}
          >
            <ClipboardPaste class="w-4 h-4" />
            Paste Into
          </button>
        </div>
        
        {#if editingColumn.type === 'day'}
          <button
            on:click={deleteColumn}
            class="w-full px-4 py-2 bg-red-100 text-red-700 rounded-lg hover:bg-red-200 flex items-center justify-center gap-2 font-medium transition-colors"
          >
            <Trash2 class="w-4 h-4" />
            Delete Column
          </button>
        {/if}
      </div>
      
      <p class="mt-4 text-xs text-slate-500">
        Tip: Drag columns by the grip handle to reorder them. Copy a column's data and paste it into another column.
      </p>
    </div>
  </div>
{/if}