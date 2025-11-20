<script lang="ts">
  import { Printer } from 'lucide-svelte';
  import National1Schedule from './National1Schedule.svelte';
  import National2EastSchedule from './National2EastSchedule.svelte';
  import National2WestSchedule from './National2WestSchedule.svelte';
  import National3Schedule from './National3Schedule.svelte';

  export let dates: any[];
  export let days: string[];
  export let startDate: Date;

  // Active unit selection
  let activeUnit: 'national1' | 'national2east' | 'national2west' | 'national3' = 'national3';

  const units = [
    { id: 'national1', label: 'National 1', component: National1Schedule },
    { id: 'national2east', label: 'National 2 East', component: National2EastSchedule },
    { id: 'national2west', label: 'National 2 West', component: National2WestSchedule },
    { id: 'national3', label: 'National 3', component: National3Schedule },
  ];

  function printSchedule() {
    window.print();
  }

  function formatDateRange() {
    if (dates.length < 7) return '';
    const firstDate = dates[0].fullDate;
    const lastDate = dates[6].fullDate;
    return `${firstDate.toLocaleDateString()} - ${lastDate.toLocaleDateString()}`;
  }

  $: activeUnitData = units.find(u => u.id === activeUnit);
</script>

<style>
  @media print {
    /* Landscape orientation for master schedule */
    @page {
      size: landscape;
      margin: 0.3in;
    }
    
    .print-hide {
      display: none !important;
    }

    :global(.master-schedule-table) {
      font-size: 6.5pt;
    }

    :global(.master-schedule-table th),
    :global(.master-schedule-table td) {
      padding: 1px 2px;
    }

    :global(.master-schedule-table textarea) {
      border: none !important;
      background: transparent !important;
      padding: 0 !important;
      min-height: auto !important;
      font-size: 6pt !important;
    }

    .unit-header {
      font-size: 10pt;
      margin-bottom: 4px;
    }
  }
</style>

<div class="master-schedule-container bg-white rounded-lg shadow-lg p-6 mb-6">
  <!-- Header with Print Button -->
  <div class="flex items-center justify-between mb-6 print-hide">
    <div>
      <h2 class="text-2xl font-bold text-slate-800">Master Daily Schedule</h2>
      <p class="text-slate-600 mt-1">Week of {formatDateRange()}</p>
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
        on:click={printSchedule}
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
    <h1 class="unit-header text-center font-bold text-slate-900">{activeUnitData?.label} - Master Daily Schedule</h1>
    <p class="text-center text-[8pt] text-slate-600">Week of {formatDateRange()}</p>
  </div>

  <!-- Unit Tabs -->
  <div class="mb-6 print-hide">
    <div class="border-b border-slate-200">
      <div class="flex gap-1">
        {#each units as unit}
          <button
            on:click={() => activeUnit = unit.id}
            class="px-4 py-2 font-medium transition-colors {activeUnit === unit.id 
              ? 'text-blue-600 border-b-2 border-blue-600 bg-blue-50' 
              : 'text-slate-600 hover:text-slate-900 hover:bg-slate-50'}"
          >
            {unit.label}
          </button>
        {/each}
      </div>
    </div>
  </div>

  <!-- Instructions -->
  <div class="mb-4 p-4 bg-blue-50 border border-blue-200 rounded-lg print-hide">
    <p class="text-sm text-blue-800">
      <strong>📝 {activeUnitData?.label}:</strong> Weekday times are in the first column before Mon-Fri. Weekend times are next to Sat-Sun. Click any cell to edit activities.
    </p>
  </div>

  <!-- Active Unit Schedule -->
  {#if activeUnit === 'national1'}
    <National1Schedule {dates} onPrint={printSchedule} />
  {:else if activeUnit === 'national2east'}
    <National2EastSchedule {dates} onPrint={printSchedule} />
  {:else if activeUnit === 'national2west'}
    <National2WestSchedule {dates} onPrint={printSchedule} />
  {:else if activeUnit === 'national3'}
    <National3Schedule {dates} onPrint={printSchedule} />
  {/if}

  <!-- Print Footer Info -->
  <div class="mt-6 text-sm text-slate-600 print-hide">
    <p class="mb-2"><strong>🖨️ Printing:</strong> Click "Print" to print directly. The schedule automatically uses landscape orientation.</p>
    <p><strong>📄 For Print Shops:</strong> Click "Download PDF", then in the print dialog select "Save as PDF" as the destination. Give the PDF to your print shop to print at larger sizes (e.g., 11x17" or larger).</p>
  </div>

  <!-- Print-only footer -->
  <div class="hidden print:block mt-2 text-[7pt] text-slate-600 text-center">
    <p>Printed on {new Date().toLocaleDateString()}</p>
  </div>
</div>