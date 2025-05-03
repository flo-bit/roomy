<script lang="ts">
  import { Button, Checkbox, Toolbar } from "bits-ui";
  import { format } from "date-fns";
  import { getContext } from "svelte";
  import { getProfile } from "$lib/profile.svelte";
  import Icon from "@iconify/svelte";
  import { user } from "$lib/user.svelte";
  import { outerWidth } from "svelte/reactivity/window";
  import Drawer from "./Drawer.svelte";
  import { getContentHtml, type Item } from "$lib/tiptap/editor";
  import { Announcement, Message, type EntityIdStr } from "@roomy-chat/sdk";
  import { g } from "$lib/global.svelte";
  import { derivePromise } from "$lib/utils.svelte";
  import type { JSONContent } from "@tiptap/core";
  import ChatInput from "./ChatInput.svelte";
  import toast from "svelte-french-toast";
  import { Prose, Avatar, Button as FoxButton } from "@fuxui/base";
  import { PopoverEmojiPicker } from "@fuxui/social";
  import RelativeTime from "svelte-relative-time";

  type Props = {
    message: Message | Announcement;
    mergeWithPrevious?: boolean;
  };

  let { message, mergeWithPrevious = false }: Props = $props();

  let messageRepliedTo = derivePromise(undefined, async () => {
    if (g.roomy && message.replyTo) {
      return await g.roomy.open(Message, message.replyTo);
    }
  });
  let relatedThreads = derivePromise([], async () => {
    if (message instanceof Announcement) {
      return await message.relatedThreads.items();
    }
    return [];
  });
  let relatedMessages = derivePromise([], async () => {
    if (message instanceof Announcement) {
      return await message.relatedMessages.items();
    }
    return [];
  });

  let isMobile = $derived((outerWidth.current ?? 0) < 640);
  let isDrawerOpen = $state(false);

  let isSelected = $state(false);
  let isThreading: { value: boolean } = getContext("isThreading");
  let users: { value: Item[] } = getContext("users");
  let contextItems: { value: Item[] } = getContext("contextItems");

  // Editing state
  let isEditing = $state(false);
  let editMessageContent: JSONContent = $state({});

  let mayDelete = $derived(
    message.matches(Message) &&
      (g.isAdmin ||
        (user.agent &&
          message
            .forceCast(Message)
            .authors((x) => x.toArray().includes(user.agent!.assertDid)))),
  );

  let mayEdit = $derived(
    message.matches(Message) &&
      user.agent &&
      message
        .forceCast(Message)
        .authors((x) => x.toArray())
        .includes(user.agent?.assertDid),
  );

  const selectMessage = getContext("selectMessage") as (
    message: Message,
  ) => void;
  const removeSelectedMessage = getContext("removeSelectedMessage") as (
    message: Message,
  ) => void;
  const setReplyTo = getContext("setReplyTo") as (message: Message) => void;
  const scrollToMessage = getContext("scrollToMessage") as (
    id: EntityIdStr,
  ) => void;

  function deleteMessage() {
    message.softDeleted = true;
    message.commit();
  }

  function startEditing() {
    if (message instanceof Message) {
      try {
        // Parse the message body JSON to get a plain object
        const parsedContent = JSON.parse(message.bodyJson) as JSONContent;

        // Create a deep copy to ensure we're not working with a Proxy object
        // This keeps all content including images intact
        editMessageContent = JSON.parse(
          JSON.stringify(parsedContent),
        ) as JSONContent;

        isEditing = true;
      } catch (error) {
        console.error("Error starting message edit:", error);
        toast.error("Failed to edit message", { position: "bottom-end" });
      }
    }
  }

  function saveEditedMessage() {
    if (
      message instanceof Message &&
      Object.keys(editMessageContent).length > 0
    ) {
      try {
        // Ensure we're working with a plain object, not a Proxy
        const plainContent = JSON.parse(
          JSON.stringify(editMessageContent),
        ) as JSONContent;

        // Update the message
        message.bodyJson = JSON.stringify(plainContent);

        // Add an updatedDate field to track edits
        // @ts-ignore - Adding custom property for edit tracking
        message.updatedDate = new Date();

        message.commit();
        isEditing = false;
        toast.success("Message updated", { position: "bottom-end" });
      } catch (error) {
        console.error("Error saving edited message:", error);
        toast.error("Failed to save message", { position: "bottom-end" });
      }
    }
  }

  function cancelEditing() {
    isEditing = false;
    editMessageContent = {};
  }

  function isMessageEdited(msg: Message): boolean {
    // @ts-ignore - Check for custom property
    return !!msg.updatedDate;
  }

  function getEditedTime(msg: Message): string {
    // @ts-ignore - Access custom property
    if (msg.updatedDate) {
      // @ts-ignore - Access custom property
      return format(msg.updatedDate, "PPpp");
    }
    return "";
  }

  function pickedEmoji(emoji: string) {
    if (!user.agent) return;
    message.reactions.toggle(emoji, user.agent.assertDid);
    message.commit();
  }

  function updateSelect() {
    const m = message.tryCast(Message);
    if (isSelected) {
      m && selectMessage(m);
    } else {
      m && removeSelectedMessage(m);
    }
  }

  function toggleReaction(reaction: string) {
    if (!user.agent) return;
    message.reactions.toggle(reaction, user.agent.assertDid);
    message.commit();
  }

  function scrollToReply() {
    if (message.matches(Announcement) || !message.replyTo) {
      return;
    }
    scrollToMessage(message.replyTo);
  }

  $effect(() => {
    if (!isThreading.value) {
      isSelected = false;
    }
  });


  let shiftDown = $state(false);
  function onKeydown({ shiftKey }: KeyboardEvent) {
    shiftDown = shiftKey;
  }
  function onKeyup({ shiftKey }: KeyboardEvent) {
    shiftDown = shiftKey;
  }

  function getAnnouncementHtml(announcement: Announcement) {
    if (!g.space) return "";
    const schema = {
      type: "doc",
      content: [] as Record<string, unknown>[],
    } satisfies JSONContent;

    switch (announcement.kind) {
      case "threadCreated": {
        schema.content.push({
          type: "paragraph",
          content: [
            { type: "text", text: "A new thread has been created: " },
            {
              type: "channelThreadMention",
              attrs: {
                id: JSON.stringify({
                  id: relatedThreads.value[0]?.id,
                  space: g.space.id,
                  type: "thread",
                }),
                label: relatedThreads.value[0]?.name || "loading...",
              },
            },
          ],
        });
        break;
      }
      case "messageMoved": {
        schema.content.push({
          type: "paragraph",
          content: [
            { type: "text", text: "Moved to: " },
            {
              type: "channelThreadMention",
              attrs: {
                id: JSON.stringify({
                  id: relatedThreads.value[0]?.id,
                  space: g.space.id,
                  type: "thread",
                }),
                label: relatedThreads.value[0]?.name || "loading...",
              },
            },
          ],
        });
        break;
      }
      case "messageDeleted": {
        schema.content.push({
          type: "paragraph",
          content: [{ type: "text", text: "This message has been deleted" }],
        });
        break;
      }
    }

    return getContentHtml(schema);
  }

  function getPlainTextContent(content: JSONContent): string {
    try {
      if (!content) return "Edit message...";

      // Extract text from content
      let text = "";

      // Handle direct text content
      if (content.text) {
        text += content.text;
      }

      // Recursively extract text from content array
      if (content.content && Array.isArray(content.content)) {
        for (const node of content.content) {
          if (node.text) {
            text += `${node.text} `;
          } else if (node.content) {
            text += `${getPlainTextContent(node)} `;
          }
        }
      }

      return text.trim() || "Edit message...";
    } catch (error) {
      console.error("Error extracting plain text:", error);
      return "Edit message...";
    }
  }
</script>

<svelte:window onkeydown={onKeydown} onkeyup={onKeyup} />

<div id={message.id} class={`flex flex-col max-w-screen py-0`}>
  <div
    class={`relative group w-full h-fit flex flex-col gap-2 px-2 py-2 dark:hover:bg-white/2 hover:bg-base-200/30 rounded-2xl`}
  >
    {#if message instanceof Announcement}
      {@render announcementView(message)}
    {:else if message instanceof Message}
      {@render replyBanner()}
      {@render messageView(message)}
    {/if}

    {#if Object.keys(message.reactions.all()).length > 0}
      <div class="flex gap-2 flex-wrap pl-14">
        {#each Object.keys(message.reactions.all()) as reaction}
          {@render reactionToggle(reaction)}
        {/each}

        <PopoverEmojiPicker
          onpicked={(emoji) => {
            pickedEmoji(emoji.unicode);
          }}
        >
          {#snippet child({ props })}
            <FoxButton variant="ghost" size="icon" {...props}>
              <Icon icon="lucide:smile-plus" class="text-primary" />
            </FoxButton>
          {/snippet}
        </PopoverEmojiPicker>
      </div>
    {/if}
  </div>
</div>

{#snippet announcementView(announcement: Announcement)}
  {@const relatedMessage = relatedMessages.value[0]}
  {@render toolbar()}
  <div class="flex flex-col gap-4">
    {#if announcement.kind === "threadCreated"}
      <Button.Root
        onclick={() => {
          if (isMobile) {
            isDrawerOpen = true;
          }
        }}
        class="flex flex-col text-start gap-2 w-full min-w-0"
      >
        <section class="flex items-center gap-2 flex-wrap w-fit">
          {#if message.createdDate}
            {@render timestamp(message.createdDate)}
          {/if}
        </section>

        <Prose>
          <div
            class="text-sm italic min-w-0 max-w-full overflow-hidden text-ellipsis"
          >
            {@html getAnnouncementHtml(announcement)}
          </div>
        </Prose>
      </Button.Root>
    {:else if announcement.kind === "messageMoved"}
      {#if !mergeWithPrevious}
        <Button.Root
          onclick={() => {
            if (isMobile) {
              isDrawerOpen = true;
            }
          }}
          class="cursor-pointer flex gap-2 text-start w-full items-center text-info-content px-4 py-1 bg-info rounded-t text-base-900 dark:text-base-100"
        >
          <Icon icon="prime:reply" width="12px" height="12px" />
          <Prose>
            <div
              class="text-sm italic dz-prose-invert chat min-w-0 max-w-full overflow-hidden text-ellipsis"
            >
              {@html getAnnouncementHtml(announcement)}
            </div>
          </Prose>
          {#if message.createdDate}
            {@render timestamp(message.createdDate)}
          {/if}
        </Button.Root>
      {/if}
      <div
        class="ml-0 border-l-2 border-info pt-0.5 pl-2 border-base-200 dark:border-base-800"
      >
        {#if relatedMessage}
          {@render messageView(relatedMessage)}
        {/if}
      </div>
    {/if}
  </div>
{/snippet}

{#snippet messageView(msg: Message)}
  <!-- doesn't change after render, so $derived is not necessary -->
  {@const authorProfile = getProfile(msg.authors((x) => x.get(0)))}

  {#await authorProfile then authorProfile}
    {@render toolbar(authorProfile)}

    <div class="flex gap-4 group">
      {#if !mergeWithPrevious}
        <a
          href={`https://bsky.app/profile/${authorProfile.handle}`}
          title={authorProfile.handle}
          target="_blank"
        >
          <Avatar src={authorProfile.avatarUrl} />
        </a>
        <!-- {:else}
        <div class="w-11">
          {#if message.createdDate}
            <span
              class="opacity-0 text-[8px] text-base-900 dark:text-base-100 transition-opacity duration-200 whitespace-nowrap group-hover:opacity-100"
            >
              {format(message.createdDate, "pp")}
            </span>
          {/if}
        </div> -->
      {:else}
        <div class="w-11 h-1"></div>
      {/if}

      {#if isEditing && message === msg}
        <div class="flex flex-col w-full gap-2">
          {#if !mergeWithPrevious}
            <section class="flex items-center gap-2 flex-wrap w-fit">
              <a
                href={`https://bsky.app/profile/${authorProfile.handle}`}
                target="_blank"
                class="text-primary hover:underline"
              >
                <h5
                  class="font-bold text-accent-700 dark:text-accent-400"
                  title={authorProfile.handle}
                >
                  {authorProfile.displayName || authorProfile.handle}
                </h5>
              </a>
              <!-- {@render timestamp(message.createdDate || new Date())} -->
            </section>
          {/if}

          <div class="w-full">
            <ChatInput
              bind:content={editMessageContent}
              users={users.value || []}
              context={contextItems.value || []}
              onEnter={saveEditedMessage}
              placeholder={message instanceof Message
                ? getPlainTextContent(JSON.parse(message.bodyJson))
                : "Edit message..."}
              editMode={true}
            />
          </div>

          <div class="flex justify-end gap-2 mt-2">
            <FoxButton variant="secondary" size="sm" onclick={cancelEditing}>
              Cancel
            </FoxButton>
            <FoxButton variant="primary" size="sm" onclick={saveEditedMessage}>
              Save
            </FoxButton>
          </div>
        </div>
      {:else}
        <Button.Root
          onclick={() => {
            if (isMobile) {
              isDrawerOpen = true;
            }
          }}
          class="flex flex-col text-start gap-2 w-full min-w-0"
        >
          {#if !mergeWithPrevious}
            <section class="flex items-center gap-2 flex-wrap w-fit">
              <a
                href={`https://bsky.app/profile/${authorProfile.handle}`}
                target="_blank"
                class="text-accent-600 dark:text-accent-400 hover:underline"
              >
                <h5 class="font-bold" title={authorProfile.handle}>
                  {authorProfile.displayName || authorProfile.handle}
                </h5>
              </a>
              {@render timestamp(message.createdDate || new Date())}
            </section>
          {/if}

          <div class="flex flex-col gap-1">
            <div class="[&_img]:rounded-2xl">
              <Prose>
                {@html getContentHtml(JSON.parse(msg.bodyJson))}
              </Prose>
            </div>

            {#if isMessageEdited(msg)}
              <div class="relative group/edit">
                <span
                  class="text-xs text-base-500 italic flex items-center gap-1 hover:text-base-300 cursor-default"
                >
                  <Icon icon="mdi:pencil" width="12px" height="12px" />
                  <span>edited</span>
                </span>

                <!-- Tooltip that appears on hover -->
                <div
                  class="absolute bottom-full left-0 mb-2 opacity-0 group-hover/edit:opacity-100 transition-opacity duration-200 bg-base-300 p-3 rounded shadow-lg text-xs z-10 min-w-[200px]"
                >
                  <div class="flex flex-col gap-1">
                    <p class="font-semibold">Message edited</p>
                    <p>
                      Original: {format(msg.createdDate || new Date(), "PPpp")}
                    </p>
                    <p>Edited: {getEditedTime(msg)}</p>
                  </div>

                  <!-- Arrow pointing down -->
                  <div
                    class="absolute -bottom-1 left-3 w-2 h-2 bg-base-300 rotate-45"
                  ></div>
                </div>
              </div>
            {/if}
          </div>
          <!-- TODO: images. -->
          <!-- {#if msg.images?.length}
          <div class="flex flex-wrap gap-2 mt-2">
            {#each msg.images as image}
              <img
                src={image.source}
                alt={image.alt || ""}
                class="max-w-md max-h-64 rounded-lg object-cover"
                loading="lazy"
              />
            {/each}
          </div>
        {/if} -->
        </Button.Root>
      {/if}
    </div>
  {/await}
{/snippet}

{#snippet toolbar(authorProfile?: { handle: string; avatarUrl: string })}
  {#if !g.isBanned}
    {#if isMobile}
      <Drawer bind:isDrawerOpen>
        <div class="flex gap-4 justify-center mb-4">
          <FoxButton
            onclick={() => {
              toggleReaction("👍");
              isDrawerOpen = false;
            }}
            variant="ghost"
            size="icon"
          >
            👍
          </FoxButton>
          <FoxButton
            onclick={() => {
              toggleReaction("😂");
              isDrawerOpen = false;
            }}
            variant="ghost"
            size="icon"
          >
            😂
          </FoxButton>

          <PopoverEmojiPicker
            onpicked={(emoji) => {
              pickedEmoji(emoji.unicode);
            }}
          >
            {#snippet child({ props })}
              <FoxButton variant="ghost" size="icon" {...props}>
                <Icon icon="lucide:smile-plus" class="text-primary" />
              </FoxButton>
            {/snippet}
          </PopoverEmojiPicker>
        </div>

        {#if authorProfile}
          <div class="flex flex-col gap-2 w-full">
            {#if message instanceof Message}
              <FoxButton
                onclick={() => {
                  setReplyTo(message);
                  isDrawerOpen = false;
                }}
                class="dz-join-item dz-btn w-full"
              >
                <Icon icon="fa6-solid:reply" />
                Reply
              </FoxButton>
            {/if}
            {#if mayEdit}
              <FoxButton
                onclick={() => {
                  startEditing();
                  isDrawerOpen = false;
                }}
                class="dz-join-item dz-btn w-full"
              >
                <Icon icon="tabler:edit" />
                Edit
              </FoxButton>
            {/if}
            {#if mayDelete}
              <FoxButton
                onclick={() => deleteMessage()}
                class="dz-join-item dz-btn dz-btn-error w-full"
              >
                <Icon icon="tabler:trash" />
                Delete
              </FoxButton>
            {/if}
          </div>
        {/if}
      </Drawer>
    {:else if !isEditing}
      <Toolbar.Root
        class={`hidden group-hover:flex absolute gap-1 -top-2 right-0 bg-base-200 dark:bg-base-800 p-1.5 px-2 rounded-2xl items-center`}
      >
        <FoxButton
          variant="ghost"
          size="icon"
          onclick={() => toggleReaction("👍")}>👍</FoxButton
        >

        <FoxButton
          variant="ghost"
          size="icon"
          onclick={() => toggleReaction("😂")}>😂</FoxButton
        >

        <PopoverEmojiPicker
          onpicked={(emoji) => {
            pickedEmoji(emoji.unicode);
          }}
        >
          {#snippet child({ props })}
            <FoxButton variant="ghost" size="icon" {...props}>
              <Icon icon="lucide:smile-plus" class="text-primary" />
            </FoxButton>
          {/snippet}
        </PopoverEmojiPicker>
        {#if mayEdit}
          <FoxButton variant="ghost" size="icon" onclick={() => startEditing()}>
            <Icon icon="tabler:edit" />
          </FoxButton>
        {/if}

        {#if shiftDown && mayDelete}
          <FoxButton
            variant="ghost"
            size="icon"
            onclick={() => deleteMessage()}
          >
            <Icon icon="tabler:trash" color="red" />
          </FoxButton>
        {/if}

        {#if authorProfile && message instanceof Message}
          <FoxButton
            variant="ghost"
            size="icon"
            onclick={() => setReplyTo(message)}
          >
            <Icon icon="fa6-solid:reply" />
          </FoxButton>
        {/if}
      </Toolbar.Root>
    {/if}

    {#if isThreading.value && message instanceof Message}
      <Checkbox.Root
        onCheckedChange={updateSelect}
        bind:checked={isSelected}
        class="absolute right-4 inset-y-0"
      >
        {#snippet children({ checked }: { checked: boolean })}
          <div
            class="border border-primary bg-base-100 text-primary-content size-4 rounded items-center cursor-pointer"
          >
            {#if checked}
              <Icon
                icon="material-symbols:check-rounded"
                class="bg-primary size-3.5"
              />
            {/if}
          </div>
        {/snippet}
      </Checkbox.Root>
    {/if}
  {/if}
{/snippet}

{#snippet timestamp(date: Date)}
  <RelativeTime
    {date}
    locale="en"
    class="text-xs text-base-600 dark:text-base-400"
  />
{/snippet}

{#snippet reactionToggle(reaction: string)}
  {@const reactions = message.reactions.all()[reaction]}
  {#if reactions}
    {#await Promise.all([...reactions.values()].map( (x) => getProfile(x), )) then profilesThatReacted}
      <FoxButton
        onclick={() => toggleReaction(reaction)}
        variant="ghost"
        title={profilesThatReacted
          .map((x) => x.displayName || x.handle)
          .join(", ")}
      >
        {reaction}
        {message.reactions.all()[reaction]?.size}
      </FoxButton>
    {/await}
  {/if}
{/snippet}

{#snippet replyBanner()}
  {@const profileRepliedTo =
    messageRepliedTo.value &&
    getProfile(messageRepliedTo.value.authors((x) => x.get(0)))}
  {#await profileRepliedTo then profileRepliedTo}
    {#if messageRepliedTo.value && profileRepliedTo}
      <Button.Root
        onclick={scrollToReply}
        class="cursor-pointer flex flex-col gap-2 text-sm text-start w-full items-start text-base-900 dark:text-base-100 px-4 py-2 bg-base-900/20 rounded-2xl dark:bg-base-100/5"
      >
        <div class="flex basis-1/2 md:basis-auto gap-2 items-center">
          <Icon icon="prime:reply" width="12px" height="12px" />
          <Avatar class="size-4" src={profileRepliedTo.avatarUrl} />
          <h5
            class="text-accent-600 dark:text-accent-400 font-medium text-ellipsis"
          >
            {profileRepliedTo.handle}
          </h5>
        </div>
        <p class="line-clamp-1 basis-1/2 md:basis-auto overflow-hidden italic">
          {@html getContentHtml(JSON.parse(messageRepliedTo.value.bodyJson))}
        </p>
      </Button.Root>
    {/if}
  {/await}
{/snippet}
