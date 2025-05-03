<script lang="ts">
  import { g } from "$lib/global.svelte";
  import { derivePromise, navigate } from "$lib/utils.svelte";
  import { cn } from "@fuxui/base";
  import { Image, Space } from "@roomy-chat/sdk";
  import { AvatarMarble } from "svelte-boring-avatars";

  let {
    space,
    handleClick,
  }: { space: Space; handleClick?: (e: MouseEvent) => void } = $props();

  const spaceImage = derivePromise(null, async () => {
    if (space.image && g.roomy) {
      return (await g.roomy.open(Image, space.image)) as Image;
    }
  });

  let isCurrent = $derived(g.space?.id === space.id);
</script>

<button
  onclick={(e: MouseEvent) => {
    navigate({ space: space.handles((x) => x.get(0)) || space.id });
    handleClick?.(e);
  }}
  data-sveltekit-keepfocus
  class={cn(
    "cursor-pointer p-2 rounded-full",
    isCurrent ? "" : "opacity-70 hover:opacity-100",
  )}
>
  <div class={cn("p-1 rounded-full", isCurrent ? "bg-accent-500" : "")}>
    {#if spaceImage.value?.uri}
      <img
        src={spaceImage.value?.uri}
        alt={space.name}
        class="w-10 h-10 object-cover rounded-full object-center"
      />
    {:else if space && space.id}
      <div class="w-10 h-10">
        <AvatarMarble name={space.id} />
      </div>
    {:else}
      <div class="w-10 h-10 bg-base-300 rounded-full"></div>
    {/if}
  </div>
  <span class="text-xs text-base-900 dark:text-base-100">{space.name}</span>
</button>
