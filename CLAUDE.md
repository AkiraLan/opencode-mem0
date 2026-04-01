<!-- GSD:project-start source:PROJECT.md -->
## Project

**opencode-mem0**

An OpenCode plugin that provides persistent memory capabilities using the Mem0 Platform API. It automatically injects context (user profile, project knowledge, relevant memories) into agent conversations and captures knowledge explicitly via trigger phrases like "remember this".

The plugin bridges OpenCode agents with Mem0's hosted memory backend, enabling cross-session continuity of knowledge and project-specific context awareness.

**Core Value:** **Memories are reliably captured and retrieved to make every conversation feel continuous and contextually aware.**

If memory persistence fails or context feels "fresh" every session, the plugin has failed its primary purpose.

### Constraints

- **Tech Stack**: Must remain compatible with OpenCode plugin SDK (`@opencode-ai/plugin`)
- **Backend**: Mem0 Platform API only — no local fallback
- **Privacy**: Cannot store raw PII; identifiers must be hashed
- **Compatibility**: Must work with non-Claude models (affects compaction logic)
- **Async Behavior**: Memory operations are async; failures may be silent
<!-- GSD:project-end -->

<!-- GSD:stack-start source:codebase/STACK.md -->
## Technology Stack

## Overview
## Languages
- **TypeScript 5.9.3** - All source code written in TypeScript with strict mode enabled
- **ESNext** - Target module system with modern JavaScript features
- **JSON/JSONC** - Configuration files with comments support (`mem0.jsonc`)
- **YAML** - GitHub Actions workflow definitions
## Runtime
- **Bun 1.3.5+** - Primary runtime and package manager
- **Node.js 20+** - Compatible target environment (based on `@types/node` dependency)
- **Bun** - Used for dependency management and script execution
- Lockfile: `bun.lock` (present)
## Frameworks & Core Dependencies
- `@opencode-ai/plugin@^1.0.162` (resolved to 1.0.191) - OpenCode plugin SDK
- **TypeScript 5.9.3** - Primary compiler with strict type checking
- **@types/bun@1.3.5** - Bun runtime type definitions
- **@types/node@20.19.27** - Node.js type definitions
## Key Dependencies
- `@opencode-ai/plugin` - Core plugin API and tool framework
- `@opencode-ai/sdk` - SDK types and interfaces (Part, etc.)
- `zod` - Runtime schema validation for tool arguments
- Native Node.js modules: `fs`, `path`, `os`, `readline`, `url`
- Native `fetch` API for HTTP requests to Mem0
## Configuration
- Target: ESNext with ESNext lib
- Module: ESNext with bundler resolution
- Strict mode enabled with additional safety checks
- Output: `./dist` directory with declarations and source maps
- Root directory: `./src`
- Key strict flags:
- Config files: `~/.config/opencode/mem0.jsonc` or `mem0.json`
- Environment variables supported:
## Build System
- `dist/index.js` - Main plugin entry point
- `dist/cli.js` - CLI installer entry point
- `dist/*.d.ts` - Type declarations
- `dist/*.d.ts.map` - Declaration source maps
## Platform Requirements
- Bun runtime installed
- Node.js 20+ compatibility
- Git for version control
- Node.js environment (plugin runs inside OpenCode)
- Network access to Mem0 API (`https://api.mem0.ai` by default)
- Write access to `~/.config/opencode/` for configuration
## CLI Tool
- `install` - Interactive/non-interactive plugin installation
## Plugin Hooks
- `chat.message` - Inject memory context and detect trigger phrases
- `event` - Handle compaction events
- `tool.mem0` - Memory management tool (add, search, list, forget, feedback, help)
<!-- GSD:stack-end -->

<!-- GSD:conventions-start source:CONVENTIONS.md -->
## Conventions

## TypeScript Configuration
- `target`: ESNext - Uses latest JavaScript features
- `module`: ESNext with bundler resolution
- `strict`: true - All strict type-checking options enabled
- `noUncheckedIndexedAccess`: true - Requires explicit handling of potentially undefined index access
- `noFallthroughCasesInSwitch`: true - Prevents accidental switch fallthrough
- `noImplicitOverride`: true - Requires explicit override keyword
- `verbatimModuleSyntax`: true - Preserves import/export syntax exactly as written
- `declaration`: true - Generates `.d.ts` declaration files
- `allowJs`: true - Allows importing JavaScript files
- `noUnusedLocals`: false - Allows unused local variables
- `noUnusedParameters`: false - Allows unused function parameters
- `noPropertyAccessFromIndexSignature`: false - Allows dot notation for index signatures
## Naming Conventions
### Files
- **Implementation files**: PascalCase for classes, camelCase for utilities (`client.ts`, `compaction.ts`)
- **Type definition files**: Lowercase with index barrel (`types/index.ts`)
- **Service modules**: Descriptive names in `src/services/` directory
### Functions
- **CamelCase**: All functions use camelCase (`getMemoryClient`, `stripPrivateContent`)
- **Factory functions**: Use `create` prefix (`createCompactionHook`)
- **Getters**: Use `get` prefix (`getUserId`, `getScopes`)
- **Check functions**: Use `is` prefix (`isConfigured`, `isSupportedModel`)
- **Predicate type guards**: Use `has` prefix for type narrowing (`hasAddResultsField`)
### Variables
- **Constants**: UPPER_SNAKE_CASE for true constants (`TIMEOUT_MS`, `CLAUDE_DEFAULT_CONTEXT_LIMIT`)
- **Configuration**: Upper snake case with env var mapping (`MEM0_API_KEY`, `CONFIG`)
- **Private class members**: Standard camelCase (no underscore prefix convention observed)
- **Type aliases**: PascalCase with descriptive names (`MemoryScopeContext`, `CompactionOptions`)
### Types and Interfaces
- **Interfaces**: Use `I` prefix for abstract interfaces (`IMemoryBackendClient`)
- **Type aliases**: Descriptive PascalCase (`MemoryType`, `MemorySector`)
- **Result types**: Suffix with `Result` (`SearchMemoriesResult`, `AddMemoryResult`)
- **Options/Config**: Suffix with `Options` or `Config` (`CompactionOptions`, `Mem0Config`)
### Enums (Union Types)
## Code Style Patterns
### Import Organization
- Always use `.js` extension in imports (ESM requirement)
- Use `node:` prefix for Node.js built-ins
- Use `type` imports for TypeScript types (`import type { Plugin }`)
### Function Design
- Functions are generally focused (20-40 lines ideal)
- Large files broken into logical sections with clear comments
- Async functions use explicit return type annotations
- Use destructuring for multiple parameters
- Options objects for configuration-heavy functions
- Type guards for runtime validation
### Error Handling
## TypeScript Patterns
### Type Guards
### Discriminated Unions
### Utility Types
## Logging Patterns
- Always include context object as second parameter
- Use descriptive message prefixes (e.g., `Mem0.addMemory:`, `[compaction]`)
- Log at entry/exit of async operations
- Log errors with full error message
## Configuration Patterns
## String and Regex Patterns
- Define patterns as constants at module level
- Use descriptive names (`CODE_BLOCK_PATTERN`, `DEFAULT_MEMORY_KEYWORD_PATTERN`)
- Include `g` flag for global matching when needed
- Comment complex patterns
## Privacy and Security
## Documentation Comments
## Architecture Patterns
### Singleton Pattern
### Adapter Pattern
### Factory Pattern
## File Organization
## Commit Message Conventions
- `docs:` - Documentation updates
- `refactor:` - Code restructuring
- `chore:` - Build/process changes
- `fix:` - Bug fixes
- `feat:` - New features (inferred)
## Key Decisions and Rationale
## Maintenance Notes
<!-- GSD:conventions-end -->

<!-- GSD:architecture-start source:ARCHITECTURE.md -->
## Architecture

## Overview
## High-Level Architecture
```
```
## Core Components
### 1. Plugin Entry Point (`src/index.ts`)
- Detects memory trigger keywords using configurable regex patterns
- Injects context on first message of each session only
- Fetches and formats: user profile, relevant user memories, project memories
- Nudges agent to use `mem0` tool when keywords detected
- Registers `mem0` tool with 7 modes: add, search, profile, list, forget, feedback, help
- Enforces configuration validation before execution
- Sanitizes content via privacy filter before storage
- Delegates to compaction hook for session lifecycle events
- Handles: `message.updated`, `session.deleted`, `session.idle`
### 2. Service Layer (`src/services/`)
#### Memory Client (`client.ts`)
```typescript
```
- Singleton instance via `getMemoryClient()` factory
- Scope encoding: `{prefix}:{userId}[:{projectId}]` for Mem0 user_id field
- Automatic org/project header injection for workspace routing
- Sector filtering via tags array (first element = sector)
- Response normalization for multiple Mem0 API versions
#### Context Formatter (`context.ts`)
```
- {static fact 1}
- {static fact 2}
- {dynamic fact 1}
- [{score}%] {memory content}
- [{sector}][{score}%] {memory content}
```
- Profile facts split by age (>7 days = static, ≤7 days = dynamic)
- Project memories shown with relevance scores
- User search results include sector annotations
#### Scope Management (`tags.ts`)
```typescript
```
- Privacy (no raw PII in Mem0)
- Stability (same inputs = same IDs)
- Length constraints (16-char hex)
#### Compaction Hook (`compaction.ts`)
- Token usage ratio exceeds threshold (default 0.8)
- Minimum 50K tokens used
- Cooldown period elapsed (30s)
- Model is Claude (opus/sonnet/haiku)
```typescript
```
#### Privacy Filter (`privacy.ts`)
- Strips `<private>...</private>` tags → `[REDACTED]`
- Rejects fully private content
- Applied in `mem0 add` tool handler
#### Logger (`logger.ts`)
#### JSONC Parser (`jsonc.ts`)
- Handles `//` and `/* */` comments
- Preserves string contents (URLs with //)
- Removes trailing commas
### 3. Configuration (`src/config.ts`)
```typescript
```
### 4. Type Definitions (`src/types/index.ts`)
- `episodic`: Events, experiences, temporal sequences
- `semantic`: Facts, concepts, general knowledge (default)
- `procedural`: Skills, how-to knowledge, processes
- `emotional`: Feelings, sentiments, reactions
- `reflective`: Meta-cognition, insights, patterns
- `project-config`: Tech stack, commands, tooling
- `architecture`: Codebase structure, components
- `error-solution`: Known issues and fixes
- `preference`: Coding style preferences
- `learned-pattern`: Project conventions
- `conversation`: Session summaries
### 5. CLI Installer (`src/cli.ts`)
- Registers plugin in `~/.config/opencode/opencode.jsonc`
- Creates `/mem0-init` slash command
- Generates `~/.config/opencode/mem0.jsonc`
- Configures Oh My OpenCode integration (disables conflicting hooks)
- Supports interactive (`--tui`) and automated (`--no-tui`) modes
## Data Flows
### First Message Context Injection
```
```
### Memory Addition Flow
```
```
### Context Compaction Flow
```
```
## Key Abstractions
### Memory Scope Context
```typescript
```
### Scope Separation Strategy
- **User scope**: Cross-project knowledge (preferences, workflows)
- **Project scope**: Directory-specific knowledge (architecture, conventions)
### Hierarchical Semantic Graph (HSG)
- Sector stored as first tag in Mem0 metadata
- Enables filtered retrieval by memory "category"
- Default: `semantic` for facts/concepts
## Error Handling Strategy
- All operations logged to `~/.opencode-supermemory.log`
- Timestamps and structured data for debugging
- Errors include full error messages
## Security Considerations
- Never logged or exposed in error messages
- Loaded from env var or config file only
- Config file is user-owned (`~/.config/`)
- `<private>` tag filtering before storage
- SHA-256 hashing of identifiers (email, path)
- No raw PII sent to Mem0
- Redirect validation in HTTP client
- Rejects cross-origin redirects
- Manual redirect handling (fetch redirect: "manual")
## Extension Points
```jsonc
```
```jsonc
```
<!-- GSD:architecture-end -->

<!-- GSD:workflow-start source:GSD defaults -->
## GSD Workflow Enforcement

Before using Edit, Write, or other file-changing tools, start work through a GSD command so planning artifacts and execution context stay in sync.

Use these entry points:
- `/gsd:quick` for small fixes, doc updates, and ad-hoc tasks
- `/gsd:debug` for investigation and bug fixing
- `/gsd:execute-phase` for planned phase work

Do not make direct repo edits outside a GSD workflow unless the user explicitly asks to bypass it.
<!-- GSD:workflow-end -->



<!-- GSD:profile-start -->
## Developer Profile

> Profile not yet configured. Run `/gsd:profile-user` to generate your developer profile.
> This section is managed by `generate-claude-profile` -- do not edit manually.
<!-- GSD:profile-end -->
