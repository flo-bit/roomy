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

  const { children } = $props();
  const spaces = derivePromise(
    [],
    async () => (await g.roomy?.spaces.items()) || [],
  );

  onMount(async () => {
    await user.init();

    if (!dev && browser) {
      posthog.init("phc_j80ksIuoxjfjRI7rPBmTLWx79rntg4Njz6Dixc3I3ik", {
        api_host: "https://roomy.chat/ingest",
        person_profiles: "identified_only", // or 'always' to create profiles for anonymous users as well
      });
    }
  });

  const isSpacesVisible = Toggle({ value: false, key: "isSpacesVisible" });
  setContext("isSpacesVisible", isSpacesVisible);

  let themeColor = $state({
    value:
      getComputedStyle(document.querySelector("html")!).getPropertyValue(
        "--color-base-300",
      ) ?? "#e6ddac",
  });
  setContext("themeColor", themeColor);

  const isSidebarVisible = Toggle({ value: false, key: "isSidebarVisible" });
  setContext("isSidebarVisible", isSidebarVisible);
  // hide on navigation
  afterNavigate(() => {
    if (
      page.params.space &&
      (page.params.channel || page.params.thread) &&
      isSidebarVisible.value
    )
      isSidebarVisible.toggle();
  });
</script>

<svelte:head>
  <meta name="theme-color" content={themeColor.value} />
  <meta name="msapplication-navbutton-color" content={themeColor.value} />
  <meta name="msapplication-TileColor" content={themeColor.value} />
  <title>Roomy</title>
</svelte:head>

{#if dev}
  <!-- Displays rendering scanner for debugging.
       Uncomment then recomment before committing. -->
  <!-- <RenderScan /> -->
{/if}

<!-- Container -->
<div class="flex w-screen h-screen max-h-screen overflow-clip gap-0">
  <Toaster />
  <div
    class="{page.params.space &&
      (isSidebarVisible.value
        ? 'flex z-1 absolute w-full'
        : 'hidden')} sm:w-auto sm:relative sm:flex h-full overflow-clip gap-0
      "
  >
    <!-- Content -->
    <div class="flex bg-base-300 h-full">
      <ServerBar
        {spaces}
        visible={isSpacesVisible.value || !page.params.space}
      />
      {#if page.params.space}
        <SidebarMain />
      {/if}
    </div>
    <!-- Overlay -->
    {#if page.params.space}
      <button
        onclick={() => {
          isSidebarVisible.toggle();
        }}
        aria-label="toggle navigation"
        class="{!isSidebarVisible.value
          ? 'hidden w-full'
          : 'sm:hidden'} cursor-pointer grow-2 h-full bg-black/10"
      ></button>
    {/if}
  </div>

  {@render children()}
</div>
