<script lang="ts">
  // Room Restriction Log component
</script>

<div class="min-h-screen bg-slate-200 flex items-start justify-center pt-2 pb-8">
  <!-- Sheet -->
  <div class="bg-white text-gray-900 w-[8.5in] h-[11in] shadow border border-gray-400 pt-4 px-8 pb-8 text-[0.8rem] leading-tight">
    <!-- Header -->
    <div class="text-center mb-3">
      <h1 class="text-base font-semibold tracking-wide">ABRAXAS ACADEMY</h1>
      <h2 class="text-lg font-bold tracking-wide mt-1">ROOM RESTRICTION DOOR LOG</h2>
    </div>

    <!-- Program checkboxes -->
    <div class="flex gap-8 justify-center mb-3">
      <label class="inline-flex items-center gap-1">
        <input type="checkbox" class="w-4 h-4 border-gray-700">
        <span>Detention Program</span>
      </label>
      <label class="inline-flex items-center gap-1">
        <input type="checkbox" class="w-4 h-4 border-gray-700">
        <span>Treatment Program</span>
      </label>
    </div>

    <!-- Resident / Restriction info -->
    <div class="space-y-1.5 mb-3">
      <div class="flex gap-4">
        <div class="flex-1 flex items-center gap-2">
          <span class="font-semibold whitespace-nowrap">Resident Name:</span>
          <div class="flex-1 border-b border-gray-700 h-4"></div>
        </div>
        <div class="w-40 flex items-center gap-2">
          <span class="font-semibold whitespace-nowrap">Dorm/Room:</span>
          <div class="flex-1 border-b border-gray-700 h-4"></div>
        </div>
      </div>

      <div class="flex gap-4">
        <div class="flex-1 flex items-center gap-2">
          <span class="font-semibold whitespace-nowrap">Date of Restriction:</span>
          <div class="flex-1 border-b border-gray-700 h-4"></div>
        </div>
      </div>

      <div class="flex gap-4">
        <div class="flex-1 flex items-center gap-2">
          <span class="font-semibold whitespace-nowrap">Start Time:</span>
          <div class="w-24 border-b border-gray-700 h-4"></div>
          <span class="ml-2 text-[0.7rem] flex items-center gap-1">
            <span class="inline-flex items-center gap-1">
              <input type="checkbox" class="w-3 h-3 border-gray-700">
              <span>AM</span>
            </span>
            <span class="inline-flex items-center gap-1 ml-2">
              <input type="checkbox" class="w-3 h-3 border-gray-700">
              <span>PM</span>
            </span>
          </span>
        </div>

        <div class="flex-1 flex items-center gap-2">
          <span class="font-semibold whitespace-nowrap">End Time:</span>
          <div class="w-24 border-b border-gray-700 h-4"></div>
          <span class="ml-2 text-[0.7rem] flex items-center gap-1">
            <span class="inline-flex items-center gap-1">
              <input type="checkbox" class="w-3 h-3 border-gray-700">
              <span>AM</span>
            </span>
            <span class="inline-flex items-center gap-1 ml-2">
              <input type="checkbox" class="w-3 h-3 border-gray-700">
              <span>PM</span>
            </span>
          </span>
        </div>
      </div>
    </div>

    <!-- Instructions -->
    <div class="border border-gray-700 p-2.5 mb-3 text-[0.7rem] leading-snug">
      <p class="font-semibold mb-1">Instructions (Room Restriction Door Log)</p>
      <ol class="list-decimal list-inside space-y-0.5">
        <li>Use this sheet any time a resident is secured in a room and the restriction is expected to exceed 1 hour.</li>
        <li>Staff must complete visual safety checks at least every 15 minutes and document the time and their initials on this sheet.</li>
        <li>The on-duty supervisor must be notified immediately.</li>
        <li>The supervisor must then notify an administrator and email administration with the resident's name, reason for continued restriction, and interventions attempted.</li>
        <li>This sheet must remain posted on the resident's door for the duration of the restriction and then turned in according to facility policy.</li>
        <li>A member of the Clinical team and a member of Administration should come to the unit, review the situation, and sign below, unless no one from that department is available/on site.</li>
        <li>If the restriction exceeds 4 hours, start a new sheet immediately and continue checks without gaps.</li>
      </ol>
    </div>

    <!-- Observation table -->
    <div class="mb-3">
      <table class="w-full border border-gray-900 text-[0.65rem]">
        <thead>
          <tr class="bg-gray-100">
            <th class="border border-gray-900 px-1 py-0.5 w-20 text-left">Time</th>
            <th class="border border-gray-900 px-1 py-0.5 w-20 text-left">Staff Initials</th>
            <th class="border border-gray-900 px-1 py-0.5 text-left">Status / Notes</th>
          </tr>
        </thead>
        <tbody>
          {#each Array(16) as _, i}
            <tr>
              <td class="border border-gray-900 px-1 py-0.5">&nbsp;</td>
              <td class="border border-gray-900 px-1 py-0.5">&nbsp;</td>
              <td class="border border-gray-900 px-1 py-0.5">&nbsp;</td>
            </tr>
          {/each}
        </tbody>
      </table>
    </div>

    <!-- Sign-off section -->
    <div class="space-y-1 text-[0.7rem] mt-3">
      <div class="pt-1.5">
        <p class="font-semibold mb-1">Supervisor Review</p>
        <div class="flex gap-4 mb-1">
          <div class="flex-1 flex items-center gap-1.5">
            <span class="whitespace-nowrap">Printed Name:</span>
            <div class="flex-1 border-b border-gray-700 h-4"></div>
          </div>
          <div class="flex-1 flex items-center gap-1.5">
            <span class="whitespace-nowrap">Signature:</span>
            <div class="flex-1 border-b border-gray-700 h-4"></div>
          </div>
          <div class="w-36 flex items-center gap-1.5">
            <span class="whitespace-nowrap">Date/Time:</span>
            <div class="flex-1 border-b border-gray-700 h-4"></div>
          </div>
        </div>
      </div>

      <div class="pt-1.5">
        <p class="font-semibold mb-1">Clinical Team Review</p>
        <div class="flex gap-4 mb-1">
          <div class="flex-1 flex items-center gap-1.5">
            <span class="whitespace-nowrap">Printed Name:</span>
            <div class="flex-1 border-b border-gray-700 h-4"></div>
          </div>
          <div class="flex-1 flex items-center gap-1.5">
            <span class="whitespace-nowrap">Signature:</span>
            <div class="flex-1 border-b border-gray-700 h-4"></div>
          </div>
          <div class="w-36 flex items-center gap-1.5">
            <span class="whitespace-nowrap">Date/Time:</span>
            <div class="flex-1 border-b border-gray-700 h-4"></div>
          </div>
        </div>
      </div>

      <div class="pt-1.5">
        <p class="font-semibold mb-1">Administration Review</p>
        <div class="flex gap-4 mb-1">
          <div class="flex-1 flex items-center gap-1.5">
            <span class="whitespace-nowrap">Printed Name:</span>
            <div class="flex-1 border-b border-gray-700 h-4"></div>
          </div>
          <div class="flex-1 flex items-center gap-1.5">
            <span class="whitespace-nowrap">Signature:</span>
            <div class="flex-1 border-b border-gray-700 h-4"></div>
          </div>
          <div class="w-36 flex items-center gap-1.5">
            <span class="whitespace-nowrap">Date/Time:</span>
            <div class="flex-1 border-b border-gray-700 h-4"></div>
          </div>
        </div>
      </div>
    </div>
  </div>
</div>