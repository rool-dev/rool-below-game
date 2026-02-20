# The Rool Below

A Rogue-like game build on Rool Spaces. Uses the `@rool-dev/svelte` which is a thin Svelte wapper on top of the core `@rool-dev/sdk` package for all data and AI operations.

## Technology Stack

- **Framework**: Svelte 5
- **Styling**: TailwindCSS v4
- **Language**: TypeScript
- **Package manager**: pnpm


For SDK documentation, **always read the README first**:

```
node_modules/@rool-dev/svelte/README.md  # The svelte wrapper
node_modules/@rool-dev/sdk/README.md  # The core SDK, reexposed by the wrapper
```

The README contains complete API documentation including:
- RoolClient and RoolSpace APIs
- Authentication patterns
- Object and relation operations
- AI operations (prompt, effort levels)
- Conversation management
- Events, undo/redo


## Architecture

All data is stored in a single publicly accessible Rool Space (link access)

The app uses crud operations to maintain the world: `createObject`and `updateObject`, and manages player locations with `link` and `unlink`

The Rool AI agent `space.prompt()` is used to let the palyer interact with the spaces as part of the gameplay.