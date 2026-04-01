# opencode-mem0

## What This Is

An OpenCode plugin that provides persistent memory capabilities using the Mem0 Platform API. It automatically injects context (user profile, project knowledge, relevant memories) into agent conversations and captures knowledge explicitly via trigger phrases like "remember this".

The plugin bridges OpenCode agents with Mem0's hosted memory backend, enabling cross-session continuity of knowledge and project-specific context awareness.

## Core Value

**Memories are reliably captured and retrieved to make every conversation feel continuous and contextually aware.**

If memory persistence fails or context feels "fresh" every session, the plugin has failed its primary purpose.

## Requirements

### Validated

- ✓ Automatic context injection on first message of each session
- ✓ Memory keyword detection and nudging (remember, save this, note that, etc.)
- ✓ Tool-based memory operations (add, search, list, forget, feedback, help)
- ✓ Scope separation (user vs project)
- ✓ HSG sector support (episodic, semantic, procedural, emotional, reflective)
- ✓ Context compaction with summary preservation
- ✓ Privacy filtering (<private> tag stripping)
- ✓ CLI installer for plugin registration

### Active

- [ ] Diagnose and resolve 'remember xxx' persistence failures
- [ ] Improve memory-write reliability and observability
- [ ] Align user expectations with actual memory behavior
- [ ] Enhance error visibility when Mem0 API calls fail

### Out of Scope

- Local/self-hosted memory backend — Mem0 Platform API is the supported backend
- Memory migration from other backends — manual export/import only
- Real-time collaborative memory — single-user scope only
- Memory versioning/history — Mem0 handles this internally

## Context

**Technical Environment:**
- TypeScript/Bun-based OpenCode plugin
- Mem0 Platform API (`https://api.mem0.ai`) as hosted backend
- Uses Hierarchical Semantic Graph (HSG) for memory organization
- SHA-256 hashed identifiers for privacy

**Known Issues to Address:**
- User reports 'remember xxx' commands may not persist memories
- User is NOT using OpenCode's compaction (using third-party plugin instead)
- User is NOT using Claude model (affects compaction behavior)
- Need to investigate: API call failures, async timing, scope routing, feedback visibility

**User Preferences:**
- Practical, prioritized advice over exhaustive documentation
- Focus on memory-write reliability and observability
- Read-only architecture/product recommendations (no exact code diffs)
- Want to understand why memories may not persist

**Prior Work:**
- Forked from happycastle114/opencode-openmemory
- Originally adapted from opencode-supermemory
- Existing codebase map in `.planning/codebase/`

## Constraints

- **Tech Stack**: Must remain compatible with OpenCode plugin SDK (`@opencode-ai/plugin`)
- **Backend**: Mem0 Platform API only — no local fallback
- **Privacy**: Cannot store raw PII; identifiers must be hashed
- **Compatibility**: Must work with non-Claude models (affects compaction logic)
- **Async Behavior**: Memory operations are async; failures may be silent

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Use Mem0 Platform API vs local storage | Hosted vector search, embeddings, HSG | — Pending validation of reliability |
| SHA-256 hashing for scope IDs | Privacy compliance, stable identifiers | ✓ Good |
| Tool-based memory capture vs automatic | Explicit control, privacy preservation | ✓ Good |
| Compaction only for Claude models | Model-specific summarization behavior | ⚠️ Revisit for non-Claude users |

## Evolution

This document evolves at phase transitions and milestone boundaries.

**After each phase transition** (via `/gsd-transition`):
1. Requirements invalidated? → Move to Out of Scope with reason
2. Requirements validated? → Move to Validated with phase reference
3. New requirements emerged? → Add to Active
4. Decisions to log? → Add to Key Decisions
5. "What This Is" still accurate? → Update if drifted

**After each milestone** (via `/gsd-complete-milestone`):
1. Full review of all sections
2. Core Value check — still the right priority?
3. Audit Out of Scope — reasons still valid?
4. Update Context with current state

---
*Last updated: 2025-04-01 after initialization*
