<script lang="ts">
  import { GripVertical, Plus, Trash2 } from 'lucide-svelte';

  export let dates: any[];
  export let onPrint: () => void;

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
  let nextId = 16;

  const dayNames = ['Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday', 'Saturday', 'Sunday'];

  function initializeSchedule() {
    timeSlots.forEach(slot => {
      scheduleData[slot.id] = {};
      for (let i = 0; i < 7; i++) {
        scheduleData[slot.id][i] = i < 5 ? slot.weekdays : slot.weekend;
      }
    });
  }

  initializeSchedule();

  function onDragStart(slotId: number, e: DragEvent) {
    dragItem = slotId;
    if (e.dataTransfer) {
      e.dataTransfer.effectAllowed = 'move';
      e.dataTransfer.setData('text/plain', String(slotId));
    }
  }

  function onDragOver(idx: number, e: DragEvent) {
    e.preventDefault();
    if (e.dataTransfer) e.dataTransfer.dropEffect = 'move';
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

  function addRow() {
    const newSlot = { id: nextId++, weekdayTime: '', weekendTime: '', weekdays: '', weekend: '' };
    timeSlots = [...timeSlots, newSlot];
    scheduleData[newSlot.id] = {};
    for (let i = 0; i < 7; i++) {
      scheduleData[newSlot.id][i] = '';
    }
  }

  function deleteRow(slotId: number) {
    if (timeSlots.length <= 1) {
      alert('Cannot delete the last row');
      return;
    }
    timeSlots = timeSlots.filter(s => s.id !== slotId);
    delete scheduleData[slotId];
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
          <td class="border-2 border-slate-900 bg-slate-50 px-1 py-1 text-center cursor-move print:hidden">
            <GripVertical class="w-4 h-4 text-slate-400 inline-block" />
          </td>
          <td class="border-2 border-slate-900 bg-slate-100 px-1 py-1 align-top">
            <input
              type="text"
              bind:value={slot.weekdayTime}
              class="w-full px-1 py-1 text-[8px] font-semibold border border-slate-300 rounded focus:ring-1 focus:ring-blue-500 focus:border-transparent print:border-0"
              placeholder="Time..."
            />
          </td>
          {#each Array(5) as _, dayIndex}
            <td class="editable-cell border-2 border-slate-900 px-1 py-1 align-top">
              <textarea
                bind:value={scheduleData[slot.id][dayIndex]}
                class="w-full min-h-[45px] px-1 py-1 text-[10px] border border-slate-300 rounded focus:ring-1 focus:ring-blue-500 focus:border-transparent resize-y print:border-0 print:p-0 print:min-h-0"
              />
            </td>
          {/each}
          <td class="border-2 border-slate-900 bg-slate-100 px-1 py-1 align-top">
            <input
              type="text"
              bind:value={slot.weekendTime}
              class="w-full px-1 py-1 text-[8px] font-semibold border border-slate-300 rounded focus:ring-1 focus:ring-blue-500 focus:border-transparent print:border-0"
              placeholder="Time..."
            />
          </td>
          {#each Array(2) as _, weekendIdx}
            {@const dayIndex = weekendIdx + 5}
            <td class="editable-cell border-2 border-slate-900 px-1 py-1 align-top">
              <textarea
                bind:value={scheduleData[slot.id][dayIndex]}
                class="w-full min-h-[45px] px-1 py-1 text-[10px] border border-slate-300 rounded focus:ring-1 focus:ring-blue-500 focus:border-transparent resize-y print:border-0 print:p-0 print:min-h-0"
              />
            </td>
          {/each}
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
  
  <div class="mt-4 print:hidden">
    <button
      on:click={addRow}
      class="px-4 py-2 bg-blue-600 text-white rounded-lg hover:bg-blue-700 flex items-center gap-2 font-medium text-sm transition-colors"
    >
      <Plus class="w-4 h-4" />
      Add Row
    </button>
  </div>
</div>