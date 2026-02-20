<script lang="ts">
  import Icon from '@iconify/svelte';

  type Direction = 'N' | 'NE' | 'E' | 'SE' | 'S' | 'SW' | 'W' | 'NW';

  interface Props {
    onMove: (direction: Direction) => void;
    disabled?: boolean;
  }

  let { onMove, disabled = false }: Props = $props();

  const DIRECTION_ICONS: Record<Direction, string> = {
    'N': 'mdi:arrow-up',
    'NE': 'mdi:arrow-top-right',
    'E': 'mdi:arrow-right',
    'SE': 'mdi:arrow-bottom-right',
    'S': 'mdi:arrow-down',
    'SW': 'mdi:arrow-bottom-left',
    'W': 'mdi:arrow-left',
    'NW': 'mdi:arrow-top-left',
  };

  // Grid layout: NW N NE / W . E / SW S SE
  const GRID: (Direction | null)[][] = [
    ['NW', 'N', 'NE'],
    ['W', null, 'E'],
    ['SW', 'S', 'SE'],
  ];
</script>

<div class="bg-stone-900 rounded-lg p-3 border-2 border-stone-700 shadow-[inset_0_0_15px_rgba(0,0,0,0.5)]">
  <h3 class="text-xs font-semibold text-amber-600 uppercase tracking-wider mb-2 flex items-center gap-1.5">
    <Icon icon="mdi:compass" class="w-4 h-4" />
    Move
  </h3>

  <div class="grid grid-cols-3 gap-1.5">
    {#each GRID as row}
      {#each row as dir}
        {#if dir}
          <button
            class="aspect-square flex items-center justify-center bg-stone-800 border border-stone-600 hover:bg-stone-700 hover:border-amber-700 disabled:opacity-50 disabled:cursor-not-allowed rounded transition-colors"
            onclick={() => onMove(dir)}
            {disabled}
            title={dir}
          >
            <Icon icon={DIRECTION_ICONS[dir]} class="w-5 h-5 text-stone-400" />
          </button>
        {:else}
          <div class="aspect-square flex items-center justify-center bg-stone-950 rounded border border-stone-700">
            <div class="w-2 h-2 bg-orange-500 rounded-full shadow-[0_0_6px_rgba(249,115,22,0.8)]"></div>
          </div>
        {/if}
      {/each}
    {/each}
  </div>

  <p class="text-xs text-stone-500 text-center mt-2 italic">Click or use arrow keys</p>
</div>
