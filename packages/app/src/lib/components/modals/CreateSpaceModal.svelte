<script lang="ts">
  import {
    type RoomyAccount,
    RoomyEntityList,
    createSpace,
    createSpaceList,
    publicGroup,
  } from "@roomy-chat/sdk";
  import { navigate } from "$lib/utils.svelte";
  import { Modal, Input, Button, Heading } from "@fuxui/base";
  import Icon from "@iconify/svelte";
  import { co } from "jazz-tools";

  let {
    open = $bindable(false),
    me,
  }: {
    open: boolean;
    me: co.loaded<typeof RoomyAccount> | undefined | null;
  } = $props();

  let newSpaceName = $state("");

  async function createSpaceSubmit(evt: Event) {
    evt.preventDefault();
    if (!newSpaceName) return;

    if (me?.profile && me.profile.newJoinedSpacesTest === undefined) {
      me.profile.newJoinedSpacesTest = RoomyEntityList.create(
        [],
        publicGroup("reader"),
      );
    }

    const space = await createSpace(newSpaceName);

    me?.profile?.newJoinedSpacesTest?.push(space);

    newSpaceName = "";

    open = false;

    navigate({ space: space.id });
  }
</script>

<Modal bind:open>
  <form
    id="createSpace"
    class="flex flex-col gap-4"
    onsubmit={createSpaceSubmit}
  >
    <Heading>Create a new space</Heading>
    <Input bind:value={newSpaceName} placeholder="Name" type="text" required />
    <Button
      type="submit"
      disabled={!newSpaceName}
      class="w-full justify-start"
      size="lg"
    >
      <Icon icon="basil:plus-outline" font-size="2em" />
      Create Space
    </Button>
  </form>
</Modal>
