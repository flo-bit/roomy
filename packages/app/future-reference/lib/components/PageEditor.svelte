<script lang="ts">
  import { onMount, getContext, tick } from "svelte";
  import { codeToHtml } from "shiki";
  import { toast } from "svelte-french-toast";
  import { Button } from "bits-ui";
  import { page } from "$app/state";
  import Icon from "@iconify/svelte";
  import Link from "@tiptap/extension-link";
  import { BlockNoteEditor } from "@blocknote/core";
  import "@blocknote/core/style.css";
  // import { page } from "$app/state";
  import { focusOnRender } from "$lib/actions/useFocusOnRender.svelte";
  import Dialog from "$lib/components/Dialog.svelte";
  import { outerWidth } from "svelte/reactivity/window";
  import { CoState } from "jazz-tools/svelte";
  import { Page, Space, isSpaceAdmin } from "@roomy-chat/sdk";
  let isMobile = $derived((outerWidth.current ?? 0) < 640);

  const pg = $derived(new CoState(Page, page.params.page));

  const space = $derived(new CoState(Space, page.params.space));

  let isEditingPage = $state(false);

  let pageRenderedHtml = $state("");
  let processedHtml = $state("");
  let editorElement: HTMLElement | null = $state(null);
  let editor: BlockNoteEditor | null;

  interface UserItem {
    value: string;
    label: string;
    category: string;
  }

  let users = getContext<{ value: UserItem[] }>("users");

  let slashMenuVisible = $state(false);
  let slashMenuPosition = $state({ x: 0, y: 0 });
  let slashCommands = $state([
    {
      name: "Heading 1",
      icon: "tabler:h-1",
      action: () =>
        editor?.updateBlock(editor?.getTextCursorPosition()?.block.id, {
          type: "heading",
          props: { level: 1 },
        }),
    },
    {
      name: "Heading 2",
      icon: "tabler:h-2",
      action: () =>
        editor?.updateBlock(editor?.getTextCursorPosition()?.block.id, {
          type: "heading",
          props: { level: 2 },
        }),
    },
    {
      name: "Heading 3",
      icon: "tabler:h-3",
      action: () =>
        editor?.updateBlock(editor?.getTextCursorPosition()?.block.id, {
          type: "heading",
          props: { level: 3 },
        }),
    },
    {
      name: "Bulleted List",
      icon: "tabler:list",
      action: () =>
        editor?.updateBlock(editor?.getTextCursorPosition()?.block.id, {
          type: "bulletListItem",
        }),
    },
    {
      name: "Numbered List",
      icon: "tabler:list-numbers",
      action: () =>
        editor?.updateBlock(editor?.getTextCursorPosition()?.block.id, {
          type: "numberedListItem",
        }),
    },
    {
      name: "Checklist",
      icon: "tabler:checkbox",
      action: () =>
        editor?.updateBlock(editor?.getTextCursorPosition()?.block.id, {
          type: "checkListItem",
        }),
    },
    {
      name: "Code Block",
      icon: "tabler:code",
      action: () =>
        editor?.updateBlock(editor?.getTextCursorPosition()?.block.id, {
          type: "codeBlock",
        }),
    },
  ]);

  let mentionMenuVisible = $state(false);
  let mentionMenuPosition = $state({ x: 0, y: 0 });
  let filteredUsers = $state<any[]>([]);
  let mentionQuery = $state("");

  let hashMenuVisible = $state(false);
  let hashMenuPosition = $state({ x: 0, y: 0 });
  let filteredItems = $state<
    { id: string; name: string; type: "thread" | "channel" }[]
  >([]);
  // Commenting out for now because it is unused, but we might want to use it somewhere later.
  // let hashQuery = $state("");

  let selectionTooltipVisible = $state(false);
  let selectionTooltipPosition = $state({ x: 0, y: 0 });
  let formatCommands = $state([
    {
      name: "Bold",
      icon: "tabler:bold",
      action: () => editor?.toggleStyles({ bold: true }),
    },
    {
      name: "Italic",
      icon: "tabler:italic",
      action: () => editor?.toggleStyles({ italic: true }),
    },
    {
      name: "Underline",
      icon: "tabler:underline",
      action: () => editor?.toggleStyles({ underline: true }),
    },
    {
      name: "Strike",
      icon: "tabler:strikethrough",
      action: () => editor?.toggleStyles({ strike: true }),
    },
    { name: "Link", icon: "tabler:link", action: () => addLinkToSelection() },
  ]);

  let isUrlPromptDialogOpen = $state(false);
  let urlInputElement: HTMLInputElement | null = $state(null);
  let urlPromptCallback: ((url: string) => void) | null = $state(null);

  let isDeleteDialogOpen = $state(false);

  function setEditingPage(value: boolean) {
    isEditingPage = value;

    if (value) {
      if (editor) {
        editor = null;
      }
      setTimeout(initBlockNoteEditor, 100);
    } else if (editor) {
      editor = null;
    }
  }

  function showDeleteDialog(event: Event) {
    event.stopPropagation();
    isDeleteDialogOpen = true;
  }

  function confirmDeletePage() {
    if (!pg.current) return;
    pg.current.softDeleted = true;

    isEditingPage = false; // Close the editor to remove cached page
    isDeleteDialogOpen = false;
  }

  const EditorHandler = (editor: BlockNoteEditor) => {
    try {
      const cursorPosition = editor.getTextCursorPosition();
      if (!cursorPosition) return;

      const block = cursorPosition.block;
      const content = Array.isArray(block.content) ? block.content[0] : null;

      if (content && content.type === "text") {
        // Handle slash commands
        if (content.text === "/") {
          slashMenuVisible = true;
          const { top, left } = editor.getSelectionBoundingBox()?.toJSON();
          const menuHeight = 300;
          const viewportHeight = window.innerHeight;
          const scrollY = window.scrollY;
          const wouldOverflow =
            top + 20 + menuHeight > viewportHeight + scrollY;
          const yPosition = wouldOverflow
            ? Math.max(scrollY, top - menuHeight - 10)
            : top + 20;
          slashMenuPosition = { x: left, y: yPosition };
          mentionMenuVisible = false;
          hashMenuVisible = false;
        }
        // Handle mentions
        else if (content.text.endsWith("@")) {
          mentionMenuVisible = true;
          mentionQuery = "";
          const { top, left } = editor.getSelectionBoundingBox()?.toJSON();
          const menuHeight = 300;
          const viewportHeight = window.innerHeight;
          const scrollY = window.scrollY;
          const wouldOverflow =
            top + 20 + menuHeight > viewportHeight + scrollY;
          const yPosition = wouldOverflow
            ? Math.max(scrollY, top - menuHeight - 10)
            : top + 20;
          mentionMenuPosition = { x: left, y: yPosition };
          updateFilteredUsers();
          slashMenuVisible = false;
          hashMenuVisible = false;
        }
        // Handle hashtags
        else if (content.text.endsWith("#")) {
          hashMenuVisible = true;
          // hashQuery = "";
          const { top, left } = editor.getSelectionBoundingBox()?.toJSON();
          const menuHeight = 300;
          const viewportHeight = window.innerHeight;
          const scrollY = window.scrollY;
          const wouldOverflow =
            top + 20 + menuHeight > viewportHeight + scrollY;
          const yPosition = wouldOverflow
            ? Math.max(scrollY, top - menuHeight - 10)
            : top + 20;
          hashMenuPosition = { x: left, y: yPosition };
          slashMenuVisible = false;
          mentionMenuVisible = false;
        }
        // Check if typing in an active mention
        else if (mentionMenuVisible) {
          const text = content.text;
          const atIndex = text.lastIndexOf("@");

          if (atIndex !== -1) {
            mentionQuery = text.substring(atIndex + 1);
          } else {
            mentionMenuVisible = false;
          }
        }
        // Check if typing in an active hashtag
        else if (hashMenuVisible) {
          const text = content.text;
          const hashIndex = text.lastIndexOf("#");

          if (hashIndex !== -1) {
            // hashQuery = text.substring(hashIndex + 1);
          } else {
            hashMenuVisible = false;
          }
        } else {
          slashMenuVisible = false;
          mentionMenuVisible = false;
          hashMenuVisible = false;
        }
      } else {
        slashMenuVisible = false;
        mentionMenuVisible = false;
        hashMenuVisible = false;
      }
    } catch (e) {
      console.error("Error in EditorHandler:", e);
      slashMenuVisible = false;
      mentionMenuVisible = false;
      hashMenuVisible = false;
    }
  };

  const SelectionHandler = (editor: BlockNoteEditor) => {
    try {
      const selection = editor.getSelection();
      if (selection) {
        const rect = editor.getSelectionBoundingBox()?.toJSON();
        if (rect) {
          selectionTooltipPosition = {
            x: rect.left + rect.width / 2,
            y: rect.top - 50,
          };
          selectionTooltipVisible = true;
        }
      } else {
        selectionTooltipVisible = false;
      }
    } catch (e) {
      console.error("Error in SelectionHandler:", e);
      selectionTooltipVisible = false;
    }
  };

  function executeSlashCommand(command: {
    name?: string;
    icon?: string;
    action: any;
  }) {
    if (!editor) return;
    const pos = editor.getTextCursorPosition();
    command.action();
    editor.updateBlock(pos.block.id, { content: [] });
    editor.setTextCursorPosition(pos.block.id, "start");
    slashMenuVisible = false;
  }

  function executeFormatCommand(command: {
    name?: string;
    icon?: string;
    action: any;
  }) {
    if (!editor) return;
    command.action();
  }

  // Filter users based on mention query
  function updateFilteredUsers() {
    if (!users?.value?.length) {
      filteredUsers = [];
      return;
    }

    const query = mentionQuery.toLowerCase();
    filteredUsers = users.value
      .filter((user) => user.label?.toLowerCase().includes(query))
      .slice(0, 10); // Limit to 10 results
  }

  // Insert user mention
  function insertUserMention(user: UserItem) {
    if (!editor) return;

    try {
      const cursorPosition = editor.getTextCursorPosition();
      if (!cursorPosition) return;

      const block = cursorPosition.block;
      const content = Array.isArray(block.content) ? block.content[0] : null;

      if (content && content.type === "text") {
        const text = content.text;
        const atIndex = text.lastIndexOf("@");

        if (atIndex !== -1) {
          const newText = text.substring(0, atIndex) + `@${user.label}`;

          editor.updateBlock(block.id, {
            content: [
              {
                type: "text",
                text: newText,
                styles: {
                  backgroundColor: "#8b5cf6",
                },
              },
            ],
          });
          editor.insertBlocks(
            [{ type: "paragraph", content: "" }],
            block.id,
            "after",
          );
          editor.setSelection(
            block.id,
            editor.getNextBlock(block.id)?.id ?? block.id,
          );
          editor.createLink(
            user.label.startsWith("https://")
              ? user.label
              : `https://${user.label}`,
          );

          editor.setTextCursorPosition(block.id, "end");
        }
      }

      mentionMenuVisible = false;
    } catch (e) {
      console.error("Error inserting mention:", e);
      mentionMenuVisible = false;
    }
  }

  function insertHashLink(item: {
    id: string;
    name: string;
    type: "thread" | "channel";
  }) {
    if (!editor) return;

    try {
      const cursorPosition = editor.getTextCursorPosition();
      if (!cursorPosition) return;

      const block = cursorPosition.block;
      const content = Array.isArray(block.content) ? block.content[0] : null;

      if (content && content.type === "text") {
        const text = content.text;
        const hashIndex = text.lastIndexOf("#");

        if (hashIndex !== -1) {
          const newText = text.substring(0, hashIndex) + `#${item.name}`;

          editor.updateBlock(block.id, {
            content: [
              {
                type: "text",
                text: newText,
                styles: {
                  backgroundColor: "#8b5cf6",
                  textColor: "white",
                },
              },
            ],
          });

          editor.insertBlocks(
            [{ type: "paragraph", content: "" }],
            block.id,
            "after",
          );

          editor.setSelection(
            block.id,
            editor.getNextBlock(block.id)?.id ?? block.id,
          );

          const linkId =
            item.type === "channel"
              ? `/${page.params.space}/${item.id}`
              : `/${page.params.space}/thread/${item.id}`;
          editor.createLink(linkId);

          editor.setTextCursorPosition(block.id, "end");
        }
      }

      hashMenuVisible = false;
    } catch (e) {
      console.error("Error inserting hash link:", e);
      hashMenuVisible = false;
    }
  }

  async function initBlockNoteEditor() {
    if (!editorElement) return;
    try {
      editorElement.innerHTML = "";
      editor = BlockNoteEditor.create({
        _extensions: {
          link: Link.configure({
            HTMLAttributes: {
              target: null,
            },
          }),
        },
      });

      editor.mount(editorElement);
      editor.onChange(EditorHandler);
      editor.onSelectionChange(SelectionHandler);
      editorElement.addEventListener("click", (e) => {
        const blockElement = (e.target as Element)?.closest(".bn-block");
        if (blockElement) {
          const blockRect = blockElement.getBoundingClientRect();
          if (e.clientX < blockRect.left + 24) {
            e.preventDefault();
            e.stopPropagation();
            const blockId = blockElement.getAttribute("data-id");
            if (!blockId) return;
            slashMenuPosition = { x: blockRect.left + 30, y: e.clientY };
            slashMenuVisible = true;
          }
        }
      });
      if (pg.current?.body) {
        try {
          console.log("wat", pg.current.body);
          const parsedContent = JSON.parse(pg.current.body);
          setTimeout(() => {
            if (editor && editor.document) {
              editor.replaceBlocks(editor.document, parsedContent);
            }
          }, 200);
        } catch (e) {
          console.error("Failed to parse page content", e);
        }
      }
    } catch (e) {
      console.error("Failed to initialize BlockNote editor", e);
    }
  }

  async function processCodeBlocks() {
    if (!pageRenderedHtml) {
      processedHtml = pageRenderedHtml;
      return;
    }

    try {
      // Process the HTML to find code blocks and apply syntax highlighting
      const parser = new DOMParser();
      const doc = parser.parseFromString(pageRenderedHtml, "text/html");

      // Find all code blocks with language class
      const codeBlocks = doc.querySelectorAll("pre code");

      for (const codeBlock of Array.from(codeBlocks)) {
        const languageMatch = codeBlock.className.match(/language-(\w+)/);
        if (!languageMatch) continue;

        const language = languageMatch[1] || "txt";
        const code = codeBlock.textContent || "";

        try {
          const highlightedCode = await codeToHtml(code, {
            lang: language,
            theme: "vitesse-dark",
          });

          // Replace the code block with highlighted version
          const preElement = codeBlock.parentElement;
          if (preElement) {
            const tempContainer = document.createElement("div");
            tempContainer.innerHTML = highlightedCode;

            const labelDiv = document.createElement("div");
            labelDiv.className = "code-language-label";
            labelDiv.textContent = formatLanguageName(language);

            const shikiElement = tempContainer.firstElementChild;
            if (shikiElement) {
              const wrapper = document.createElement("div");
              wrapper.className = "code-block-wrapper";
              wrapper.appendChild(labelDiv);
              wrapper.appendChild(shikiElement);

              preElement.replaceWith(wrapper);
            } else {
              preElement.replaceWith(tempContainer.firstElementChild!);
            }
          }
        } catch (e) {
          console.error(
            `Failed to highlight code for language ${language}:`,
            e,
          );
        }
      }

      processedHtml = doc.body.innerHTML;
    } catch (e) {
      console.error("Failed to process code blocks:", e);
      processedHtml = pageRenderedHtml;
    }
  }

  // Function to format language name for display
  function formatLanguageName(lang: string): string {
    const languageMap: Record<string, string> = {
      js: "JavaScript",
      ts: "TypeScript",
      jsx: "React JSX",
      tsx: "React TSX",
      html: "HTML",
      css: "CSS",
      scss: "SCSS",
      sass: "Sass",
      py: "Python",
      rb: "Ruby",
      rs: "Rust",
      go: "Go",
      java: "Java",
      kt: "Kotlin",
      cs: "C#",
      cpp: "C++",
      c: "C",
      php: "PHP",
      sh: "Shell",
      bash: "Bash",
      zsh: "Zsh",
      sql: "SQL",
      json: "JSON",
      yml: "YAML",
      yaml: "YAML",
      md: "Markdown",
      svelte: "Svelte",
    };

    return languageMap[lang] || lang.charAt(0).toUpperCase() + lang.slice(1);
  }

  $effect(() => {
    pageRenderedHtml;
    processCodeBlocks();
  });

  $effect(() => {
    isEditingPage;
    if (!pg.current?.body) return;
    if (pg.current.body == "{}") {
      pageRenderedHtml = "";
      processedHtml = "";
      return;
    }
    console.log("page body", pg.current.body);
    const json = JSON.parse(pg.current.body);
    try {
      const rendererEditor = BlockNoteEditor.create();
      rendererEditor
        .blocksToFullHTML(json)
        .then((html) => {
          pageRenderedHtml = html;
        })
        .catch(() => {
          pageRenderedHtml = "";
        });
    } catch (_e) {
      pageRenderedHtml = "";
    }
  });

  async function savePageContent() {
    if (!editor || !pg.current) return;
    try {
      const res = JSON.stringify(editor.document);
      pg.current.body = res;

      setEditingPage(false);
      toast.success("Page saved successfully", { position: "bottom-end" });
    } catch (e) {
      console.error("Failed to save page content", e);
      toast.error("Failed to save page content", { position: "bottom-end" });
    }
  }

  function addLinkToSelection() {
    if (!editor) return;
    const selection = editor.getSelection();
    if (!selection) {
      toast.error("Please select text to create a link", {
        position: "bottom-end",
      });
      return;
    }

    if (urlInputElement) {
      urlInputElement.value = "https://";
    }
    isUrlPromptDialogOpen = true;

    urlPromptCallback = (url) => {
      if (url) {
        try {
          if (!url.startsWith("http://") && !url.startsWith("https://")) {
            url = "https://" + url;
          }
          editor?.createLink(url);
          selectionTooltipVisible = false;
        } catch (e) {
          toast.error("Failed to add link", { position: "bottom-end" });
        }
      }
    };
  }

  function submitUrlPrompt() {
    if (urlPromptCallback && urlInputElement) {
      urlPromptCallback(urlInputElement.value);
      tick().then(() => {
        if (urlInputElement) {
          urlInputElement.value = ""; // Reset the input value after submission
        }
        urlPromptCallback = null;
        isUrlPromptDialogOpen = false;
      });
    }
  }

  $effect(() => {
    if (editor) {
      editor = null;
    }
  });

  onMount(async () => {
    // const existingLink = document.querySelector(
    //   'link[href*="@blocknote/core/style.css"]',
    // );
    // if (!existingLink) {
    //   const link = document.createElement("link");
    //   link.rel = "stylesheet";
    //   link.href = "https://unpkg.com/@blocknote/core@latest/style.css";
    //   document.head.appendChild(link);
    // }
    // if (pageRenderedHtml) {
    //   await processCodeBlocks();
    // }
  });
</script>

<header class="dz-navbar">
  <div class="dz-navbar-start flex w-full justify-between gap-4">
    <label
      for="sidebar-left"
      class="btn btn-ghost p-1 shrink-0 drawer-button sm:hidden"
      title="Open sidebar"
    >
      <Icon
        icon="material-symbols:book-outline"
        font-size="1.75rem"
        class="shrink-0"
      />
    </label>

    <div class="flex gap-2 items-center" class:dz-input={isEditingPage}>
      <Icon icon={"material-symbols:thread-unread-rounded"} />
      {#if !isEditingPage}
        <h4
          class={`${isMobile && "line-clamp-1 overflow-hidden text-ellipsis"} text-base-content text-lg font-bold flex gap-2 items-center`}
        >
          {pg.current?.name}
        </h4>
      {:else if pg.current?.name}
        <input
          type="text"
          bind:value={pg.current.name}
          class="text-base-content text-lg font-bold p-0 flex-1 mr-2"
          placeholder="Page title"
          required
        />
      {/if}
    </div>
  </div>
  <div class="flex gap-3">
    {#if !isEditingPage}
      <Button.Root
        onclick={() => setEditingPage(true)}
        class=" max-w-fit dz-btn dz-btn-primary"
      >
        <Icon icon="tabler:edit" />
        Edit
      </Button.Root>
    {:else}
      <div class="pl-3 flex gap-2 grow">
        <Button.Root
          onclick={() => setEditingPage(false)}
          class="dz-btn dz-btn-ghost"
        >
          Cancel
        </Button.Root>
        <Button.Root onclick={savePageContent} class="dz-btn dz-btn-primary">
          Save
        </Button.Root>
      </div>
    {/if}

    {#if isSpaceAdmin(space.current)}
      <button
        class="dz-btn dz-btn-error dz-delete-btn group-hover:block"
        onclick={showDeleteDialog}
      >
        <Icon icon="tabler:trash" />
      </button>
    {/if}
  </div>
</header>
<!-- If no page selected, show guidance -->
<div class="dz-divider"></div>

{#if !isEditingPage}
  <section class="page-content h-full overflow-y-auto">
    <div class="page-rendered p-4">
      <div class="page-html text-base-content">
        {#if pg.current && !pg.current.softDeleted}
          {@html processedHtml}
        {:else}
          <p class="text-base-content/70">No content available.</p>
        {/if}
      </div>
    </div>
  </section>
{:else}
  <section class="page-editor-container p-4 h-full overflow-y-auto">
    <div
      class="page-editor bg-base-300/20 rounded-lg border border-base-content/30 p-4 h-auto {isEditingPage
        ? 'edit-mode'
        : ''}"
    >
      <div
        class="permanent-formatting-toolbar bg-base-300 border border-base-content/20 rounded-lg shadow-lg p-1 mb-4 flex items-center"
      >
        {#each formatCommands as command}
          <button
            class="btn btn-ghost btn-square btn-sm"
            title={command.name}
            onclick={() => executeFormatCommand(command)}
          >
            <Icon icon={command.icon} class="text-xl" />
          </button>
        {/each}
        <div class="ml-2 border-l border-base-content/20 h-6"></div>
        {#each slashCommands as command, i}
          {#if i < 3}
            <button
              class="btn btn-ghost btn-square btn-sm"
              title={command.name}
              onclick={() => executeSlashCommand(command)}
            >
              <Icon icon={command.icon} class="text-xl" />
            </button>
          {/if}
        {/each}
      </div>

      <div bind:this={editorElement} class="min-h-[400px]"></div>

      {#if slashMenuVisible && isEditingPage}
        <div
          class="slash-menu bg-base-300 border border-base-content/20 rounded shadow-lg absolute z-50"
          style="left: {slashMenuPosition.x}px; top: {slashMenuPosition.y}px;"
        >
          <ul class="py-1">
            {#each slashCommands as command}
              <li>
                <button
                  class="flex items-center gap-2 w-full text-left px-4 py-2 hover:bg-base-200 text-base-content"
                  onclick={() => executeSlashCommand(command)}
                >
                  <Icon icon={command.icon} class="text-xl" />
                  <span>{command.name}</span>
                </button>
              </li>
            {/each}
          </ul>
        </div>
      {/if}

      {#if mentionMenuVisible && isEditingPage}
        <div
          class="mention-menu bg-base-300 border border-base-content/20 rounded shadow-lg absolute z-50"
          style="left: {mentionMenuPosition.x}px; top: {mentionMenuPosition.y}px;"
        >
          <div class="py-1 max-h-[300px] overflow-y-auto min-w-[200px]">
            {#if filteredUsers.length > 0}
              <ul>
                {#each filteredUsers as user}
                  <li>
                    <button
                      class="flex items-center gap-2 w-full text-left px-4 py-2 hover:bg-base-200 text-base-content"
                      onclick={() => insertUserMention(user)}
                    >
                      <div
                        class="w-6 h-6 rounded-full bg-primary flex items-center justify-center overflow-hidden"
                      >
                        <span class="text-xs font-bold text-primary-content"
                          >{user.label[0]?.toUpperCase() || "?"}</span
                        >
                      </div>
                      <span>{user.label}</span>
                    </button>
                  </li>
                {/each}
              </ul>
            {:else}
              <div class="px-4 py-2 text-base-content/70">No users found</div>
            {/if}
          </div>
        </div>
      {/if}

      {#if hashMenuVisible && isEditingPage}
        <div
          class="hash-menu bg-base-300 border border-base-content/20 rounded shadow-lg absolute z-50"
          style="left: {hashMenuPosition.x}px; top: {hashMenuPosition.y}px;"
        >
          <div class="py-1 max-h-[300px] overflow-y-auto min-w-[250px]">
            {#if filteredItems.length > 0}
              <ul>
                {#each filteredItems as item}
                  <li>
                    <button
                      class="flex items-center gap-2 w-full text-left px-4 py-2 hover:bg-base-200 text-base-content"
                      onclick={() => insertHashLink(item)}
                    >
                      <div
                        class="w-6 h-6 rounded flex items-center justify-center overflow-hidden"
                      >
                        <Icon
                          icon={item.type === "channel"
                            ? "tabler:hash"
                            : "tabler:message-circle"}
                          class="text-lg text-primary"
                        />
                      </div>
                      <span>{item.name}</span>
                      <span class="text-xs text-base-content/70 ml-auto"
                        >{item.type}</span
                      >
                    </button>
                  </li>
                {/each}
              </ul>
            {:else}
              <div class="px-4 py-2 text-base-content/70">
                No channels or threads found
              </div>
            {/if}
          </div>
        </div>
      {/if}

      {#if selectionTooltipVisible && isEditingPage}
        <div
          class="tooltip-animate bg-base-300 border border-base-content/20 rounded shadow-lg absolute z-50 flex items-center justify-center p-1"
          style="left: {selectionTooltipPosition.x}px; top: {selectionTooltipPosition.y}px; transform: translateX(-50%);"
        >
          {#each formatCommands as command}
            <button
              class="btn btn-ghost btn-square btn-sm"
              title={command.name}
              onclick={() => {
                command.action();
                selectionTooltipVisible = false;
              }}
            >
              <Icon icon={command.icon} />
            </button>
          {/each}
        </div>
      {/if}
    </div>
  </section>
{/if}

<Dialog
  title="Add Link"
  description="Embed link into text"
  bind:isDialogOpen={isUrlPromptDialogOpen}
>
  <form onsubmit={submitUrlPrompt} class="flex flex-col gap-4">
    <input
      type="text"
      bind:this={urlInputElement}
      use:focusOnRender
      placeholder="https://example.com"
      class="dz-input dz-bordered w-full"
      required
    />
    <div class="flex justify-end gap-3 mt-2">
      <button type="submit" class="dz-btn dz-btn-primary">Add Link</button>
    </div>
  </form>
</Dialog>

<Dialog
  title="Confirm Page Deletion"
  description="Are you sure you want to delete <b>{pg.current
    ?.name}</b>?</br></br><b>Note:</b> Deletes are not permanent and only hide the data from view. The data is still publicly accessible."
  bind:isDialogOpen={isDeleteDialogOpen}
>
  <div class="flex justify-end gap-3">
    <button class="dz-btn dz-btn-error" onclick={confirmDeletePage}
      >Delete</button
    >
  </div>
</Dialog>

<style>
  :global(.bn-block) {
    position: relative;
    transition: padding-left 0.15s ease;
    padding-left: 24px;
  }

  :global(.page-editor-container .edit-mode .bn-block::before) {
    content: "+";
    position: absolute;
    left: 6px;
    top: 50%;
    transform: translateY(-50%);
    color: hsl(var(--p));
    font-size: 16px;
    font-weight: bold;
    width: 18px;
    height: 18px;
    border-radius: 3px;
    display: flex;
    align-items: center;
    justify-content: center;
    opacity: 0;
    transition:
      opacity 0.15s ease,
      background-color 0.15s ease;
    cursor: pointer;
    z-index: 10;
    background-color: transparent;
  }

  :global(.page-editor-container .edit-mode .bn-block:hover::before) {
    opacity: 1;
  }

  :global(.page-editor-container .edit-mode .bn-block::before:hover) {
    background-color: hsl(var(--p) / 0.2);
  }

  :global(.page-editor) {
    color: hsl(var(--bc));
  }

  .slash-menu,
  .hash-menu,
  .mention-menu {
    min-width: 200px;
    max-width: 350px;
    max-height: 300px;
    overflow-y: auto;
    z-index: 100;
  }

  .page-editor {
    overflow: visible;
  }

  :global(.page-rendered input[type="checkbox"]) {
    pointer-events: none;
  }

  :global([type="checkbox"]) {
    pointer-events: auto;
    cursor: pointer;
  }

  :global(.bn-inline-content a) {
    color: hsl(var(--p));
    text-decoration: underline;
    cursor: pointer;
  }

  :global(input[type="checkbox"]) {
    accent-color: hsl(var(--p));
    width: 16px;
    height: 16px;
    margin-right: 8px;
  }

  .tooltip-animate {
    animation: tooltipFadeIn 0.2s ease;
  }

  @keyframes tooltipFadeIn {
    from {
      opacity: 0;
      transform: translateY(5px) translateX(-50%);
    }
    to {
      opacity: 1;
      transform: translateY(0) translateX(-50%);
    }
  }

  .permanent-formatting-toolbar {
    position: sticky;
    top: 0;
    z-index: 20;
    display: flex;
    flex-wrap: wrap;
    gap: 2px;
  }

  :global(.shiki) {
    margin: 0;
    padding: 1em;
    border-radius: 0.5em;
    overflow-x: auto;
    font-family: "Fira Code", monospace;
    font-size: 0.9em;
    line-height: 1.6;
    tab-size: 2;
  }

  :global(.shiki code) {
    display: block;
    padding: 0.5em;
    background: transparent;
  }

  :global(.page-rendered pre) {
    margin: 1em 0;
    border-radius: 0.5em;
    overflow: hidden;
    background-color: hsl(var(--n));
  }

  :global(.page-rendered code:not([class])) {
    padding: 0.2em 0.4em;
    margin: 0;
    background-color: hsl(var(--p) / 0.2);
    border-radius: 3px;
    font-family: "Fira Code", monospace;
    font-size: 0.85em;
  }

  :global(.code-block-wrapper) {
    position: relative;
    margin-top: 0.5em;
    width: inherit;
  }

  :global(.code-language-label) {
    position: absolute;
    top: 0;
    right: 0;
    background-color: hsl(var(--p) / 0.7);
    color: hsl(var(--pc));
    padding: 2px 8px;
    font-size: 0.75rem;
    border-bottom-left-radius: 4px;
    font-family: "Fira Code", monospace;
    z-index: 10;
    user-select: none;
  }

  :global(.bn-inline-content [data-mention]) {
    color: hsl(var(--p));
    background-color: hsl(var(--p) / 0.1);
    padding: 0 2px;
    border-radius: 2px;
    font-weight: 500;
  }

  :global(
    .bn-inline-content a[href^="channel-"],
    .bn-inline-content a[href^="thread/"],
    .bn-inline-content a[href^="thread-"]
  ) {
    color: hsl(var(--p)) !important;
    font-weight: 500;
    text-decoration: none !important;
    padding: 0 2px;
    background-color: hsl(var(--p) / 0.1);
    border-radius: 3px;
  }

  :global(
    .bn-inline-content a[href^="channel-"]:hover,
    .bn-inline-content a[href^="thread/"]:hover,
    .bn-inline-content a[href^="thread-"]:hover
  ) {
    background-color: hsl(var(--p) / 0.2);
  }
</style>
