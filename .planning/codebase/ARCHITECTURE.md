# Architecture

**Analysis Date:** 2026-04-01

## Overview

opencode-mem0 is an OpenCode plugin that provides persistent memory capabilities using the Mem0 Platform API. It follows an event-driven plugin architecture with automatic context injection, explicit memory capture via trigger phrases, and preemptive context compaction when approaching token limits.

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        OpenCode Plugin System                           │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                   │
│  │ chat.message │  │    tool      │  │    event     │                   │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘                   │
└─────────┼─────────────────┼─────────────────┼───────────────────────────┘
          │                 │                 │
          ▼                 ▼                 ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                     OpenMemoryPlugin (`src/index.ts`)                   │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                     Hook Handlers                                │    │
│  │  • Memory keyword detection (regex-based)                        │    │
│  │  • First-message context injection                               │    │
│  │  • Tool registration (mem0)                                      │    │
│  │  • Event handling for compaction                                 │    │
│  └─────────────────────────────────────────────────────────────────┘    │
└─────────────────────────┬───────────────────────────────────────────────┘
                          │
          ┌───────────────┼───────────────┐
          │               │               │
          ▼               ▼               ▼
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│ Memory Client   │ │   Context       │ │  Compaction     │
│ (`client.ts`)   │ │   Service       │ │  Hook           │
│                 │ │ (`context.ts`)  │ │ (`compaction.ts`)│
│ • Mem0 REST API │ │                 │ │                 │
│ • CRUD ops      │ │ • Format        │ │ • Token limit   │
│ • Search        │ │ • Inject        │ │   monitoring    │
│ • Profile       │ │ • Prioritize    │ │ • Summary       │
│                 │ │                 │ │   preservation  │
└────────┬────────┘ └─────────────────┘ └─────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│                  Mem0 Platform API                      │
│              https://api.mem0.ai                        │
│  • Embeddings & vector search                           │
│  • Hierarchical Semantic Graph (HSG)                    │
│  • Scoped memory namespaces                             │
└─────────────────────────────────────────────────────────┘
```

## Core Components

### 1. Plugin Entry Point (`src/index.ts`)

The main plugin export implementing the OpenCode Plugin interface. Registers three primary capabilities:

**Chat Message Hook (`chat.message`)**
- Detects memory trigger keywords using configurable regex patterns
- Injects context on first message of each session only
- Fetches and formats: user profile, relevant user memories, project memories
- Nudges agent to use `mem0` tool when keywords detected

**Tool Registration (`tool`)**
- Registers `mem0` tool with 7 modes: add, search, profile, list, forget, feedback, help
- Enforces configuration validation before execution
- Sanitizes content via privacy filter before storage

**Event Hook (`event`)**
- Delegates to compaction hook for session lifecycle events
- Handles: `message.updated`, `session.deleted`, `session.idle`

### 2. Service Layer (`src/services/`)

#### Memory Client (`client.ts`)
Singleton adapter pattern implementing `IMemoryBackendClient` interface:

```typescript
interface IMemoryBackendClient {
  searchMemories(query, scope, options)
  addMemory(content, scope, options)
  listMemories(scope, options)
  deleteMemory(memoryId, scope)
  getProfile(scope, query?)
  createFeedback?(memoryId, feedback, reason)
}
```

**Key design decisions:**
- Singleton instance via `getMemoryClient()` factory
- Scope encoding: `{prefix}:{userId}[:{projectId}]` for Mem0 user_id field
- Automatic org/project header injection for workspace routing
- Sector filtering via tags array (first element = sector)
- Response normalization for multiple Mem0 API versions

#### Context Formatter (`context.ts`)
Transforms memory data into prompt-ready context strings:

**Output format:**
```
[OPENMEMORY]

User Profile:
- {static fact 1}
- {static fact 2}

Recent Context:
- {dynamic fact 1}

Project Knowledge:
- [{score}%] {memory content}

Relevant Memories:
- [{sector}][{score}%] {memory content}
```

**Prioritization rules:**
- Profile facts split by age (>7 days = static, ≤7 days = dynamic)
- Project memories shown with relevance scores
- User search results include sector annotations

#### Scope Management (`tags.ts`)
Generates stable identifiers for memory scoping:

```typescript
getScopes(directory) => {
  user: { userId: sha256(git_email || USER) },
  project: { userId, projectId: sha256(directory) }
}
```

**Rationale:** SHA-256 hashing ensures:
- Privacy (no raw PII in Mem0)
- Stability (same inputs = same IDs)
- Length constraints (16-char hex)

#### Compaction Hook (`compaction.ts`)
Preemptive context window management for Claude models:

**Trigger conditions:**
- Token usage ratio exceeds threshold (default 0.8)
- Minimum 50K tokens used
- Cooldown period elapsed (30s)
- Model is Claude (opus/sonnet/haiku)

**Compaction flow:**
1. Fetch project memories from Mem0
2. Inject structured compaction prompt with memories
3. Trigger OpenCode session summarization
4. Capture summary and save as memory
5. Resume conversation automatically

**State management:**
```typescript
interface CompactionState {
  lastCompactionTime: Map<sessionID, timestamp>
  compactionInProgress: Set<sessionID>
  summarizedSessions: Set<sessionID>
}
```

#### Privacy Filter (`privacy.ts`)
Content sanitization before storage:
- Strips `<private>...</private>` tags → `[REDACTED]`
- Rejects fully private content
- Applied in `mem0 add` tool handler

#### Logger (`logger.ts`)
Simple file-based logging to `~/.opencode-supermemory.log`

#### JSONC Parser (`jsonc.ts`)
Comment-stripping JSON parser for config files:
- Handles `//` and `/* */` comments
- Preserves string contents (URLs with //)
- Removes trailing commas

### 3. Configuration (`src/config.ts`)

**Configuration hierarchy (highest priority first):**
1. Environment variables (`MEM0_API_KEY`, etc.)
2. `~/.config/opencode/mem0.jsonc`
3. `~/.config/opencode/mem0.json`
4. Built-in defaults

**Key configuration options:**
```typescript
interface Config {
  apiUrl: string              // Mem0 API endpoint
  apiKey: string              // Authentication token
  orgId?: string              // Workspace org
  projectId?: string          // Workspace project
  keywordPatterns: string[]   // Custom memory triggers
  filterPrompt: string        // Storage guidance
  compactionThreshold: number // Token ratio trigger (0-1)
  similarityThreshold: number // Search relevance floor
  maxMemories: number         // User search results limit
  maxProjectMemories: number  // Project list limit
  maxProfileItems: number     // Profile facts limit
  minSalience: number         // Minimum memory relevance
  injectProfile: boolean      // Include profile in context
  scopePrefix: string         // Namespace prefix
  defaultSector: MemorySector // Default HSG sector
}
```

### 4. Type Definitions (`src/types/index.ts`)

**Memory Sectors (HSG model):**
- `episodic`: Events, experiences, temporal sequences
- `semantic`: Facts, concepts, general knowledge (default)
- `procedural`: Skills, how-to knowledge, processes
- `emotional`: Feelings, sentiments, reactions
- `reflective`: Meta-cognition, insights, patterns

**Memory Types (application layer):**
- `project-config`: Tech stack, commands, tooling
- `architecture`: Codebase structure, components
- `error-solution`: Known issues and fixes
- `preference`: Coding style preferences
- `learned-pattern`: Project conventions
- `conversation`: Session summaries

### 5. CLI Installer (`src/cli.ts`)

Standalone installer tool for plugin setup:
- Registers plugin in `~/.config/opencode/opencode.jsonc`
- Creates `/mem0-init` slash command
- Generates `~/.config/opencode/mem0.jsonc`
- Configures Oh My OpenCode integration (disables conflicting hooks)
- Supports interactive (`--tui`) and automated (`--no-tui`) modes

## Data Flows

### First Message Context Injection
```
User sends first message
        │
        ▼
┌─────────────────────┐
│ Detect memory       │──No──▶ Continue
│ keyword?            │
└────────┬────────────┘
         │ Yes
         ▼
┌─────────────────────┐
│ Inject nudge to     │
│ use mem0 tool       │
└─────────────────────┘
         │
         ▼
┌─────────────────────┐
│ Parallel fetch:     │
│ • Profile           │
│ • User memories     │
│ • Project memories  │
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│ Format context      │
│ (formatContextFor   │
│  Prompt)            │
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│ Prepend to message  │
│ parts as synthetic  │
│ text part           │
└─────────────────────┘
```

### Memory Addition Flow
```
Agent calls mem0 tool
   mode: "add"
        │
        ▼
┌─────────────────────┐
│ Validate config     │
│ (isConfigured)      │
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│ Sanitize content    │
│ (stripPrivate       │
│  Content)           │
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│ Determine scope     │
│ (user vs project)   │
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│ Call Mem0 API       │
│ POST /v1/memories   │
│ with metadata:      │
│ • type              │
│ • tags (incl sector)│
│ • scope             │
│ • source            │
└─────────────────────┘
```

### Context Compaction Flow
```
Message updated event
        │
        ▼
┌─────────────────────┐
│ Role = assistant?   │──No──▶ Return
│ Finish = true?      │
└────────┬────────────┘
         │ Yes
         ▼
┌─────────────────────┐
│ Get token usage     │
│ (input+output+cache)│
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│ usage/limit >       │──No──▶ Return
│ threshold?          │
└────────┬────────────┘
         │ Yes
         ▼
┌─────────────────────┐
│ Cooldown elapsed?   │──No──▶ Return
└────────┬────────────┘
         │ Yes
         ▼
┌─────────────────────┐
│ Fetch project       │
│ memories            │
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│ Inject compaction   │
│ prompt with         │
│ memories            │
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│ Trigger summarize   │
│ API call            │
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│ On summary message: │
│ Save as memory      │
│ type=conversation   │
└─────────────────────┘
```

## Key Abstractions

### Memory Scope Context
```typescript
interface MemoryScopeContext {
  userId: string;      // Hashed git email or USER env
  projectId?: string;  // Hashed working directory (optional)
}
```

Used throughout to route operations to correct Mem0 namespace.

### Scope Separation Strategy
- **User scope**: Cross-project knowledge (preferences, workflows)
- **Project scope**: Directory-specific knowledge (architecture, conventions)

Both share the same `userId` prefix but project scope appends `:{projectId}`.

### Hierarchical Semantic Graph (HSG)
Mem0's organizational model mapped to plugin types:
- Sector stored as first tag in Mem0 metadata
- Enables filtered retrieval by memory "category"
- Default: `semantic` for facts/concepts

## Error Handling Strategy

**Graceful degradation principles:**
1. Config validation: Plugin silently skips if unconfigured
2. API failures: Return structured error JSON, don't crash
3. Timeout handling: 30s timeout on all Mem0 API calls
4. Missing data: Empty arrays instead of null/undefined
5. Partial failures: Log errors but continue with available data

**Logging approach:**
- All operations logged to `~/.opencode-supermemory.log`
- Timestamps and structured data for debugging
- Errors include full error messages

## Security Considerations

**API Key handling:**
- Never logged or exposed in error messages
- Loaded from env var or config file only
- Config file is user-owned (`~/.config/`)

**Content privacy:**
- `<private>` tag filtering before storage
- SHA-256 hashing of identifiers (email, path)
- No raw PII sent to Mem0

**Cross-origin protection:**
- Redirect validation in HTTP client
- Rejects cross-origin redirects
- Manual redirect handling (fetch redirect: "manual")

## Extension Points

**Custom memory triggers:**
Add regex patterns to `mem0.jsonc`:
```jsonc
{
  "keywordPatterns": ["\\bremember this\\b", "\\bnote this\\b"]
}
```

**Custom filter guidance:**
```jsonc
{
  "filterPrompt": "Only store durable preferences, workflows, and project conventions."
}
```

**Alternative backends:**
Implement `IMemoryBackendClient` interface and replace `Mem0RESTClient` in `client.ts`.

---

*Architecture analysis: 2026-04-01*
