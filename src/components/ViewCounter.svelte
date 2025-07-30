<script>
  import { onMount } from "svelte";

  export let url = "";
  export let className = "";
  export let viewsText = "次浏览";

  let views = 0;
  let loading = true;

  async function fetchViews() {
    try {
      const counterUrl = window?.PUBLIC_COUNTER_URL || import.meta?.env?.PUBLIC_COUNTER_URL;
      if (!counterUrl) {
        loading = false;
        return;
      }
      
      const response = await fetch(`${counterUrl}${url}`, {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
        },
      });
      
      if (response.ok) {
        const data = await response.json();
        views = data.views || 0;
      }
    } catch (error) {
      console.error("Failed to fetch views:", error);
    } finally {
      loading = false;
    }
  }

  onMount(() => {
    fetchViews();
  });
</script>

{#if !loading}
  <div class="flex flex-row items-center {className}">
    <div class="transition h-6 w-6 rounded-md bg-black/5 dark:bg-white/10 text-black/50 dark:text-white/50 flex items-center justify-center mr-2">
      <svg class="w-4 h-4" fill="currentColor" viewBox="0 0 24 24" aria-label="views">
        <path d="M12 4.5C7 4.5 2.73 7.61 1 12c1.73 4.39 6 7.5 11 7.5s9.27-3.11 11-7.5c-1.73-4.39-6-7.5-11-7.5zM12 17c-2.76 0-5-2.24-5-5s2.24-5 5-5 5 2.24 5 5-2.24 5-5 5zm0-8c-1.66 0-3 1.34-3 3s1.34 3 3 3 3-1.34 3-3-1.34-3-3-3z"/>
      </svg>
    </div>
    <div class="text-sm">
      {views} {viewsText}
    </div>
  </div>
{/if}
</script>
