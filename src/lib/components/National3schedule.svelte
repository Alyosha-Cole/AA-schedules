<script lang="ts">
  import { Printer, GripVertical, Plus, Trash2, Download, Upload } from 'lucide-svelte';

  export let dates: any[];
  export let onPrint: () => void;

  // Combined time slots with weekday/weekend indicators - now mutable
  let timeSlots = [
    { id: 1, weekdayTime: '7:00-7:15 AM', weekendTime: '8:00-8:25 AM', weekdays: 'Wake up-Hygiene', weekend: 'Wake up-Hygiene' },
    { id: 2, weekdayTime: '7:15-7:30 AM', weekendTime: '8:30-9:00 AM', weekdays: 'Room Clean', weekend: 'Breakfast/Med pass' },
    { id: 3, weekdayTime: '7:30-8:00 AM', weekendTime: '9:15-10:30 AM', weekdays: 'Morning Check-in', weekend: 'Staff led group' },
    { id: 4, weekdayTime: '8:00-8:25 AM', weekendTime: '11:00 AM-12:00 PM', weekdays: 'Med pass', weekend: 'Rec Time' },
    { id: 5, weekdayTime: '8:30-8:50 AM', weekendTime: '12:00-12:30 PM', weekdays: 'Goal of the day', weekend: 'Major Clean-Up' },
    { id: 6, weekdayTime: '9:00-11:25 AM', weekendTime: '12:30-12:50 PM', weekdays: 'School', weekend: 'Lunch' },
    { id: 7, weekdayTime: '11:00-1:00 PM', weekendTime: '1:00-2:00 PM', weekdays: 'Lunch Periods', weekend: 'Writing Skills' },
    { id: 8, weekdayTime: '1:00-2:30 PM', weekendTime: '2:00-2:50 PM', weekdays: 'School', weekend: 'TV Program' },
    { id: 9, weekdayTime: '2:30-3:20 PM', weekendTime: '3:00-4:00 PM', weekdays: 'Group Session', weekend: 'Rec-Time' },
    { id: 10, weekdayTime: '3:20-4:20 PM', weekendTime: '4:00-5:00 PM', weekdays: 'Rec-Time', weekend: 'Board Games' },
    { id: 11, weekdayTime: '4:30-4:50 PM', weekendTime: '5:00-5:20 PM', weekdays: 'Unstructured Time', weekend: 'Dinner' },
    { id: 12, weekdayTime: '5:00-5:20 PM', weekendTime: '5:30-5:50 PM', weekdays: 'Dinner', weekend: 'Unstructured time' },
    { id: 13, weekdayTime: '5:30-6:30 PM', weekendTime: '6:00-7:00 PM', weekdays: 'Showers', weekend: 'Showers' },
    { id: 14, weekdayTime: '6:30-7:45 PM', weekendTime: '7:00-7:45 PM', weekdays: 'PM Check-in/Med pass', weekend: 'PM Check-in/Med pass' },
    { id: 15, weekdayTime: '8:00 PM', weekendTime: '8:00 PM', weekdays: 'Bed Time', weekend: 'Bed Time' },
  ];

  // Schedule data: [slotId][dayIndex] = activity description
  let scheduleData: Record<number, Record<number, string>> = {};

  // Drag state
  let dragItem: number | null = null;
  let dragOverIndex: number | null = null;

  // Next ID for new rows
  let nextId = 16;

  // File input for import
  let fileInput: HTMLInputElement;

  // Day names for display (no dates)
  const dayNames = ['Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday', 'Saturday', 'Sunday'];

  // Initialize with defaults
  function initializeSchedule() {
    timeSlots.forEach(slot => {
      scheduleData[slot.id] = {};
      for (let i = 0; i < 7; i++) {
        if (i < 5) {
          scheduleData[slot.id][i] = slot.weekdays;
        } else {
          scheduleData[slot.id][i] = slot.weekend;
        }
      }
    });
  }

  initializeSchedule();

  // Drag and drop handlers
  function onDragStart(slotId: number, e: DragEvent) {
    dragItem = slotId;
    if (e.dataTransfer) {
      e.dataTransfer.effectAllowed = 'move';
      e.dataTransfer.setData('text/plain', String(slotId));
    }
  }

  function onDragOver(idx: number, e: DragEvent) {
    e.preventDefault();
    if (e.dataTransfer) {
      e.dataTransfer.dropEffect = 'move';
    }
    dragOverIndex = idx;
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
        timeSlots = newSlots;
      }
    }
    dragItem = null;
  }

  function onDragEnd() {
    dragOverIndex = null;
    dragItem = null;
  }

  // Add new row
  function addRow() {
    const newSlot = {
      id: nextId++,
      weekdayTime: '',
      weekendTime: '',
      weekdays: '',
      weekend: ''
    };
    timeSlots = [...timeSlots, newSlot];
    
    // Initialize schedule data for new row
    scheduleData[newSlot.id] = {};
    for (let i = 0; i < 7; i++) {
      scheduleData[newSlot.id][i] = '';
    }
  }

  // Delete row
  function deleteRow(slotId: number) {
    if (timeSlots.length <= 1) {
      alert('Cannot delete the last row');
      return;
    }
    timeSlots = timeSlots.filter(s => s.id !== slotId);
    delete scheduleData[slotId];
  }

  // Export to CSV (Excel-compatible)
  function exportToExcel() {
    // Create CSV header
    const dayNamesHeader = ['Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday', 'Saturday', 'Sunday'];
    let csv = 'Weekday Time,' + dayNamesHeader.slice(0, 5).join(',') + ',Weekend Time,' + dayNamesHeader.slice(5, 7).join(',') + '\n';
    
    // Add each row
    timeSlots.forEach(slot => {
      const row = [
        slot.weekdayTime || '',
        scheduleData[slot.id][0] || '',
        scheduleData[slot.id][1] || '',
        scheduleData[slot.id][2] || '',
        scheduleData[slot.id][3] || '',
        scheduleData[slot.id][4] || '',
        slot.weekendTime || '',
        scheduleData[slot.id][5] || '',
        scheduleData[slot.id][6] || ''
      ];
      // Escape commas in cell values
      csv += row.map(cell => `"${cell.replace(/"/g, '""')}"`).join(',') + '\n';
    });

    // Download
    const blob = new Blob([csv], { type: 'text/csv;charset=utf-8;' });
    const link = document.createElement('a');
    const url = URL.createObjectURL(blob);
    link.setAttribute('href', url);
    link.setAttribute('download', `National3_Schedule_${new Date().toISOString().split('T')[0]}.csv`);
    link.style.visibility = 'hidden';
    document.body.appendChild(link);
    link.click();
    document.body.removeChild(link);
  }

  // Import from CSV/Excel
  function importFromExcel(event: Event) {
    const input = event.target as HTMLInputElement;
    const file = input.files?.[0];
    if (!file) return;

    const reader = new FileReader();
    reader.onload = (e) => {
      const text = e.target?.result as string;
      const lines = text.split('\n');
      
      if (lines.length < 2) {
        alert('Invalid file format');
        return;
      }

      // Skip header row
      const dataLines = lines.slice(1).filter(line => line.trim());
      
      // Clear existing data
      timeSlots = [];
      scheduleData = {};
      
      // Parse each row
      dataLines.forEach((line, index) => {
        const cells = parseCSVLine(line);
        if (cells.length < 9) return; // Skip incomplete rows
        
        const newId = index + 1;
        const newSlot = {
          id: newId,
          weekdayTime: cells[0],
          weekendTime: cells[6],
          weekdays: cells[1], // Default from Monday
          weekend: cells[7]   // Default from Saturday
        };
        
        timeSlots.push(newSlot);
        
        // Set schedule data for all days
        scheduleData[newId] = {
          0: cells[1], // Mon
          1: cells[2], // Tue
          2: cells[3], // Wed
          3: cells[4], // Thu
          4: cells[5], // Fri
          5: cells[7], // Sat
          6: cells[8]  // Sun
        };
      });
      
      nextId = timeSlots.length + 1;
      timeSlots = timeSlots; // Trigger reactivity
      
      alert('Schedule imported successfully!');
    };
    
    reader.readAsText(file);
    input.value = ''; // Reset input
  }

  // Parse CSV line handling quoted fields with commas
  function parseCSVLine(line: string): string[] {
    const result = [];
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
  }

  .drag-row.dragging {
    opacity: 0.5;
    background-color: rgba(59, 130, 246, 0.1);
  }

  .drag-row.drag-over {
    border-top: 3px solid #3b82f6;
  }

  .drag-row {
    cursor: move;
  }

  @media print {
    .drag-row {
      cursor: default;
    }
  }
</style>

<div class="overflow-x-auto">
  <table class="master-schedule-table w-full border-collapse text-[10px]">
    <thead>
      <tr>
        <th class="border-2 border-slate-900 bg-slate-700 text-white px-1 py-1 w-[30px] print:hidden"></th>
        <th class="border-2 border-slate-900 bg-slate-700 text-white px-1 py-1 text-left font-bold text-[9px] w-[85px]">
          Weekday Time
        </th>
        {#each dayNames.slice(0, 5) as dayName}
          <th class="border-2 border-slate-900 bg-slate-700 text-white px-1 py-1 text-center font-bold">
            <div class="font-bold text-[9px]">{dayName}</div>
          </th>
        {/each}
        <th class="border-2 border-slate-900 bg-slate-700 text-white px-1 py-1 text-left font-bold text-[9px] w-[85px]">
          Weekend Time
        </th>
        {#each dayNames.slice(5, 7) as dayName}
          <th class="border-2 border-slate-900 bg-slate-700 text-white px-1 py-1 text-center font-bold">
            <div class="font-bold text-[9px]">{dayName}</div>
          </th>
        {/each}
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
          
          <!-- Weekday Time (editable) -->
          <td class="border-2 border-slate-900 bg-slate-100 px-1 py-1 align-top">
            <input
              type="text"
              bind:value={slot.weekdayTime}
              class="w-full px-1 py-1 text-[8px] font-semibold border border-slate-300 rounded focus:ring-1 focus:ring-blue-500 focus:border-transparent print:border-0"
              placeholder="Time..."
            />
          </td>
          
          <!-- Weekday activities (Mon-Fri) -->
          {#each Array(5) as _, dayIndex}
            <td class="editable-cell border-2 border-slate-900 px-1 py-1 align-top">
              <textarea
                bind:value={scheduleData[slot.id][dayIndex]}
                class="w-full min-h-[45px] px-1 py-1 text-[10px] border border-slate-300 rounded focus:ring-1 focus:ring-blue-500 focus:border-transparent resize-y print:border-0 print:p-0 print:min-h-0"
              />
            </td>
          {/each}
          
          <!-- Weekend Time (editable) -->
          <td class="border-2 border-slate-900 bg-slate-100 px-1 py-1 align-top">
            <input
              type="text"
              bind:value={slot.weekendTime}
              class="w-full px-1 py-1 text-[8px] font-semibold border border-slate-300 rounded focus:ring-1 focus:ring-blue-500 focus:border-transparent print:border-0"
              placeholder="Time..."
            />
          </td>
          
          <!-- Weekend activities (Sat-Sun) -->
          {#each Array(2) as _, weekendIdx}
            {@const dayIndex = weekendIdx + 5}
            <td class="editable-cell border-2 border-slate-900 px-1 py-1 align-top">
              <textarea
                bind:value={scheduleData[slot.id][dayIndex]}
                class="w-full min-h-[45px] px-1 py-1 text-[10px] border border-slate-300 rounded focus:ring-1 focus:ring-blue-500 focus:border-transparent resize-y print:border-0 print:p-0 print:min-h-0"
              />
            </td>
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
  
  <!-- Add Row Button -->
  <div class="mt-4 print:hidden">
    <button
      on:click={addRow}
      class="px-4 py-2 bg-blue-600 text-white rounded-lg hover:bg-blue-700 flex items-center gap-2 font-medium text-sm transition-colors"
    >
      <Plus class="w-4 h-4" />
      Add Row
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
        accept=".csv"
        bind:this={fileInput}
        on:change={importFromExcel}
        class="hidden"
      />
    </label>
  </div>
</div>