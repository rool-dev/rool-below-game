<script lang="ts">
  import Icon from '@iconify/svelte';
  import type { RoolObject } from '@rool-dev/svelte';

  interface PlayerObject extends RoolObject {
    type: 'player';
    name: string;
    hp: number;
  }

  interface ItemObject extends RoolObject {
    type: 'item';
    name: string;
    description: string;
  }

  interface Props {
    player: PlayerObject;
    inventory: ItemObject[];
  }

  let { player, inventory }: Props = $props();

  let isDead = $derived((player.hp ?? 10) <= 0);
  let hpPercent = $derived(Math.max(0, Math.min(100, ((player.hp ?? 10) / 10) * 100)));

  function getHpColor(hp: number): string {
    if (hp <= 0) return 'bg-stone-700';
    if (hp <= 3) return 'bg-red-500';
    if (hp <= 6) return 'bg-orange-500';
    return 'bg-green-500';
  }
</script>

<div class="bg-stone-900 rounded-lg p-3 border-2 border-stone-700 shadow-[inset_0_0_15px_rgba(0,0,0,0.5)]">
  <!-- Player name -->
  <div class="flex items-center gap-2 mb-3">
    <div class="w-8 h-8 bg-orange-600 rounded-full flex items-center justify-center shadow-[0_0_8px_rgba(249,115,22,0.5)] {isDead ? 'opacity-50' : ''}">
      <Icon icon={isDead ? 'mdi:skull' : 'mdi:account'} class="w-5 h-5 text-white" />
    </div>
    <span class="font-medium text-amber-100 truncate flex-1 {isDead ? 'line-through opacity-50' : ''}">{player.name}</span>
  </div>

  <!-- HP bar -->
  <div class="mb-3">
    <div class="flex items-center justify-between text-xs mb-1">
      <span class="text-stone-500 uppercase tracking-wider flex items-center gap-1">
        <Icon icon="mdi:heart" class="w-3.5 h-3.5 {isDead ? 'text-stone-600' : 'text-red-500'}" />
        HP
      </span>
      <span class="{isDead ? 'text-stone-600' : 'text-stone-400'}">{Math.max(0, player.hp ?? 10)}/10</span>
    </div>
    <div class="h-2 bg-stone-800 rounded-full overflow-hidden">
      <div
        class="h-full transition-all duration-300 {getHpColor(player.hp ?? 10)}"
        style="width: {hpPercent}%"
      ></div>
    </div>
    {#if isDead}
      <p class="text-xs text-stone-500 italic mt-1">Move to respawn</p>
    {/if}
  </div>

  <!-- Inventory -->
  <div>
    <h3 class="text-xs font-semibold text-amber-600 uppercase tracking-wider mb-2 flex items-center gap-1.5">
      <Icon icon="mdi:bag-personal" class="w-4 h-4" />
      Inventory
    </h3>
    {#if inventory.length === 0}
      <p class="text-xs text-stone-600 italic">Empty</p>
    {:else}
      <div class="space-y-1">
        {#each inventory as item}
          <div class="flex items-center gap-2 text-sm bg-stone-800/50 rounded px-2 py-1" title={item.description}>
            <Icon icon="mdi:cube-outline" class="w-3.5 h-3.5 text-yellow-500 shrink-0" />
            <span class="text-stone-300 truncate">{item.name}</span>
          </div>
        {/each}
      </div>
    {/if}
  </div>
</div>
