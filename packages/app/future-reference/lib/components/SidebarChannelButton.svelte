<script lang="ts">
  import { page } from "$app/state";
  import {
    type Channel,
    type RoomyAccount,
    type Space,
    publicGroup,
    isSpaceAdmin,
  } from "@roomy-chat/sdk";
  import { navigate, navigateSync } from "$lib/utils.svelte";
  import Icon from "@iconify/svelte";
  import { Button } from "@fuxui/base";
  import { co, z } from "jazz-tools";

  let {
    channel,
    deleteItem,
    space,
    lastReadDate,
    me,
  }: {
    channel: co.loaded<typeof Channel> | undefined | null;
    deleteItem: (channel: co.loaded<typeof Channel>) => void;
    space: co.loaded<typeof Space> | undefined | null;
    lastReadDate: Date | undefined | null;
    me: co.loaded<typeof RoomyAccount> | undefined | null;
  } = $props();

  const latestEntriesByAccount = $derived(
    Object.values(channel?.mainThread?.timeline?.perAccount ?? {}).sort(
      (a, b) => a.madeAt.getTime() - b.madeAt.getTime(),
    ),
  );

  let isNew = $derived.by(() => {
    if (!lastReadDate) return latestEntriesByAccount.length !== 0;
    if (latestEntriesByAccount.length === 0) return false;
    let date = latestEntriesByAccount.at(-1)?.madeAt;
    if (!date) return false;

    return new Date(lastReadDate) < date;
  });

  const channelNotifications = $derived(
    me?.profile?.roomyInbox?.filter(
      (x) => x?.channelId === channel?.id && !x?.read,
    ).length,
  );

  // Cache the user ID to prevent excessive reactivity
  const myUserId = $derived(me?.id);
  const mySubscriptions = $derived(me?.profile?.threadSubscriptions);

  const channelThreads = $derived.by(() => {
    if (!myUserId || !channel?.subThreads) return [];

    // Use a stable filter to prevent layout shifts
    const filteredThreads = [];
    const subThreadsArray = Array.from(channel.subThreads); // Create stable array reference

    // Check if the current thread is in this channel
    const currentThreadId = page.params?.thread;
    const hasCurrentThread =
      currentThreadId && subThreadsArray.some((t) => t?.id === currentThreadId);

    if (currentThreadId) {
      console.log("🔍 Sidebar filtering for channel:", channel?.name, {
        channelId: channel?.id,
        currentThreadId,
        hasCurrentThread,
        totalSubThreads: subThreadsArray.length,
        threadIds: subThreadsArray.map((t) => t?.id).slice(0, 3), // First 3 for debugging
      });
    }

    for (const thread of subThreadsArray) {
      if (!thread || thread.softDeleted) continue;

      // Debug log for current thread
      if (thread.id === currentThreadId) {
        console.log("🔍 Processing current thread:", {
          threadId: thread.id,
          threadName: thread.name,
          channelName: channel?.name,
        });
      }

      // Check if explicitly unsubscribed
      try {
        const isUnsubscribed = mySubscriptions?.some?.(
          (subId) => subId === `unsubscribe:${thread.id}`,
        );
        if (isUnsubscribed) {
          console.log("❌ Thread unsubscribed:", thread.id);
          continue;
        }
      } catch (e) {
        // Ignore subscription check errors to prevent breaking the channel list
      }

      let shouldInclude = false;

      // Check if I created the thread
      const threadCreator = thread._edits?.name?.by?.id;
      const isCreator = threadCreator === myUserId;
      if (isCreator) {
        console.log("✅ Sidebar: Including thread (creator):", {
          threadId: thread.id,
          threadName: thread.name,
          threadCreator,
          myUserId,
          match: threadCreator === myUserId,
        });
        shouldInclude = true;
      } else if (thread.id === currentThreadId) {
        console.log(
          "❓ Sidebar: Current thread not included by creator check:",
          {
            threadId: thread.id,
            threadName: thread.name,
            threadCreator,
            myUserId,
            isMatch: threadCreator === myUserId,
            hasEdits: !!thread._edits,
            hasNameEdit: !!thread._edits?.name,
            hasBy: !!thread._edits?.name?.by,
          },
        );
      }

      // Check if I have viewed/read this thread
      const hasRead = me?.root?.lastRead?.[thread.id];
      if (!shouldInclude && hasRead) {
        console.log("✅ Sidebar: Including thread due to lastRead:", {
          threadId: thread.id,
          threadName: thread.name,
          lastReadTime: hasRead,
        });
        shouldInclude = true;
      }

      // Check if I sent any messages in this thread
      if (!shouldInclude) {
        const threadTimeline = thread.timeline;
        if (threadTimeline?.perAccount) {
          const myMessages = threadTimeline.perAccount[myUserId];
          if (myMessages && myMessages.length > 0) {
            shouldInclude = true;
          } else {
            // Also check all timeline entries for my account ID
            const allEntries = Object.values(threadTimeline.perAccount);
            for (const accountEntries of allEntries) {
              if (accountEntries.by?.id === myUserId) {
                shouldInclude = true;
                break;
              }
            }
          }
        }
      }

      if (shouldInclude) {
        filteredThreads.push(thread);
      }
    }

    return filteredThreads;
  });

  function unsubscribeFromThread(threadId: string) {
    try {
      if (!me?.profile) return;

      if (!me.profile.threadSubscriptions) {
        me.profile.threadSubscriptions = co
          .list(z.string())
          .create([], publicGroup("writer"));
      }
      me.profile.threadSubscriptions.push(`unsubscribe:${threadId}`);
    } catch (e) {
      console.error("Failed to unsubscribe from thread:", e);
    }
  }

  const threadNotifications = $derived.by(() => {
    if (!channelThreads) return 0;
    return channelThreads.reduce((total, thread) => {
      const threadNotifs =
        me?.profile?.roomyInbox?.filter(
          (x) => x?.threadId === thread?.id && !x?.read,
        ).length || 0;
      return total + threadNotifs;
    }, 0);
  });

  const totalNotifications = $derived(
    (channelNotifications || 0) + (threadNotifications || 0),
  );
</script>

{#if channel && !channel.softDeleted}
  <!-- Channel -->
  <div class="group flex items-center gap-1 relative">
    <!-- <Button.Root
      href={navigateSync({
        space: page.params.space!,
        channel: channel.id,
      })}
      class="flex-1 cursor-pointer px-1 dz-btn dz-btn-ghost justify-start border {channel.id === page.params.channel
        ? 'border-primary text-primary'
        : 'border-transparent'}"
    > -->

    <Button
      href={navigateSync({
        space: page.params.space!,
        channel: channel.id,
      })}
      variant="ghost"
      class="w-full justify-start"
      data-current={channel.id === page.params.channel}
    >
      <Icon
        icon={channel.channelType === "feeds"
          ? "basil:feed-outline"
          : channel.channelType === "links"
            ? "basil:link-outline"
            : "basil:comment-solid"}
        class="shrink-0"
      />
      <span class="truncate">{channel?.name || "..."}</span>
      {#if channel.channelType === "feeds"}
        <span
          class="text-xs bg-primary/20 text-primary px-1 py-0.5 rounded shrink-0"
          >FEEDS</span
        >
      {:else if channel.channelType === "links"}
        <span
          class="text-xs bg-secondary/20 text-secondary px-1 py-0.5 rounded shrink-0"
          >LINKS</span
        >
      {/if}

      {#if totalNotifications}
        <span
          class="inline-flex items-center justify-center bg-primary font-bold text-base-100 rounded-full size-6"
        >
          {totalNotifications}
        </span>
      {/if}
    </Button>
    <!-- </Button.Root> -->

    {#if isNew}
      <div class="absolute top-1 left-1 size-2 bg-primary rounded-full"></div>
    {/if}
    {#if isSpaceAdmin(space)}
      <Button
        title="Delete"
        class="opacity-0 group-hover:opacity-100 transition-opacity cursor-pointer dz-btn dz-btn-ghost dz-btn-circle text-error hover:bg-error/10"
        onclick={() => deleteItem(channel)}
      >
        <Icon icon="lucide:x" class="size-4" />
      </Button>
    {/if}
  </div>

  <!-- Threads under channel -->
  <div
    class="ml-6 overflow-hidden"
    style="min-height: 0; max-height: {channelThreads.length > 0
      ? 'none'
      : '0'}; transition: max-height 200ms ease-in-out;"
  >
    {#if channelThreads.length > 0}
      <div class="flex flex-col gap-1">
        {#each channelThreads as thread}
          {#if thread}
            {@const threadNotifs = me?.profile?.roomyInbox?.filter(
              (x) => x?.threadId === thread.id && !x?.read,
            ).length}
            {@const isThreadNew = (() => {
              const lastRead = me?.root?.lastRead?.[thread.id];
              if (!lastRead) return false;
              const latestEntries = Object.values(
                thread?.timeline?.perAccount ?? {},
              ).sort((a, b) => a.madeAt.getTime() - b.madeAt.getTime());
              if (latestEntries.length === 0) return false;
              const latestDate = latestEntries.at(-1)?.madeAt;
              return latestDate ? new Date(lastRead) < latestDate : false;
            })()}
            <div class="group flex items-center gap-1 relative">
              <Button
                variant="ghost"
                data-current={thread.id === page.params.thread}
                title={thread.name}
                onclick={() => {
                  navigate({
                    space: page.params.space!,
                    thread: thread.id,
                  });
                }}
                class="flex-1 cursor-pointer px-1 dz-btn dz-btn-ghost justify-start border {thread.id ===
                page.params.thread
                  ? 'border-primary text-primary'
                  : 'border-transparent'} group-hover:pr-7"
              >
                <h4
                  class="flex justify-start items-center w-full gap-2 px-2 text-sm"
                >
                  <Icon icon="tabler:hash" class="shrink-0" />
                  <span class="truncate"
                    >{thread.name?.replace(/^💬\s*/, "") || "..."}</span
                  >

                  {#if threadNotifs}
                    <span
                      class="inline-flex items-center justify-center bg-primary font-bold text-base-100 rounded-full size-5 text-xs shrink-0 ml-auto"
                    >
                      {threadNotifs}
                    </span>
                  {/if}
                </h4>
              </Button>

              <Button
                title="Unsubscribe from thread"
                class="absolute right-0 top-0 bottom-0 w-6 opacity-0 group-hover:opacity-100 transition-opacity cursor-pointer flex items-center justify-center text-error hover:bg-error/10 rounded"
                onclick={(e) => {
                  e.stopPropagation();
                  unsubscribeFromThread(thread.id);
                }}
              >
                <Icon icon="tabler:x" class="size-3" />
              </Button>

              {#if isThreadNew}
                <div
                  class="absolute top-1 left-1 size-2 bg-primary rounded-full"
                ></div>
              {/if}
            </div>
          {/if}
        {/each}
      </div>
    {/if}
  </div>
{/if}
