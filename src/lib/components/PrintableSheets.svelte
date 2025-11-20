<script lang="ts">
  import { FileText, Printer } from 'lucide-svelte';
  import RoomRestrictionLog from './RoomRestrictionLog.svelte';

  // Active sheet tab
  let activeSheet: 'room-restriction' = 'room-restriction';

  const sheets = [
    { id: 'room-restriction', label: 'Room Restriction Log' },
    // Add more sheets here as needed
  ];

  function printSheet() {
    // STEP 1: Remove any landscape styles from other components
    const landscapeStyle = document.getElementById('master-schedule-landscape');
    if (landscapeStyle) {
      landscapeStyle.remove();
    }
    
    // STEP 2: Inject portrait print style
    const portraitStyle = document.createElement('style');
    portraitStyle.id = 'printable-sheets-portrait';
    portraitStyle.textContent = `
      @media print {
        @page {
          size: portrait !important;
          margin: 0.5in !important;
        }
      }
    `;
    document.head.appendChild(portraitStyle);
    
    // STEP 3: Mark printable sheet for printing
    const sheetEl = document.querySelector('.printable-sheet-container');
    if (sheetEl) {
      sheetEl.classList.add('sheet-print-target');
      
      // STEP 4: Trigger print
      setTimeout(() => {
        window.print();
        
        // STEP 5: Cleanup after print dialog closes
        setTimeout(() => {
          sheetEl.classList.remove('sheet-print-target');
          const styleEl = document.getElementById('printable-sheets-portrait');
          if (styleEl) styleEl.remove();
        }, 500);
      }, 100);
    }
  }

  function downloadPDF() {
    // Remove any landscape styles
    const landscapeStyle = document.getElementById('master-schedule-landscape');
    if (landscapeStyle) {
      landscapeStyle.remove();
    }
    
    // Inject portrait style
    const portraitStyle = document.createElement('style');
    portraitStyle.id = 'printable-sheets-portrait';
    portraitStyle.textContent = `
      @media print {
        @page {
          size: portrait !important;
          margin: 0.5in !important;
        }
      }
    `;
    document.head.appendChild(portraitStyle);
    
    // Mark for printing
    const sheetEl = document.querySelector('.printable-sheet-container');
    if (sheetEl) {
      sheetEl.classList.add('sheet-print-target');
    }
    
    // Show instructions for saving as PDF
    alert('In the print dialog:\n1. Select "Save as PDF" or "Microsoft Print to PDF" as the printer\n2. Verify portrait orientation is selected\n3. Click Save and name your file (e.g., "Room_Restriction_Log.pdf")');
    
    setTimeout(() => {
      window.print();
      
      // Cleanup - longer timeout for PDF save
      setTimeout(() => {
        if (sheetEl) {
          sheetEl.classList.remove('sheet-print-target');
        }
        const styleEl = document.getElementById('printable-sheets-portrait');
        if (styleEl) styleEl.remove();
      }, 1000);
    }, 100);
  }
</script>

<style>
  @media print {
    /* Hide ALL printable sheet containers by default */
    :global(.printable-sheet-container) {
      display: none !important;
    }
    
    /* Show ONLY the sheet being printed */
    :global(.printable-sheet-container.sheet-print-target) {
      display: block !important;
    }

    /* Hide ALL master schedules during sheet print */
    :global(.master-schedule-container) {
      display: none !important;
    }

    /* Hide ALL staff schedules during sheet print */
    :global([data-schedule-id]) {
      display: none !important;
    }

    .print-hide {
      display: none !important;
    }

    /* Ensure proper portrait layout */
    .printable-sheet-container {
      background: white !important;
      min-height: 100vh !important;
      padding: 0 !important;
      margin: 0 !important;
      box-shadow: none !important;
      border-radius: 0 !important;
    }

    /* Ensure sheet content is properly displayed */
    :global(.bg-slate-200) {
      background: white !important;
      padding: 0 !important;
      margin: 0 !important;
      min-height: auto !important;
    }

    /* Optimize the sheet itself */
    :global(.bg-white.text-gray-900) {
      width: 100% !important;
      max-width: 100% !important;
      height: auto !important;
      box-shadow: none !important;
      border: none !important;
      margin: 0 !important;
      padding: 0.5in !important;
    }

    /* Ensure proper spacing and borders */
    :global(table) {
      border-collapse: collapse !important;
      page-break-inside: auto !important;
    }

    :global(tr) {
      page-break-inside: avoid !important;
      page-break-after: auto !important;
    }

    /* Ensure borders are visible */
    :global(.border),
    :global(.border-gray-400),
    :global(.border-gray-700),
    :global(.border-gray-900) {
      border-color: #1e293b !important;
      print-color-adjust: exact !important;
      -webkit-print-color-adjust: exact !important;
    }

    /* Ensure checkboxes are visible */
    :global(input[type="checkbox"]) {
      border: 1px solid #1e293b !important;
      print-color-adjust: exact !important;
      -webkit-print-color-adjust: exact !important;
    }

    /* Remove any gray backgrounds */
    :global(body) {
      background: white !important;
    }
  }
</style>

<div class="printable-sheet-container bg-white rounded-lg shadow-lg mb-6 p-6">
  <!-- Header with Print Buttons -->
  <div class="flex items-center justify-between mb-6 print-hide">
    <div class="flex items-center gap-3">
      <FileText class="w-6 h-6 text-blue-600" />
      <h2 class="text-2xl font-bold text-slate-800">Printable Sheets</h2>
    </div>
    <div class="flex gap-2">
      <button
        on:click={printSheet}
        class="px-4 py-2 bg-blue-600 text-white rounded-lg hover:bg-blue-700 flex items-center gap-2 font-medium transition-colors"
        title="Print sheet"
      >
        <Printer class="w-4 h-4" />
        Print
      </button>
      <button
        on:click={downloadPDF}
        class="px-4 py-2 bg-green-600 text-white rounded-lg hover:bg-green-700 flex items-center gap-2 font-medium transition-colors"
        title="Save as PDF"
      >
        <svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
          <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 10v6m0 0l-3-3m3 3l3-3m2 8H7a2 2 0 01-2-2V5a2 2 0 012-2h5.586a1 1 0 01.707.293l5.414 5.414a1 1 0 01.293.707V19a2 2 0 01-2 2z" />
        </svg>
        Download PDF
      </button>
    </div>
  </div>

  <!-- Sheet Tabs -->
  <div class="border-b border-slate-200 mb-6 print-hide">
    <div class="flex gap-1">
      {#each sheets as sheet}
        <button
          on:click={() => activeSheet = sheet.id}
          class="px-4 py-2 font-medium transition-colors {activeSheet === sheet.id 
            ? 'text-blue-600 border-b-2 border-blue-600 bg-blue-50' 
            : 'text-slate-600 hover:text-slate-900 hover:bg-slate-50'}"
        >
          {sheet.label}
        </button>
      {/each}
    </div>
  </div>

  <!-- Instructions -->
  <div class="mb-4 p-4 bg-blue-50 border border-blue-200 rounded-lg print-hide">
    <p class="text-sm text-blue-800">
      <strong>📄 Printable Sheet:</strong> This sheet is designed to be printed on standard 8.5" x 11" paper in portrait orientation. Click Print to print directly, or Download PDF to save for later.
    </p>
  </div>

  <!-- Active Sheet Content -->
  {#if activeSheet === 'room-restriction'}
    <RoomRestrictionLog />
  {/if}

  <!-- Print Footer Info -->
  <div class="mt-6 text-sm text-slate-600 print-hide">
    <p class="mb-2"><strong>🖨️ Printing:</strong> Click "Print" to print directly. The sheet automatically uses portrait orientation and fits on standard 8.5" x 11" paper.</p>
    <p><strong>📄 Save as PDF:</strong> Click "Download PDF", then in the print dialog select "Save as PDF" as the destination to save a digital copy.</p>
  </div>
</div>