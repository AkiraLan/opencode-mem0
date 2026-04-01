# Codebase Structure

**Analysis Date:** 2026-04-01

## Directory Layout

```
opencode-mem0/
├── src/                          # Source code (TypeScript)
│   ├── index.ts                  # Plugin entry point
│   ├── cli.ts                    # CLI installer tool
│   ├── config.ts                 # Configuration loader
│   ├── types/
│   │   └── index.ts              # Type definitions
│   └── services/
│       ├── client.ts             # Mem0 API client
│       ├── compaction.ts         # Context compaction hook
│       ├── context.ts            # Context formatting
│       ├── jsonc.ts              # JSONC parser
│       ├── logger.ts             # File logging utility
│       ├── privacy.ts            # Content filtering
│       └── tags.ts               # Scope ID generation
├── dist/                         # Compiled output (not in repo)
├── scripts/                      # Build/utility scripts
├── .planning/                    # Documentation
│   └── codebase/
│       ├── ARCHITECTURE.md       # This file
│       └── STRUCTURE.md          # This file
├── package.json                  # Package manifest
├── tsconfig.json                 # TypeScript config
├── bun.lock                      # Bun lockfile
└── README.md                     # User documentation
```

## File Purposes

### Core Plugin Files

**`src/index.ts`** (654 lines)
- Main plugin export: `OpenMemoryPlugin`
- Implements OpenCode Plugin interface
- Registers hooks: `chat.message`, `tool`, `event`
- Contains memory keyword detection logic
- Formats and injects context on first message
- Tool implementation for `mem0` with 7 modes

**`src/cli.ts`** (618 lines)
- CLI entry point (`#!/usr/bin/env node`)
- `install` command for plugin setup
- Modifies `~/.config/opencode/opencode.jsonc`
- Creates `~/.config/opencode/mem0.jsonc`
- Registers `/mem0-init` slash command
- Handles Oh My OpenCode compatibility

### Configuration

**`src/config.ts`** (134 lines)
- Loads config from JSONC files or environment
- Provides typed `CONFIG` object with defaults
- Validation helpers for config values
- `isConfigured()` check for API key presence

**Key config files accessed:**
- `~/.config/opencode/mem0.jsonc` (preferred)
- `~/.config/opencode/mem0.json` (fallback)

### Types

**`src/types/index.ts`** (127 lines)
- Core type definitions
- `IMemoryBackendClient` interface (adapter pattern)
- Memory sectors, types, scopes
- API response types
- Conversation message types

### Services (`src/services/`)

**`client.ts`** (510 lines)
- `Mem0RESTClient` class implementing `IMemoryBackendClient`
- REST API calls to Mem0 Platform
- Scope encoding/decoding
- Response normalization
- Singleton pattern via `getMemoryClient()`
- Exported facade: `openMemoryClient`

**`compaction.ts`** (552 lines)
- Context compaction hook implementation
- Token usage monitoring
- Preemptive summarization trigger
- Summary capture and storage
- State management for compaction lifecycle

**`context.ts`** (76 lines)
- `formatContextForPrompt()` function
- Transforms memory data to prompt text
- Handles profile, project, and user memories
- Score/salience formatting

**`tags.ts`** (56 lines)
- Scope ID generation
- Git email extraction
- SHA-256 hashing for privacy
- Legacy tag functions for compatibility

**`privacy.ts`** (12 lines)
- `<private>` tag detection
- Content redaction
- Fully-private content rejection

**`logger.ts`** (15 lines)
- Simple file logger
- Writes to `~/.opencode-supermemory.log`
- Timestamped entries

**`jsonc.ts`** (137 lines)
- `stripJsoncComments()` - removes // and /* */ comments
- `stripJsonTrailingCommas()` - removes trailing commas
- `parseJsonc()` - combined parser
- String-aware parsing (preserves URLs in strings)

### Build Configuration

**`tsconfig.json`** (35 lines)
- Target: ESNext
- Module: ESNext with bundler resolution
- Strict TypeScript enabled
- Output: `./dist` directory
- Declaration files generated

**`package.json`**
- Type: module (ESM)
- Main: `dist/index.js`
- Bin: `dist/cli.js` as `opencode-mem0`
- Build script uses Bun bundler
- Single dependency: `@opencode-ai/plugin`

## Module Relationships

### Import Graph

```
src/index.ts
├── @opencode-ai/plugin (external)
├── ./config.js
│   └── ./services/jsonc.js
├── ./services/client.js
│   ├── ./config.js
│   ├── ./logger.js
│   └── ../types/index.js
├── ./services/context.js
│   ├── ./config.js
│   └── ../types/index.js
├── ./services/tags.js
│   ├── ./config.js
│   └── ../types/index.js
├── ./services/privacy.js
├── ./services/compaction.js
│   ├── ./client.js
│   ├── ./logger.js
│   ├── ./config.js
│   └── ../types/index.js
└── ./services/logger.js

src/cli.ts
├── ./services/jsonc.js
└── node:fs, node:path, node:os, node:url, node:readline
```

### Service Dependencies

| Service | Depends On | Used By |
|---------|-----------|---------|
| `client.ts` | `config.ts`, `logger.ts`, `types/index.ts` | `index.ts`, `compaction.ts` |
| `compaction.ts` | `client.ts`, `logger.ts`, `config.ts`, `types/index.ts` | `index.ts` |
| `context.ts` | `config.ts`, `types/index.ts` | `index.ts` |
| `tags.ts` | `config.ts`, `types/index.ts` | `index.ts`, `compaction.ts` |
| `privacy.ts` | - | `index.ts` |
| `logger.ts` | - | `client.ts`, `compaction.ts`, `index.ts` |
| `jsonc.ts` | - | `config.ts`, `cli.ts` |

## Naming Conventions

### Files
- **Source files**: `kebab-case.ts` (all files)
- **Entry points**: `index.ts`, `cli.ts`
- **Services**: descriptive nouns (`client.ts`, `logger.ts`)

### Types
- **Interfaces**: PascalCase with `I` prefix for backend abstractions (`IMemoryBackendClient`)
- **Types**: PascalCase (`MemoryScopeType`, `MemorySector`)
- **Enums**: Not used; use union types instead

### Functions
- **Exports**: camelCase (`formatContextForPrompt`, `getMemoryClient`)
- **Private helpers**: camelCase (no underscore prefix convention)
- **Methods**: camelCase

### Constants
- **Top-level config**: UPPER_SNAKE_CASE (`MEM0_API_KEY`, `CONFIG_FILES`)
- **Internal constants**: PascalCase or camelCase depending on usage

## Where to Add New Code

### New Memory Tool Modes
**Location**: `src/index.ts` (tool.execute switch statement)

Add new case in the switch statement around line 308-610:
```typescript
case "newmode": {
  // Implementation
  return JSON.stringify({ success: true, ... });
}
```

Update tool schema args (around line 260-284) if new parameters needed.

### New API Client Methods
**Location**: `src/services/client.ts`

Add to `Mem0RESTClient` class, then expose via `openMemoryClient` facade:
```typescript
// In class
async newMethod(): Promise<NewResult> { ... }

// In facade
newMethod: () => getMemoryClient().newMethod(),
```

Update types in `src/types/index.ts`:
```typescript
export interface NewResult { ... }
```

### New Configuration Options
**Location**: `src/config.ts`

1. Add to `Mem0Config` interface (line 13-31)
2. Add to `DEFAULTS` object (line 41-56)
3. Add to `CONFIG` export with validation (line 115-130)
4. Document in README.md

### New Event Handlers
**Location**: `src/index.ts` (event hook)

Add condition in `event` handler (around line 623-627) or extend compaction hook.

### New Privacy Filters
**Location**: `src/services/privacy.ts`

Add function, then call from `index.ts` tool handler before `addMemory`.

## Special Directories

### `~/.opencode/` (Runtime)
- **Purpose**: OpenCode message storage
- **Used by**: `compaction.ts`
- **Contents**: Session messages and parts (JSON files)
- **Created**: Automatically by OpenCode
- **Plugin subdirs**: `messages/`, `parts/`

### `~/.config/opencode/` (Configuration)
- **Purpose**: User configuration
- **Plugin files**:
  - `mem0.jsonc` - Plugin config (created by CLI)
  - `opencode.jsonc` - OpenCode config (modified by CLI)
  - `oh-my-opencode.json` - Oh My OpenCode config (modified by CLI)

### `dist/` (Build Output)
- **Generated**: Yes (by `bun run build`)
- **Committed**: No (in .gitignore)
- **Contents**: Compiled JS + type declarations

## Build Process

```bash
# Development
bun run dev           # TypeScript watch mode
bun run typecheck     # Type checking only

# Production
bun run build         # Full build
```

**Build steps:**
1. `bun build ./src/index.ts --outdir ./dist --target node`
2. `bun build ./src/cli.ts --outfile ./dist/cli.js --target node`
3. `tsc --emitDeclarationOnly` (generate .d.ts files)

**Output:**
- `dist/index.js` - Plugin bundle
- `dist/cli.js` - CLI bundle
- `dist/*.d.ts` - Type declarations

## Testing Strategy

**No test suite currently exists.**

**Recommended test additions:**
- Unit tests for `jsonc.ts` parsing edge cases
- Unit tests for `privacy.ts` content filtering
- Integration tests for `client.ts` with mocked Mem0 API
- E2E tests for CLI installer

**Suggested location**: `tests/` directory with Vitest or Bun's test runner.

---

*Structure analysis: 2026-04-01*
