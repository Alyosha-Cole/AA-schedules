<script lang="ts">
  import { Printer, Plus, Pencil, Trash2, X, Copy } from 'lucide-svelte';
  import ScheduleTable from './ScheduleTable.svelte';

  export let dates: any[];
  export let days: string[];
  export let startDate: Date;

  // Default column structure (shared across all units)
  const defaultColumns = [
    { id: 'weekdayTime', name: 'Weekday Time', type: 'time' as const, group: 'weekday' },
    { id: 'mon', name: 'Monday', type: 'day' as const, dayIndex: 0, group: 'weekday' },
    { id: 'tue', name: 'Tuesday', type: 'day' as const, dayIndex: 1, group: 'weekday' },
    { id: 'wed', name: 'Wednesday', type: 'day' as const, dayIndex: 2, group: 'weekday' },
    { id: 'thu', name: 'Thursday', type: 'day' as const, dayIndex: 3, group: 'weekday' },
    { id: 'fri', name: 'Friday', type: 'day' as const, dayIndex: 4, group: 'weekday' },
    { id: 'weekendTime', name: 'Weekend Time', type: 'time' as const, group: 'weekend' },
    { id: 'sat', name: 'Saturday', type: 'day' as const, dayIndex: 5, group: 'weekend' },
    { id: 'sun', name: 'Sunday', type: 'day' as const, dayIndex: 6, group: 'weekend' },
  ];

  // Default time slots for each unit type
  const defaultTimeSlots = {
    national1: [
      { id: 1, weekdayTime: '7:00-7:15 AM', weekendTime: '8:00-8:25 AM', weekdays: 'Wake up-Hygiene', weekend: 'Wake up-Hygiene' },
      { id: 2, weekdayTime: '7:15-7:30 AM', weekendTime: '8:30-9:00 AM', weekdays: 'Room Clean', weekend: 'Breakfast/Med pass' },
      { id: 3, weekdayTime: '7:30-8:00 AM', weekendTime: '9:15-10:30 AM', weekdays: 'Morning Check-in', weekend: 'Staff led group' },
      { id: 4, weekdayTime: '8:00-8:25 AM', weekendTime: '11:00 AM-12:00 PM', weekdays: 'Med pass', weekend: 'Rec Time' },
      { id: 5, weekdayTime: '8:30-8:50 AM', weekendTime: '12:00-12:50 PM', weekdays: 'Goal of the day', weekend: 'Major Clean-Up' },
      { id: 6, weekdayTime: '9:00-11:00 AM', weekendTime: '1:00-1:20 PM', weekdays: 'School', weekend: 'Lunch' },
      { id: 7, weekdayTime: '11:00-1:00 PM', weekendTime: '1:30-2:00 PM', weekdays: 'Lunch Periods', weekend: 'Writing Skills' },
      { id: 8, weekdayTime: '1:00-2:30 PM', weekendTime: '2:00-2:50 PM', weekdays: 'School', weekend: 'TV Program' },
      { id: 9, weekdayTime: '3:00-4:00 PM', weekendTime: '3:00-4:00 PM', weekdays: 'Group Session', weekend: 'Rec-Time' },
      { id: 10, weekdayTime: '4:20-5:20 PM', weekendTime: '4:10-5:00 PM', weekdays: 'Rec-Time', weekend: 'Board Games' },
      { id: 11, weekdayTime: '5:30-5:50 PM', weekendTime: '5:00-5:20 PM', weekdays: 'Dinner', weekend: 'Unstructured time' },
      { id: 12, weekdayTime: '6:00-7:00 PM', weekendTime: '5:30-5:50 PM', weekdays: 'Showers', weekend: 'Dinner' },
      { id: 13, weekdayTime: '7:00-7:45 PM', weekendTime: '6:00-7:00 PM', weekdays: 'PM Check-in/Med pass', weekend: 'Showers' },
      { id: 14, weekdayTime: '8:00 PM', weekendTime: '7:00-7:45 PM', weekdays: 'Bed Time', weekend: 'PM Check-in/Med pass' },
      { id: 15, weekdayTime: '', weekendTime: '8:00 PM', weekdays: '', weekend: 'Bed Time' },
    ],
    national2east: [
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
    ],
    national2west: [
      { id: 1, weekdayTime: '7:00-7:15 AM', weekendTime: '8:00-8:25 AM', weekdays: 'Wake up-Hygiene', weekend: 'Wake up-Hygiene' },
      { id: 2, weekdayTime: '7:15-7:30 AM', weekendTime: '8:30-9:00 AM', weekdays: 'Room Clean', weekend: 'Breakfast/Med pass' },
      { id: 3, weekdayTime: '7:30-8:00 AM', weekendTime: '9:15-10:30 AM', weekdays: 'Morning Check-in', weekend: 'Staff led group' },
      { id: 4, weekdayTime: '8:00-8:25 AM', weekendTime: '11:00 AM-12:00 PM', weekdays: 'Med pass', weekend: 'Rec Time' },
      { id: 5, weekdayTime: '8:30-8:50 AM', weekendTime: '12:00-12:20 PM', weekdays: 'Goal of the day', weekend: 'Lunch' },
      { id: 6, weekdayTime: '9:00-12:00 PM', weekendTime: '12:30-1:30 PM', weekdays: 'School', weekend: 'Major Clean-up' },
      { id: 7, weekdayTime: '11:00-1:00 PM', weekendTime: '1:30-2:00 PM', weekdays: 'Lunch Periods', weekend: 'Writing Skills' },
      { id: 8, weekdayTime: '1:00-2:30 PM', weekendTime: '2:00-2:50 PM', weekdays: 'School', weekend: 'TV Program' },
      { id: 9, weekdayTime: '2:30-3:20 PM', weekendTime: '3:00-4:00 PM', weekdays: 'Rec-Time', weekend: 'Rec-Time' },
      { id: 10, weekdayTime: '3:20-4:20 PM', weekendTime: '4:10-4:30 PM', weekdays: 'Showers', weekend: 'Unstructured time' },
      { id: 11, weekdayTime: '4:30-4:50 PM', weekendTime: '4:30-4:50 PM', weekdays: 'Dinner', weekend: 'Dinner' },
      { id: 12, weekdayTime: '5:00-6:00 PM', weekendTime: '5:00-5:50 PM', weekdays: 'Group', weekend: 'Unstructured time' },
      { id: 13, weekdayTime: '6:00-6:30 PM', weekendTime: '6:00-7:00 PM', weekdays: 'Unstructured Time', weekend: 'Showers' },
      { id: 14, weekdayTime: '6:30-7:45 PM', weekendTime: '7:00-7:45 PM', weekdays: 'PM Check-in/Med pass', weekend: 'PM Check-in/Med pass' },
      { id: 15, weekdayTime: '8:00 PM', weekendTime: '8:00 PM', weekdays: 'Bed Time', weekend: 'Bed Time' },
    ],
    national3: [
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
    ],
    blank: [
      { id: 1, weekdayTime: '', weekendTime: '', weekdays: '', weekend: '' },
      { id: 2, weekdayTime: '', weekendTime: '', weekdays: '', weekend: '' },
      { id: 3, weekdayTime: '', weekendTime: '', weekdays: '', weekend: '' },
      { id: 4, weekdayTime: '', weekendTime: '', weekdays: '', weekend: '' },
      { id: 5, weekdayTime: '', weekendTime: '', weekdays: '', weekend: '' },
    ]
  };

  // Unit type definition
  interface Unit {
    id: string;
    name: string;
    columns: typeof defaultColumns;
    timeSlots: typeof defaultTimeSlots.national1;
    scheduleData: Record<number, Record<number, string>>;
    nextRowId: number;
  }

  // Initialize schedule data from time slots
  function initializeScheduleData(timeSlots: typeof defaultTimeSlots.national1): Record<number, Record<number, string>> {
    const data: Record<number, Record<number, string>> = {};
    timeSlots.forEach(slot => {
      data[slot.id] = {};
      for (let i = 0; i < 7; i++) {
        data[slot.id][i] = i < 5 ? slot.weekdays : slot.weekend;
      }
    });
    return data;
  }

  // Create a new unit with given template
  function createUnit(id: string, name: string, template: keyof typeof defaultTimeSlots = 'blank'): Unit {
    const timeSlots = JSON.parse(JSON.stringify(defaultTimeSlots[template]));
    return {
      id,
      name,
      columns: JSON.parse(JSON.stringify(defaultColumns)),
      timeSlots,
      scheduleData: initializeScheduleData(timeSlots),
      nextRowId: timeSlots.length + 1
    };
  }

  // Initialize with default units
  let units: Unit[] = [
    createUnit('national1', 'National 1', 'national1'),
    createUnit('national2east', 'National 2 East', 'national2east'),
    createUnit('national2west', 'National 2 West', 'national2west'),
    createUnit('national3', 'National 3', 'national3'),
  ];

  // Active unit
  let activeUnitId = 'national3';
  $: activeUnit = units.find(u => u.id === activeUnitId) || units[0];

  // Column clipboard (shared across all units)
  let columnClipboard: { columnId: string; data: Record<number, string>; timeData?: string[] } | null = null;

  // Unit management modal
  let showUnitModal = false;
  let editingUnit: Unit | null = null;
  let unitModalMode: 'add' | 'edit' = 'add';
  let unitName = '';
  let unitTemplate: keyof typeof defaultTimeSlots = 'blank';

  // === UNIT MANAGEMENT ===
  function openAddUnitModal() {
    unitModalMode = 'add';
    editingUnit = null;
    unitName = '';
    unitTemplate = 'blank';
    showUnitModal = true;
  }

  function openEditUnitModal(unit: Unit) {
    unitModalMode = 'edit';
    editingUnit = unit;
    unitName = unit.name;
    showUnitModal = true;
  }

  function closeUnitModal() {
    showUnitModal = false;
    editingUnit = null;
    unitName = '';
  }

  function saveUnit() {
    if (!unitName.trim()) {
      alert('Please enter a unit name');
      return;
    }

    if (unitModalMode === 'add') {
      const newId = `unit_${Date.now()}`;
      const newUnit = createUnit(newId, unitName.trim(), unitTemplate);
      units = [...units, newUnit];
      activeUnitId = newId;
    } else if (editingUnit) {
      units = units.map(u => 
        u.id === editingUnit!.id 
          ? { ...u, name: unitName.trim() }
          : u
      );
    }
    closeUnitModal();
  }

  function deleteUnit(unitId: string) {
    if (units.length <= 1) {
      alert('Cannot delete the last unit');
      return;
    }
    
    if (!confirm('Are you sure you want to delete this unit?')) return;
    
    units = units.filter(u => u.id !== unitId);
    if (activeUnitId === unitId) {
      activeUnitId = units[0].id;
    }
    closeUnitModal();
  }

  function duplicateUnit(unit: Unit) {
    const newId = `unit_${Date.now()}`;
    const newUnit: Unit = {
      ...JSON.parse(JSON.stringify(unit)),
      id: newId,
      name: `${unit.name} (Copy)`
    };
    units = [...units, newUnit];
    activeUnitId = newId;
  }

  // === SCHEDULE TABLE EVENT HANDLERS ===
  function handleUpdateColumns(event: CustomEvent) {
    units = units.map(u => 
      u.id === activeUnitId 
        ? { ...u, columns: event.detail }
        : u
    );
  }

  function handleUpdateTimeSlots(event: CustomEvent) {
    units = units.map(u => 
      u.id === activeUnitId 
        ? { ...u, timeSlots: event.detail }
        : u
    );
  }

  function handleUpdateScheduleData(event: CustomEvent) {
    units = units.map(u => 
      u.id === activeUnitId 
        ? { ...u, scheduleData: event.detail }
        : u
    );
  }

  function handleUpdateClipboard(event: CustomEvent) {
    columnClipboard = event.detail;
  }

  function handleAddRow() {
    const unit = units.find(u => u.id === activeUnitId);
    if (!unit) return;

    const newSlot = {
      id: unit.nextRowId,
      weekdayTime: '',
      weekendTime: '',
      weekdays: '',
      weekend: ''
    };

    const newScheduleData = { ...unit.scheduleData };
    newScheduleData[newSlot.id] = {};
    for (let i = 0; i < 7; i++) {
      newScheduleData[newSlot.id][i] = '';
    }

    units = units.map(u => 
      u.id === activeUnitId 
        ? { 
            ...u, 
            timeSlots: [...u.timeSlots, newSlot],
            scheduleData: newScheduleData,
            nextRowId: u.nextRowId + 1
          }
        : u
    );
  }

  function handleDeleteRow(event: CustomEvent) {
    const slotId = event.detail;
    const unit = units.find(u => u.id === activeUnitId);
    if (!unit) return;

    const newScheduleData = { ...unit.scheduleData };
    delete newScheduleData[slotId];

    units = units.map(u => 
      u.id === activeUnitId 
        ? { 
            ...u, 
            timeSlots: u.timeSlots.filter(s => s.id !== slotId),
            scheduleData: newScheduleData
          }
        : u
    );
  }

  function handleAddColumn() {
    const unit = units.find(u => u.id === activeUnitId);
    if (!unit) return;

    const newId = `custom_${Date.now()}`;
    const existingDayIndexes = unit.columns.filter(c => c.type === 'day').map(c => c.dayIndex ?? -1);
    const newDayIndex = Math.max(...existingDayIndexes) + 1;

    const newCol = {
      id: newId,
      name: `Day ${newDayIndex + 1}`,
      type: 'day' as const,
      dayIndex: newDayIndex,
      group: 'custom'
    };

    const newScheduleData = { ...unit.scheduleData };
    unit.timeSlots.forEach(slot => {
      if (newScheduleData[slot.id]) {
        newScheduleData[slot.id] = { ...newScheduleData[slot.id], [newDayIndex]: '' };
      }
    });

    units = units.map(u => 
      u.id === activeUnitId 
        ? { 
            ...u, 
            columns: [...u.columns, newCol],
            scheduleData: newScheduleData
          }
        : u
    );
  }

  function handleImportData(event: CustomEvent) {
    const { dataLines, columns: importedColumns } = event.detail as { 
      dataLines: string[][]; 
      columns: typeof defaultColumns;
    };
    
    const unit = units.find(u => u.id === activeUnitId);
    if (!unit) return;

    // Build new time slots and schedule data from imported data
    const newTimeSlots: typeof defaultTimeSlots.national1 = [];
    const newScheduleData: Record<number, Record<number, string>> = {};
    
    dataLines.forEach((cells, index) => {
      const newId = index + 1;
      
      // Find time columns and day columns based on current column structure
      let weekdayTime = '';
      let weekendTime = '';
      
      importedColumns.forEach((col, colIdx) => {
        if (col.id === 'weekdayTime' && cells[colIdx]) {
          weekdayTime = cells[colIdx];
        } else if (col.id === 'weekendTime' && cells[colIdx]) {
          weekendTime = cells[colIdx];
        }
      });
      
      const newSlot = {
        id: newId,
        weekdayTime,
        weekendTime,
        weekdays: '',
        weekend: ''
      };
      
      newTimeSlots.push(newSlot);
      
      // Build schedule data for this row
      newScheduleData[newId] = {};
      importedColumns.forEach((col, colIdx) => {
        if (col.type === 'day' && col.dayIndex !== undefined && cells[colIdx] !== undefined) {
          newScheduleData[newId][col.dayIndex] = cells[colIdx];
        }
      });
    });
    
    // Update the unit with imported data including the column names
    units = units.map(u => 
      u.id === activeUnitId 
        ? { 
            ...u, 
            columns: importedColumns,  // Use the imported columns with updated names
            timeSlots: newTimeSlots,
            scheduleData: newScheduleData,
            nextRowId: newTimeSlots.length + 1
          }
        : u
    );
  }

  // === PRINT FUNCTIONS ===
  function printSchedule() {
    const portraitStyle = document.getElementById('staff-schedule-portrait');
    if (portraitStyle) portraitStyle.remove();
    
    const landscapeStyle = document.createElement('style');
    landscapeStyle.id = 'master-schedule-landscape';
    landscapeStyle.textContent = `
      @media print {
        @page {
          size: landscape !important;
          margin: 0.4in 0.5in !important;
        }
      }
    `;
    document.head.appendChild(landscapeStyle);
    
    const masterEl = document.querySelector('.master-schedule-container');
    if (masterEl) {
      masterEl.classList.add('facility-print-target');
      
      setTimeout(() => {
        window.print();
        
        setTimeout(() => {
          masterEl.classList.remove('facility-print-target');
          const styleEl = document.getElementById('master-schedule-landscape');
          if (styleEl) styleEl.remove();
        }, 500);
      }, 100);
    }
  }

  function downloadPDF() {
    const portraitStyle = document.getElementById('staff-schedule-portrait');
    if (portraitStyle) portraitStyle.remove();
    
    const landscapeStyle = document.createElement('style');
    landscapeStyle.id = 'master-schedule-landscape';
    landscapeStyle.textContent = `
      @media print {
        @page {
          size: landscape !important;
          margin: 0.4in 0.5in !important;
        }
      }
    `;
    document.head.appendChild(landscapeStyle);
    
    const masterEl = document.querySelector('.master-schedule-container');
    if (masterEl) {
      masterEl.classList.add('facility-print-target');
    }
    
    alert('In the print dialog:\n1. Select "Save as PDF" or "Microsoft Print to PDF" as the printer\n2. Verify landscape orientation is selected\n3. Click Save and name your file\n\nThis PDF can be taken to a print shop for large format printing.');
    
    setTimeout(() => {
      window.print();
      
      setTimeout(() => {
        if (masterEl) {
          masterEl.classList.remove('facility-print-target');
        }
        const styleEl = document.getElementById('master-schedule-landscape');
        if (styleEl) styleEl.remove();
      }, 1000);
    }, 100);
  }
</script>

<style>
  @media print {
    :global(.master-schedule-container) {
      display: none !important;
    }
    
    :global(.master-schedule-container.facility-print-target) {
      display: flex !important;
      flex-direction: column !important;
      justify-content: center !important;
      align-items: center !important;
    }

    :global([data-schedule-id]) {
      display: none !important;
    }

    .print-hide {
      display: none !important;
    }

    .master-schedule-container {
      background: white !important;
      min-height: 100vh !important;
      height: 100vh !important;
      display: flex !important;
      flex-direction: column !important;
      justify-content: center !important;
      padding: 0 !important;
      margin: 0 !important;
      box-shadow: none !important;
      border-radius: 0 !important;
    }

    .overflow-x-auto {
      overflow: visible !important;
      width: 100% !important;
      max-width: 100% !important;
      display: flex !important;
      justify-content: center !important;
    }

    :global(.master-schedule-table) {
      font-size: 6.5pt !important;
      background: white !important;
      width: 98% !important;
      max-width: 98% !important;
      table-layout: fixed !important;
    }

    :global(.master-schedule-table th),
    :global(.master-schedule-table td) {
      padding: 2px 2px !important;
      background-clip: padding-box !important;
      overflow: hidden !important;
      word-wrap: break-word !important;
    }

    :global(.master-schedule-table th:nth-child(1)),
    :global(.master-schedule-table td:nth-child(1)),
    :global(.master-schedule-table th:nth-child(7)),
    :global(.master-schedule-table td:nth-child(7)) {
      width: 9% !important;
      font-size: 5.5pt !important;
    }

    :global(.master-schedule-table th:nth-child(2)),
    :global(.master-schedule-table td:nth-child(2)),
    :global(.master-schedule-table th:nth-child(3)),
    :global(.master-schedule-table td:nth-child(3)),
    :global(.master-schedule-table th:nth-child(4)),
    :global(.master-schedule-table td:nth-child(4)),
    :global(.master-schedule-table th:nth-child(5)),
    :global(.master-schedule-table td:nth-child(5)),
    :global(.master-schedule-table th:nth-child(6)),
    :global(.master-schedule-table td:nth-child(6)),
    :global(.master-schedule-table th:nth-child(8)),
    :global(.master-schedule-table td:nth-child(8)),
    :global(.master-schedule-table th:nth-child(9)),
    :global(.master-schedule-table td:nth-child(9)) {
      width: 11.5% !important;
    }

    :global(.master-schedule-table textarea),
    :global(.master-schedule-table input) {
      border: none !important;
      background: transparent !important;
      padding: 0 !important;
      margin: 0 !important;
      min-height: auto !important;
      font-size: 6pt !important;
      line-height: 1.1 !important;
    }

    .unit-header {
      font-size: 11pt !important;
      margin-bottom: 6px !important;
      text-align: center !important;
    }

    .print-footer {
      font-size: 7pt !important;
      margin-top: 6px !important;
      text-align: center !important;
    }

    :global(body) {
      background: white !important;
    }

    :global(.master-schedule-table),
    :global(.master-schedule-table th),
    :global(.master-schedule-table td) {
      border-color: #1e293b !important;
      print-color-adjust: exact !important;
      -webkit-print-color-adjust: exact !important;
    }
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
    min-width: 400px;
    max-width: 90vw;
    box-shadow: 0 25px 50px -12px rgba(0, 0, 0, 0.25);
  }
</style>

<div class="master-schedule-container bg-white rounded-lg shadow-lg p-6 mb-6">
  <!-- Header with Print Button -->
  <div class="flex items-center justify-between mb-6 print-hide">
    <div>
      <h2 class="text-2xl font-bold text-slate-800">Master Daily Schedule</h2>
      <p class="text-slate-600 mt-1">Recurring Daily Schedule Template</p>
    </div>
    <div class="flex gap-2">
      <button
        on:click={printSchedule}
        class="px-4 py-2 bg-blue-600 text-white rounded-lg hover:bg-blue-700 flex items-center gap-2 font-medium transition-colors"
        title="Print master schedule"
      >
        <Printer class="w-4 h-4" />
        Print
      </button>
      <button
        on:click={downloadPDF}
        class="px-4 py-2 bg-green-600 text-white rounded-lg hover:bg-green-700 flex items-center gap-2 font-medium transition-colors"
        title="Save as PDF for print shop"
      >
        <svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
          <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 10v6m0 0l-3-3m3 3l3-3m2 8H7a2 2 0 01-2-2V5a2 2 0 012-2h5.586a1 1 0 01.707.293l5.414 5.414a1 1 0 01.293.707V19a2 2 0 01-2 2z" />
        </svg>
        Download PDF
      </button>
    </div>
  </div>

  <!-- Print-only header -->
  <div class="hidden print:block mb-2">
    <h1 class="unit-header font-bold text-slate-900">{activeUnit?.name} - Master Daily Schedule</h1>
  </div>

  <!-- Unit Tabs -->
  <div class="mb-6 print-hide">
    <div class="border-b border-slate-200">
      <div class="flex gap-1 flex-wrap items-center">
        {#each units as unit (unit.id)}
          <div class="relative group flex items-center">
            <button
              on:click={() => activeUnitId = unit.id}
              class="px-4 py-2 font-medium transition-colors {activeUnitId === unit.id 
                ? 'text-blue-600 border-b-2 border-blue-600 bg-blue-50' 
                : 'text-slate-600 hover:text-slate-900 hover:bg-slate-50'}"
            >
              {unit.name}
            </button>
            <button
              on:click={() => openEditUnitModal(unit)}
              class="p-1 rounded opacity-0 group-hover:opacity-100 hover:bg-blue-100 transition-all -ml-1"
              title="Edit unit"
              aria-label="Edit {unit.name}"
            >
              <Pencil class="w-3 h-3" />
            </button>
          </div>
        {/each}
        
        <!-- Add Unit Button -->
        <button
          on:click={openAddUnitModal}
          class="px-3 py-2 text-slate-500 hover:text-blue-600 hover:bg-blue-50 rounded-lg transition-colors flex items-center gap-1"
          title="Add new unit"
        >
          <Plus class="w-4 h-4" />
          Add Unit
        </button>
      </div>
    </div>
  </div>

  <!-- Instructions -->
  <div class="mb-4 p-4 bg-blue-50 border border-blue-200 rounded-lg print-hide">
    <p class="text-sm text-blue-800">
      <strong>📝 {activeUnit?.name}:</strong> Drag rows or columns to reorder. Click the pencil icon on column headers to rename, copy, or delete columns. Click any cell to edit activities.
    </p>
  </div>

  <!-- Active Unit Schedule -->
  {#if activeUnit}
    <ScheduleTable
      columns={activeUnit.columns}
      timeSlots={activeUnit.timeSlots}
      scheduleData={activeUnit.scheduleData}
      unitName={activeUnit.name}
      {columnClipboard}
      on:updateColumns={handleUpdateColumns}
      on:updateTimeSlots={handleUpdateTimeSlots}
      on:updateScheduleData={handleUpdateScheduleData}
      on:updateClipboard={handleUpdateClipboard}
      on:addRow={handleAddRow}
      on:deleteRow={handleDeleteRow}
      on:addColumn={handleAddColumn}
      on:importData={handleImportData}
    />
  {/if}

  <!-- Print Footer Info -->
  <div class="mt-6 text-sm text-slate-600 print-hide">
    <p class="mb-2"><strong>🖨️ Printing:</strong> Click "Print" to print directly. The schedule automatically uses landscape orientation and fills the page.</p>
    <p><strong>📄 For Print Shops:</strong> Click "Download PDF", then in the print dialog select "Save as PDF" as the destination.</p>
  </div>

  <!-- Print-only footer -->
  <div class="hidden print:block mt-2 text-slate-600 print-footer">
    <p>Printed on {new Date().toLocaleDateString()}</p>
  </div>
</div>

<!-- Unit Management Modal -->
{#if showUnitModal}
  <div 
    class="modal-overlay" 
    on:click={closeUnitModal} 
    on:keydown={(e) => e.key === 'Escape' && closeUnitModal()}
    role="dialog"
    aria-modal="true"
    aria-labelledby="unit-modal-title"
  >
    <div class="modal-content" on:click|stopPropagation role="document">
      <div class="flex items-center justify-between mb-4">
        <h3 id="unit-modal-title" class="text-lg font-bold text-slate-800">
          {unitModalMode === 'add' ? 'Add New Unit' : 'Edit Unit'}
        </h3>
        <button 
          on:click={closeUnitModal}
          class="p-1 rounded hover:bg-slate-100 transition-colors"
          aria-label="Close modal"
        >
          <X class="w-5 h-5 text-slate-500" />
        </button>
      </div>
      
      <!-- Unit Name -->
      <div class="mb-4">
        <label for="unit-name-input" class="block text-sm font-medium text-slate-700 mb-1">Unit Name</label>
        <input
          id="unit-name-input"
          type="text"
          bind:value={unitName}
          class="w-full px-3 py-2 border border-slate-300 rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-transparent"
          placeholder="Enter unit name..."
        />
      </div>
      
      <!-- Template Selection (only for new units) -->
      {#if unitModalMode === 'add'}
        <div class="mb-4">
          <label for="unit-template-select" class="block text-sm font-medium text-slate-700 mb-1">Start from Template</label>
          <select
            id="unit-template-select"
            bind:value={unitTemplate}
            class="w-full px-3 py-2 border border-slate-300 rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-transparent"
          >
            <option value="blank">Blank Schedule</option>
            <option value="national1">National 1 Template</option>
            <option value="national2east">National 2 East Template</option>
            <option value="national2west">National 2 West Template</option>
            <option value="national3">National 3 Template</option>
          </select>
        </div>
      {/if}
      
      <!-- Action Buttons -->
      <div class="flex flex-col gap-2">
        <button
          on:click={saveUnit}
          class="w-full px-4 py-2 bg-blue-600 text-white rounded-lg hover:bg-blue-700 flex items-center justify-center gap-2 font-medium transition-colors"
        >
          {unitModalMode === 'add' ? 'Create Unit' : 'Save Changes'}
        </button>
        
        {#if unitModalMode === 'edit' && editingUnit}
          <div class="flex gap-2">
            <button
              on:click={() => duplicateUnit(editingUnit)}
              class="flex-1 px-4 py-2 bg-slate-100 text-slate-700 rounded-lg hover:bg-slate-200 flex items-center justify-center gap-2 font-medium transition-colors"
            >
              <Copy class="w-4 h-4" />
              Duplicate
            </button>
            
            <button
              on:click={() => deleteUnit(editingUnit.id)}
              class="flex-1 px-4 py-2 bg-red-100 text-red-700 rounded-lg hover:bg-red-200 flex items-center justify-center gap-2 font-medium transition-colors"
            >
              <Trash2 class="w-4 h-4" />
              Delete
            </button>
          </div>
        {/if}
      </div>
    </div>
  </div>
{/if}