<script lang="ts">
  import Icon from '@iconify/svelte';
  import SvelteMarkdown from '@humanspeak/svelte-markdown';
  import type { Interaction } from '@rool-dev/svelte';

  interface Props {
    interactions: Interaction[];
    onSend: (text: string) => Promise<void>;
    disabled?: boolean;
  }

  let { interactions, onSend, disabled = false }: Props = $props();

  // Filter to only show prompt interactions (not link/unlink/createObject system ops)
  let chatInteractions = $derived(
    interactions.filter(i => i.operation === 'prompt')
  );

  let messageInput = $state('');
  let isSending = $state(false);
  let messagesContainer: HTMLElement | null = $state(null);

  // Auto-scroll chat
  $effect(() => {
    if (chatInteractions.length > 0 && messagesContainer) {
      setTimeout(() => {
        messagesContainer?.scrollTo({ top: messagesContainer.scrollHeight, behavior: 'smooth' });
      }, 0);
    }
  });

  async function sendMessage() {
    if (!messageInput.trim() || isSending || disabled) return;

    const text = messageInput.trim();
    messageInput = '';
    isSending = true;

    try {
      await onSend(text);
    } finally {
      isSending = false;
    }
  }

  function handleKeydown(e: KeyboardEvent) {
    if (e.key === 'Enter' && !e.shiftKey) {
      e.preventDefault();
      sendMessage();
    }
  }

  function formatTime(timestamp: number): string {
    return new Date(timestamp).toLocaleTimeString('en-US', { hour: 'numeric', minute: '2-digit' });
  }
</script>

<div class="bg-stone-900 rounded-lg border-2 border-stone-700 flex flex-col h-80 shadow-[inset_0_0_20px_rgba(0,0,0,0.5)]">
  <div class="px-3 py-2 border-b-2 border-stone-700 shrink-0 bg-stone-800/50">
    <h3 class="text-xs font-semibold text-amber-500 uppercase tracking-wider flex items-center gap-1.5">
      <Icon icon="mdi:wizard-hat" class="w-4 h-4" />
      Dungeon Master
    </h3>
  </div>

  <!-- Messages -->
  <div class="flex-1 overflow-auto p-3 space-y-3 bg-gradient-to-b from-stone-900/50 to-stone-950/50" bind:this={messagesContainer}>
    {#if chatInteractions.length === 0}
      <div class="text-center text-stone-500 text-sm py-4 italic">
        <p>Speak to the dungeon master...</p>
        <p class="mt-1 text-stone-600">"look around" or "examine the walls"</p>
      </div>
    {/if}

    {#each chatInteractions as interaction}
      <!-- User message -->
      <div class="flex flex-col items-end">
        <div class="flex items-center gap-2 mb-0.5 text-xs text-stone-500">
          <span>{interaction.userName ?? 'You'}</span>
          <span>{formatTime(interaction.timestamp)}</span>
        </div>
        <div class="max-w-[85%] bg-amber-900/80 border border-amber-700/50 text-amber-100 rounded rounded-br-none px-3 py-1.5">
          <p class="text-sm">{interaction.input}</p>
        </div>
      </div>

      <!-- AI response -->
      {#if interaction.output}
        <div class="flex justify-start">
          <div class="max-w-[85%] bg-stone-800/80 border border-stone-600 rounded rounded-bl-none px-3 py-1.5">
            <div class="markdown-output text-sm text-stone-200">
              <SvelteMarkdown source={interaction.output} />
            </div>
          </div>
        </div>
      {:else}
        <div class="flex justify-start">
          <div class="max-w-[85%] bg-stone-800/80 border border-stone-600 rounded rounded-bl-none px-3 py-1.5">
            {#if interaction.toolCalls && interaction.toolCalls.length > 0}
              <div class="flex items-center gap-2 text-sm text-stone-400">
                <Icon icon="mdi:loading" class="w-4 h-4 animate-spin text-amber-600" />
                <span class="font-mono text-xs">{interaction.toolCalls[interaction.toolCalls.length - 1]?.name}</span>
              </div>
            {:else}
              <div class="flex items-center gap-2 text-stone-400 italic">
                <Icon icon="mdi:loading" class="w-4 h-4 animate-spin text-amber-600" />
                <span class="text-sm">Pondering...</span>
              </div>
            {/if}
          </div>
        </div>
      {/if}
    {/each}
  </div>

  <!-- Input -->
  <div class="p-3 border-t-2 border-stone-700 shrink-0 bg-stone-800/50">
    <div class="flex items-center gap-2">
      <input
        type="text"
        class="flex-1 px-3 py-2 bg-stone-800 border border-stone-600 rounded text-stone-200 placeholder-stone-500 focus:ring-2 focus:ring-amber-600 focus:border-amber-600 focus:outline-none text-sm"
        placeholder="What do you do?"
        bind:value={messageInput}
        onkeydown={handleKeydown}
        disabled={isSending || disabled}
      />
      <button
        class="px-3 py-2 text-sm font-medium text-amber-100 bg-amber-800 border border-amber-700 rounded hover:bg-amber-700 disabled:opacity-50 disabled:cursor-not-allowed transition-colors"
        onclick={sendMessage}
        disabled={isSending || disabled || !messageInput.trim()}
      >
        {#if isSending}
          <Icon icon="mdi:loading" class="w-5 h-5 animate-spin" />
        {:else}
          <Icon icon="mdi:send" class="w-5 h-5" />
        {/if}
      </button>
    </div>
  </div>
</div>

<style>
  @reference "tailwindcss";

  .markdown-output :global(p) {
    @apply leading-relaxed my-1.5;
  }

  .markdown-output :global(p:first-child) {
    @apply mt-0;
  }

  .markdown-output :global(p:last-child) {
    @apply mb-0;
  }

  .markdown-output :global(strong) {
    @apply font-semibold text-amber-400;
  }

  .markdown-output :global(em) {
    @apply italic text-stone-400;
  }
</style>
