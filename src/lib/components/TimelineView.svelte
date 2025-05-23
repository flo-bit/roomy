<script lang="ts">
  import { getContext, setContext } from "svelte";
  import toast from "svelte-french-toast";
  import { user } from "$lib/user.svelte";
  import { getContentHtml, type Item } from "$lib/tiptap/editor";
  import { outerWidth } from "svelte/reactivity/window";

  import Icon from "@iconify/svelte";
  import ChatArea from "$lib/components/ChatArea.svelte";
  import ChatInput from "$lib/components/ChatInput.svelte";
  import AvatarImage from "$lib/components/AvatarImage.svelte";
  import { Button, Tabs } from "bits-ui";

  import { derivePromise } from "$lib/utils.svelte";
  import { g } from "$lib/global.svelte";
  import {
    Announcement,
    Channel,
    Message,
    Thread,
    Timeline,
    Category,
  } from "@roomy-chat/sdk";
  import type { JSONContent } from "@tiptap/core";
  import { getProfile } from "$lib/profile.svelte";
  import TimelineToolbar from "$lib/components/TimelineToolbar.svelte";
  import { Avatar } from "@fuxui/base";

  let isMobile = $derived((outerWidth.current ?? 0) < 640);

  let users: { value: Item[] } = getContext("users");
  let contextItems: { value: Item[] } = getContext("contextItems");

  let messageInput: JSONContent = $state({});

  // thread maker
  let isThreading = $state({ value: false });
  let threadTitleInput = $state("");
  let selectedMessages: Message[] = $state([]);
  setContext("isThreading", isThreading);
  setContext("selectMessage", (message: Message) => {
    selectedMessages.push(message);
  });
  setContext("removeSelectedMessage", (message: Message) => {
    selectedMessages = selectedMessages.filter((m) => m != message);
  });

  $effect(() => {
    if (!isThreading.value && selectedMessages.length > 0) {
      selectedMessages = [];
    }
  });

  // Reply Utils
  let replyingTo = $state() as Message | undefined;
  setContext("setReplyTo", (message: Message) => {
    replyingTo = message;
  });

  async function createThread(e: SubmitEvent) {
    e.preventDefault();
    if (!g.roomy || !g.space || !g.channel) return;

    const thread = await g.roomy.create(Thread);

    // messages can be selected in any order
    // sort them on create based on their position from the channel
    let channelMessageIds = g.channel.timeline.ids();
    selectedMessages.sort((a, b) => {
      return channelMessageIds.indexOf(a.id) - channelMessageIds.indexOf(b.id);
    });

    for (const message of selectedMessages) {
      // move selected message ID from channel to thread timeline
      thread.timeline.push(message);
      const index = g.channel.timeline.ids().indexOf(message.id);
      g.channel.timeline.remove(index);

      // create an Announcement about the move for each message
      const announcement = await g.roomy.create(Announcement);
      announcement.kind = "messageMoved";
      announcement.relatedMessages.push(message);
      announcement.relatedThreads.push(thread);
      announcement.commit();
      g.channel.timeline.insert(index, announcement);
    }

    // TODO: decide whether the thread needs a reference to it's original channel. That might be
    // confusing because it's messages could have come from multiple channels?
    thread.name = threadTitleInput;
    thread.commit();

    // create an Announcement about the new Thread in current channel
    const announcement = await g.roomy.create(Announcement);
    announcement.kind = "threadCreated";
    announcement.relatedThreads.push(thread);
    announcement.commit();

    g.channel.timeline.push(announcement);

    // If this is a channel ( the alternative would be a thread )
    if (g.channel instanceof Channel) {
      g.channel.threads.push(thread);
    }

    g.channel.commit();

    g.space.threads.push(thread);
    g.space.commit();

    threadTitleInput = "";
    isThreading.value = false;
    toast.success("Thread created", { position: "bottom-end" });
  }

  async function sendMessage() {
    if (!g.roomy || !g.space || !g.channel || !user.agent) return;

    // Image upload is now handled in ChatInput.svelte

    const message = await g.roomy.create(Message);
    message.authors(
      (authors) => user.agent && authors.push(user.agent.assertDid),
    );
    message.bodyJson = JSON.stringify(messageInput);
    message.createdDate = new Date();
    message.commit();
    if (replyingTo) message.replyTo = replyingTo;

    // Images are now handled by TipTap in the message content
    // Limit image size in message input to 300x300

    g.channel.timeline.push(message);
    g.channel.commit();

    messageInput = {};
    replyingTo = undefined;
  }

  //
  // Settings Dialog
  //

  // Image upload permissions are now handled in ChatInput.svelte
  let channelNameInput = $state("");
  let channelCategoryInput = $state(undefined) as undefined | string;
  $effect(() => {
    if (!g.space) return;

    channelNameInput = g.channel?.name || "";
    channelCategoryInput = undefined;
    g.space &&
      g.space.sidebarItems.items().then((items) => {
        for (const item of items) {
          const category = item.tryCast(Category);
          if (
            category &&
            typeof category.channels === "object" &&
            typeof category.channels.ids === "function" &&
            g.channel &&
            category.channels.ids().includes(g.channel.id)
          ) {
            channelCategoryInput = category.id;
            return;
          }
        }
      });
  });
  // Image upload is now handled in ChatInput.svelte
  let relatedThreads = derivePromise([], async () =>
    g.channel && g.channel instanceof Channel
      ? await g.channel.threads.items()
      : [],
  );
</script>

{#if g.space && g.channel}
  <ChatArea timeline={g.channel.forceCast(Timeline)} />
  <div class="flex items-center">
    {#if !isMobile || !isThreading.value}
      <section class="grow flex flex-col">
        {#if replyingTo}
          <div
            class="flex justify-between bg-secondary text-secondary-content rounded-t-lg px-4 py-2"
          >
            <div class="flex flex-col gap-1">
              <h5
                class="flex gap-2 items-center text-base-700 dark:text-base-400 text-sm"
              >
                Replying to
                {#await getProfile(replyingTo.authors( (x) => x.get(0), )) then profile}
                  <Avatar src={profile.avatarUrl} class="size-5" />
                  <strong class="text-accent-700 dark:text-accent-400"
                    >{profile.handle}</strong
                  >
                {/await}
              </h5>
              <p
                class="text-base-900 dark:text-base-100 text-ellipsis italic pb-2"
              >
                {@html getContentHtml(JSON.parse(replyingTo.bodyJson))}
              </p>
            </div>
            <Button.Root
              type="button"
              onclick={() => (replyingTo = undefined)}
              class="dz-btn dz-btn-circle dz-btn-ghost"
            >
              <Icon icon="zondicons:close-solid" />
            </Button.Root>
          </div>
        {/if}
        <div class="relative">
          <!-- TODO: get all users that has joined the server -->
          {#if g.roomy && g.roomy.spaces.ids().includes(g.space.id)}
            <ChatInput
              bind:content={messageInput}
              users={users.value}
              context={contextItems.value}
              onEnter={sendMessage}
            />
          {:else}
            <Button.Root
              class="w-full dz-btn"
              onclick={() => {
                if (g.space && g.roomy) {
                  g.roomy.spaces.push(g.space);
                  g.roomy.commit();
                }
              }}>join Space To Chat</Button.Root
            >
          {/if}

          <!-- Image upload button is now in ChatInput.svelte -->
        </div>
      </section>
    {/if}
  </div>
{/if}
