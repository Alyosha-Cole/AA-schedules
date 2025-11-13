<script lang="ts">
  import { Lock } from 'lucide-svelte';
  import { createEventDispatcher } from 'svelte';

  export let passwordInput: string;
  export let passwordError: boolean;

  const dispatch = createEventDispatcher();

  function handleSubmit() {
    dispatch('submit');
  }
</script>

<div class="min-h-screen bg-gradient-to-br from-blue-500 to-blue-700 flex items-center justify-center p-6">
  <div class="bg-white rounded-lg shadow-2xl p-8 max-w-md w-full">
    <div class="text-center mb-8">
      <div class="inline-flex items-center justify-center w-20 h-20 bg-blue-100 rounded-full mb-4">
        <Lock class="w-10 h-10 text-blue-600" aria-hidden="true" />
      </div>
      <h1 class="text-3xl font-bold text-slate-800 mb-2">Scheduler Login</h1>
      <p class="text-slate-600">Enter password to access the scheduler</p>
    </div>
    
    <form on:submit|preventDefault={handleSubmit} class="space-y-4">
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