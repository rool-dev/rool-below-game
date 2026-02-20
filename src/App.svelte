<script lang="ts">
  import { createRool, type ReactiveSpace, type ReactiveCollection, type RoolObject, type CurrentUser } from '@rool-dev/svelte';
  import Icon from '@iconify/svelte';
  import MiniMap from './MiniMap.svelte';
  import RoomView from './RoomView.svelte';
  import Navigation from './Navigation.svelte';
  import PlayerInfo from './PlayerInfo.svelte';
  import Chat from './Chat.svelte';

  const rool = createRool();
  rool.init();

  const GRID_SIZE = 10;
  const START_X = 5;
  const START_Y = 5;

  // Types
  interface GameObject extends RoolObject {
    type: 'game';
    grid: (string | null)[][];
    backstory: string; // The dungeon's lore and history
    journal: string[]; // Major events that unfold during the adventure
  }

  interface RoomObject extends RoolObject {
    type: 'room';
    name: string;
    description: string;
    image?: string;
  }

  interface PlayerObject extends RoolObject {
    type: 'player';
    name: string;
    hp: number; // 0-10, where 10 is full health and 0 is dead
    _userId: string;
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

  type Direction = 'N' | 'NE' | 'E' | 'SE' | 'S' | 'SW' | 'W' | 'NW';

  const DIRECTION_OFFSETS: Record<Direction, [number, number]> = {
    'N':  [0, -1],
    'NE': [1, -1],
    'E':  [1, 0],
    'SE': [1, 1],
    'S':  [0, 1],
    'SW': [-1, 1],
    'W':  [-1, 0],
    'NW': [-1, -1],
  };

  const DIRECTION_NAMES: Record<Direction, string> = {
    'N': 'north',
    'NE': 'northeast',
    'E': 'east',
    'SE': 'southeast',
    'S': 'south',
    'SW': 'southwest',
    'W': 'west',
    'NW': 'northwest',
  };

  // State
  let currentSpace = $state<ReactiveSpace | null>(null);
  let currentUser = $state<CurrentUser | null>(null);
  let game = $state<GameObject | null>(null);
  let currentRoom = $state<RoomObject | null>(null);
  let player = $state<PlayerObject | null>(null);
  let playerPosition = $state<{ x: number; y: number } | null>(null); // Current player position
  let roomContents = $state<RoolObject[]>([]); // All objects in current room
  let inventory = $state<ItemObject[]>([]); // Items the player is carrying
  let allPlayers = $state<ReactiveCollection | null>(null); // All players in game
  let isMoving = $state(false);
  let isCreatingRoom = $state(false);
  let wallMessage = $state<string | null>(null);
  let narratorMessage = $state<string | null>(null); // Ephemeral dungeon master narration
  let showUserMenu = $state(false);
  let linkCopied = $state(false);

  // Derived from room contents
  let itemsInRoom = $derived(roomContents.filter(o => o.type === 'item') as ItemObject[]);
  let npcsInRoom = $derived(roomContents.filter(o => o.type === 'npc') as NpcObject[]);
  let playersInRoom = $derived(roomContents.filter(o => o.type === 'player' && o.id !== player?.id) as PlayerObject[]);

  // Derived
  let currentInteractions = $derived(currentSpace?.interactions ?? []);

  // URL handling
  function getSpaceIdFromUrl(): string | null {
    const params = new URLSearchParams(window.location.search);
    return params.get('space');
  }

  function setSpaceIdInUrl(spaceId: string) {
    const url = new URL(window.location.href);
    url.searchParams.set('space', spaceId);
    history.replaceState(null, '', url.toString());
  }

  // Sign in handler
  function signIn() {
    rool.login('The Rool Below');
  }

  // Load current user on auth
  $effect(() => {
    if (rool.authenticated && !currentUser) {
      rool.getCurrentUser().then(user => {
        currentUser = user;
      });
    }
  });

  // Load space from URL on auth
  $effect(() => {
    const spaceId = getSpaceIdFromUrl();
    if (spaceId && rool.authenticated && currentUser && !currentSpace) {
      joinGame(spaceId);
    }
  });

  // Real-time sync
  $effect(() => {
    if (!currentSpace) return;

    const unsubUpdate = currentSpace.on('objectUpdated', ({ objectId, object }) => {
      const obj = object as RoolObject;
      if (obj.type === 'game') game = obj as GameObject;
      if (objectId === currentRoom?.id) currentRoom = obj as RoomObject;
      if (objectId === player?.id) player = obj as PlayerObject;
    });

    const unsubLinked = currentSpace.on('linked', ({ sourceId }) => {
      // If something was linked to our room or player, refresh
      if (sourceId === currentRoom?.id) fetchRoomContents();
      if (sourceId === player?.id) {
        fetchInventory();
        fetchRoomContents(); // Item picked up may have been in room
      }
    });

    const unsubUnlinked = currentSpace.on('unlinked', ({ sourceId }) => {
      // If something was unlinked from our room or player, refresh
      if (sourceId === currentRoom?.id) fetchRoomContents();
      if (sourceId === player?.id) {
        fetchInventory();
        fetchRoomContents(); // Item dropped may now be in room
      }
    });

    return () => {
      unsubUpdate();
      unsubLinked();
      unsubUnlinked();
    };
  });

  // Fetch all contents of current room
  async function fetchRoomContents() {
    if (!currentSpace || !currentRoom) {
      roomContents = [];
      return;
    }
    roomContents = await currentSpace.getChildren(currentRoom.id, 'contains');
  }

  // Fetch player inventory
  async function fetchInventory() {
    if (!currentSpace || !player) {
      inventory = [];
      return;
    }
    const children = await currentSpace.getChildren(player.id, 'carrying');
    inventory = children.filter(o => o.type === 'item') as ItemObject[];
  }

  // Create empty grid
  function createEmptyGrid(): (string | null)[][] {
    return Array.from({ length: GRID_SIZE }, () =>
      Array.from({ length: GRID_SIZE }, () => null)
    );
  }

  // Find a room's position in the grid
  function findRoomPosition(grid: (string | null)[][], roomId: string): { x: number; y: number } | null {
    for (let y = 0; y < grid.length; y++) {
      for (let x = 0; x < grid[y].length; x++) {
        if (grid[y][x] === roomId) {
          return { x, y };
        }
      }
    }
    return null;
  }

  // System instruction for the AI
  function buildSystemInstruction(room: RoomObject): string {
    return `You are the dungeon master for "${room.name}". ${room.description}

SCHEMA (strict):
- player.hp: number 0-10 (10=full, 0=dead). Adjust when players take damage or heal.
- item: { type:'item', name, description } — link room->'contains'->item, or player->'carrying'->item
- npc: { type:'npc', name, description } — link room->'contains'->npc

RULES:
- The UI shows room details, items, NPCs, players. Don't repeat what's visible.
- Be dangerous. Traps, ambushes, consequences. Reduce update the player's hp field when bad things happen.
- Create items/NPCs only if fewer than 2 exist in the room. Do not exceed 2 items and 2 NPCs per room.
- Keep responses to 1-2 sentences. Atmospheric and darkly humorous.
- Stay consistent with the dungeon's backstory (stored in the game object).

JOURNAL: Append to game.journal ONLY for major events (one sentence max):
- Player death or near-death (hp ≤ 2)
- Defeating a named enemy/NPC
- Major discovery related to the backstory
Do NOT log: room entries, item pickups, routine combat, minor damage.

DO NOT: describe the room, create rooms, modify the grid, or link/unlink players (app manages player location).`;
  }

  // Switch to a room's conversation (each room has its own chat history)
  async function enterRoomConversation(space: ReactiveSpace, roomId: string) {
    space.conversationId = roomId;
    // Always set the system instruction to ensure it matches the current room
    const room = await space.getObject(roomId) as RoomObject;
    if (room) {
      await space.setSystemInstruction(buildSystemInstruction(room));
    }
  }

  // Create a new game
  async function createGame() {
    if (!currentUser) return;

    const space = await rool.createSpace('The Rool Below');
    currentSpace = space;
    isCreatingRoom = true; // Show "Discovering" spinner while creating starting room
    await space.setLinkAccess('editor');

    // Create game object with empty grid, backstory and journal
    const emptyGrid = createEmptyGrid();
    const { object: gameObj } = await space.createObject({
      data: {
        id: 'game',
        type: 'game',
        grid: emptyGrid,
        backstory: '{{2-3 sentence dark history of this mysterious dungeon: who built it, what happened here, what evil lurks within}}',
        journal: []
      }
    });

    // Create starting room that fits the backstory
    const { object: startRoom } = await space.createObject({
      data: {
        type: 'room',
        name: '{{evocative dungeon entrance name}}',
        description: '{{2-3 sentence atmospheric description of dungeon entrance}}',
        image: '{{dark fantasy dungeon entrance, pixel art style}}'
      }
    });

    // Update game with starting room
    emptyGrid[START_Y][START_X] = startRoom.id;
    await space.updateObject('game', { data: { grid: emptyGrid } });

    // Create player
    const { object: newPlayer } = await space.createObject({
      data: {
        type: 'player',
        name: currentUser.name ?? currentUser.email,
        hp: 10,
        _userId: currentUser.id,
      }
    });

    // Link player to starting room (room contains player)
    await space.link(startRoom.id, 'contains', newPlayer.id);

    // Enter room's conversation
    await enterRoomConversation(space, startRoom.id);

    // Update state (include backstory from generated game)
    game = { ...gameObj, grid: emptyGrid } as GameObject;
    game.grid[START_Y][START_X] = startRoom.id;
    currentRoom = startRoom as RoomObject;
    player = newPlayer as PlayerObject;
    playerPosition = { x: START_X, y: START_Y };
    isCreatingRoom = false;

    // Set up reactive collection for all players
    allPlayers = space.collection({ where: { type: 'player' } });

    setSpaceIdInUrl(space.id);
    await fetchRoomContents();
    await fetchInventory();
  }

  // Join an existing game
  async function joinGame(spaceId: string) {
    if (!currentUser) return;

    try {
      const space = await rool.openSpace(spaceId);
      currentSpace = space;

      // Load game
      const gameObj = await space.getObject('game');
      if (!gameObj) {
        console.error('No game found in space');
        return;
      }
      game = gameObj as GameObject;

      // Check if player already exists for this user (by user ID)
      const { objects: existingPlayers } = await space.findObjects({
        where: { type: 'player', _userId: currentUser.id }
      });

      if (existingPlayers.length > 0) {
        // Resume existing player - find their room from link (room contains player)
        player = existingPlayers[0] as PlayerObject;
        const rooms = await space.getParents(player.id, 'contains');
        if (rooms.length > 0) {
          currentRoom = rooms[0] as RoomObject;
          playerPosition = findRoomPosition(game.grid, currentRoom.id);
        }
      } else {
        // Find a starting position (first non-null room or center)
        let startX = START_X;
        let startY = START_Y;
        let startRoomId: string | null = null;

        // Find first available room
        outer: for (let y = 0; y < GRID_SIZE; y++) {
          for (let x = 0; x < GRID_SIZE; x++) {
            if (game.grid[y]?.[x]) {
              startX = x;
              startY = y;
              startRoomId = game.grid[y][x];
              break outer;
            }
          }
        }

        if (!startRoomId) {
          console.error('No rooms found in space');
          return;
        }

        // Create new player
        const { object: newPlayer } = await space.createObject({
          data: {
            type: 'player',
            name: currentUser.name ?? currentUser.email,
            hp: 10,
            _userId: currentUser.id,
          }
        });

        await space.link(startRoomId, 'contains', newPlayer.id);
        player = newPlayer as PlayerObject;
        currentRoom = await space.getObject(startRoomId) as RoomObject;
        playerPosition = { x: startX, y: startY };
      }

      // Enter room's conversation
      if (currentRoom) {
        await enterRoomConversation(space, currentRoom.id);
      }

      // Set up reactive collection for all players
      allPlayers = space.collection({ where: { type: 'player' } });

      setSpaceIdInUrl(space.id);
      await fetchRoomContents();
      await fetchInventory();
    } catch (err) {
      console.error('Failed to join game:', err);
      // Clear URL if space doesn't exist
      const url = new URL(window.location.href);
      url.searchParams.delete('space');
      history.replaceState(null, '', url.toString());
    }
  }

  // Move in a direction
  async function move(direction: Direction) {
    if (!currentSpace || !player || !game || !playerPosition || isMoving) return;

    const [dx, dy] = DIRECTION_OFFSETS[direction];
    const newX = playerPosition.x + dx;
    const newY = playerPosition.y + dy;

    // Check boundaries
    if (newX < 0 || newX >= GRID_SIZE || newY < 0 || newY >= GRID_SIZE) {
      wallMessage = 'You hit a wall.';
      setTimeout(() => wallMessage = null, 2000);
      return;
    }

    isMoving = true;
    wallMessage = null;
    narratorMessage = null; // Clear previous room's narrator message

    // Respawn if dead
    if (player.hp <= 0) {
      await currentSpace.updateObject(player.id, { data: { hp: 5 } });
      player = { ...player, hp: 5 };
    }

    // Save the current room before any changes (for unlinking later)
    const previousRoom = currentRoom;

    try {
      let roomId = game.grid[newY]?.[newX];

      // Create room if it doesn't exist
      if (!roomId) {
        isCreatingRoom = true;
        currentRoom = null; // Clear room to show loading state

        const { object: newRoom } = await currentSpace.createObject({
          data: {
            type: 'room',
            name: '{{evocative dungeon room name}}',
            description: '{{2-3 sentence atmospheric description}}',
            image: '{{dark fantasy dungeon room, pixel art style}}'
          },
          prompt: `Create a room to the ${DIRECTION_NAMES[direction]} of "${previousRoom?.name}". ${previousRoom?.description}`
        });

        roomId = newRoom.id;
        isCreatingRoom = false;

        // Update game grid
        const newGrid = game.grid.map(row => [...row]);
        newGrid[newY][newX] = roomId;
        await currentSpace.updateObject('game', { data: { grid: newGrid } });
        game = { ...game, grid: newGrid };
      }

      // Unlink from previous room (using saved reference)
      // Use try/catch in case the link was already removed (e.g., by AI)
      if (previousRoom) {
        try {
          await currentSpace.unlink(previousRoom.id, 'contains', player.id);
        } catch (e) {
          console.warn('Could not unlink from previous room (may already be unlinked):', e);
        }
      }

      // Link to new room (room contains player)
      await currentSpace.link(roomId, 'contains', player.id);

      // Switch to new room's conversation
      await enterRoomConversation(currentSpace, roomId);

      // Update local state
      playerPosition = { x: newX, y: newY };
      currentRoom = await currentSpace.getObject(roomId) as RoomObject;

      await fetchRoomContents();

      // Let the dungeon master react to the player's movement (ephemeral so it doesn't clutter chat)
      currentSpace.prompt(
        `${player.name} enters from the ${DIRECTION_NAMES[getOppositeDirection(direction)]}.`,
        { objectIds: [roomId, player.id], ephemeral: true }
      ).then(({ message }) => {
        fetchRoomContents();
        fetchInventory();
        // Show narrator message (stays until next event)
        if (message) {
          narratorMessage = message;
        }
      });
    } finally {
      isMoving = false;
      isCreatingRoom = false;
    }
  }

  // Get opposite direction for "enters from" phrasing
  function getOppositeDirection(dir: Direction): Direction {
    const opposites: Record<Direction, Direction> = {
      'N': 'S', 'S': 'N', 'E': 'W', 'W': 'E',
      'NE': 'SW', 'SW': 'NE', 'NW': 'SE', 'SE': 'NW'
    };
    return opposites[dir];
  }

  // Send chat message
  async function sendMessage(text: string) {
    if (!currentSpace || !currentRoom) return;
    narratorMessage = null; // Clear narrator message when user chats
    await currentSpace.prompt(text, { objectIds: [currentRoom.id, player!.id] });
    await fetchRoomContents();
    await fetchInventory();
  }

  // Copy share link
  async function copyShareLink() {
    if (!currentSpace) return;
    const url = new URL(window.location.href);
    url.searchParams.set('space', currentSpace.id);
    await navigator.clipboard.writeText(url.toString());
    linkCopied = true;
    setTimeout(() => linkCopied = false, 2000);
  }

  // Logout
  function logout() {
    currentSpace?.close();
    currentSpace = null;
    currentRoom = null;
    player = null;
    game = null;
    rool.logout();
  }

  // Keyboard navigation
  const KEY_TO_DIRECTION: Record<string, Direction> = {
    'ArrowUp': 'N',
    'ArrowDown': 'S',
    'ArrowLeft': 'W',
    'ArrowRight': 'E',
    // Numpad for diagonals
    'Numpad8': 'N',
    'Numpad9': 'NE',
    'Numpad6': 'E',
    'Numpad3': 'SE',
    'Numpad2': 'S',
    'Numpad1': 'SW',
    'Numpad4': 'W',
    'Numpad7': 'NW',
  };

  function handleKeydown(e: KeyboardEvent) {
    // Don't navigate if typing in an input
    if (e.target instanceof HTMLInputElement || e.target instanceof HTMLTextAreaElement) {
      return;
    }

    const direction = KEY_TO_DIRECTION[e.code];
    if (direction) {
      e.preventDefault();
      move(direction);
    }
  }

  // Set up keyboard listener
  $effect(() => {
    if (currentSpace) {
      window.addEventListener('keydown', handleKeydown);
      return () => window.removeEventListener('keydown', handleKeydown);
    }
  });
</script>

{#if rool.authenticated === null}
  <!-- Checking auth state -->
  <div class="h-dvh bg-stone-950 flex items-center justify-center">
    <div class="text-center">
      <div class="animate-spin rounded-full h-12 w-12 border-b-2 border-amber-600 mx-auto mb-4"></div>
      <p class="text-stone-500 italic">Loading...</p>
    </div>
  </div>
{:else if rool.authenticated === false}
  <!-- Splash screen for unauthenticated users -->
  <div class="h-dvh bg-stone-950 flex flex-col items-center justify-center p-4 overflow-auto">
    <div class="max-w-md w-full text-center">
      <!-- Game artwork -->
      <div class="mb-8">
        <img
          src="below.png"
          alt="The Rool Below"
          class="w-full max-w-sm mx-auto rounded-lg shadow-2xl shadow-amber-900/30 border-2 border-stone-800"
        />
      </div>

      <!-- Title and tagline -->
      <h1 class="text-4xl font-bold text-amber-100 mb-2 drop-shadow-[0_0_15px_rgba(251,191,36,0.4)]">
        The Rool Below
      </h1>
      <p class="text-stone-400 italic mb-8">
        A collaborative dungeon adventure
      </p>

      <!-- Sign in button -->
      <button
        class="w-full max-w-xs py-4 text-lg font-medium text-amber-100 bg-gradient-to-r from-amber-800 to-red-900 rounded-lg border-2 border-amber-700 hover:from-amber-700 hover:to-red-800 transition-all shadow-lg shadow-amber-900/50"
        onclick={signIn}
      >
        <span class="flex items-center justify-center gap-2">
          <Icon icon="mdi:login" class="w-6 h-6" />
          Sign In to Play
        </span>
      </button>

      <p class="text-stone-600 text-sm mt-6">
        Explore a procedural dungeon with friends.<br/>
        AI generates new rooms as you venture deeper.
      </p>
    </div>
  </div>
{:else if !currentUser}
  <!-- Loading user data -->
  <div class="h-dvh bg-stone-950 flex items-center justify-center">
    <div class="text-center">
      <div class="animate-spin rounded-full h-12 w-12 border-b-2 border-amber-600 mx-auto mb-4"></div>
      <p class="text-stone-500 italic">Loading...</p>
    </div>
  </div>
{:else if !currentSpace && !getSpaceIdFromUrl()}
  <!-- Landing page -->
  <div class="h-dvh bg-stone-950 flex flex-col items-center justify-center p-4">
    <div class="text-center mb-8">
      <div class="w-20 h-20 bg-gradient-to-br from-amber-700 to-red-900 rounded-lg flex items-center justify-center mx-auto mb-4 shadow-lg shadow-amber-900/50 border-2 border-amber-800">
        <Icon icon="mdi:stairs-down" class="w-10 h-10 text-amber-200" />
      </div>
      <h1 class="text-3xl font-bold text-amber-100 mb-2 drop-shadow-[0_0_10px_rgba(251,191,36,0.3)]">The Rool Below</h1>
      <p class="text-stone-400 italic">A collaborative dungeon adventure</p>
    </div>

    <div class="flex flex-col gap-4 w-full max-w-xs">
      <button
        class="py-4 text-lg font-medium text-amber-100 bg-gradient-to-r from-amber-800 to-red-900 rounded-lg border-2 border-amber-700 hover:from-amber-700 hover:to-red-800 transition-all shadow-lg shadow-amber-900/50"
        onclick={createGame}
      >
        <span class="flex items-center justify-center gap-2">
          <Icon icon="mdi:stairs-down" class="w-6 h-6" />
          Descend
        </span>
      </button>

      <p class="text-center text-stone-500 text-sm italic">
        Or join a friend via their share link
      </p>
    </div>

    <p class="text-stone-600 text-sm mt-8">
      Logged in as {currentUser.name ?? currentUser.email}
    </p>
  </div>
{:else if !currentSpace}
  <!-- Loading space -->
  <div class="h-dvh bg-stone-950 flex items-center justify-center">
    <div class="text-center">
      <div class="animate-spin rounded-full h-12 w-12 border-b-2 border-amber-600 mx-auto mb-4"></div>
      <p class="text-stone-500 italic">Entering dungeon...</p>
    </div>
  </div>
{:else}
  <!-- Game view -->
  <div class="h-dvh bg-stone-950 flex flex-col overflow-hidden">
    <!-- Header -->
    <header class="flex items-center justify-between px-4 py-3 border-b-2 border-stone-800 bg-stone-900/50">
      <div class="flex items-center gap-3">
        <div class="w-8 h-8 bg-gradient-to-br from-amber-700 to-red-900 rounded flex items-center justify-center border border-amber-800">
          <Icon icon="mdi:sword-cross" class="w-4 h-4 text-amber-200" />
        </div>
        <h1 class="text-lg font-semibold text-amber-100">The Rool Below</h1>
        {#if allPlayers?.objects && allPlayers.objects.length > 0}
          <div class="hidden sm:flex items-center gap-1.5 text-xs text-stone-500">
            <Icon icon="mdi:account-group" class="w-4 h-4" />
            <span>{allPlayers.objects.length} adventurer{allPlayers.objects.length !== 1 ? 's' : ''}</span>
          </div>
        {/if}
      </div>

      <div class="flex items-center gap-2">
        <button
          class="flex items-center justify-center gap-1.5 px-3 py-1.5 text-sm bg-stone-800 border rounded transition-colors sm:min-w-24 {linkCopied ? 'text-green-400 border-green-700' : 'text-stone-300 border-stone-700 hover:bg-stone-700'}"
          onclick={copyShareLink}
        >
          {#if linkCopied}
            <Icon icon="mdi:check" class="w-4 h-4" />
            <span class="hidden sm:inline">Copied!</span>
          {:else}
            <Icon icon="mdi:share-variant" class="w-4 h-4" />
            <span class="hidden sm:inline">Share</span>
          {/if}
        </button>

        <!-- User menu dropdown -->
        <div class="relative">
          <button
            class="flex items-center gap-1.5 px-3 py-1.5 text-sm text-stone-300 bg-stone-800 border border-stone-700 rounded hover:bg-stone-700 transition-colors"
            onclick={() => showUserMenu = !showUserMenu}
          >
            <Icon icon="mdi:account" class="w-4 h-4" />
            <span class="hidden sm:inline max-w-32 truncate">{player?.name}</span>
            <Icon icon="mdi:chevron-down" class="w-4 h-4" />
          </button>

          {#if showUserMenu}
            <!-- Backdrop to close menu -->
            <button
              class="fixed inset-0 z-10"
              onclick={() => showUserMenu = false}
              aria-label="Close menu"
            ></button>

            <!-- Menu -->
            <div class="absolute right-0 top-full mt-1 w-48 bg-stone-800 rounded border-2 border-stone-700 shadow-xl z-20 overflow-hidden">
              <div class="px-3 py-2 border-b border-stone-700">
                <p class="text-sm text-stone-300 truncate">{player?.name}</p>
                <p class="text-xs text-stone-500 truncate">{currentUser?.email}</p>
              </div>
              <button
                class="w-full flex items-center gap-2 px-3 py-2 text-sm text-stone-300 hover:bg-stone-700 transition-colors"
                onclick={() => { showUserMenu = false; logout(); }}
              >
                <Icon icon="mdi:logout" class="w-4 h-4" />
                Log out
              </button>
            </div>
          {/if}
        </div>
      </div>
    </header>

    <!-- Main content -->
    <main class="flex-1 flex flex-col md:flex-row min-h-0 p-4 gap-4 overflow-auto md:overflow-hidden">
      <!-- Desktop: Left sidebar | Mobile: after room view -->
      <div class="order-2 md:order-1 flex flex-col gap-4 md:w-64 shrink-0">
        <div class="flex flex-row md:flex-col gap-4">
          {#if game && playerPosition}
            <div class="flex-1 md:flex-none">
              <MiniMap grid={game.grid} playerX={playerPosition.x} playerY={playerPosition.y} />
            </div>
          {/if}

          <div class="flex-1 md:flex-none">
            <Navigation onMove={move} disabled={isMoving} />
          </div>
        </div>

        {#if player}
          <PlayerInfo {player} {inventory} />
        {/if}

        {#if wallMessage}
          <div class="text-center text-amber-500 text-sm animate-pulse hidden md:block italic">
            {wallMessage}
          </div>
        {/if}
      </div>

      <!-- Room + Chat -->
      <div class="order-1 md:order-2 flex-1 flex flex-col gap-4 min-w-0">
        <!-- Room view -->
        <div class="flex-1 min-h-64 md:min-h-0 relative">
          {#if currentRoom && currentSpace}
            <RoomView
              room={currentRoom}
              {playersInRoom}
              {itemsInRoom}
              {npcsInRoom}
              {narratorMessage}
              space={currentSpace}
              {isMoving}
            />
          {:else if isCreatingRoom}
            <!-- Creating new room -->
            <div class="h-full bg-stone-900 rounded-lg border-2 border-stone-700 flex flex-col items-center justify-center gap-4 shadow-[inset_0_0_30px_rgba(0,0,0,0.5)]">
              <div class="relative">
                <div class="w-24 h-24 border-4 border-stone-700 border-t-amber-600 rounded-full animate-spin"></div>
                <div class="absolute inset-0 flex items-center justify-center">
                  <Icon icon="mdi:map-marker-question" class="w-10 h-10 text-amber-500" />
                </div>
              </div>
              <div class="text-center">
                <p class="text-lg font-medium text-amber-200">Discovering new area...</p>
                <p class="text-sm text-stone-500 mt-1 italic">The dungeon reveals its secrets</p>
              </div>
            </div>
          {:else}
            <div class="h-full bg-stone-900 rounded-lg border-2 border-stone-700 flex items-center justify-center text-stone-500 italic">
              <Icon icon="mdi:loading" class="w-6 h-6 animate-spin mr-2 text-amber-600" />
              Loading room...
            </div>
          {/if}
        </div>

        {#if wallMessage}
          <div class="text-center text-amber-500 text-sm animate-pulse md:hidden italic">
            {wallMessage}
          </div>
        {/if}

        <Chat
          interactions={currentInteractions}
          onSend={sendMessage}
          disabled={isMoving}
        />
      </div>
    </main>
  </div>
{/if}
