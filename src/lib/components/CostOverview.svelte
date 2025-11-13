<script lang="ts">
  export let schedules: any[];
  export let costAnalysis: any;
</script>

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