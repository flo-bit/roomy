<script lang="ts">
  import Icon from "@iconify/svelte";

  import { g } from "$lib/global.svelte";
  import { user } from "$lib/user.svelte";
  import { derivePromise } from "$lib/utils.svelte";
  import { Button, Heading, Paragraph, Subheading } from "@fuxui/base";

  import { BlueskyLogin } from "@fuxui/social";
  import SpaceIcon from "$lib/fox/SpaceIcon.svelte";

  const spaces = derivePromise(
    undefined,
    async () => await g.roomy?.spaces.items(),
  );
</script>

<div class=" min-h-screen flex items-center justify-center">
  <div class="flex flex-col gap-8 items-center">
    <Heading class="text-5xl font-bold text-center">Hello Roomy</Heading>
    <Paragraph
      class="text-lg font-medium max-w-2xl text-center px-4 text-pretty"
    >
      A digital gardening platform for communities. Built on the AT Protocol.
      Flourish in Spaces, curating knowledge and conversations together.
    </Paragraph>
    <div class="divider"></div>

    {#if !user.session}
      <BlueskyLogin
        login={async (handle) => {
          await user.loginWithHandle(handle);
          return true;
        }}
      />
      />
    {:else if !spaces.value}
      <span class="dz-loading dz-loading-spinner mx-auto w-25"></span>
    {:else if spaces.value.length > 0}
      <Subheading class="text-3xl font-bold">Your Spaces</Subheading>
      <section class="flex flex-wrap justify-center gap-4 max-w-5xl px-2">
        {#each spaces.value as space}
          <SpaceIcon {space} />
        {/each}
      </section>
    {:else if spaces.value.length === 0}
      <Paragraph class="text-lg font-medium text-center px-2">
        You don't have any spaces yet. Create one to get started!
      </Paragraph>
    {:else}
      <Paragraph>No servers found.</Paragraph>
    {/if}
  </div>
</div>
