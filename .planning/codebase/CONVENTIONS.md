# Coding Conventions

**Analysis Date:** 2025-04-01

**Overview:** This document outlines the coding standards, patterns, and practices used in the opencode-mem0 project. The codebase follows modern TypeScript best practices with strict compiler settings, functional programming patterns, and comprehensive error handling.

---

## TypeScript Configuration

**Compiler:** TypeScript 5.7.3 with strict mode enabled

**Key Settings** (`tsconfig.json`):
- `target`: ESNext - Uses latest JavaScript features
- `module`: ESNext with bundler resolution
- `strict`: true - All strict type-checking options enabled
- `noUncheckedIndexedAccess`: true - Requires explicit handling of potentially undefined index access
- `noFallthroughCasesInSwitch`: true - Prevents accidental switch fallthrough
- `noImplicitOverride`: true - Requires explicit override keyword
- `verbatimModuleSyntax`: true - Preserves import/export syntax exactly as written
- `declaration`: true - Generates `.d.ts` declaration files
- `allowJs`: true - Allows importing JavaScript files

**Relaxed Settings** (for development convenience):
- `noUnusedLocals`: false - Allows unused local variables
- `noUnusedParameters`: false - Allows unused function parameters
- `noPropertyAccessFromIndexSignature`: false - Allows dot notation for index signatures

---

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
```typescript
// Prefer string literal unions over enum declarations
export type MemoryType =
  | "project-config"
  | "architecture"
  | "error-solution"
  | "preference"
  | "learned-pattern"
  | "conversation";

export type MemorySector = 
  | "episodic"
  | "semantic"
  | "procedural"
  | "emotional"
  | "reflective";
```

---

## Code Style Patterns

### Import Organization
**Order observed in files:**
1. External dependencies (`@opencode-ai/plugin`)
2. Node.js built-in modules (`node:fs`, `node:path`)
3. Internal services (`./services/client.js`)
4. Types (`./types/index.js`)
5. Utilities and config (`../config.js`)

**Key conventions:**
- Always use `.js` extension in imports (ESM requirement)
- Use `node:` prefix for Node.js built-ins
- Use `type` imports for TypeScript types (`import type { Plugin }`)

### Function Design

**Size and Complexity:**
- Functions are generally focused (20-40 lines ideal)
- Large files broken into logical sections with clear comments
- Async functions use explicit return type annotations

**Parameters:**
- Use destructuring for multiple parameters
- Options objects for configuration-heavy functions
- Type guards for runtime validation

**Examples:**
```typescript
// From src/services/client.ts - focused, single-purpose
function withTimeout<T>(promise: Promise<T>, ms: number): Promise<T> {
  return Promise.race([
    promise,
    new Promise<T>((_, reject) =>
      setTimeout(() => reject(new Error(`Timeout after ${ms}ms`)), ms)
    ),
  ]);
}

// From src/services/tags.ts - options object pattern
export function getScopes(directory: string): { 
  user: MemoryScopeContext; 
  project: MemoryScopeContext 
} {
  const userId = getUserId();
  const projectId = getProjectId(directory);
  return { user: { userId }, project: { userId, projectId } };
}
```

### Error Handling

**Strategy:** Centralized error handling with structured logging

**Patterns:**
1. **Try-catch with error normalization:**
```typescript
// From src/services/client.ts
try {
  const response = await this.fetch("/v2/memories/search", { ... });
  // ... processing
} catch (error) {
  const errorMessage = error instanceof Error ? error.message : String(error);
  log("Mem0.searchMemories: error", { error: errorMessage });
  return { success: false, results: [], total: 0, error: errorMessage };
}
```

2. **Result pattern for operations:**
```typescript
// All backend operations return structured results
interface AddMemoryResult {
  success: boolean;
  id?: string;
  sector?: MemorySector;
  error?: string;
}
```

3. **Graceful degradation:**
   - Never throw from plugin hooks
   - Always return sensible defaults on failure
   - Log errors for debugging

---

## TypeScript Patterns

### Type Guards
```typescript
// From src/services/client.ts
function isRecord(value: unknown): value is Record<string, unknown> {
  return typeof value === "object" && value !== null;
}

function hasAddResultsField(data: unknown): data is Mem0AddResponse {
  return typeof data === "object" && data !== null && "results" in data;
}
```

### Discriminated Unions
```typescript
// From src/types/index.ts
export interface MemoryItem {
  id: string;
  content: string;
  score?: number;
  salience?: number;
  sector?: MemorySector;
  tags?: string[];
  metadata?: Record<string, unknown>;
  createdAt?: string;
  updatedAt?: string;
}
```

### Utility Types
```typescript
// Type-safe property access with null checks
const metadataTags = metadata?.tags;
const tags = Array.isArray(metadataTags)
  ? metadataTags.filter((tag): tag is string => typeof tag === "string")
  : record.categories;
```

---

## Logging Patterns

**Location:** `src/services/logger.ts`

**Pattern:** Structured JSON logging to file
```typescript
export function log(message: string, data?: unknown) {
  const timestamp = new Date().toISOString();
  const line = data 
    ? `[${timestamp}] ${message}: ${JSON.stringify(data)}\n`
    : `[${timestamp}] ${message}\n`;
  appendFileSync(LOG_FILE, line);
}
```

**Usage:**
- Always include context object as second parameter
- Use descriptive message prefixes (e.g., `Mem0.addMemory:`, `[compaction]`)
- Log at entry/exit of async operations
- Log errors with full error message

**Examples:**
```typescript
log("Plugin init", { directory, scopes, configured: isConfigured() });
log("chat.message: processing", { messagePreview: userMessage.slice(0, 100) });
log("Mem0.searchMemories: error", { error: errorMessage });
```

---

## Configuration Patterns

**Location:** `src/config.ts`

**Pattern:** Environment + File-based configuration with validation

```typescript
// Layered config: File config → Environment variables → Defaults
export const MEM0_API_KEY = fileConfig.apiKey ?? process.env.MEM0_API_KEY;

// Validation functions for each type
function readUnitInterval(value: unknown, fallback: number): number {
  return typeof value === "number" && Number.isFinite(value) && value >= 0 && value <= 1
    ? value
    : fallback;
}

// Centralized CONFIG object
export const CONFIG = {
  apiUrl: MEM0_API_URL,
  // ... validated values
};
```

---

## String and Regex Patterns

**Regex Definitions:**
- Define patterns as constants at module level
- Use descriptive names (`CODE_BLOCK_PATTERN`, `DEFAULT_MEMORY_KEYWORD_PATTERN`)
- Include `g` flag for global matching when needed
- Comment complex patterns

**From `src/index.ts`:**
```typescript
const CODE_BLOCK_PATTERN = /```[\s\S]*?```/g;
const INLINE_CODE_PATTERN = /`[^`]+`/g;
const DEFAULT_MEMORY_KEYWORD_PATTERN =
  /\b(remember|memorize|save\s+this|note\s+this|keep\s+in\s+mind|don'?t\s+forget|learn\s+this|store\s+this|record\s+this|make\s+a\s+note|take\s+note|jot\s+down|commit\s+to\s+memory|remember\s+that|never\s+forget|always\s+remember)\b/i;
```

---

## Privacy and Security

**Location:** `src/services/privacy.ts`

**Pattern:** Content redaction for sensitive data
```typescript
export function stripPrivateContent(content: string): string {
  return content.replace(/<private>[\s\S]*?<\/private>/gi, "[REDACTED]");
}
```

---

## Documentation Comments

**Style:** JSDoc for exported functions, inline for complex logic

```typescript
/**
 * Strips comments from JSONC content while respecting string boundaries.
 * Handles // and /* comments, URLs in strings, and escaped quotes.
 */
export function stripJsoncComments(content: string): string {
  // Implementation...
}

// From src/services/compaction.ts - inline comment for complex logic
// Count consecutive backslashes before this quote
let backslashCount = 0;
let j = i - 1;
while (j >= 0 && content[j] === "\\") {
  backslashCount++;
  j--;
}
```

---

## Architecture Patterns

### Singleton Pattern
```typescript
// From src/services/client.ts
let clientInstance: Mem0RESTClient | null = null;

export function getMemoryClient(): Mem0RESTClient {
  if (!clientInstance) {
    clientInstance = new Mem0RESTClient();
  }
  return clientInstance;
}
```

### Adapter Pattern
```typescript
// Abstract interface for different memory backends
export interface IMemoryBackendClient {
  searchMemories(query: string, scope: MemoryScopeContext, options?: {...}): Promise<SearchMemoriesResult>;
  addMemory(content: string, scope: MemoryScopeContext, options?: {...}): Promise<AddMemoryResult>;
  // ...
}
```

### Factory Pattern
```typescript
// From src/services/compaction.ts
export function createCompactionHook(
  ctx: CompactionContext,
  tags: { user: string; project: string },
  scopes: { user: MemoryScopeContext; project: MemoryScopeContext },
  options?: CompactionOptions
) {
  const state: CompactionState = { ... };
  // Return object with methods
  return { async event({ event }) { ... } };
}
```

---

## File Organization

```
src/
├── index.ts          # Plugin entry point, main logic
├── cli.ts            # CLI installer and commands
├── config.ts         # Configuration loading and validation
├── types/
│   └── index.ts      # Type definitions and interfaces
└── services/
    ├── client.ts     # Mem0 API client
    ├── compaction.ts # Context compaction logic
    ├── context.ts    # Context formatting for prompts
    ├── jsonc.ts      # JSONC parsing utilities
    ├── logger.ts     # Logging service
    ├── privacy.ts    # Privacy/content filtering
    └── tags.ts       # Scope and tag generation
```

---

## Commit Message Conventions

**Pattern:** Conventional Commits format

**Types observed:**
- `docs:` - Documentation updates
- `refactor:` - Code restructuring
- `chore:` - Build/process changes
- `fix:` - Bug fixes
- `feat:` - New features (inferred)

**Examples from git history:**
```
docs: update Explicit Memory Saving section with regex support info
refactor: rename openmemory references to mem0
chore: rename package from opencode-openmemory to opencode-mem0
fix: add prt_ prefix to Part IDs to comply with OpenCode SDK validation
```

---

## Key Decisions and Rationale

1. **ESM-only (`"type": "module"`)**: Modern JavaScript, tree-shaking support, aligns with Node.js direction

2. **Bun as build tool**: Fast builds, built-in TypeScript support, single executable output

3. **Strict TypeScript**: Catches errors at compile time, improves maintainability, better IDE support

4. **No testing framework**: Project currently has no test files; relies on manual testing and TypeScript strictness

5. **JSONC for config**: Supports comments in configuration files, better user experience

6. **File-based logging**: Simple, persistent, no external dependencies, user-accessible for debugging

7. **No linting tools**: No ESLint or Prettier configured; relies on TypeScript strictness and manual code review

---

## Maintenance Notes

**When adding new features:**
1. Add types to `src/types/index.ts` first
2. Implement in appropriate service module
3. Export from `src/index.ts` if needed
4. Update CLI if configuration changes
5. Follow existing error handling patterns
6. Add logging calls for debugging

**When modifying configuration:**
1. Add to `Mem0Config` interface
2. Add default value to `DEFAULTS`
3. Add validation function if needed
4. Export from `CONFIG` object
5. Support both file and environment variable sources

**Build command:**
```bash
bun run build
```

**Type checking:**
```bash
bun run typecheck
```

---

*Convention analysis: 2025-04-01*
