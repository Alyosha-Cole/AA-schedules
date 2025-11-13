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
  const regularRate = 25;
  const overtimeRate = 37.5;
  const requiredStaff = 10;
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

  // ===== INITIAL STATE =====
  let staff = [
    { id: 1, name: 'MOODY' }, { id: 2, name: 'SHAWELL' }, { id: 3, name: 'MILLER' },
    { id: 4, name: 'NGUYEN' }, { id: 5, name: 'JACKSON' }, { id: 6, name: 'MACRINA' },
    { id: 7, name: 'GARNER' }, { id: 8, name: 'DOUGLAS' }, { id: 9, name: 'MORALES' },
    { id: 10, name: 'ANDERSON' }, { id: 11, name: 'CLARK' }, { id: 12, name: 'ALLEN' },
    { id: 13, name: 'BROWN' }, { id: 14, name: 'HOLMES' }, { id: 15, name: 'FOLGEMAN' },
    { id: 16, name: 'HOLLAND' }, { id: 17, name: 'SANTANA' }, { id: 18, name: 'BRIDGES' },
    { id: 19, name: 'TRYTHALL' }, { id: 20, name: 'GILBERT' }, { id: 21, name: 'FLOWERS' },
    { id: 22, name: 'HYDEN' }, { id: 23, name: 'NORMAN' }, { id: 24, name: 'DAVIS' },
  ];

  // ===== SAVE/LOAD SYSTEM =====
  const STORAGE_KEY = 'juvenile-facility-scheduler-data';
  let lastSaved: Date | null = null;
  let saveStatus = '';
  let fileInput: HTMLInputElement;

  $: if (typeof window !== 'undefined' && isAuthenticated) {
    saveToLocalStorage();
  }

  function saveToLocalStorage() {
    try {
      const data = {
        staff,
        schedules,
        startDate: startDate.toISOString(),
        editingOpen,
        collapsedSchedules,
        version: '1.0',
        savedAt: new Date().toISOString()
      };
      localStorage.setItem(STORAGE_KEY, JSON.stringify(data));
      lastSaved = new Date();
      saveStatus = 'Auto-saved';
      setTimeout(() => saveStatus = '', 2000);
    } catch (e) {
      console.error('Failed to save to localStorage:', e);
      saveStatus = 'Save failed';
    }
  }

  function loadFromLocalStorage() {
    try {
      const stored = localStorage.getItem(STORAGE_KEY);
      if (stored) {
        const data = JSON.parse(stored);
        staff = data.staff || staff;
        schedules = data.schedules || schedules;
        startDate = data.startDate ? new Date(data.startDate) : startDate;
        editingOpen = data.editingOpen || {};
        collapsedSchedules = data.collapsedSchedules || {};
        lastSaved = data.savedAt ? new Date(data.savedAt) : null;
        saveStatus = 'Loaded from auto-save';
        setTimeout(() => saveStatus = '', 3000);
        return true;
      }
    } catch (e) {
      console.error('Failed to load from localStorage:', e);
    }
    return false;
  }

  function exportToJSON() {
    const data = {
      staff,
      schedules,
      startDate: startDate.toISOString(),
      version: '1.0',
      exportedAt: new Date().toISOString(),
      metadata: {
        totalStaff: staff.length,
        totalSchedules: schedules.length
      }
    };
    
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
    setTimeout(() => saveStatus = '', 3000);
  }

  function importFromJSON(event: Event) {
    const input = event.target as HTMLInputElement;
    const file = input.files?.[0];
    if (!file) return;

    const reader = new FileReader();
    reader.onload = (e) => {
      try {
        const data = JSON.parse(e.target?.result as string);
        
        if (!data.staff || !data.schedules) {
          throw new Error('Invalid file format');
        }

        staff = data.staff;
        schedules = data.schedules;
        startDate = data.startDate ? new Date(data.startDate) : new Date(2025, 11, 1);
        
        editingOpen = {};
        collapsedSchedules = {};
        
        saveStatus = 'Imported successfully';
        setTimeout(() => saveStatus = '', 3000);
        
        saveToLocalStorage();
      } catch (error) {
        console.error('Failed to import:', error);
        saveStatus = 'Import failed - invalid file';
        setTimeout(() => saveStatus = '', 3000);
      }
    };
    reader.readAsText(file);
    
    input.value = '';
  }

  function triggerImport() {
    fileInput.click();
  }

  function clearAllData() {
    if (confirm('Are you sure you want to clear all data? This cannot be undone.')) {
      localStorage.removeItem(STORAGE_KEY);
      
      staff = [
        { id: 1, name: 'MOODY' }, { id: 2, name: 'SHAWELL' }, { id: 3, name: 'MILLER' },
        { id: 4, name: 'NGUYEN' }, { id: 5, name: 'JACKSON' }, { id: 6, name: 'MACRINA' },
        { id: 7, name: 'GARNER' }, { id: 8, name: 'DOUGLAS' }, { id: 9, name: 'MORALES' },
        { id: 10, name: 'ANDERSON' }, { id: 11, name: 'CLARK' }, { id: 12, name: 'ALLEN' },
        { id: 13, name: 'BROWN' }, { id: 14, name: 'HOLMES' }, { id: 15, name: 'FOLGEMAN' },
        { id: 16, name: 'HOLLAND' }, { id: 17, name: 'SANTANA' }, { id: 18, name: 'BRIDGES' },
        { id: 19, name: 'TRYTHALL' }, { id: 20, name: 'GILBERT' }, { id: 21, name: 'FLOWERS' },
        { id: 22, name: 'HYDEN' }, { id: 23, name: 'NORMAN' }, { id: 24, name: 'DAVIS' },
      ];
      
      schedules = [
        autoAssignForSchedule({
          id: 1,
          name: '12-Hour 2-2-3 Rotation (Default)',
          type: '12-hour',
          rotation: '2-2-3',
          assignments: {},
          simStaff: [],
          simCounter: 0,
          displayOrder: []
        }, staff),
        autoAssignForSchedule({
          id: 2,
          name: '12-Hour 4-3 Rotation',
          type: '12-hour',
          rotation: '4-3',
          assignments: {},
          simStaff: [],
          simCounter: 0,
          displayOrder: []
        }, staff),
        autoAssignForSchedule({
          id: 3,
          name: '10-Hour Rotating Days Off (OT)',
          type: '10-hour',
          assignments: {},
          simStaff: [],
          simCounter: 0,
          displayOrder: []
        }, staff),
      ];
      
      startDate = new Date(2025, 11, 1);
      editingOpen = {};
      collapsedSchedules = {};
      lastSaved = null;
      
      saveStatus = 'All data cleared';
      setTimeout(() => saveStatus = '', 3000);
    }
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

  function buildAssignments12h_A_B(staffList) {
    const assignments: Record<number, any> = {};
    staffList.forEach((s, idx) => {
      assignments[s.id] = { team: idx % 2 === 0 ? 'A' : 'B' };
    });
    return assignments;
  }

  function buildAssignments10h_AM_PM(staffList) {
    const assignments: Record<number, any> = {};
    const half = Math.ceil(staffList.length / 2);
    staffList.forEach((s, idx) => {
      const shift = idx < half ? 'AM' : 'PM';
      const cycle = DAYS_OFF_CYCLE[idx % DAYS_OFF_CYCLE.length];
      assignments[s.id] = { shift, daysOff: [...cycle] };
    });
    return assignments;
  }

  function autoAssignForSchedule(schedule, staffList) {
    const base = (schedule.type === '12-hour')
      ? { ...schedule, assignments: buildAssignments12h_A_B(staffList) }
      : { ...schedule, assignments: buildAssignments10h_AM_PM(staffList) };
    return {
      ...base,
      displayOrder: base.displayOrder?.length ? base.displayOrder : staffList.map(s => s.id)
    };
  }

  function ensureAllSchedulesAssigned() {
    schedules = schedules.map(s => {
      const withAssign = autoAssignForSchedule(s, staff);
      const realIds = staff.map(p => p.id);
      const sims = withAssign.simStaff?.map(sim => sim.id) ?? [];
      const allIds = new Set([...realIds, ...sims]);
      const cleaned = (withAssign.displayOrder || []).filter(id => allIds.has(id));
      const missing = [...allIds].filter(id => !cleaned.includes(id));
      return { ...withAssign, displayOrder: [...cleaned, ...missing] };
    });
  }

  // ===== SCHEDULES =====
  let schedules = [
    autoAssignForSchedule({
      id: 1,
      name: '12-Hour 2-2-3 Rotation (Default)',
      type: '12-hour',
      rotation: '2-2-3',
      assignments: {},
      simStaff: [],
      simCounter: 0,
      displayOrder: []
    }, staff),

    autoAssignForSchedule({
      id: 2,
      name: '12-Hour 4-3 Rotation',
      type: '12-hour',
      rotation: '4-3',
      assignments: {},
      simStaff: [],
      simCounter: 0,
      displayOrder: []
    }, staff),

    autoAssignForSchedule({
      id: 3,
      name: '10-Hour Rotating Days Off (OT)',
      type: '10-hour',
      assignments: {},
      simStaff: [],
      simCounter: 0,
      displayOrder: []
    }, staff),
  ];

  let newStaffName = '';
  let newScheduleName = '';

  let editingOpen: Record<number, boolean> = {};
  function toggleEditing(id: number) {
    editingOpen = { ...editingOpen, [id]: !editingOpen[id] };
  }

  let collapsedSchedules: Record<number, boolean> = {};
  function toggleCollapse(id: number) {
    collapsedSchedules = { ...collapsedSchedules, [id]: !collapsedSchedules[id] };
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
  const getInversePattern = (pattern: number[]) => pattern.map(d => (d ? 0 : 1));

  // ===== CRUD (REAL STAFF) =====
  function addStaff() {
    const name = newStaffName.trim();
    if (!name) return;
    const newId = Math.max(0, ...staff.map(s => s.id)) + 1;
    staff = [...staff, { id: newId, name }];
    newStaffName = '';
    ensureAllSchedulesAssigned();
  }

  function deleteStaff(id: number) {
    staff = staff.filter(s => s.id !== id);
    schedules = schedules.map(s => {
      const nextAssignments = Object.fromEntries(
        Object.entries(s.assignments).filter(([sid]) => parseInt(sid) !== id)
      );
      const nextOrder = (s.displayOrder || []).filter((x) => x !== id);
      return { ...s, assignments: nextAssignments, displayOrder: nextOrder };
    });
  }

  function updateStaffName(id: number, name: string) {
    staff = staff.map(s => s.id === id ? { ...s, name } : s);
  }

  // ===== CRUD (SCHEDULES) =====
  function addSchedule() {
    const name = (newScheduleName.trim() || '12-Hour 2-2-3');
    const newId = Math.max(0, ...schedules.map(s => s.id)) + 1;
    let newSchedule = {
      id: newId,
      name,
      type: '12-hour',
      rotation: '2-2-3',
      assignments: {},
      simStaff: [],
      simCounter: 0,
      displayOrder: staff.map(s => s.id)
    };
    newSchedule = autoAssignForSchedule(newSchedule, staff);
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
        updated = autoAssignForSchedule(updated, staff);
      }

      if (field === 'rotation' && updated.type === '12-hour') {
        const allHaveTeam = staff.every(p => updated.assignments[p.id]?.team);
        if (!allHaveTeam) {
          updated = { ...updated, assignments: buildAssignments12h_A_B(staff) };
        }
      }

      return updated;
    });
  }

  function updateAssignment(scheduleId: number, staffId: number, field: string, value: any) {
    schedules = schedules.map(sch => {
      if (sch.id !== scheduleId) return sch;
      const assignments = { ...sch.assignments };
      const prev = assignments[staffId] || (sch.type === '12-hour' ? { team: 'A' } : { shift: 'AM', daysOff: [6,0] });
      assignments[staffId] = { ...prev, [field]: value };
      return { ...sch, assignments };
    });
  }

  function toggleDayOff(scheduleId: number, staffId: number, dayIndex: number) {
    schedules = schedules.map(sch => {
      if (sch.id !== scheduleId) return sch;
      const assignments = { ...sch.assignments };
      const prev = assignments[staffId] || { shift: 'AM', daysOff: [] };
      const daysOff = Array.isArray(prev.daysOff) ? [...prev.daysOff] : [];
      const idx = daysOff.indexOf(dayIndex);
      if (idx > -1) daysOff.splice(idx, 1); else daysOff.push(dayIndex);
      assignments[staffId] = { ...prev, daysOff: daysOff.sort((a,b)=>a-b) };
      return { ...sch, assignments };
    });
  }

  function updateDaysOffPattern(scheduleId: number, staffId: number, pattern: string) {
    const daysOff = DAYS_OFF_PATTERNS[pattern] || [6, 0];
    schedules = schedules.map(sch => {
      if (sch.id !== scheduleId) return sch;
      const assignments = { ...sch.assignments };
      const prev = assignments[staffId] || { shift: 'AM', daysOff: [6, 0] };
      assignments[staffId] = { ...prev, daysOff };
      return { ...sch, assignments };
    });
  }

  // ===== SIMULATED STAFF =====
  function nextSimId(schedule) {
    return (schedule.simCounter || 0) + 1;
  }

  function addSimStaff(scheduleId: number, kind: 'A'|'B'|'AM'|'PM') {
    schedules = schedules.map(s => {
      if (s.id !== scheduleId) return s;
      const simCounter = nextSimId(s);
      const base = {
        id: `sim-${simCounter}`,
        name: `[SIM] ${kind} #${simCounter}`
      };
      const sim =
        s.type === '12-hour'
          ? { ...base, team: kind }
          : { ...base, shift: kind, daysOff: [6,0] };
      const displayOrder = [...(s.displayOrder || []), sim.id];
      return {
        ...s,
        simCounter,
        simStaff: [...s.simStaff, sim],
        displayOrder
      };
    });
  }

  function deleteSimStaff(scheduleId: number, simId: string) {
    schedules = schedules.map(s => {
      if (s.id !== scheduleId) return s;
      return {
        ...s,
        simStaff: s.simStaff.filter(sim => sim.id !== simId),
        displayOrder: (s.displayOrder || []).filter(x => x !== simId)
      };
    });
  }

  function updateSimStaff(scheduleId: number, simId: string, field: string, value: any) {
    schedules = schedules.map(s => {
      if (s.id !== scheduleId) return s;
      const simStaff = s.simStaff.map(sim => sim.id === simId ? { ...sim, [field]: value } : sim);
      return { ...s, simStaff };
    });
  }

  function toggleSimDayOff(scheduleId: number, simId: string, dayIndex: number) {
    schedules = schedules.map(s => {
      if (s.id !== scheduleId) return s;
      const simStaff = s.simStaff.map(sim => {
        if (sim.id !== simId) return sim;
        const daysOff = Array.isArray(sim.daysOff) ? [...sim.daysOff] : [];
        const idx = daysOff.indexOf(dayIndex);
        if (idx > -1) daysOff.splice(idx, 1); else daysOff.push(dayIndex);
        return { ...sim, daysOff: daysOff.sort((a,b)=>a-b) };
      });
      return { ...s, simStaff };
    });
  }

  function updateSimDaysOffPattern(scheduleId: number, simId: string, pattern: string) {
    const daysOff = DAYS_OFF_PATTERNS[pattern] || [6, 0];
    schedules = schedules.map(s => {
      if (s.id !== scheduleId) return s;
      const simStaff = s.simStaff.map(sim => {
        if (sim.id !== simId) return sim;
        return { ...sim, daysOff };
      });
      return { ...s, simStaff };
    });
  }

  function toggleScheduleDay(scheduleId: number, staffId: number, dayIndex: number) {
    schedules = schedules.map(sch => {
      if (sch.id !== scheduleId) return sch;

      const newAssignments: Record<string, any> = {};
      for (const key of Object.keys(sch.assignments)) {
        const a = sch.assignments[key];
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
        const current = generateSchedule({ ...sch, assignments: newAssignments }, staffId);
        newAssignments[staffId].manualSchedule = [...current];
      }
      const cur = newAssignments[staffId].manualSchedule[dayIndex];
      newAssignments[staffId].manualSchedule[dayIndex] = cur === 1 ? 0 : 1;

      return { ...sch, assignments: newAssignments, lastEdited: Date.now() };
    });
  }

  function toggleScheduleDayForSim(scheduleId: number, simId: string, dayIndex: number) {
    schedules = schedules.map(s => {
      if (s.id !== scheduleId) return s;
      const simStaff = s.simStaff.map(sim => {
        if (sim.id !== simId) return sim;
        const manual = sim.manualSchedule
          ? [...sim.manualSchedule]
          : generateScheduleForAssignment(s, sim);
        manual[dayIndex] = manual[dayIndex] === 1 ? 0 : 1;
        return { ...sim, manualSchedule: manual };
      });
      return { ...s, simStaff, lastEdited: Date.now() };
    });
  }

  function resetManualEdits(scheduleId: number) {
    schedules = schedules.map(sch => {
      if (sch.id !== scheduleId) return sch;
      const assignments: Record<string, any> = {};
      for (const key of Object.keys(sch.assignments)) {
        const a = { ...sch.assignments[key] };
        delete a.manualSchedule;
        assignments[key] = a;
      }
      const simStaff = sch.simStaff.map(sim => {
        const copy = { ...sim };
        delete copy.manualSchedule;
        return copy;
      });
      return { ...sch, assignments, simStaff };
    });
  }

  function reorderList(scheduleId: number, fromId: string | number, toId: string | number) {
    schedules = schedules.map(s => {
      if (s.id !== scheduleId) return s;
      const order = [...(s.displayOrder || [])];
      const fromIdx = order.indexOf(fromId);
      const toIdx = order.indexOf(toId);
      if (fromIdx === -1 || toIdx === -1 || fromIdx === toIdx) return s;
      const [moved] = order.splice(fromIdx, 1);
      order.splice(toIdx, 0, moved);
      return { ...s, displayOrder: order };
    });
  }

  function moveStaffInSchedule(scheduleId: number, staffId: string | number, direction: 'up' | 'down') {
    schedules = schedules.map(s => {
      if (s.id !== scheduleId) return s;
      const order = [...(s.displayOrder || [])];
      const idx = order.indexOf(staffId);
      if (idx === -1) return s;
      
      const newIdx = direction === 'up' ? idx - 1 : idx + 1;
      if (newIdx < 0 || newIdx >= order.length) return s;
      
      [order[idx], order[newIdx]] = [order[newIdx], order[idx]];
      return { ...s, displayOrder: order };
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

  function generateSchedule(schedule, staffId: number) {
    const assignment = schedule.assignments[staffId];
    if (!assignment) return Array(14).fill(0);
    if (assignment.manualSchedule) return assignment.manualSchedule;
    return generateScheduleForAssignment(schedule, assignment);
  }

  function calculateScheduleCosts(schedule) {
    let totalCost = 0;
    let totalHours = 0;

    const rowsReal: any[] = [];
    const rowsSim: any[] = [];

    const dailyCoverage = Array(14).fill(0);
    const amCoverage = Array(14).fill(0);
    const pmCoverage = Array(14).fill(0);

    const hoursPerShift = schedule.type === '12-hour' ? 12 : 10;

    for (const s of staff) {
      const scheduleData = generateSchedule(schedule, s.id);
      const a = schedule.assignments[s.id] || {};
      const staffTotalHours = scheduleData.reduce((sum, d) => sum + d * hoursPerShift, 0);

      let regularHours, overtimeHours, staffCost;
      if (staffTotalHours > 80) {
        regularHours = 80;
        overtimeHours = staffTotalHours - 80;
        staffCost = (regularHours * regularRate) + (overtimeHours * overtimeRate);
      } else {
        regularHours = staffTotalHours;
        overtimeHours = 0;
        staffCost = staffTotalHours * regularRate;
      }

      totalCost += staffCost;
      totalHours += staffTotalHours;

      rowsReal.push({
        id: s.id,
        name: s.name,
        hours: staffTotalHours,
        regularHours,
        overtimeHours,
        cost: staffCost,
        schedule: scheduleData,
        team: a.team,
        shift: a.shift,
        simulated: false,
        isEdited: !!a.manualSchedule
      });

      scheduleData.forEach((on, idx) => {
        if (schedule.type === '12-hour') {
          dailyCoverage[idx] += on;
        } else {
          if (a.shift === 'AM') amCoverage[idx] += on;
          if (a.shift === 'PM') pmCoverage[idx] += on;
        }
      });
    }

    for (const sim of schedule.simStaff || []) {
      const sched = sim.manualSchedule ? sim.manualSchedule : generateScheduleForAssignment(schedule, sim);
      const simHours = sched.reduce((sum, d) => sum + d * hoursPerShift, 0);

      let regularHours, overtimeHours, simCost;
      if (simHours > 80) {
        regularHours = 80;
        overtimeHours = simHours - 80;
        simCost = (regularHours * regularRate) + (overtimeHours * overtimeRate);
      } else {
        regularHours = simHours;
        overtimeHours = 0;
        simCost = simHours * regularRate;
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
        if (schedule.type === '12-hour') {
          dailyCoverage[idx] += on;
        } else {
          if (sim.shift === 'AM') amCoverage[idx] += on;
          if (sim.shift === 'PM') pmCoverage[idx] += on;
        }
      });
    }

    const mapById = new Map<string | number, any>();
    [...rowsReal, ...rowsSim].forEach(row => mapById.set(row.id, row));
    const ordered: any[] = [];
    const order = schedule.displayOrder || [];

    for (const id of order) {
      if (mapById.has(id)) ordered.push(mapById.get(id));
      mapById.delete(id);
    }
    for (const leftover of mapById.values()) ordered.push(leftover);

    let gapOTHours = 0;
    if (schedule.type === '12-hour') {
      dailyCoverage.forEach(c => {
        if (c < requiredStaff) gapOTHours += (requiredStaff - c) * 12;
      });
    } else {
      amCoverage.forEach(c => { if (c < requiredStaff) gapOTHours += (requiredStaff - c) * 10; });
      pmCoverage.forEach(c => { if (c < requiredStaff) gapOTHours += (requiredStaff - c) * 10; });
    }
    const gapOTCost = gapOTHours * overtimeRate;
    totalCost += gapOTCost;

    return {
      totalCost,
      totalHours,
      staffDetails: ordered,
      dailyCoverage,
      amCoverage,
      pmCoverage,
      gapOTHours,
      gapOTCost
    };
  }

  $: allScheduleCosts = schedules.map(s => calculateScheduleCosts(s));
  
  $: costAnalysis = {
    twoWeekCosts: allScheduleCosts.map(c => c.totalCost),
    yearlyCosts: allScheduleCosts.map(c => c.totalCost * 26),
    avgHoursPerStaff: allScheduleCosts.map(c => {
      const totalStaff = staff.length + (schedules.find((_, i) => i === allScheduleCosts.indexOf(c))?.simStaff?.length || 0);
      return totalStaff > 0 ? (c.totalHours / totalStaff).toFixed(1) : 0;
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
    {regularRate}
    {overtimeRate}
    {requiredStaff}
    {startDate}
    {saveStatus}
    {lastSaved}
    {staff}
    {schedules}
    {dates}
    {days}
    {costAnalysis}
    {editingOpen}
    {collapsedSchedules}
    bind:newStaffName
    bind:newScheduleName
    bind:fileInput
    on:logout={logout}
    on:export={exportToJSON}
    on:import={triggerImport}
    on:importFile={importFromJSON}
    on:clearAll={clearAllData}
    on:shiftPeriod={(e) => shiftPeriod(e.detail)}
    on:setStartDate={setStartFromInput}
    on:addStaff={addStaff}
    on:deleteStaff={(e) => deleteStaff(e.detail)}
    on:updateStaffName={(e) => updateStaffName(e.detail.id, e.detail.name)}
    on:addSchedule={addSchedule}
    on:deleteSchedule={(e) => deleteSchedule(e.detail)}
    on:updateSchedule={(e) => updateSchedule(e.detail.id, e.detail.field, e.detail.value)}
    on:toggleEditing={(e) => toggleEditing(e.detail)}
    on:toggleCollapse={(e) => toggleCollapse(e.detail)}
    on:addSimStaff={(e) => addSimStaff(e.detail.scheduleId, e.detail.kind)}
    on:deleteSimStaff={(e) => deleteSimStaff(e.detail.scheduleId, e.detail.simId)}
    on:updateSimStaff={(e) => updateSimStaff(e.detail.scheduleId, e.detail.simId, e.detail.field, e.detail.value)}
    on:updateAssignment={(e) => updateAssignment(e.detail.scheduleId, e.detail.staffId, e.detail.field, e.detail.value)}
    on:toggleDayOff={(e) => toggleDayOff(e.detail.scheduleId, e.detail.staffId, e.detail.dayIndex)}
    on:toggleSimDayOff={(e) => toggleSimDayOff(e.detail.scheduleId, e.detail.simId, e.detail.dayIndex)}
    on:toggleScheduleDay={(e) => toggleScheduleDay(e.detail.scheduleId, e.detail.staffId, e.detail.dayIndex)}
    on:toggleScheduleDayForSim={(e) => toggleScheduleDayForSim(e.detail.scheduleId, e.detail.simId, e.detail.dayIndex)}
    on:resetManualEdits={(e) => resetManualEdits(e.detail)}
    on:reorderList={(e) => reorderList(e.detail.scheduleId, e.detail.fromId, e.detail.toId)}
    on:moveStaff={(e) => moveStaffInSchedule(e.detail.scheduleId, e.detail.staffId, e.detail.direction)}
    on:updateDaysOffPattern={(e) => updateDaysOffPattern(e.detail.scheduleId, e.detail.staffId, e.detail.pattern)}
    on:updateSimDaysOffPattern={(e) => updateSimDaysOffPattern(e.detail.scheduleId, e.detail.simId, e.detail.pattern)}
    {generateSchedule}
    {generateScheduleForAssignment}
    {calculateScheduleCosts}
    {getDaysOffPattern}
  />
{/if}