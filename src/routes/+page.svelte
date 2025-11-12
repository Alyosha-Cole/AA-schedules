<script lang="ts">
  import { Calendar, Plus, Trash2, Edit2, ChevronDown, ChevronRight, Download, Upload, Save, Lock } from 'lucide-svelte';
  import { onMount } from 'svelte';

  // ===== PASSWORD PROTECTION =====
  const CORRECT_PASSWORD = 'AAscheduler2025'; // Change this to your desired password
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
  const overtimeRate = 37.5; // 1.5x
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

  let dragItem: { scheduleId: number; id: string | number } | null = null;

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

  function onDragStartRow(scheduleId: number, id: string | number, e: DragEvent) {
    dragItem = { scheduleId, id };
    (e.currentTarget as HTMLElement).classList.add('opacity-60');
    e.dataTransfer?.setData('text/plain', String(id));
    e.dataTransfer?.setDragImage(new Image(), 0, 0);
  }
  function onDragOverRow(e: DragEvent) { e.preventDefault(); }
  function onDropRow(scheduleId: number, id: string | number, e: DragEvent) {
    e.preventDefault();
    if (dragItem && dragItem.scheduleId === scheduleId) {
      reorderList(scheduleId, dragItem.id, id);
    }
  }
  function onDragEndRow(e: DragEvent) {
    (e.currentTarget as HTMLElement).classList.remove('opacity-60');
    dragItem = null;
  }

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
  function toInputValue(d: Date) {
    const y = d.getFullYear();
    const m = String(d.getMonth() + 1).padStart(2, '0');
    const dd = String(d.getDate()).padStart(2, '0');
    return `${y}-${m}-${dd}`;
  }
</script>

<style>
  :root {
    --sticky-top: 0px;
  }
</style>

{#if !isAuthenticated}
<div class="min-h-screen bg-gradient-to-br from-blue-500 to-blue-700 flex items-center justify-center p-6">
  <div class="bg-white rounded-lg shadow-2xl p-8 max-w-md w-full">
    <div class="text-center mb-8">
      <div class="inline-flex items-center justify-center w-20 h-20 bg-blue-100 rounded-full mb-4">
        <Lock class="w-10 h-10 text-blue-600" aria-hidden="true" />
      </div>
      <h1 class="text-3xl font-bold text-slate-800 mb-2">Scheduler Login</h1>
      <p class="text-slate-600">Enter password to access the scheduler</p>
    </div>
    
    <form on:submit|preventDefault={checkPassword} class="space-y-4">
      <div>
        <label for="password" class="block text-sm font-medium text-slate-700 mb-2">
          Password
        </label>
        <input
          id="password"
          type="password"
          bind:value={passwordInput}
          class="w-full px-4 py-3 border-2 border-slate-300 rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-blue-500 transition-colors"
          placeholder="Enter password"
          autocomplete="current-password"
        />
        {#if passwordError}
          <p class="mt-2 text-sm text-red-600">Incorrect password. Please try again.</p>
        {/if}
      </div>
      
      <button
        type="submit"
        class="w-full bg-blue-600 text-white py-3 px-4 rounded-lg hover:bg-blue-700 transition-colors font-medium text-lg"
      >
        Login
      </button>
    </form>
    
    <div class="mt-6 p-4 bg-slate-50 rounded-lg">
      <p class="text-xs text-slate-600 text-center">
        🔒 Sessions expire when you close your browser
      </p>
    </div>
  </div>
</div>
{:else}
<div class="min-h-screen bg-gradient-to-br from-slate-50 to-slate-100 p-6">
  <div class="max-w-7xl mx-auto">
    <div class="bg-white rounded-lg shadow-lg p-6 mb-6">
      <div class="flex items-start justify-between mb-4">
        <div class="flex-1">
          <h1 class="text-3xl font-bold text-slate-800 mb-2 flex items-center gap-2">
            <Calendar class="text-blue-600" aria-hidden="true" />
            Juvenile Facility Staffing Scheduler
          </h1>
          <p class="text-slate-600">Compare Different Schedule Configurations</p>
        </div>
        
        <div class="flex flex-col gap-2 items-end">
          <div class="flex gap-2">
            <button
              on:click={logout}
              class="px-4 py-2 bg-slate-600 text-white rounded-lg hover:bg-slate-700 flex items-center gap-2 font-medium text-sm"
              title="Logout"
            >
              <Lock class="w-4 h-4" aria-hidden="true" />
              Logout
            </button>
            
            <button
              on:click={exportToJSON}
              class="px-4 py-2 bg-green-600 text-white rounded-lg hover:bg-green-700 flex items-center gap-2 font-medium text-sm"
              title="Export schedule to JSON file"
            >
              <Download class="w-4 h-4" aria-hidden="true" />
              Export JSON
            </button>
            
            <button
              on:click={() => fileInput.click()}
              class="px-4 py-2 bg-blue-600 text-white rounded-lg hover:bg-blue-700 flex items-center gap-2 font-medium text-sm"
              title="Import schedule from JSON file"
            >
              <Upload class="w-4 h-4" aria-hidden="true" />
              Import JSON
            </button>
            <input
              type="file"
              accept=".json"
              bind:this={fileInput}
              on:change={importFromJSON}
              class="hidden"
              aria-label="Import JSON file"
            />
            
            <button
              on:click={clearAllData}
              class="px-4 py-2 bg-red-600 text-white rounded-lg hover:bg-red-700 flex items-center gap-2 font-medium text-sm"
              title="Clear all data and reset to defaults"
            >
              <Trash2 class="w-4 h-4" aria-hidden="true" />
              Clear All
            </button>
          </div>
          
          {#if saveStatus}
            <div class="flex items-center gap-2 text-sm text-green-600">
              <Save class="w-4 h-4" aria-hidden="true" />
              <span>{saveStatus}</span>
            </div>
          {:else if lastSaved}
            <div class="text-xs text-slate-500">
              Last saved: {lastSaved.toLocaleTimeString()}
            </div>
          {/if}
        </div>
      </div>
      
      <div class="mt-2 text-sm text-slate-600 flex gap-4 flex-wrap">
        <span>Regular Rate: <strong>${regularRate}/hr</strong></span>
        <span aria-hidden="true">•</span>
        <span>Overtime Rate: <strong>${overtimeRate}/hr (1.5x)</strong> after 80 hours/2 weeks</span>
        <span aria-hidden="true">•</span>
        <span>Gap OT Rate: <strong>${overtimeRate}/hr</strong></span>
      </div>

      <div class="mt-4 flex flex-wrap items-center gap-2">
        <label class="text-sm text-slate-700">2-Week Start:</label>
        <input
          type="date"
          class="px-3 py-2 border border-slate-300 rounded focus:ring-2 focus:ring-blue-500 focus:border-transparent"
          on:change={setStartFromInput}
          value={toInputValue(startDate)}
          aria-label="Two week start date"
        />
        <button class="px-3 py-2 bg-slate-100 rounded border border-slate-300" on:click={()=>shiftPeriod(-14)} aria-label="Previous two weeks">−14d</button>
        <button class="px-3 py-2 bg-slate-100 rounded border border-slate-300" on:click={()=>shiftPeriod(+14)} aria-label="Next two weeks">+14d</button>
      </div>
      
      <div class="mt-4 p-3 bg-blue-50 border border-blue-200 rounded-lg">
        <div class="flex items-start gap-2 text-sm text-blue-800">
          <Save class="w-5 h-5 flex-shrink-0 mt-0.5" aria-hidden="true" />
          <div>
            <div class="font-semibold mb-1">💾 Auto-Save Enabled</div>
            <div>Your changes are automatically saved to your browser. Use <strong>Export JSON</strong> to save a copy you can share with others, and <strong>Import JSON</strong> to load saved schedules.</div>
          </div>
        </div>
      </div>
    </div>

    <div class="bg-white rounded-lg shadow-lg p-6 mb-6">
      <h2 class="text-xl font-bold text-slate-800 mb-4">Cost Analysis Overview</h2>
      <div class="overflow-x-auto">
        <table class="w-full text-sm">
          <thead>
            <tr class="border-b-2 border-slate-300">
              <th class="text-left py-3 px-4 font-semibold">Schedule Name</th>
              <th class="text-right py-3 px-4 font-semibold">2-Week Cost</th>
              <th class="text-right py-3 px-4 font-semibold">Yearly Cost (26 periods)</th>
              <th class="text-right py-3 px-4 font-semibold">Avg Hours/Staff (2 weeks)</th>
            </tr>
          </thead>
          <tbody>
            {#each schedules as schedule, idx (schedule.id)}
              <tr class="border-b border-slate-200 hover:bg-slate-50">
                <td class="py-3 px-4 font-medium">{schedule.name}</td>
                <td class="py-3 px-4 text-right text-green-600 font-semibold">
                  ${costAnalysis.twoWeekCosts[idx].toLocaleString()}
                </td>
                <td class="py-3 px-4 text-right text-blue-600 font-semibold">
                  ${costAnalysis.yearlyCosts[idx].toLocaleString()}
                </td>
                <td class="py-3 px-4 text-right font-semibold">
                  {costAnalysis.avgHoursPerStaff[idx]} hrs
                </td>
              </tr>
            {/each}
          </tbody>
        </table>
      </div>
    </div>

    <div class="bg-white rounded-lg shadow-lg p-6 mb-6">
      <div class="flex items-center justify-between mb-4 gap-2">
        <h2 class="text-xl font-bold text-slate-800">Staff Roster</h2>
        <div class="text-sm text-slate-500">Order per schedule is set inside "Assign".</div>
      </div>

      <div class="flex gap-2 mb-4">
        <input
          type="text"
          placeholder="Enter staff name"
          aria-label="Staff name"
          bind:value={newStaffName}
          on:keydown={(e)=> e.key === 'Enter' && addStaff()}
          class="flex-1 px-4 py-2 border border-slate-300 rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-transparent"
        />
        <button
          on:click={addStaff}
          class="px-6 py-2 bg-blue-600 text-white rounded-lg hover:bg-blue-700 flex items-center gap-2 font-medium"
          aria-label="Add staff"
        >
          <Plus class="w-5 h-5" aria-hidden="true" />
          Add Staff
        </button>
      </div>

      <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 2xl:grid-cols-4 gap-2">
        {#each staff as s, index (s.id)}
          <div class="flex items-center gap-2 border border-slate-200 rounded-lg p-2 bg-white">
            <span class="w-6 text-right text-slate-500 text-sm">{index + 1}.</span>
            <input
              type="text"
              bind:value={s.name}
              on:input={(e)=> updateStaffName(s.id, (e.currentTarget as HTMLInputElement).value)}
              class="flex-1 px-2 py-1 border border-slate-300 rounded text-sm focus:ring-2 focus:ring-blue-500 focus:border-transparent"
              aria-label={`Name for ${s.name}`}
            />
            <button on:click={()=> deleteStaff(s.id)} class="p-1 text-red-600 hover:bg-red-50 rounded transition-colors" aria-label={`Delete ${s.name}`}>
              <Trash2 class="w-4 h-4" aria-hidden="true" />
            </button>
          </div>
        {/each}
      </div>
    </div>

    <div class="bg-white rounded-lg shadow-lg p-6 mb-6">
      <h2 class="text-xl font-bold text-slate-800 mb-4">Schedule Configurations</h2>

      <div class="flex gap-2 mb-4">
        <input
          type="text"
          placeholder="Enter schedule name"
          aria-label="Schedule name"
          bind:value={newScheduleName}
          on:keydown={(e)=> e.key === 'Enter' && addSchedule()}
          class="flex-1 px-4 py-2 border border-slate-300 rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-transparent"
        />
        <button
          on:click={addSchedule}
          class="px-6 py-2 bg-green-600 text-white rounded-lg hover:bg-green-700 flex items-center gap-2 font-medium"
          aria-label="Add schedule"
        >
          <Plus class="w-5 h-5" aria-hidden="true" />
          Add Schedule
        </button>
      </div>

      <div class="space-y-2">
        {#each schedules as schedule (schedule.id)}
          <div class="flex items-center gap-4 border border-slate-200 rounded-lg p-3 bg-white">
            <input
              type="text"
              bind:value={schedule.name}
              on:input={(e)=> updateSchedule(schedule.id, 'name', (e.currentTarget as HTMLInputElement).value)}
              class="flex-1 px-3 py-2 border border-slate-300 rounded font-medium focus:ring-2 focus:ring-blue-500 focus:border-transparent"
              aria-label={`Schedule name for ${schedule.name}`}
            />
            <select
              bind:value={schedule.type}
              on:change={(e)=> updateSchedule(schedule.id, 'type', (e.currentTarget as HTMLSelectElement).value)}
              class="px-3 py-2 border border-slate-300 rounded focus:ring-2 focus:ring-blue-500 focus:border-transparent"
              aria-label="Shift length"
            >
              <option value="12-hour">12-Hour Shifts</option>
              <option value="10-hour">10-Hour Shifts</option>
            </select>
            {#if schedule.type === '12-hour'}
              <select
                bind:value={schedule.rotation}
                on:change={(e)=> updateSchedule(schedule.id, 'rotation', (e.currentTarget as HTMLSelectElement).value)}
                class="px-3 py-2 border border-slate-300 rounded focus:ring-2 focus:ring-blue-500 focus:border-transparent"
                aria-label="12-hour rotation"
              >
                <option value="2-2-3">2-2-3 Rotation</option>
                <option value="4-3">4-3 Rotation (B: Sun–Tue, A: Wed–Fri; Sat alternates)</option>
              </select>
            {/if}
            <button
              on:click={()=> toggleEditing(schedule.id)}
              class="px-4 py-2 bg-blue-500 text-white rounded hover:bg-blue-600 flex items-center gap-2"
              aria-pressed={!!editingOpen[schedule.id]}
              aria-controls={`assign-panel-${schedule.id}`}
            >
              <Edit2 class="w-4 h-4" aria-hidden="true" />
              {editingOpen[schedule.id] ? 'Done' : 'Assign'}
            </button>
            <button on:click={()=> deleteSchedule(schedule.id)} class="p-2 text-red-600 hover:bg-red-50 rounded transition-colors" aria-label={`Delete ${schedule.name}`}>
              <Trash2 class="w-5 h-5" aria-hidden="true" />
            </button>
          </div>
        {/each}
      </div>
    </div>

    {#each schedules as schedule (schedule.id + '-' + (schedule.lastEdited || 0) + '-' + startDate.getTime())}
      {@const costs = calculateScheduleCosts(schedule)}
      {@const minCoverage = schedule.type === '12-hour'
        ? Math.min(...costs.dailyCoverage)
        : Math.min(Math.min(...costs.amCoverage), Math.min(...costs.pmCoverage))}
      {@const assignedStaff = Object.keys(schedule.assignments).length}
      {@const minStaffNeeded = requiredStaff * 2}
      {@const isCollapsed = collapsedSchedules[schedule.id]}

      <div class="bg-white rounded-lg shadow-lg p-6 mb-6">
        <div class="flex items-center justify-between mb-4 pb-4 border-b-2 border-slate-200">
          <div class="flex items-center gap-3">
            <button
              on:click={() => toggleCollapse(schedule.id)}
              class="p-2 hover:bg-slate-100 rounded transition-colors"
              aria-expanded={!isCollapsed}
              aria-label={isCollapsed ? 'Expand schedule' : 'Collapse schedule'}
            >
              {#if isCollapsed}
                <ChevronRight class="w-6 h-6 text-slate-600" aria-hidden="true" />
              {:else}
                <ChevronDown class="w-6 h-6 text-slate-600" aria-hidden="true" />
              {/if}
            </button>
            
            <div class="flex flex-col gap-1">
              <h2 class="text-2xl font-bold text-slate-800">{schedule.name}</h2>
              <div class="text-sm text-slate-600">
                Assigned (real): {assignedStaff} / {minStaffNeeded} needed
              </div>
            </div>
          </div>

          <div class="flex items-center gap-2">
            <button
              on:click={()=> toggleEditing(schedule.id)}
              class="px-4 py-2 bg-blue-500 text-white rounded hover:bg-blue-600 flex items-center gap-2"
              aria-pressed={!!editingOpen[schedule.id]}
              aria-controls={`assign-panel-${schedule.id}`}
            >
              <Edit2 class="w-4 h-4" aria-hidden="true" />
              {editingOpen[schedule.id] ? 'Done' : 'Assign'}
            </button>

            <div class="flex gap-4">
              <div class="text-right">
                <div class="text-3xl font-bold text-green-600">${costs.totalCost.toLocaleString()}</div>
                <div class="text-sm text-slate-600">Total 2-Week Cost</div>
                {#if costs.gapOTHours > 0}
                  <div class="text-xs text-orange-600 mt-1">(includes ${costs.gapOTCost.toLocaleString()} gap OT)</div>
                {/if}
              </div>
              <div class="text-right">
                <div class="text-3xl font-bold text-blue-600">{costs.totalHours}</div>
                <div class="text-sm text-slate-600">Scheduled Hours</div>
              </div>
              <div class="text-right">
                <div class={`text-3xl font-bold ${minCoverage >= requiredStaff ? 'text-emerald-600' : 'text-red-600'}`}>
                  {minCoverage}/{requiredStaff}
                </div>
                <div class="text-sm text-slate-600">
                  {schedule.type === '12-hour' ? 'Min Coverage (any day)' : 'Min AM/PM Coverage'}
                </div>
              </div>
            </div>
          </div>
        </div>

        {#if !isCollapsed}
          {#if editingOpen[schedule.id]}
            <div id={`assign-panel-${schedule.id}`} class="mb-4 p-4 bg-slate-50 rounded-lg" tabindex="-1">
              <h3 class="font-bold mb-3">Assign & Order (drag to reorder)</h3>

              <div class="mb-3 flex gap-2">
                {#if schedule.type === '12-hour'}
                  <button class="px-3 py-1 bg-amber-600 text-white rounded" on:click={()=> addSimStaff(schedule.id, 'A')}>+ [SIM] Team A</button>
                  <button class="px-3 py-1 bg-amber-600 text-white rounded" on:click={()=> addSimStaff(schedule.id, 'B')}>+ [SIM] Team B</button>
                {:else}
                  <button class="px-3 py-1 bg-amber-600 text-white rounded" on:click={()=> addSimStaff(schedule.id, 'AM')}>+ [SIM] AM</button>
                  <button class="px-3 py-1 bg-amber-600 text-white rounded" on:click={()=> addSimStaff(schedule.id, 'PM')}>+ [SIM] PM</button>
                {/if}
              </div>

              <div class="grid grid-cols-1 md:grid-cols-2 gap-3">
                {#each schedule.displayOrder as rowId, idx (rowId)}
                  {@const isSim = String(rowId).startsWith('sim-')}
                  {@const simObj = isSim ? schedule.simStaff.find(s => s.id === rowId) : null}
                  {@const realObj = !isSim ? staff.find(p => p.id === rowId) : null}
                  {@const a = !isSim
                    ? (schedule.assignments[rowId] || (schedule.type === '12-hour' ? { team: 'A' } : { shift: 'AM', daysOff: [6,0] }))
                    : (simObj || {})}

                  <div
                    class={`flex items-start gap-3 border rounded-lg p-3 bg-white ${isSim ? 'border-amber-300' : 'border-slate-200'} cursor-grab`}
                    draggable="true"
                    on:dragstart={(e)=> onDragStartRow(schedule.id, rowId, e)}
                    on:dragover={onDragOverRow}
                    on:drop={(e)=> onDropRow(schedule.id, rowId, e)}
                    on:dragend={onDragEndRow}
                  >
                    <span class="w-6 text-right text-slate-500 text-sm select-none">{idx + 1}.</span>

                    <div class="flex-1">
                      <div class="flex items-center gap-2 mb-2">
                        {#if isSim}
                          <span class="text-xs bg-amber-600 text-white px-2 py-0.5 rounded">SIM</span>
                          <input
                            class="px-2 py-1 border rounded w-full"
                            bind:value={simObj.name}
                            on:input={(e)=> updateSimStaff(schedule.id, simObj.id, 'name', (e.currentTarget as HTMLInputElement).value)}
                            aria-label={`Name for ${simObj.id}`}
                          />
                          <button class="p-1 text-red-600 hover:bg-red-50 rounded" on:click={()=> deleteSimStaff(schedule.id, simObj.id)} aria-label={`Delete ${simObj.name}`}>
                            <Trash2 class="w-4 h-4" aria-hidden="true" />
                          </button>
                        {:else}
                          <div class="font-medium">{realObj?.name ?? '(Unknown)'}</div>
                        {/if}
                      </div>

                      {#if schedule.type === '12-hour'}
                        <div class="flex items-center gap-2">
                          <label class="text-sm" for={`team-${schedule.id}-${rowId}`}>Team:</label>
                          <select
                            id={`team-${schedule.id}-${rowId}`}
                            class="px-3 py-1 border rounded focus:ring-2 focus:ring-blue-500"
                            value={a.team || 'A'}
                            on:change={(e)=> {
                              const val = (e.currentTarget as HTMLSelectElement).value;
                              if (isSim) updateSimStaff(schedule.id, simObj.id, 'team', val);
                              else updateAssignment(schedule.id, rowId as number, 'team', val);
                            }}
                          >
                            <option value="A">A</option>
                            <option value="B">B</option>
                          </select>
                        </div>
                      {:else}
                        <div class="flex items-center gap-2 mb-2">
                          <label class="text-sm" for={`shift-${schedule.id}-${rowId}`}>Shift:</label>
                          <select
                            id={`shift-${schedule.id}-${rowId}`}
                            class="px-3 py-1 border rounded focus:ring-2 focus:ring-blue-500"
                            value={a.shift || 'AM'}
                            on:change={(e)=> {
                              const val = (e.currentTarget as HTMLSelectElement).value;
                              if (isSim) updateSimStaff(schedule.id, simObj.id, 'shift', val);
                              else updateAssignment(schedule.id, rowId as number, 'shift', val);
                            }}
                          >
                            <option value="AM">AM (7a-5p)</option>
                            <option value="PM">PM (1p-11p)</option>
                          </select>
                        </div>

                        <div class="flex items-center gap-2" role="group" aria-label="Days off">
                          <span class="text-sm">Days Off:</span>
                          {#each days as d, di}
                            <button
                              class={`px-2 py-1 rounded text-xs font-medium transition-colors ${
                                (a.daysOff || []).includes(di)
                                  ? 'bg-red-500 text-white'
                                  : 'bg-slate-200 text-slate-700 hover:bg-slate-300'
                              }`}
                              title={`${d} - ${(a.daysOff || []).includes(di) ? 'Day Off' : 'Working'}`}
                              on:click={()=>{
                                if (isSim) toggleSimDayOff(schedule.id, simObj.id, di);
                                else toggleDayOff(schedule.id, rowId as number, di);
                              }}
                              aria-pressed={(a.daysOff || []).includes(di)}
                            >
                              {d[0]}
                            </button>
                          {/each}
                        </div>
                      {/if}
                    </div>
                  </div>
                {/each}
              </div>
            </div>
          {/if}

          <div class="mb-3 flex items-center justify-between">
            <div class="p-3 bg-blue-50 border border-blue-200 rounded-lg flex-1 mr-2">
              <div class="flex items-center gap-2 text-sm text-blue-800">
                <span class="font-semibold">💡 Tip:</span>
                <span>Click any cell to toggle ON/OFF (works for both real and simulated rows).</span>
              </div>
            </div>
            <button
              on:click={()=> resetManualEdits(schedule.id)}
              class="px-4 py-3 bg-orange-500 text-white rounded-lg hover:bg-orange-600 transition-colors text-sm font-medium whitespace-nowrap"
            >
              Reset All Edits
            </button>
          </div>

          <div class="overflow-x-auto">
            <table class="w-full border-separate border-spacing-0 text-sm">
              <thead>
                <tr>
                  <th class="border border-slate-300 px-3 py-2 text-left font-semibold sticky left-0 bg-slate-100 z-50 top-[var(--sticky-top)]">#</th>
                  <th class="border border-slate-300 px-3 py-2 text-left font-semibold sticky left-[3.5rem] bg-slate-100 z-50 top-[var(--sticky-top)]">Staff</th>
                  {#each dates as d}
                    <th class="border border-slate-300 px-2 py-2 text-center min-w-[56px] sticky top-[var(--sticky-top)] bg-slate-100 z-40">
                      <div class="text-xs font-semibold">{d.date}</div>
                      <div class="text-xs text-slate-600">{d.dayName}</div>
                    </th>
                  {/each}
                  <th class="border border-slate-300 px-3 py-2 text-center font-semibold sticky top-[var(--sticky-top)] bg-slate-100 z-40">Hours</th>
                  <th class="border border-slate-300 px-3 py-2 text-center font-semibold sticky top-[var(--sticky-top)] bg-slate-100 z-40">Cost</th>
                </tr>
              </thead>
              <tbody>
                {#each costs.staffDetails as detail, idx (detail.id)}
                  {@const shiftLabel = schedule.type === '12-hour'
                    ? '8:30a-8:30p'
                    : (detail.shift === 'AM' ? '7a-5p' : '1p-11p')}
                  <tr class={`hover:bg-slate-50 ${detail.simulated ? 'bg-amber-50' : ''}`}>
                    <td class="border border-slate-300 px-3 py-2 font-medium sticky left-0 bg-white w-14 text-right text-slate-500 z-20">
                      {idx + 1}.
                    </td>
                    <td class="border border-slate-300 px-3 py-2 font-medium sticky left-[3.5rem] bg-white z-20">
                      {detail.name}
                      {#if detail.simulated}
                        <span class="ml-2 text-xs bg-amber-600 text-white px-2 py-0.5 rounded">SIM</span>
                      {/if}
                      {#if schedule.type === '12-hour' && detail.team}
                        <div class="text-xs text-slate-500">Team {detail.team}</div>
                      {/if}
                      {#if schedule.type === '10-hour' && detail.shift}
                        <div class="text-xs text-slate-500">{detail.shift} Shift</div>
                      {/if}
                      {#if detail.isEdited}
                        <span class="ml-2 text-xs bg-yellow-400 text-yellow-900 px-2 py-0.5 rounded">EDITED</span>
                      {/if}
                    </td>

                    {#each detail.schedule as working, dIdx}
                      <td
                        class={`border border-slate-300 px-1 py-2 text-center cursor-pointer transition-colors ${
                          working ? 'bg-green-100 hover:bg-green-200' : 'bg-red-50 hover:bg-red-100'
                        }`}
                        title={`Click to toggle ${dates[dIdx].dayName} (${working ? 'ON' : 'OFF'})`}
                        on:click={()=>{
                          if (String(detail.id).startsWith('sim-')) {
                            toggleScheduleDayForSim(schedule.id, detail.id as string, dIdx);
                          } else {
                            toggleScheduleDay(schedule.id, detail.id as number, dIdx);
                          }
                        }}
                      >
                        {#if working}
                          <div class="text-xs font-semibold text-green-800">{shiftLabel}</div>
                        {:else}
                          <div class="text-xs text-red-600">OFF</div>
                        {/if}
                      </td>
                    {/each}

                    <td class="border border-slate-300 px-3 py-2 text-center font-semibold">
                      {detail.hours}
                      {#if detail.overtimeHours > 0}
                        <div class="text-xs text-orange-600">
                          ({detail.regularHours} reg + {detail.overtimeHours} OT)
                        </div>
                      {/if}
                    </td>
                    <td class="border border-slate-300 px-3 py-2 text-center font-semibold text-green-700">
                      ${detail.cost.toLocaleString()}
                      {#if detail.overtimeHours > 0}
                        <div class="text-xs text-slate-600">
                          (${(detail.regularHours * regularRate).toLocaleString()} + ${(detail.overtimeHours * overtimeRate).toLocaleString()})
                        </div>
                      {/if}
                    </td>
                  </tr>
                {/each}

                {#if schedule.type === '12-hour'}
                  <tr class="bg-slate-100 font-bold">
                    <td class="border border-slate-300 px-3 py-2 sticky left-0 bg-slate-100 z-10"></td>
                    <td class="border border-slate-300 px-3 py-2 sticky left-[3.5rem] bg-slate-100 z-10">Daily Coverage</td>
                    {#each costs.dailyCoverage as c}
                      <td class={`border border-slate-300 px-2 py-2 text-center ${c >= requiredStaff ? 'bg-green-200 text-green-800' : 'bg-red-200 text-red-800'}`}>
                        {c}/{requiredStaff}
                      </td>
                    {/each}
                    <td class="border border-slate-300 px-3 py-2 text-center bg-slate-100">{costs.totalHours}</td>
                    <td class="border border-slate-300 px-3 py-2 text-center text-green-700 bg-slate-100">
                      ${costs.totalCost.toLocaleString()}
                    </td>
                  </tr>
                {:else}
                  <tr class="bg-blue-50 font-bold">
                    <td class="border border-slate-300 px-3 py-2 sticky left-0 bg-blue-50 z-10"></td>
                    <td class="border border-slate-300 px-3 py-2 sticky left-[3.5rem] bg-blue-50 z-10">AM Coverage</td>
                    {#each costs.amCoverage as c}
                      <td class={`border border-slate-300 px-2 py-2 text-center ${c >= requiredStaff ? 'bg-green-200 text-green-800' : 'bg-red-200 text-red-800'}`}>
                        {c}/{requiredStaff}
                      </td>
                    {/each}
                    <td class="border border-slate-300 px-3 py-2 text-center bg-blue-50" rowspan="2">
                      {costs.totalHours}
                      {#if costs.gapOTHours > 0}
                        <div class="text-xs text-orange-600">+{costs.gapOTHours} gap</div>
                      {/if}
                    </td>
                    <td class="border border-slate-300 px-3 py-2 text-center text-green-700 bg-blue-50" rowspan="2">
                      ${costs.totalCost.toLocaleString()}
                    </td>
                  </tr>
                  <tr class="bg-purple-50 font-bold">
                    <td class="border border-slate-300 px-3 py-2 sticky left-0 bg-purple-50 z-10"></td>
                    <td class="border border-slate-300 px-3 py-2 sticky left-[3.5rem] bg-purple-50 z-10">PM Coverage</td>
                    {#each costs.pmCoverage as c}
                      <td class={`border border-slate-300 px-2 py-2 text-center ${c >= requiredStaff ? 'bg-green-200 text-green-800' : 'bg-red-200 text-red-800'}`}>
                        {c}/{requiredStaff}
                      </td>
                    {/each}
                  </tr>
                {/if}
              </tbody>
            </table>
          </div>
        {/if}
      </div>
    {/each}
  </div>
</div>
{/if}