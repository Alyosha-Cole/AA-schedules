<script lang="ts">
  import { onMount } from 'svelte';
  import AuthGate from '$lib/components/AuthGate.svelte';
  import SchedulerApp from '$lib/components/SchedulerApp.svelte';

  // ===== PASSWORD PROTECTION =====
  const CORRECT_PASSWORD = 'AAscheduler2025';
  let isAuthenticated = false;
  let passwordInput = '';
  let passwordError = false;

  function checkPassword() {
    if (passwordInput === CORRECT_PASSWORD) {
      isAuthenticated = true;
      passwordError = false;
      sessionStorage.setItem('scheduler-auth', 'true');
    } else {
      passwordError = true;
      passwordInput = '';
    }
  }

  function logout() {
    isAuthenticated = false;
    sessionStorage.removeItem('scheduler-auth');
    passwordInput = '';
  }

  // ===== CONFIG =====
  const days = ['Mon', 'Tue', 'Wed', 'Thu', 'Fri', 'Sat', 'Sun'];

  // ===== PERIOD (2 weeks) =====
  let startDate = new Date(2025, 11, 1);

  function fmtMD(d: Date) {
    return `${d.getMonth() + 1}/${d.getDate()}`;
  }
  function addDays(base: Date, n: number) {
    const d = new Date(base);
    d.setDate(d.getDate() + n);
    return d;
  }

  $: dates = Array.from({ length: 14 }, (_, i) => {
    const date = addDays(startDate, i);
    const jsDay = date.getDay();
    return {
      dayName: days[jsDay === 0 ? 6 : jsDay - 1],
      date: fmtMD(date),
      dayIndex: i,
      fullDate: date
    };
  });

  // ===== STAFF POSITIONS =====
  let staffPositions = [
    {
      id: 1,
      name: 'Floor Staff',
      rate: 25,
      overtimeRate: 37.5,
      color: '#3b82f6', // blue
      people: [
        { id: 1, name: 'MOODY' }, { id: 2, name: 'SHAWELL' }, { id: 3, name: 'MILLER' },
        { id: 4, name: 'NGUYEN' }, { id: 5, name: 'JACKSON' }, { id: 6, name: 'MACRINA' },
        { id: 7, name: 'GARNER' }, { id: 8, name: 'DOUGLAS' }, { id: 9, name: 'MORALES' },
        { id: 10, name: 'ANDERSON' }, { id: 11, name: 'CLARK' }, { id: 12, name: 'ALLEN' },
        { id: 13, name: 'BROWN' }, { id: 14, name: 'HOLMES' }, { id: 15, name: 'FOLGEMAN' },
        { id: 16, name: 'HOLLAND' }, { id: 17, name: 'SANTANA' }, { id: 18, name: 'BRIDGES' },
        { id: 19, name: 'TRYTHALL' }, { id: 20, name: 'GILBERT' }, { id: 21, name: 'FLOWERS' },
        { id: 22, name: 'HYDEN' }, { id: 23, name: 'NORMAN' }, { id: 24, name: 'DAVIS' },
      ],
      simCounter: 0
    },
    {
      id: 2,
      name: 'Supervisors',
      rate: 30,
      overtimeRate: 30,
      color: '#9333ea', // purple
      people: [
        { id: 1, name: 'SPOFFORD' }, { id: 2, name: 'ROSNER' }, { id: 3, name: 'DUNCAN' },
        { id: 4, name: 'SHUMATE' }, { id: 5, name: 'DORE' }, { id: 6, name: 'ZAVALA' },
      ],
      simCounter: 0
    }
  ];

  let newPositionName = '';
  let newStaffNames: Record<number, string> = {};

  // ===== SAVE/LOAD SYSTEM =====
  const STORAGE_KEY = 'juvenile-facility-scheduler-data';
  let lastSaved: Date | null = null;
  let saveStatus = '';
  let fileInput: HTMLInputElement;
  let isInitialLoad = true; // Prevent auto-save during initial component load
  let scheduleUpdateCounter = 0; // Track mutations to trigger auto-save

  function saveToLocalStorage() {
    try {
      const data = {
        staffPositions,
        schedules,
        startDate: startDate.toISOString(),
        editingOpen,
        collapsedSchedules,
        version: '3.0',
        savedAt: new Date().toISOString()
      };
      localStorage.setItem(STORAGE_KEY, JSON.stringify(data));
      lastSaved = new Date();
      saveStatus = 'Auto-saved';
      console.log('✓ Auto-saved to localStorage at', lastSaved.toLocaleTimeString());
      setTimeout(() => saveStatus = '', 2000);
    } catch (e) {
      console.error('✗ Failed to save to localStorage:', e);
      saveStatus = 'Save failed';
    }
  }

  function handleScheduleMutation(event: CustomEvent) {
    if (!isInitialLoad) {
      const updatedSchedule = event.detail;
      
      if (updatedSchedule) {
        // Find and replace the updated schedule in the array
        schedules = schedules.map(s => 
          s.id === updatedSchedule.id ? updatedSchedule : s
        );
      } else {
        // If no schedule passed, just trigger reactivity
        schedules = schedules;
      }
      
      // Increment counter to trigger auto-save
      scheduleUpdateCounter++;
    }
  }

  function loadFromLocalStorage() {
    try {
      const stored = localStorage.getItem(STORAGE_KEY);
      if (stored) {
        const data = JSON.parse(stored);
        staffPositions = data.staffPositions || staffPositions;
        schedules = data.schedules || schedules;
        startDate = data.startDate ? new Date(data.startDate) : startDate;
        editingOpen = data.editingOpen || {};
        collapsedSchedules = data.collapsedSchedules || {};
        lastSaved = data.savedAt ? new Date(data.savedAt) : null;
        saveStatus = 'Loaded from auto-save';
        console.log('✓ Loaded from localStorage. Last saved:', lastSaved?.toLocaleString());
        console.log('  - Schedules loaded:', schedules.length);
        console.log('  - Staff positions loaded:', staffPositions.length);
        setTimeout(() => saveStatus = '', 3000);
        return true;
      }
      console.log('ℹ No saved data found in localStorage');
    } catch (e) {
      console.error('✗ Failed to load from localStorage:', e);
    }
    return false;
  }

  function exportToJSON() {
    const data = {
      staffPositions,
      schedules,
      startDate: startDate.toISOString(),
      version: '3.0',
      exportedAt: new Date().toISOString(),
      metadata: {
        totalPositions: staffPositions.length,
        totalSchedules: schedules.length
      }
    };
    
    console.log('📤 Exporting current state to JSON');
    console.log('  - Schedules being exported:', schedules.length);
    console.log('  - Staff positions being exported:', staffPositions.length);
    console.log('  - Total schedule assignments:', schedules.reduce((sum, s) => {
      return sum + Object.keys(s.positionAssignments || {}).length;
    }, 0));
    
    const blob = new Blob([JSON.stringify(data, null, 2)], { type: 'application/json' });
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url;
    a.download = `schedule-${new Date().toISOString().split('T')[0]}.json`;
    document.body.appendChild(a);
    a.click();
    document.body.removeChild(a);
    URL.revokeObjectURL(url);
    
    saveStatus = 'Exported to JSON';
    console.log('✓ Export complete!');
    setTimeout(() => saveStatus = '', 3000);
  }

  function importFromJSON(event: CustomEvent) {
    // Handle the event - it might be wrapped in a CustomEvent
    const originalEvent = event.detail || event;
    const input = originalEvent.target as HTMLInputElement;
    
    console.log('🔍 Import triggered');
    console.log('  - Event:', event);
    console.log('  - Original event:', originalEvent);
    console.log('  - Input element:', input);
    
    const file = input?.files?.[0];
    if (!file) {
      console.log('⚠️ No file selected');
      return;
    }

    console.log('📁 File selected:', file.name);

    const reader = new FileReader();
    reader.onload = (e) => {
      try {
        const data = JSON.parse(e.target?.result as string);
        
        if (!data.staffPositions || !data.schedules) {
          throw new Error('Invalid file format');
        }

        console.log('📥 Importing JSON file:', file.name);
        console.log('  - Schedules to import:', data.schedules.length);
        console.log('  - Staff positions to import:', data.staffPositions.length);

        staffPositions = data.staffPositions;
        schedules = data.schedules;
        startDate = data.startDate ? new Date(data.startDate) : new Date(2025, 11, 1);
        
        editingOpen = {};
        collapsedSchedules = {};
        
        saveStatus = 'Imported successfully';
        console.log('✓ Import successful! Data loaded and will auto-save');
        setTimeout(() => saveStatus = '', 3000);
        
        // Reactive statement will trigger auto-save
      } catch (error) {
        console.error('✗ Failed to import:', error);
        saveStatus = 'Import failed - invalid file';
        setTimeout(() => saveStatus = '', 3000);
      }
    };
    reader.readAsText(file);
    
    // Clear input to allow re-importing same file
    if (input) input.value = '';
  }

  function triggerImport() {
    fileInput.click();
  }

  function clearAllData() {
    if (confirm('Are you sure you want to clear all data? This cannot be undone.')) {
      localStorage.removeItem(STORAGE_KEY);
      
      staffPositions = [
        {
          id: 1,
          name: 'Floor Staff',
          rate: 25,
          overtimeRate: 37.5,
          color: '#3b82f6',
          people: [
            { id: 1, name: 'MOODY' }, { id: 2, name: 'SHAWELL' }, { id: 3, name: 'MILLER' },
            { id: 4, name: 'NGUYEN' }, { id: 5, name: 'JACKSON' }, { id: 6, name: 'MACRINA' },
            { id: 7, name: 'GARNER' }, { id: 8, name: 'DOUGLAS' }, { id: 9, name: 'MORALES' },
            { id: 10, name: 'ANDERSON' }, { id: 11, name: 'CLARK' }, { id: 12, name: 'ALLEN' },
            { id: 13, name: 'BROWN' }, { id: 14, name: 'HOLMES' }, { id: 15, name: 'FOLGEMAN' },
            { id: 16, name: 'HOLLAND' }, { id: 17, name: 'SANTANA' }, { id: 18, name: 'BRIDGES' },
            { id: 19, name: 'TRYTHALL' }, { id: 20, name: 'GILBERT' }, { id: 21, name: 'FLOWERS' },
            { id: 22, name: 'HYDEN' }, { id: 23, name: 'NORMAN' }, { id: 24, name: 'DAVIS' },
          ],
          simCounter: 0
        },
        {
          id: 2,
          name: 'Supervisors',
          rate: 30,
          overtimeRate: 30,
          color: '#9333ea',
          people: [
            { id: 1, name: 'SPOFFORD' }, { id: 2, name: 'ROSNER' }, { id: 3, name: 'DUNCAN' },
            { id: 4, name: 'SHUMATE' }, { id: 5, name: 'DORE' }, { id: 6, name: 'ZAVALA' },
          ],
          simCounter: 0
        }
      ];
      
      schedules = createDefaultSchedules();
      
      startDate = new Date(2025, 11, 1);
      editingOpen = {};
      collapsedSchedules = {};
      lastSaved = null;
      
      saveStatus = 'All data cleared';
      setTimeout(() => saveStatus = '', 3000);
    }
  }

function createDefaultSchedules() {
  // ===== SCHEDULE 1: 12-Hour 2-2-3 (84 Hours) =====
  const schedule1 = autoAssignForSchedule({
    id: 1,
    name: '12-Hour 2-2-3 Rotation (84 Hours)',
    type: '12-hour',
    rotation: '2-2-3',
    requiredCounts: {
      1: { day12: 10, am: 0, pm: 0 }, // Floor Staff
      2: { day12: 1, am: 0, pm: 0 }   // Supervisors
    },
    positionAssignments: {},
    positionDisplayOrders: {},
    positionSimStaff: {},
    positionSimCounters: {}
  });

  // For the 84-hour schedule:
  // Order staff by team: all A's first, then all B's (per position)
  for (const position of staffPositions) {
    const posId = position.id;
    const posAssignments = schedule1.positionAssignments[posId] || {};

    const meta = position.people.map((p, index) => {
      const a = posAssignments[p.id] || { team: 'A' };
      const team = (a.team || 'A') as 'A' | 'B';
      return { id: p.id, team, originalIndex: index };
    });

    meta.sort((a, b) => {
      if (a.team !== b.team) {
        return a.team === 'A' ? -1 : 1; // A team first, then B
      }
      // keep original order within each team
      return a.originalIndex - b.originalIndex;
    });

    schedule1.positionDisplayOrders[posId] = meta.map((m) => m.id);
  }

  // ===== SCHEDULE 2: 12-Hour 2-2-3 (96 Hours) =====
  const schedule2Base = autoAssignForSchedule({
    id: 2,
    name: '12-Hour 2-2-3 Rotation (96 Hours)',
    type: '12-hour',
    rotation: '2-2-3',
    requiredCounts: {
      1: { day12: 10, am: 0, pm: 0 }, // Floor Staff
      2: { day12: 1, am: 0, pm: 0 }   // Supervisors
    },
    positionAssignments: {},
    positionDisplayOrders: {},
    positionSimStaff: {},
    positionSimCounters: {}
  });

  // --- Build 96-hour patterns ---

  // Normal 2-2-3 pattern (Team A): 7 workdays in 14
  const basePatternA = getRotationPattern('2-2-3'); // [1,1,0,0,1,1,1,0,0,1,1,0,0,0]
  const basePatternB = getInversePattern(basePatternA); // Team B is opposite

  // Indices of OFF days for each team (places we can add the extra shift)
  const offDaysA = basePatternA
    .map((v, idx) => (v === 0 ? idx : -1))
    .filter((i) => i >= 0);
  const offDaysB = basePatternB
    .map((v, idx) => (v === 0 ? idx : -1))
    .filter((i) => i >= 0);

  const teamConfig: Record<'A' | 'B', {
    base: number[];
    offDays: number[];
    nextIdx: number;
  }> = {
    A: { base: basePatternA, offDays: offDaysA, nextIdx: 0 },
    B: { base: basePatternB, offDays: offDaysB, nextIdx: 0 }
  };

  // For each position (Floor Staff, Supervisors, etc.), give EVERY staff:
  //   - the normal 2-2-3 pattern for their team
  //   - plus ONE extra day (from that team's off days), rotating through the off days
  // Then order them: all A's (grouped by extra day), then all B's (grouped by extra day)
  for (const position of staffPositions) {
    const posId = position.id;
    const posAssignments = schedule2Base.positionAssignments[posId] || {};

    // Collect metadata for ordering
    const meta: { id: number; team: 'A' | 'B'; extraDayIndex: number }[] = [];

    for (const person of position.people) {
      const existing = posAssignments[person.id] || {};
      const team = (existing.team || 'A') as 'A' | 'B';
      const cfg = teamConfig[team];
      if (!cfg) continue;

      // Start from the base 2-2-3 pattern for that team
      const pattern = [...cfg.base];

      // Choose which OFF day this specific staff will pick up as their extra shift
      const offIndex = cfg.offDays[cfg.nextIdx % cfg.offDays.length];
      cfg.nextIdx += 1;

      // Flip that day from 0 → 1, so they now have 8 workdays (96 hours)
      pattern[offIndex] = 1;

      posAssignments[person.id] = {
        ...existing,
        manualSchedule: pattern
      };

      meta.push({ id: person.id, team, extraDayIndex: offIndex });
    }

    schedule2Base.positionAssignments[posId] = posAssignments;

    // Build order: A team sorted by extraDayIndex, then B team sorted by extraDayIndex
    const orderA = meta
      .filter((m) => m.team === 'A')
      .sort((a, b) => a.extraDayIndex - b.extraDayIndex)
      .map((m) => m.id);

    const orderB = meta
      .filter((m) => m.team === 'B')
      .sort((a, b) => a.extraDayIndex - b.extraDayIndex)
      .map((m) => m.id);

    schedule2Base.positionDisplayOrders[posId] = [...orderA, ...orderB];
  }

  // ===== SCHEDULE 3: 10-Hour (100 Hours) =====
  const schedule3 = autoAssignForSchedule({
    id: 3,
    name: '10-Hour (100 Hours)',
    type: '10-hour',
    requiredCounts: {
      1: { day12: 0, am: 8, pm: 10 }, // Floor Staff
      2: { day12: 0, am: 1, pm: 1 }   // Supervisors
    },
    positionAssignments: {},
    positionDisplayOrders: {},
    positionSimStaff: {},
    positionSimCounters: {}
  });

  // For the 10-hour schedule:
  // Group by shift (AM first, then PM), then by days-off pattern so people
  // who share the same pattern line up together.
  for (const position of staffPositions) {
    const posId = position.id;
    const posAssignments = schedule3.positionAssignments[posId] || {};

    const meta = position.people.map((p, index) => {
      const a = posAssignments[p.id] || { shift: 'AM', daysOff: [6, 0] };
      const shift = (a.shift || 'AM') as 'AM' | 'PM';
      const patternKey = getDaysOffPattern(a.daysOff || [6, 0]);
      return { id: p.id, shift, patternKey, originalIndex: index };
    });

    meta.sort((a, b) => {
      // AM staff first, then PM
      if (a.shift !== b.shift) {
        return a.shift === 'AM' ? -1 : 1;
      }
      // Within the same shift, group by days-off pattern
      if (a.patternKey < b.patternKey) return -1;
      if (a.patternKey > b.patternKey) return 1;
      // Keep original order within the same pattern
      return a.originalIndex - b.originalIndex;
    });

    schedule3.positionDisplayOrders[posId] = meta.map((m) => m.id);
  }

  return [schedule1, schedule2Base, schedule3];
}




  onMount(() => {
    if (sessionStorage.getItem('scheduler-auth') === 'true') {
      isAuthenticated = true;
    }
    
    if (isAuthenticated) {
      const loaded = loadFromLocalStorage();
      if (!loaded) {
        ensureAllSchedulesAssigned();
      }
    }
    
    // Allow auto-save to trigger after initial load is complete
    // Use setTimeout to ensure this happens after all reactive statements settle
    setTimeout(() => {
      isInitialLoad = false;
    }, 100);
  });

  // ===== ASSIGNMENT HELPERS =====
  const DAYS_OFF_CYCLE = [
    [6,0],
    [1,2],
    [4,5],
  ];

  const DAYS_OFF_PATTERNS = {
    'sun-mon': [6, 0],
    'mon-tue': [0, 1],
    'tue-wed': [1, 2],
    'wed-thu': [2, 3],
    'thu-fri': [3, 4],
    'fri-sat': [4, 5],
    'sat-sun': [5, 6]
  };

  function getDaysOffPattern(daysOff: number[]): string {
    if (!daysOff || daysOff.length === 0) return 'sun-mon';
    const sorted = [...daysOff].sort((a, b) => a - b);
    for (const [key, pattern] of Object.entries(DAYS_OFF_PATTERNS)) {
      if (pattern.length === sorted.length && pattern.every((v, i) => v === sorted[i])) {
        return key;
      }
    }
    return 'sun-mon';
  }

  function buildAssignments12h_A_B(peopleList) {
    const assignments: Record<number, any> = {};
    peopleList.forEach((s, idx) => {
      assignments[s.id] = { team: idx % 2 === 0 ? 'A' : 'B' };
    });
    return assignments;
  }

  function buildAssignments10h_AM_PM(peopleList) {
    const assignments: Record<number, any> = {};
    const half = Math.ceil(peopleList.length / 2);
    peopleList.forEach((s, idx) => {
      const shift = idx < half ? 'AM' : 'PM';
      const cycle = DAYS_OFF_CYCLE[idx % DAYS_OFF_CYCLE.length];
      assignments[s.id] = { shift, daysOff: [...cycle] };
    });
    return assignments;
  }

  function autoAssignForSchedule(schedule) {
  const positionAssignments: Record<number, any> = {};
  const positionDisplayOrders: Record<number, (number | string)[]> = {};
  const positionSimStaff: Record<number, any[]> = {};
  const positionSimCounters: Record<number, number> = {};

  for (const position of staffPositions) {
    // Base assignments (A/B teams for 12-hour, AM/PM+daysOff for 10-hour)
    const baseAssignments =
      schedule.type === '12-hour'
        ? buildAssignments12h_A_B(position.people)
        : buildAssignments10h_AM_PM(position.people);

    const prevAssignments = schedule.positionAssignments?.[position.id] || {};
    const mergedAssignments: Record<number, any> = {};

    for (const person of position.people) {
      const base = baseAssignments[person.id];
      const existing = prevAssignments[person.id];

      // Merge so we KEEP things like manualSchedule, daysOff, etc.
      mergedAssignments[person.id] = existing
        ? { ...base, ...existing }
        : base;
    }

    positionAssignments[position.id] = mergedAssignments;

    // Keep any existing custom order if present, otherwise default to staff order
    positionDisplayOrders[position.id] =
      schedule.positionDisplayOrders?.[position.id] ||
      position.people.map((p) => p.id);

    positionSimStaff[position.id] =
      schedule.positionSimStaff?.[position.id] || [];
    positionSimCounters[position.id] =
      schedule.positionSimCounters?.[position.id] || 0;
  }

  // Initialize required counts if not present
  if (!schedule.requiredCounts) {
    schedule.requiredCounts = {};
    for (const position of staffPositions) {
      schedule.requiredCounts[position.id] = { day12: 0, am: 0, pm: 0 };
    }
  }

  return {
    ...schedule,
    positionAssignments,
    positionDisplayOrders,
    positionSimStaff,
    positionSimCounters
  };
}


  function ensureAllSchedulesAssigned() {
    schedules = schedules.map(s => autoAssignForSchedule(s));
  }

  // ===== SCHEDULES =====
  let schedules = createDefaultSchedules();

  let newScheduleName = '';

  let editingOpen: Record<number, boolean> = {};
  function toggleEditing(id: number) {
    editingOpen = { ...editingOpen, [id]: !editingOpen[id] };
  }

  let collapsedSchedules: Record<number, boolean> = {};
  function toggleCollapse(id: number) {
    collapsedSchedules = { ...collapsedSchedules, [id]: !collapsedSchedules[id] };
  }

  // ===== AUTO-SAVE SYSTEM =====
  // This reactive statement watches ALL the variables we want to auto-save
  // It will trigger whenever any of these variables change
  // BUT it won't trigger during initial component load (to prevent overwriting imports)
  $: if (typeof window !== 'undefined' && isAuthenticated && !isInitialLoad) {
    // Access the variables to create dependencies
    schedules;
    staffPositions;
    startDate;
    editingOpen;
    collapsedSchedules;
    scheduleUpdateCounter; // This will trigger on manual mutations
    
    saveToLocalStorage();
  }

// ===== ROTATION HELPERS =====
function getRotationPattern(rotation: string) {
  switch (rotation) {
    case '2-2-3':
      return [1,1,0,0,1,1,1,0,0,1,1,0,0,0];
    default:
      return Array(14).fill(0);
  }
}

// 👇 change THIS:
function getInversePattern(pattern: number[]): number[] {
  return pattern.map((d) => (d ? 0 : 1));
}


  // ===== CRUD (STAFF POSITIONS) =====
  function addPosition() {
    const name = newPositionName.trim();
    if (!name) return;
    const newId = Math.max(0, ...staffPositions.map(p => p.id)) + 1;
    const newPosition = {
      id: newId,
      name,
      rate: 25,
      overtimeRate: 37.5,
      color: `#${Math.floor(Math.random()*16777215).toString(16)}`,
      people: [],
      simCounter: 0
    };
    staffPositions = [...staffPositions, newPosition];
    newPositionName = '';
    
    // Add to all schedules
    schedules = schedules.map(s => ({
      ...s,
      requiredCounts: { ...s.requiredCounts, [newId]: { day12: 0, am: 0, pm: 0 } },
      positionAssignments: { ...s.positionAssignments, [newId]: {} },
      positionDisplayOrders: { ...s.positionDisplayOrders, [newId]: [] },
      positionSimStaff: { ...s.positionSimStaff, [newId]: [] },
      positionSimCounters: { ...s.positionSimCounters, [newId]: 0 }
    }));
  }

  function deletePosition(id: number) {
    if (staffPositions.length <= 1) {
      alert('Cannot delete the last position type.');
      return;
    }
    staffPositions = staffPositions.filter(p => p.id !== id);
    
    // Remove from all schedules
    schedules = schedules.map(s => {
      const { [id]: _, ...restCounts } = s.requiredCounts;
      const { [id]: __, ...restAssignments } = s.positionAssignments;
      const { [id]: ___, ...restOrders } = s.positionDisplayOrders;
      const { [id]: ____, ...restSim} = s.positionSimStaff;
      const { [id]: _____, ...restCounters } = s.positionSimCounters;
      return {
        ...s,
        requiredCounts: restCounts,
        positionAssignments: restAssignments,
        positionDisplayOrders: restOrders,
        positionSimStaff: restSim,
        positionSimCounters: restCounters
      };
    });
  }

  function updatePosition(id: number, field: string, value: any) {
    staffPositions = staffPositions.map(p => 
      p.id === id ? { ...p, [field]: value } : p
    );
  }

  // ===== CRUD (STAFF IN POSITIONS) =====
  function addStaffToPosition(positionId: number) {
    const name = newStaffNames[positionId]?.trim();
    if (!name) return;
    
    staffPositions = staffPositions.map(pos => {
      if (pos.id !== positionId) return pos;
      const newId = Math.max(0, ...pos.people.map(p => p.id)) + 1;
      return {
        ...pos,
        people: [...pos.people, { id: newId, name }]
      };
    });
    
    newStaffNames[positionId] = '';
    ensureAllSchedulesAssigned();
  }

  function deleteStaffFromPosition(positionId: number, staffId: number) {
    staffPositions = staffPositions.map(pos => {
      if (pos.id !== positionId) return pos;
      return {
        ...pos,
        people: pos.people.filter(p => p.id !== staffId)
      };
    });
    
    // Remove from schedules
    schedules = schedules.map(s => {
      const assignments = { ...s.positionAssignments[positionId] };
      delete assignments[staffId];
      const orders = (s.positionDisplayOrders[positionId] || []).filter(id => id !== staffId);
      return {
        ...s,
        positionAssignments: { ...s.positionAssignments, [positionId]: assignments },
        positionDisplayOrders: { ...s.positionDisplayOrders, [positionId]: orders }
      };
    });
  }

  function updateStaffInPosition(positionId: number, staffId: number, name: string) {
    staffPositions = staffPositions.map(pos => {
      if (pos.id !== positionId) return pos;
      return {
        ...pos,
        people: pos.people.map(p => p.id === staffId ? { ...p, name } : p)
      };
    });
  }

// ... (continuing from previous)

  // ===== CRUD (SCHEDULES) =====
  function addSchedule() {
    const name = (newScheduleName.trim() || '12-Hour 2-2-3');
    const newId = Math.max(0, ...schedules.map(s => s.id)) + 1;
    
    const requiredCounts = {};
    for (const position of staffPositions) {
      requiredCounts[position.id] = { day12: 0, am: 0, pm: 0 };
    }
    
    let newSchedule = {
      id: newId,
      name,
      type: '12-hour',
      rotation: '2-2-3',
      requiredCounts,
      positionAssignments: {},
      positionDisplayOrders: {},
      positionSimStaff: {},
      positionSimCounters: {}
    };
    newSchedule = autoAssignForSchedule(newSchedule);
    schedules = [...schedules, newSchedule];
    newScheduleName = '';
  }

  function deleteSchedule(id: number) {
    schedules = schedules.filter(s => s.id !== id);
    const { [id]: _, ...rest } = editingOpen;
    editingOpen = rest;
    const { [id]: __, ...rest2 } = collapsedSchedules;
    collapsedSchedules = rest2;
  }

  function updateSchedule(id: number, field: string, value: any) {
    schedules = schedules.map(s => {
      if (s.id !== id) return s;
      let updated = { ...s, [field]: value };

      if (field === 'type') {
        if (value === '12-hour' && !updated.rotation) updated.rotation = '2-2-3';
        updated = autoAssignForSchedule(updated);
      }

      if (field === 'rotation' && updated.type === '12-hour') {
        // Reassign all positions
        for (const position of staffPositions) {
          const allHaveTeam = position.people.every(p => updated.positionAssignments[position.id]?.[p.id]?.team);
          if (!allHaveTeam) {
            updated.positionAssignments[position.id] = buildAssignments12h_A_B(position.people);
          }
        }
      }

      return updated;
    });
  }

  function updateRequiredCount(scheduleId: number, positionId: number, field: string, value: number) {
    schedules = schedules.map(s => {
      if (s.id !== scheduleId) return s;
      return {
        ...s,
        requiredCounts: {
          ...s.requiredCounts,
          [positionId]: {
            ...s.requiredCounts[positionId],
            [field]: value
          }
        }
      };
    });
  }

  function updateAssignment(scheduleId: number, positionId: number, staffId: number, field: string, value: any) {
    schedules = schedules.map(sch => {
      if (sch.id !== scheduleId) return sch;
      const positionAssignments = { ...sch.positionAssignments };
      const assignments = { ...positionAssignments[positionId] };
      const prev = assignments[staffId] || (sch.type === '12-hour' ? { team: 'A' } : { shift: 'AM', daysOff: [6,0] });
      
      // Create new assignment with updated field
      const newAssignment = { ...prev, [field]: value };
      
      // If changing team or shift, clear any manual schedule so they follow the new pattern
      // Also clear individual staff overrides to use the new team/shift defaults
      if (field === 'team' || field === 'shift') {
        delete newAssignment.manualSchedule;
        
        // Clear staff schedule overrides for this staff member
        if (sch.staffScheduleOverrides?.[positionId]?.[staffId]) {
          delete sch.staffScheduleOverrides[positionId][staffId];
          // Clean up empty objects
          if (Object.keys(sch.staffScheduleOverrides[positionId]).length === 0) {
            delete sch.staffScheduleOverrides[positionId];
          }
          if (sch.staffScheduleOverrides && Object.keys(sch.staffScheduleOverrides).length === 0) {
            delete sch.staffScheduleOverrides;
          }
        }
      }
      
      assignments[staffId] = newAssignment;
      positionAssignments[positionId] = assignments;
      return { ...sch, positionAssignments };
    });
  }

  function updateDaysOffPattern(scheduleId: number, positionId: number, staffId: number, pattern: string) {
    const daysOff = DAYS_OFF_PATTERNS[pattern] || [6, 0];
    schedules = schedules.map(sch => {
      if (sch.id !== scheduleId) return sch;
      const positionAssignments = { ...sch.positionAssignments };
      const assignments = { ...positionAssignments[positionId] };
      const prev = assignments[staffId] || { shift: 'AM', daysOff: [6, 0] };
      
      // Create new assignment with updated days off
      const newAssignment = { ...prev, daysOff };
      
      // Clear manual schedule so they follow the new days off pattern
      delete newAssignment.manualSchedule;
      
      assignments[staffId] = newAssignment;
      positionAssignments[positionId] = assignments;
      return { ...sch, positionAssignments };
    });
  }

  // ===== SIMULATED STAFF =====
  function addSimStaff(scheduleId: number, positionId: number, kind: 'A'|'B'|'AM'|'PM') {
    schedules = schedules.map(s => {
      if (s.id !== scheduleId) return s;
      
      const currentCounter = s.positionSimCounters[positionId] || 0;
      const simCounter = currentCounter + 1;
      const position = staffPositions.find(p => p.id === positionId);
      
      const base = {
        id: `sim-${positionId}-${simCounter}`,
        name: `[SIM ${position?.name}] ${kind} #${simCounter}`
      };
      
      const sim = s.type === '12-hour'
        ? { ...base, team: kind }
        : { ...base, shift: kind, daysOff: [6,0] };
      
      const simStaff = [...(s.positionSimStaff[positionId] || []), sim];
      const displayOrder = [...(s.positionDisplayOrders[positionId] || []), sim.id];
      
      return {
        ...s,
        positionSimCounters: { ...s.positionSimCounters, [positionId]: simCounter },
        positionSimStaff: { ...s.positionSimStaff, [positionId]: simStaff },
        positionDisplayOrders: { ...s.positionDisplayOrders, [positionId]: displayOrder }
      };
    });
  }

  function deleteSimStaff(scheduleId: number, positionId: number, simId: string) {
    schedules = schedules.map(s => {
      if (s.id !== scheduleId) return s;
      const simStaff = (s.positionSimStaff[positionId] || []).filter(sim => sim.id !== simId);
      const displayOrder = (s.positionDisplayOrders[positionId] || []).filter(id => id !== simId);
      return {
        ...s,
        positionSimStaff: { ...s.positionSimStaff, [positionId]: simStaff },
        positionDisplayOrders: { ...s.positionDisplayOrders, [positionId]: displayOrder }
      };
    });
  }

  function updateSimStaff(scheduleId: number, positionId: number, simId: string, field: string, value: any) {
    schedules = schedules.map(s => {
      if (s.id !== scheduleId) return s;
      
      // If changing team or shift, clear staff schedule overrides for this simulated staff
      if (field === 'team' || field === 'shift') {
        if (s.staffScheduleOverrides?.[positionId]?.[simId]) {
          const updatedOverrides = { ...s.staffScheduleOverrides };
          if (updatedOverrides[positionId]) {
            const posOverrides = { ...updatedOverrides[positionId] };
            delete posOverrides[simId];
            
            if (Object.keys(posOverrides).length === 0) {
              delete updatedOverrides[positionId];
            } else {
              updatedOverrides[positionId] = posOverrides;
            }
            
            if (Object.keys(updatedOverrides).length === 0) {
              s = { ...s };
              delete s.staffScheduleOverrides;
            } else {
              s = { ...s, staffScheduleOverrides: updatedOverrides };
            }
          }
        }
      }
      
      const simStaff = (s.positionSimStaff[positionId] || []).map(sim => {
        if (sim.id !== simId) return sim;
        
        // Create updated sim
        const updated = { ...sim, [field]: value };
        
        // If changing team or shift, clear manual schedule
        if (field === 'team' || field === 'shift') {
          delete updated.manualSchedule;
        }
        
        return updated;
      });
      return {
        ...s,
        positionSimStaff: { ...s.positionSimStaff, [positionId]: simStaff }
      };
    });
  }

  function updateSimDaysOffPattern(scheduleId: number, positionId: number, simId: string, pattern: string) {
    const daysOff = DAYS_OFF_PATTERNS[pattern] || [6, 0];
    schedules = schedules.map(s => {
      if (s.id !== scheduleId) return s;
      const simStaff = (s.positionSimStaff[positionId] || []).map(sim => {
        if (sim.id !== simId) return sim;
        
        // Create updated sim with new days off
        const updated = { ...sim, daysOff };
        
        // Clear manual schedule so they follow the new pattern
        delete updated.manualSchedule;
        
        return updated;
      });
      return {
        ...s,
        positionSimStaff: { ...s.positionSimStaff, [positionId]: simStaff }
      };
    });
  }

  function toggleScheduleDay(scheduleId: number, positionId: number, staffId: number, dayIndex: number) {
    schedules = schedules.map(sch => {
      if (sch.id !== scheduleId) return sch;

      const positionAssignments = { ...sch.positionAssignments };
      const newAssignments: Record<string, any> = { ...positionAssignments[positionId] };
      
      for (const key of Object.keys(newAssignments)) {
        const a = newAssignments[key];
        newAssignments[key] = { ...a };
        if (a.daysOff) newAssignments[key].daysOff = [...a.daysOff];
        if (a.manualSchedule) newAssignments[key].manualSchedule = [...a.manualSchedule];
      }

      if (!newAssignments[staffId]) {
        newAssignments[staffId] = sch.type === '12-hour'
          ? { team: 'A' }
          : { shift: 'AM', daysOff: [6,0] };
      }

      if (!newAssignments[staffId].manualSchedule) {
        const current = generateSchedule(sch, positionId, staffId);
        newAssignments[staffId].manualSchedule = [...current];
      }
      const cur = newAssignments[staffId].manualSchedule[dayIndex];
      newAssignments[staffId].manualSchedule[dayIndex] = cur === 1 ? 0 : 1;

      positionAssignments[positionId] = newAssignments;
      return { ...sch, positionAssignments };
    });
  }

  function toggleScheduleDayForSim(scheduleId: number, positionId: number, simId: string, dayIndex: number) {
    schedules = schedules.map(s => {
      if (s.id !== scheduleId) return s;
      const simStaff = (s.positionSimStaff[positionId] || []).map(sim => {
        if (sim.id !== simId) return sim;
        const manual = sim.manualSchedule
          ? [...sim.manualSchedule]
          : generateScheduleForAssignment(s, sim);
        manual[dayIndex] = manual[dayIndex] === 1 ? 0 : 1;
        return { ...sim, manualSchedule: manual };
      });
      return {
        ...s,
        positionSimStaff: { ...s.positionSimStaff, [positionId]: simStaff }
      };
    });
  }

  function resetManualEdits(scheduleId: number) {
    schedules = schedules.map(sch => {
      if (sch.id !== scheduleId) return sch;
      
      const positionAssignments = {};
      for (const posId of Object.keys(sch.positionAssignments)) {
        const assignments: Record<string, any> = {};
        for (const key of Object.keys(sch.positionAssignments[posId])) {
          const a = { ...sch.positionAssignments[posId][key] };
          delete a.manualSchedule;
          assignments[key] = a;
        }
        positionAssignments[posId] = assignments;
      }
      
      const positionSimStaff = {};
      for (const posId of Object.keys(sch.positionSimStaff)) {
        const simStaff = (sch.positionSimStaff[posId] || []).map(sim => {
          const copy = { ...sim };
          delete copy.manualSchedule;
          return copy;
        });
        positionSimStaff[posId] = simStaff;
      }
      
      return { ...sch, positionAssignments, positionSimStaff };
    });
  }

  function reorderList(scheduleId: number, positionId: number, fromId: string | number, toId: string | number) {
    schedules = schedules.map(s => {
      if (s.id !== scheduleId) return s;
      const order = [...(s.positionDisplayOrders[positionId] || [])];
      const fromIdx = order.indexOf(fromId);
      const toIdx = order.indexOf(toId);
      if (fromIdx === -1 || toIdx === -1 || fromIdx === toIdx) return s;
      const [moved] = order.splice(fromIdx, 1);
      order.splice(toIdx, 0, moved);
      return {
        ...s,
        positionDisplayOrders: { ...s.positionDisplayOrders, [positionId]: order }
      };
    });
  }

  // ===== GENERATION LOGIC =====
  function generateScheduleForAssignment(schedule, assignment) {
    if (!assignment) return Array(14).fill(0);

    if (schedule.type === '12-hour') {
      if (schedule.rotation === '4-3') {
        return dates.map((item, i) => {
          const d = item.fullDate;
          const jsDay = d.getDay();
          const weekIndex = Math.floor(i / 7);
          const saturdayGoesToA = (weekIndex % 2 === 0);
          if (assignment.team === 'A') {
            return (jsDay === 3 || jsDay === 4 || jsDay === 5 || (jsDay === 6 && saturdayGoesToA)) ? 1 : 0;
          } else if (assignment.team === 'B') {
            return (jsDay === 0 || jsDay === 1 || jsDay === 2 || (jsDay === 6 && !saturdayGoesToA)) ? 1 : 0;
          }
          return 0;
        });
      }

      if (schedule.rotation === '2-2-3') {
        let pattern = getRotationPattern('2-2-3');
        if (assignment.team === 'B') pattern = getInversePattern(pattern);
        return pattern.slice(0, 14);
      }
      return Array(14).fill(0);
    }

    return dates.map((item) => {
      const d = item.fullDate;
      const jsDay = d.getDay();
      const dayOfWeek = jsDay === 0 ? 6 : jsDay - 1;
      return (assignment.daysOff || []).includes(dayOfWeek) ? 0 : 1;
    });
  }

  function generateSchedule(schedule, positionId: number, staffId: number) {
    const assignment = schedule.positionAssignments[positionId]?.[staffId];
    if (!assignment) return Array(14).fill(0);
    
    // Get base schedule (either manual or generated)
    let baseSchedule: number[];
    if (assignment.manualSchedule) {
      baseSchedule = assignment.manualSchedule;
    } else {
      baseSchedule = generateScheduleForAssignment(schedule, assignment);
    }
    
    // Apply staffScheduleOverrides if they exist
    const overrides = schedule.staffScheduleOverrides?.[positionId]?.[staffId];
    if (overrides) {
      // Create a copy and apply overrides
      baseSchedule = [...baseSchedule];
      for (let dayIndex = 0; dayIndex < 14; dayIndex++) {
        if (overrides[dayIndex] !== undefined) {
          const override = overrides[dayIndex];
          // If override is OFF or PTO, set to 0 (not working)
          // Otherwise, set to 1 (working)
          if (override === 'OFF' || override === 'PTO') {
            baseSchedule[dayIndex] = 0;
          } else {
            baseSchedule[dayIndex] = 1;
          }
        }
      }
    }
    
    return baseSchedule;
  }

  // Helper function to determine which shift a time label represents
  function getShiftFromTimeLabel(label: string): 'AM' | 'PM' | 'OFF' | null {
    if (!label || label === 'OFF' || label === 'PTO') return 'OFF';
    
    // AM shifts typically start in the morning (7a, 8a, etc.)
    if (label.includes('7a') || label.includes('8a') || label.includes('8:30a')) return 'AM';
    
    // PM shifts typically start in the afternoon (1p, 2p, etc.)
    if (label.includes('1p') || label.includes('2p')) return 'PM';
    
    // Default to null if we can't determine
    return null;
  }

  function calculateScheduleCosts(schedule) {
    let totalCost = 0;
    let totalHours = 0;

    const positionDetails = {};
    const positionCoverage = {};

    const hoursPerShift = schedule.type === '12-hour' ? 12 : 10;

    // Process each position
    for (const position of staffPositions) {
      const posId = position.id;
      const rowsReal: any[] = [];
      const rowsSim: any[] = [];

      const dailyCoverage = Array(14).fill(0);
      const amCoverage = Array(14).fill(0);
      const pmCoverage = Array(14).fill(0);

      // Process real staff in this position
      for (const person of position.people) {
        const scheduleData = generateSchedule(schedule, posId, person.id);
        const a = schedule.positionAssignments[posId]?.[person.id] || {};
        const personTotalHours = scheduleData.reduce((sum, d) => sum + d * hoursPerShift, 0);

        let regularHours, overtimeHours, personCost;
        if (personTotalHours > 80) {
          regularHours = 80;
          overtimeHours = personTotalHours - 80;
          personCost = (regularHours * position.rate) + (overtimeHours * position.overtimeRate);
        } else {
          regularHours = personTotalHours;
          overtimeHours = 0;
          personCost = personTotalHours * position.rate;
        }

        totalCost += personCost;
        totalHours += personTotalHours;

        rowsReal.push({
          id: person.id,
          name: person.name,
          hours: personTotalHours,
          regularHours,
          overtimeHours,
          cost: personCost,
          schedule: scheduleData,
          team: a.team,
          shift: a.shift,
          simulated: false,
          isEdited: !!a.manualSchedule
        });

        scheduleData.forEach((on, idx) => {
          if (on === 1) { // Only count if they're working this day
            if (schedule.type === '12-hour') {
              dailyCoverage[idx] += 1;
            } else {
              // Check if there's an override for this specific day
              const override = schedule.staffScheduleOverrides?.[posId]?.[person.id]?.[idx];
              
              if (override) {
                // Use the override to determine which shift
                const shiftFromOverride = getShiftFromTimeLabel(override);
                if (shiftFromOverride === 'AM') {
                  amCoverage[idx] += 1;
                } else if (shiftFromOverride === 'PM') {
                  pmCoverage[idx] += 1;
                }
                // If 'OFF' or null, don't count in coverage
              } else {
                // No override, use assigned shift
                if (a.shift === 'AM') amCoverage[idx] += 1;
                if (a.shift === 'PM') pmCoverage[idx] += 1;
              }
            }
          }
        });
      }

      // Process simulated staff in this position
      for (const sim of (schedule.positionSimStaff[posId] || [])) {
        let sched = sim.manualSchedule ? sim.manualSchedule : generateScheduleForAssignment(schedule, sim);
        
        // Apply staffScheduleOverrides if they exist for this simulated staff
        const overrides = schedule.staffScheduleOverrides?.[posId]?.[sim.id];
        if (overrides) {
          sched = [...sched];
          for (let dayIndex = 0; dayIndex < 14; dayIndex++) {
            if (overrides[dayIndex] !== undefined) {
              const override = overrides[dayIndex];
              // If override is OFF or PTO, set to 0 (not working)
              // Otherwise, set to 1 (working)
              if (override === 'OFF' || override === 'PTO') {
                sched[dayIndex] = 0;
              } else {
                sched[dayIndex] = 1;
              }
            }
          }
        }
        
        const simHours = sched.reduce((sum, d) => sum + d * hoursPerShift, 0);

        let regularHours, overtimeHours, simCost;
        if (simHours > 80) {
          regularHours = 80;
          overtimeHours = simHours - 80;
          simCost = (regularHours * position.rate) + (overtimeHours * position.overtimeRate);
        } else {
          regularHours = simHours;
          overtimeHours = 0;
          simCost = simHours * position.rate;
        }

        totalCost += simCost;
        totalHours += simHours;

        rowsSim.push({
          id: sim.id,
          name: sim.name || `[SIM]`,
          hours: simHours,
          regularHours,
          overtimeHours,
          cost: simCost,
          schedule: sched,
          team: sim.team,
          shift: sim.shift,
          simulated: true,
          isEdited: !!sim.manualSchedule
        });

        sched.forEach((on, idx) => {
          if (on === 1) { // Only count if they're working this day
            if (schedule.type === '12-hour') {
              dailyCoverage[idx] += 1;
            } else {
              // Check if there's an override for this specific day
              const override = schedule.staffScheduleOverrides?.[posId]?.[sim.id]?.[idx];
              
              if (override) {
                // Use the override to determine which shift
                const shiftFromOverride = getShiftFromTimeLabel(override);
                if (shiftFromOverride === 'AM') {
                  amCoverage[idx] += 1;
                } else if (shiftFromOverride === 'PM') {
                  pmCoverage[idx] += 1;
                }
                // If 'OFF' or null, don't count in coverage
              } else {
                // No override, use assigned shift
                if (sim.shift === 'AM') amCoverage[idx] += 1;
                if (sim.shift === 'PM') pmCoverage[idx] += 1;
              }
            }
          }
        });
      }

      // Order rows
      const mapById = new Map<string | number, any>();
      [...rowsReal, ...rowsSim].forEach(row => mapById.set(row.id, row));
      const ordered: any[] = [];
      const order = schedule.positionDisplayOrders[posId] || [];
      for (const id of order) {
        if (mapById.has(id)) ordered.push(mapById.get(id));
        mapById.delete(id);
      }
      for (const leftover of mapById.values()) ordered.push(leftover);

      positionDetails[posId] = ordered;
      positionCoverage[posId] = {
        dailyCoverage,
        amCoverage,
        pmCoverage
      };
    }

    // Calculate gap OT
    let gapOTHours = 0;
    for (const position of staffPositions) {
      const posId = position.id;
      const required = schedule.requiredCounts[posId] || { day12: 0, am: 0, pm: 0 };
      const coverage = positionCoverage[posId];
      
      if (schedule.type === '12-hour') {
        coverage.dailyCoverage.forEach(c => {
          if (c < required.day12) gapOTHours += (required.day12 - c) * 12;
        });
      } else {
        coverage.amCoverage.forEach(c => {
          if (c < required.am) gapOTHours += (required.am - c) * 10;
        });
        coverage.pmCoverage.forEach(c => {
          if (c < required.pm) gapOTHours += (required.pm - c) * 10;
        });
      }
    }
    
    // Use the highest overtime rate for gap coverage
    const maxOvertimeRate = Math.max(...staffPositions.map(p => p.overtimeRate));
    const gapOTCost = gapOTHours * maxOvertimeRate;
    totalCost += gapOTCost;

    return {
      totalCost,
      totalHours,
      positionDetails,
      positionCoverage,
      gapOTHours,
      gapOTCost
    };
  }

  $: allScheduleCosts = schedules.map(s => calculateScheduleCosts(s));
  
  $: costAnalysis = {
    twoWeekCosts: allScheduleCosts.map(c => c.totalCost),
    yearlyCosts: allScheduleCosts.map(c => c.totalCost * 26),
    avgHoursPerStaff: allScheduleCosts.map(c => {
      const totalPeople = staffPositions.reduce((sum, pos) => sum + pos.people.length, 0);
      return totalPeople > 0 ? (c.totalHours / totalPeople).toFixed(1) : 0;
    })
  };

  function shiftPeriod(days: number) {
    startDate = addDays(startDate, days);
  }
  
  function setStartFromInput(e: Event) {
    const val = (e.currentTarget as HTMLInputElement).value;
    if (!val) return;
    const parts = val.split('-').map(Number);
    if (parts.length === 3) {
      startDate = new Date(parts[0], parts[1] - 1, parts[2]);
    }
  }
</script>

<style>
  :global(:root) {
    --sticky-top: 0px;
  }
</style>

{#if !isAuthenticated}
  <AuthGate
    bind:passwordInput
    {passwordError}
    on:submit={checkPassword}
  />
{:else}
  <SchedulerApp
    {startDate}
    {saveStatus}
    {lastSaved}
    {staffPositions}
    {schedules}
    {dates}
    {days}
    {costAnalysis}
    {editingOpen}
    {collapsedSchedules}
    bind:newPositionName
    bind:newStaffNames
    bind:newScheduleName
    bind:fileInput
    on:logout={logout}
    on:export={exportToJSON}
    on:import={triggerImport}
    on:importFile={importFromJSON}
    on:clearAll={clearAllData}
    on:scheduleUpdated={handleScheduleMutation}
    on:shiftPeriod={(e) => shiftPeriod(e.detail)}
    on:setStartDate={setStartFromInput}
    on:addPosition={addPosition}
    on:deletePosition={(e) => deletePosition(e.detail)}
    on:updatePosition={(e) => updatePosition(e.detail.id, e.detail.field, e.detail.value)}
    on:addStaffToPosition={(e) => addStaffToPosition(e.detail)}
    on:deleteStaffFromPosition={(e) => deleteStaffFromPosition(e.detail.positionId, e.detail.staffId)}
    on:updateStaffInPosition={(e) => updateStaffInPosition(e.detail.positionId, e.detail.staffId, e.detail.name)}
    on:addSchedule={addSchedule}
    on:deleteSchedule={(e) => deleteSchedule(e.detail)}
    on:updateSchedule={(e) => updateSchedule(e.detail.id, e.detail.field, e.detail.value)}
    on:updateRequiredCount={(e) => updateRequiredCount(e.detail.scheduleId, e.detail.positionId, e.detail.field, e.detail.value)}
    on:toggleEditing={(e) => toggleEditing(e.detail)}
    on:toggleCollapse={(e) => toggleCollapse(e.detail)}
    on:addSimStaff={(e) => addSimStaff(e.detail.scheduleId, e.detail.positionId, e.detail.kind)}
    on:deleteSimStaff={(e) => deleteSimStaff(e.detail.scheduleId, e.detail.positionId, e.detail.simId)}
    on:updateSimStaff={(e) => updateSimStaff(e.detail.scheduleId, e.detail.positionId, e.detail.simId, e.detail.field, e.detail.value)}
    on:updateAssignment={(e) => updateAssignment(e.detail.scheduleId, e.detail.positionId, e.detail.staffId, e.detail.field, e.detail.value)}
    on:toggleScheduleDay={(e) => toggleScheduleDay(e.detail.scheduleId, e.detail.positionId, e.detail.staffId, e.detail.dayIndex)}
    on:toggleScheduleDayForSim={(e) => toggleScheduleDayForSim(e.detail.scheduleId, e.detail.positionId, e.detail.simId, e.detail.dayIndex)}
    on:resetManualEdits={(e) => resetManualEdits(e.detail)}
    on:reorderList={(e) => reorderList(e.detail.scheduleId, e.detail.positionId, e.detail.fromId, e.detail.toId)}
    on:updateDaysOffPattern={(e) => updateDaysOffPattern(e.detail.scheduleId, e.detail.positionId, e.detail.staffId, e.detail.pattern)}
    on:updateSimDaysOffPattern={(e) => updateSimDaysOffPattern(e.detail.scheduleId, e.detail.positionId, e.detail.simId, e.detail.pattern)}
    {generateSchedule}
    {generateScheduleForAssignment}
    {calculateScheduleCosts}
    {getDaysOffPattern}
  />
{/if}