<script lang="ts">
  import { page } from "$app/state";
  import { base } from "$app/paths";
  import { Accordion, AccordionItem, Button, cn, Sidebar } from "@fuxui/base";

  import Logo from "./Logo.svelte";
  import type { Space } from "@roomy-chat/sdk";
  import { derivePromise, navigate } from "$lib/utils.svelte";
  import { g } from "$lib/global.svelte";
  import SidebarSpace from "$lib/components/SidebarSpace.svelte";
  import SpaceIcon from "./SpaceIcon.svelte";

  let sidebarItems = derivePromise([], async () => {
    if (!g.space) return [];
    return await g.space.sidebarItems.items();
  });

  let availableThreads = derivePromise([], async () =>
    ((await g.space?.threads.items()) || []).filter((x) => !x.softDeleted),
  );

  let {
    mobileOnly = false,
    spaces,
    visible,
  }: {
    mobileOnly: boolean;
    spaces: { value: Space[] };
    visible: boolean;
  } = $props();

  const handleClick = (e: MouseEvent) => {
    // close the mobile menu
    const popover = document.getElementById("mobile-menu");
    if (popover) {
      popover.hidePopover();
    }
  };
</script>

<Sidebar
  {mobileOnly}
  class="w-72 lg:w-72 xl:w-72 bg-base-100 lg:dark:bg-base-900/20"
>
  <div class="mb-24 flex flex-col items-start p-4">
    <div class={cn("flex items-center gap-2", "mb-8 px-3 py-1")}>
      <a
        class="text-base-950 dark:text-base-50 focus-visible:outline-base-900 dark:focus-visible:outline-base-50 rounded-xl font-semibold focus-visible:outline-2 focus-visible:outline-offset-4"
        href={"/home"}>🔥 roomy</a
      >
    </div>

    <div class="flex gap-2 flex-wrap mb-6">
      {#each spaces.value as space, i}
        <SpaceIcon {space} {handleClick} />
      {/each}
    </div>

    <Accordion type="multiple" class="w-full" value={["channels", "threads"]}>
      <AccordionItem
        class="border-0"
        title="Channels"
        value="channels"
        triggerClasses="text-sm px-3 py-1 font-semibold"
        contentClasses="flex flex-col gap-1 items-start px-1"
      >
        {#each sidebarItems.value as item}
          <Button
            onclick={(e: MouseEvent) => {
              navigate({
                space: page.params.space!,
                channel: item.id,
              });
              handleClick(e);
            }}
            data-current={g.channel?.id === item.id}
            variant="ghost"
            class="w-full justify-start">{item.name}</Button
          >
        {/each}
      </AccordionItem>

      <AccordionItem
        class="border-0"
        title="Threads"
        value="threads"
        triggerClasses="text-sm px-3 py-1 font-semibold"
        contentClasses="flex flex-col gap-1 items-start px-1"
      >
        {#each availableThreads.value as item}
          <Button
            onclick={(e: MouseEvent) => {
              navigate({
                space: page.params.space!,
                channel: item.id,
              });
              handleClick(e);
            }}
            variant="ghost"
            class="w-full justify-start">{item.name}</Button
          >
        {/each}
      </AccordionItem>
    </Accordion>
  </div>
</Sidebar>
