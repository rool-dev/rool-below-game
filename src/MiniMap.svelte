<script lang="ts">
  import Icon from '@iconify/svelte';

  interface Props {
    grid: (string | null)[][];
    playerX: number;
    playerY: number;
  }

  let { grid, playerX, playerY }: Props = $props();

  const GRID_SIZE = 10;
</script>

<div class="bg-stone-900 rounded-lg p-3 border-2 border-stone-700 shadow-[inset_0_0_15px_rgba(0,0,0,0.5)]">
  <h3 class="text-xs font-semibold text-amber-600 uppercase tracking-wider mb-2 flex items-center gap-1.5">
    <Icon icon="mdi:map" class="w-4 h-4" />
    Dungeon Map
  </h3>

  <div class="grid gap-px bg-stone-950 p-0.5 rounded" style="grid-template-columns: repeat({GRID_SIZE}, 1fr);">
    {#each Array(GRID_SIZE) as _, y}
      {#each Array(GRID_SIZE) as _, x}
        {@const isPlayer = x === playerX && y === playerY}
        {@const hasRoom = grid[y]?.[x] !== null}
        <div
          class="aspect-square {isPlayer
            ? 'bg-orange-500 shadow-[0_0_6px_rgba(249,115,22,0.8)]'
            : hasRoom
              ? 'bg-stone-600'
              : 'bg-stone-900'}"
          title={isPlayer
            ? 'You are here'
            : hasRoom
              ? 'Explored'
              : 'Unknown'}
        ></div>
      {/each}
    {/each}
  </div>

  <div class="flex items-center gap-4 mt-3 text-xs text-stone-500">
    <div class="flex items-center gap-1.5">
      <div class="w-2.5 h-2.5 bg-orange-500 shadow-[0_0_4px_rgba(249,115,22,0.6)]"></div>
      <span>You</span>
    </div>
    <div class="flex items-center gap-1.5">
      <div class="w-2.5 h-2.5 bg-stone-600"></div>
      <span>Explored</span>
    </div>
  </div>
</div>
