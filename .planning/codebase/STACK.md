# Technology Stack

**Analysis Date:** 2026-04-01

## Overview

opencode-mem0 is a TypeScript-based OpenCode plugin that provides persistent memory capabilities using the Mem0 Platform API. The project uses Bun as its runtime and package manager, with a modern ES module architecture targeting Node.js environments.

## Languages

**Primary:**
- **TypeScript 5.9.3** - All source code written in TypeScript with strict mode enabled
- **ESNext** - Target module system with modern JavaScript features

**Secondary:**
- **JSON/JSONC** - Configuration files with comments support (`mem0.jsonc`)
- **YAML** - GitHub Actions workflow definitions

## Runtime

**Environment:**
- **Bun 1.3.5+** - Primary runtime and package manager
- **Node.js 20+** - Compatible target environment (based on `@types/node` dependency)

**Package Manager:**
- **Bun** - Used for dependency management and script execution
- Lockfile: `bun.lock` (present)

## Frameworks & Core Dependencies

**Core Plugin Framework:**
- `@opencode-ai/plugin@^1.0.162` (resolved to 1.0.191) - OpenCode plugin SDK
  - Provides `Plugin`, `PluginInput`, `tool` exports
  - Includes `@opencode-ai/sdk@1.0.191` as transitive dependency
  - Uses `zod@4.1.8` for schema validation

**Development Tools:**
- **TypeScript 5.9.3** - Primary compiler with strict type checking
- **@types/bun@1.3.5** - Bun runtime type definitions
- **@types/node@20.19.27** - Node.js type definitions

## Key Dependencies

**Critical (Runtime):**
- `@opencode-ai/plugin` - Core plugin API and tool framework
- `@opencode-ai/sdk` - SDK types and interfaces (Part, etc.)
- `zod` - Runtime schema validation for tool arguments

**Infrastructure:**
- Native Node.js modules: `fs`, `path`, `os`, `readline`, `url`
- Native `fetch` API for HTTP requests to Mem0

## Configuration

**TypeScript Configuration (`tsconfig.json`):**
- Target: ESNext with ESNext lib
- Module: ESNext with bundler resolution
- Strict mode enabled with additional safety checks
- Output: `./dist` directory with declarations and source maps
- Root directory: `./src`
- Key strict flags:
  - `noUncheckedIndexedAccess: true`
  - `noImplicitOverride: true`
  - `noFallthroughCasesInSwitch: true`

**Environment Configuration:**
- Config files: `~/.config/opencode/mem0.jsonc` or `mem0.json`
- Environment variables supported:
  - `MEM0_API_KEY` / `OPENMEMORY_API_KEY`
  - `MEM0_API_URL` / `OPENMEMORY_API_URL`
  - `MEM0_ORG_ID` / `OPENMEMORY_ORG_ID`
  - `MEM0_PROJECT_ID` / `OPENMEMORY_PROJECT_ID`

## Build System

**Build Scripts (`package.json`):**
```bash
bun run build    # Build index.ts and cli.ts to dist/
bun run dev      # TypeScript watch mode
bun run typecheck  # Type checking without emit
```

**Build Process:**
1. `bun build ./src/index.ts --outdir ./dist --target node`
2. `bun build ./src/cli.ts --outfile ./dist/cli.js --target node`
3. `tsc --emitDeclarationOnly` - Generate `.d.ts` files

**Output:**
- `dist/index.js` - Main plugin entry point
- `dist/cli.js` - CLI installer entry point
- `dist/*.d.ts` - Type declarations
- `dist/*.d.ts.map` - Declaration source maps

## Platform Requirements

**Development:**
- Bun runtime installed
- Node.js 20+ compatibility
- Git for version control

**Production:**
- Node.js environment (plugin runs inside OpenCode)
- Network access to Mem0 API (`https://api.mem0.ai` by default)
- Write access to `~/.config/opencode/` for configuration

## CLI Tool

**Binary:** `opencode-mem0` (defined in `package.json` bin)

**Entry:** `./dist/cli.js`

**Commands:**
- `install` - Interactive/non-interactive plugin installation
  - `--no-tui` - Non-interactive mode
  - `--disable-context-recovery` - Disable Oh My OpenCode auto-compact

## Plugin Hooks

**Registered Hooks (`package.json` opencode section):**
- `chat.message` - Inject memory context and detect trigger phrases
- `event` - Handle compaction events
- `tool.mem0` - Memory management tool (add, search, list, forget, feedback, help)

---

*Stack analysis: 2026-04-01*
