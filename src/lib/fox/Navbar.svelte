<script lang="ts">
  import { Button, ThemeToggle, Navbar } from "@fuxui/base";

  import Logo from "./Logo.svelte";
  import ThemeSelectDropdown from "./ThemeSelectDropdown.svelte";
  import { g } from "$lib/global.svelte";

  let { hasSidebar = false, showLogo = true } = $props();
</script>

<Navbar {hasSidebar} class="lg:left-74">
  <div class="flex items-center gap-2">
    <Button
      variant="link"
      class="block font-normal lg:hidden backdrop-blur-none"
      popovertarget="mobile-menu"
    >
      <svg
        xmlns="http://www.w3.org/2000/svg"
        viewBox="0 0 24 24"
        fill="currentColor"
        class="size-6!"
      >
        <path
          fill-rule="evenodd"
          d="M3 9a.75.75 0 0 1 .75-.75h16.5a.75.75 0 0 1 0 1.5H3.75A.75.75 0 0 1 3 9Zm0 6.75a.75.75 0 0 1 .75-.75h16.5a.75.75 0 0 1 0 1.5H3.75a.75.75 0 0 1-.75-.75Z"
          clip-rule="evenodd"
        ></path>
      </svg>
    </Button>

    <div
      class="lg:ml-5 text-base-950 dark:text-base-50 focus-visible:outline-base-900 dark:focus-visible:outline-base-50 rounded-xl font-semibold focus-visible:outline-2 focus-visible:outline-offset-4"
    >
      {g.space?.name} - {g.channel?.name}
    </div>
  </div>
  <div class="flex items-end gap-3">
    <ThemeSelectDropdown />
    <ThemeToggle />

    <!-- <Github href="https://github.com/flo-bit/ui-kit" class="mr-2 p-2" /> -->
  </div>
</Navbar>
