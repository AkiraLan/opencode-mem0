# External Integrations

**Analysis Date:** 2026-04-01

## Overview

opencode-mem0 integrates primarily with the Mem0 Platform API for persistent memory storage and retrieval. It also interfaces with the OpenCode plugin system for context injection and compaction handling.

## APIs & External Services

**Mem0 Platform API:**
- **Base URL:** `https://api.mem0.ai` (configurable via `apiUrl`)
- **Purpose:** Primary memory storage backend - embeddings, search, and retrieval
- **Auth:** Token-based (`Authorization: Token <api_key>`)
- **Timeout:** 30 seconds for all requests
- **Endpoints Used:**
  - `POST /v2/memories/search` - Semantic memory search
  - `POST /v1/memories` - Add new memory
  - `POST /v2/memories` - List memories with pagination
  - `DELETE /v1/memories/{id}` - Delete memory
  - `POST /v1/feedback` - Submit memory feedback

**OpenCode Plugin System:**
- **Purpose:** Host environment for plugin execution
- **Integration Points:**
  - Plugin registration via `@opencode-ai/plugin`
  - Hook handlers: `chat.message`, `event`
  - Tool registration: `mem0` tool exposed to LLM
  - Context injection into conversation parts

## Data Storage

**Primary Storage:**
- **Mem0 Cloud API** - All memories stored remotely via Mem0 Platform
- **Scope Isolation:** User-scoped and project-scoped memories via user_id formatting
- **Metadata:** Memory type, sector, tags, project_id, source tracked

**Local Configuration:**
- **Location:** `~/.config/opencode/mem0.jsonc`
- **Contents:** API credentials, thresholds, limits, keyword patterns
- **Format:** JSON with comments (JSONC)

**No Local Database:**
- No SQLite, PostgreSQL, or other local database
- No file-based memory storage
- All persistence handled by Mem0 API

## Authentication & Identity

**Mem0 API Authentication:**
- **Method:** API Token in Authorization header
- **Source Priority:**
  1. `apiKey` field in `mem0.jsonc`
  2. `MEM0_API_KEY` environment variable
  3. `OPENMEMORY_API_KEY` environment variable (legacy)
- **Key Format:** `m0-...` (Mem0 platform keys)

**Workspace Routing (Optional):**
- `org_id` - Organization ID for multi-tenant workspaces
- `project_id` - Project ID for workspace segmentation
- Configured via config file or environment variables

**User Identification:**
- Scoped user IDs: `{prefix}:{userId}` or `{prefix}:{userId}:{projectId}`
- Prefix: "opencode" (configurable via `scopePrefix`)

## Memory Organization

**Hierarchical Semantic Graph (HSG) Sectors:**
- `episodic` - Events, experiences, temporal sequences
- `semantic` - Facts, concepts, general knowledge (default)
- `procedural` - Skills, how-to knowledge, processes
- `emotional` - Feelings, sentiments, reactions
- `reflective` - Meta-cognition, insights, patterns

**Memory Types:**
- `project-config` - Tech stack, commands, tooling
- `architecture` - Codebase structure, components
- `learned-pattern` - Code conventions specific to codebase
- `error-solution` - Known issues and fixes
- `preference` - Coding style preferences
- `conversation` - Session summaries

## OpenCode Integration Details

**Plugin Registration:**
- Registered in `~/.config/opencode/opencode.jsonc` via `plugin` array
- Auto-registration via CLI installer

**Hooks:**
1. **`chat.message`** (`src/index.ts:156`)
   - Injects memory context on first message of session
   - Detects memory trigger phrases ("remember this", "note that", etc.)
   - Adds synthetic parts to conversation output

2. **`event`** (`src/index.ts:623`)
   - Handles context compaction events
   - Saves conversation summaries as memories

**Tool Exposure:**
- Tool name: `mem0`
- Modes: add, search, profile, list, forget, feedback, help
- Arguments validated via Zod schemas

**Context Injection:**
- User profile (static + dynamic facts)
- Relevant user memories (semantic search)
- Recent project memories (list operation)
- Injected as synthetic text parts at conversation start

## Trigger Phrases

**Built-in Memory Keywords (Regex):**
```regex
\b(remember|memorize|save\s+this|note\s+this|keep\s+in\s+mind|don'?t\s+forget|learn\s+this|store\s+this|record\s+this|make\s+a\s+note|take\s+note|jot\s+down|commit\s+to\s+memory|remember\s+that|never\s+forget|always\s+remember)\b
```

**Custom Patterns:**
- Configurable via `keywordPatterns` array in `mem0.jsonc`
- Merged with built-in pattern using OR logic

## CI/CD & Deployment

**GitHub Actions (`.github/workflows/release.yml`):**
- Trigger: Tag push matching `v*`
- Steps:
  1. Checkout code
  2. Setup Bun with npm registry
  3. `bun install`
  4. `bun run typecheck`
  5. `bun run build`
  6. `npm publish --access public`

**Publishing:**
- **Registry:** npm (https://registry.npmjs.org/)
- **Access:** Public
- **Auth:** `NPM_TOKEN` secret
- **OIDC:** Enabled for secure publishing

## Environment Configuration

**Required for Operation:**
- `MEM0_API_KEY` - Valid Mem0 API key (or config file equivalent)

**Optional Overrides:**
- `MEM0_API_URL` - Custom API endpoint
- `MEM0_ORG_ID` / `MEM0_PROJECT_ID` - Workspace routing

**Configuration File (`~/.config/opencode/mem0.jsonc`):**
```jsonc
{
  "apiKey": "m0-...",
  "apiUrl": "https://api.mem0.ai",
  "orgId": "",
  "projectId": "",
  "keywordPatterns": [],
  "filterPrompt": "",
  "compactionThreshold": 0.8,
  "similarityThreshold": 0.6,
  "minSalience": 0.3,
  "maxMemories": 5,
  "maxProjectMemories": 10,
  "maxProfileItems": 5,
  "injectProfile": true,
  "scopePrefix": "opencode",
  "defaultSector": "semantic"
}
```

## Oh My OpenCode Compatibility

**Detection:**
- Checks for `~/.config/opencode/oh-my-opencode.json`
- Checks main config for "oh-my-opencode" references

**Integration:**
- Disables `anthropic-context-window-limit-recovery` hook to avoid conflicts
- OpenMemory handles compaction independently

## Security Considerations

**API Key Handling:**
- Never logged or exposed in output
- Loaded from config file or environment
- Validated for placeholder values ("m0-your-api-key")

**Content Sanitization:**
- `src/services/privacy.ts` strips private content
- Detects fully private content and rejects storage
- Code blocks excluded from trigger detection

**Redirect Safety:**
- Cross-origin redirects rejected
- Only same-origin redirects followed

---

*Integration audit: 2026-04-01*
