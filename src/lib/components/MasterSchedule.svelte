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
    // STEP 1: Remove any portrait styles from staff schedule
    const portraitStyle = document.getElementById('staff-schedule-portrait');
    if (portraitStyle) {
      portraitStyle.remove();
    }
    
    // STEP 2: Inject our landscape style with high specificity
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
    
    // STEP 3: Mark master schedule for printing
    const masterEl = document.querySelector('.master-schedule-container');
    if (masterEl) {
      masterEl.classList.add('facility-print-target');
      
      // STEP 4: Trigger print
      setTimeout(() => {
        window.print();
        
        // STEP 5: Cleanup after print dialog closes
        setTimeout(() => {
          masterEl.classList.remove('facility-print-target');
          const styleEl = document.getElementById('master-schedule-landscape');
          if (styleEl) styleEl.remove();
        }, 500);
      }, 100);
    }
  }

  function downloadPDF() {
    // Remove any portrait styles from staff schedule
    const portraitStyle = document.getElementById('staff-schedule-portrait');
    if (portraitStyle) {
      portraitStyle.remove();
    }
    
    // Inject landscape style
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
    
    // Mark for printing
    const masterEl = document.querySelector('.master-schedule-container');
    if (masterEl) {
      masterEl.classList.add('facility-print-target');
    }
    
    // Show instructions for saving as PDF
    alert('In the print dialog:\n1. Select "Save as PDF" or "Microsoft Print to PDF" as the printer\n2. Verify landscape orientation is selected\n3. Click Save and name your file (e.g., "National3_Schedule.pdf")\n\nThis PDF can be taken to a print shop for large format printing.');
    
    setTimeout(() => {
      window.print();
      
      // Cleanup - longer timeout for PDF save
      setTimeout(() => {
        if (masterEl) {
          masterEl.classList.remove('facility-print-target');
        }
        const styleEl = document.getElementById('master-schedule-landscape');
        if (styleEl) styleEl.remove();
      }, 1000);
    }, 100);
  }

  $: activeUnitData = units.find(u => u.id === activeUnit);
</script>

<style>
  @media print {
    /* Hide ALL master schedules by default */
    :global(.master-schedule-container) {
      display: none !important;
    }
    
    /* Show ONLY the master schedule being printed */
    :global(.master-schedule-container.facility-print-target) {
      display: flex !important;
      flex-direction: column !important;
      justify-content: center !important;
      align-items: center !important;
    }

    /* Hide ALL staff schedules during facility schedule print */
    :global([data-schedule-id]) {
      display: none !important;
    }

    .print-hide {
      display: none !important;
    }

    /* Vertically center content with equal margins */
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

    /* Table wrapper for centering */
    .overflow-x-auto {
      overflow: visible !important;
      width: 100% !important;
      max-width: 100% !important;
      display: flex !important;
      justify-content: center !important;
    }

    /* Optimize table sizing */
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

    /* Time columns - fixed narrow width */
    :global(.master-schedule-table th:nth-child(1)),
    :global(.master-schedule-table td:nth-child(1)),
    :global(.master-schedule-table th:nth-child(7)),
    :global(.master-schedule-table td:nth-child(7)) {
      width: 9% !important;
      font-size: 5.5pt !important;
    }

    /* Day columns - equal width */
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

    /* Remove any gray backgrounds */
    :global(body) {
      background: white !important;
    }

    /* Ensure table borders are visible */
    :global(.master-schedule-table),
    :global(.master-schedule-table th),
    :global(.master-schedule-table td) {
      border-color: #1e293b !important;
      print-color-adjust: exact !important;
      -webkit-print-color-adjust: exact !important;
    }
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
    <h1 class="unit-header font-bold text-slate-900">{activeUnitData?.label} - Master Daily Schedule</h1>
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
    <p class="mb-2"><strong>🖨️ Printing:</strong> Click "Print" to print directly. The schedule automatically uses landscape orientation and fills the page.</p>
    <p><strong>📄 For Print Shops:</strong> Click "Download PDF", then in the print dialog select "Save as PDF" as the destination. Give the PDF to your print shop to print at larger sizes (e.g., 11x17" or larger).</p>
  </div>

  <!-- Print-only footer -->
  <div class="hidden print:block mt-2 text-slate-600 print-footer">
    <p>Printed on {new Date().toLocaleDateString()}</p>
  </div>
</div>