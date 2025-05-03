<script lang="ts">
  import "../../app.css";
  import { onMount, setContext } from "svelte";
  import { browser, dev } from "$app/environment";

  import posthog from "posthog-js";
  import { Toaster } from "svelte-french-toast";
  import { RenderScan } from "svelte-render-scan";

  import { g } from "$lib/global.svelte";
  import { user } from "$lib/user.svelte";
  import { derivePromise, Toggle } from "$lib/utils.svelte";
  import ServerBar from "$lib/components/ServerBar.svelte";
  import SidebarMain from "$lib/components/SidebarMain.svelte";
  import { page } from "$app/state";
  import { afterNavigate } from "$app/navigation";
  import Navbar from "$lib/fox/Navbar.svelte";
  import Sidebar from "$lib/fox/Sidebar.svelte";
  const { children } = $props();
  const spaces = derivePromise(
    [],
    async () => (await g.roomy?.spaces.items()) || [],
  );

  onMount(async () => {
    await user.init();
  });
</script>

{#if dev}
  <!-- Displays rendering scanner for debugging.
       Uncomment then recomment before committing. -->
  <!-- <RenderScan /> -->
{/if}

{@render children()}
<!-- Container -->
<!-- <div class="flex w-screen h-screen max-h-screen overflow-clip gap-0">
  <Toaster />

  {@render children()}
</div> -->

<Toaster />
