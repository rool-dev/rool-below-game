<script lang="ts">
  import Icon from '@iconify/svelte';
  import type { RoolObject, ReactiveSpace } from '@rool-dev/svelte';

  interface RoomObject extends RoolObject {
    type: 'room';
    name: string;
    description: string;
    image?: string;
  }

  interface PlayerObject extends RoolObject {
    type: 'player';
    name: string;
  }

  interface ItemObject extends RoolObject {
    type: 'item';
    name: string;
    description: string;
  }

  interface NpcObject extends RoolObject {
    type: 'npc';
    name: string;
    description?: string;
  }

  interface Props {
    room: RoomObject;
    playersInRoom: PlayerObject[];
    itemsInRoom: ItemObject[];
    npcsInRoom?: NpcObject[];
    narratorMessage?: string | null;
    space: ReactiveSpace;
    isMoving?: boolean;
  }

  let { room, playersInRoom, itemsInRoom, npcsInRoom = [], narratorMessage = null, space, isMoving = false }: Props = $props();

  // Fetch and cache image URLs
  let imageUrl = $state<string | null>(null);
  let loadingImage = $state(false);

  $effect(() => {
    const mediaUrl = room.image;
    if (!mediaUrl) {
      imageUrl = null;
      return;
    }

    // Fetch the image via authenticated fetchMedia
    loadingImage = true;
    space.fetchMedia(mediaUrl).then(async (response) => {
      const blob = await response.blob();
      // Revoke previous URL if any
      if (imageUrl) {
        URL.revokeObjectURL(imageUrl);
      }
      imageUrl = URL.createObjectURL(blob);
    }).catch((err) => {
      console.error('Failed to fetch room image:', err);
      imageUrl = null;
    }).finally(() => {
      loadingImage = false;
    });

    // Cleanup on unmount or when image changes
    return () => {
      if (imageUrl) {
        URL.revokeObjectURL(imageUrl);
      }
    };
  });
</script>

<div class="h-full bg-stone-900 rounded-lg border-2 border-stone-700 overflow-hidden shadow-[inset_0_0_30px_rgba(0,0,0,0.5)]">
  <div class="h-full p-4 overflow-auto bg-gradient-to-b from-stone-900 to-stone-950">
    <!-- Title -->
    <h2 class="text-xl font-bold text-amber-200 mb-3 drop-shadow-[0_0_8px_rgba(251,191,36,0.3)] {isMoving ? 'opacity-50' : ''}">
      {room.name}
    </h2>

    <!-- Floated image with wrapped text -->
    <div class="room-content {isMoving ? 'opacity-50' : ''} transition-opacity">
      {#if room.image}
        <div class="float-left mr-4 mb-2 w-48 sm:w-64 md:w-96">
          {#if loadingImage}
            <div class="aspect-square bg-stone-800 rounded border border-stone-700 flex flex-col items-center justify-center gap-2 text-stone-600">
              <Icon icon="mdi:loading" class="w-8 h-8 animate-spin text-amber-600" />
              <span class="text-xs italic">The shadows shift...</span>
            </div>
          {:else if imageUrl}
            <img
              src={imageUrl}
              alt={room.name}
              class="w-full rounded border border-stone-700"
            />
          {:else}
            <div class="aspect-square bg-stone-800 rounded border border-stone-700 flex items-center justify-center text-stone-700">
              <Icon icon="mdi:image-off" class="w-10 h-10" />
            </div>
          {/if}
        </div>
      {/if}

      <p class="text-stone-300 leading-relaxed">
        {room.description}
      </p>

      <!-- Narrator message (ephemeral dungeon master narration) -->
      {#if narratorMessage}
        <p class="text-amber-200/80 leading-relaxed mt-3 italic">
          {narratorMessage}
        </p>
      {/if}

      <!-- Clear float before items/players -->
      <div class="clear-both"></div>
    </div>

    <!-- NPCs, Items and players -->
    {#if npcsInRoom.length > 0 || itemsInRoom.length > 0 || playersInRoom.length > 0}
      <div class="flex flex-col sm:flex-row gap-4 mt-4 pt-4 border-t border-stone-800">
        <!-- NPCs in room -->
        {#if npcsInRoom.length > 0}
          <div class="flex-1">
            <h3 class="text-sm font-semibold text-amber-600 uppercase tracking-wider mb-2 flex items-center gap-1.5">
              <Icon icon="mdi:account-alert" class="w-4 h-4" />
              Creatures
            </h3>
            <div class="space-y-2">
              {#each npcsInRoom as npc}
                <div class="bg-red-950/50 border border-red-900/50 rounded px-3 py-2">
                  <span class="text-red-400 font-medium">{npc.name}</span>
                  {#if npc.description}
                    <p class="text-sm text-stone-400 mt-0.5">{npc.description}</p>
                  {/if}
                </div>
              {/each}
            </div>
          </div>
        {/if}

        <!-- Items in room -->
        {#if itemsInRoom.length > 0}
          <div class="flex-1">
            <h3 class="text-sm font-semibold text-amber-600 uppercase tracking-wider mb-2 flex items-center gap-1.5">
              <Icon icon="mdi:treasure-chest" class="w-4 h-4" />
              Items
            </h3>
            <div class="space-y-2">
              {#each itemsInRoom as item}
                <div class="bg-stone-800/80 border border-stone-700 rounded px-3 py-2">
                  <span class="text-yellow-400 font-medium">{item.name}</span>
                  {#if item.description}
                    <p class="text-sm text-stone-400 mt-0.5">{item.description}</p>
                  {/if}
                </div>
              {/each}
            </div>
          </div>
        {/if}

        <!-- Other players in room -->
        {#if playersInRoom.length > 0}
          <div class="flex-1">
            <h3 class="text-sm font-semibold text-amber-600 uppercase tracking-wider mb-2 flex items-center gap-1.5">
              <Icon icon="mdi:account-group" class="w-4 h-4" />
              Adventurers Here
            </h3>
            <div class="flex flex-wrap gap-2">
              {#each playersInRoom as otherPlayer}
                <div class="flex items-center gap-1.5 bg-stone-800/80 border border-stone-700 rounded-full px-3 py-1">
                  <div class="w-2 h-2 bg-orange-500 rounded-full shadow-[0_0_6px_rgba(249,115,22,0.8)]"></div>
                  <span class="text-sm text-stone-300">{otherPlayer.name}</span>
                </div>
              {/each}
            </div>
          </div>
        {/if}
      </div>
    {/if}
  </div>
</div>
