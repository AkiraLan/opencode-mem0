# Testing Patterns

**Analysis Date:** 2025-04-01

**Overview:** This document describes the testing strategy, infrastructure, and patterns for the opencode-mem0 project. Currently, the project relies on TypeScript's strict type checking and manual testing rather than automated test suites.

---

## Current Testing Status

**No Automated Tests Present**

The codebase currently has:
- **0 test files** (no `.test.ts`, `.spec.ts`, or similar)
- **0 test frameworks configured** (no Jest, Vitest, Mocha, etc.)
- **0 test scripts** in `package.json`

---

## Testing Strategy

### Primary: TypeScript Strictness

The project relies heavily on TypeScript's static type checking as its primary quality assurance mechanism.

**Type Checking Command:**
```bash
bun run typecheck    # Runs: tsc --noEmit
```

**Compiler Configuration** (`tsconfig.json`):
- `strict: true` - All strict type-checking options enabled
- `noUncheckedIndexedAccess: true` - Requires explicit undefined checks
- `noFallthroughCasesInSwitch: true` - Prevents accidental switch fallthrough
- `noImplicitOverride: true` - Requires explicit override declarations
- `skipLibCheck: true` - Ignores type errors in dependencies

**What this catches:**
- Type mismatches
- Undefined/null access
- Missing return values
- Unused variables (when enabled)
- Incorrect property access

### Secondary: Manual Testing

**Build Verification:**
```bash
bun run build        # Compiles and bundles the project
```

**Build output:**
- `dist/index.js` - Main plugin bundle
- `dist/cli.js` - CLI executable
- `dist/*.d.ts` - Type declaration files

### Development Mode

**Watch Mode:**
```bash
bun run dev          # Runs: tsc --watch
```

---

## Testing Infrastructure Gaps

### Missing: Unit Tests

**Areas that would benefit from unit testing:**

1. **JSONC Parsing** (`src/services/jsonc.ts`):
   - Comment stripping edge cases
   - Trailing comma removal
   - Nested string handling
   - Escaped quote handling

2. **Configuration Loading** (`src/config.ts`):
   - Default value application
   - Environment variable precedence
   - Validation logic (unit intervals, positive integers)
   - Invalid config handling

3. **Privacy Filtering** (`src/services/privacy.ts`):
   - `<private>` tag detection
   - Content redaction
   - Fully private content detection

4. **Tag Generation** (`src/services/tags.ts`):
   - SHA256 hashing consistency
   - Git email extraction
   - Fallback user ID generation

5. **Context Formatting** (`src/services/context.ts`):
   - Memory formatting with scores
   - Empty result handling
   - Profile injection logic

### Missing: Integration Tests

**Critical integration points:**

1. **Mem0 API Client** (`src/services/client.ts`):
   - API authentication
   - Request/response handling
   - Timeout behavior
   - Redirect handling
   - Error response parsing

2. **Plugin Lifecycle** (`src/index.ts`):
   - Plugin initialization
   - Hook registration
   - Tool execution
   - Memory keyword detection
   - Context injection

3. **Compaction System** (`src/services/compaction.ts`):
   - Token threshold detection
   - Message injection
   - Summary capture
   - Memory storage

4. **CLI Installer** (`src/cli.ts`):
   - Config file parsing
   - Plugin registration
   - File operations
   - Interactive prompts

### Missing: End-to-End Tests

**User workflow scenarios:**

1. Fresh installation and setup
2. Memory capture via trigger phrases
3. Context injection on first message
4. Compaction trigger at context limit
5. Tool execution (add, search, list, forget)

---

## Recommended Testing Framework

**Recommendation: Vitest**

**Rationale:**
- Native ESM support (matches project configuration)
- TypeScript support out of the box
- Fast execution ( aligns with Bun philosophy)
- Built-in mocking capabilities
- Compatible with Bun runtime

**Alternative: Node.js Test Runner**
- Built into Node.js 18+
- No additional dependencies
- Native ESM support
- Simpler but less feature-rich

---

## Testing Patterns to Implement

### Pattern 1: Result Type Testing

```typescript
// For functions returning Result types
describe("addMemory", () => {
  it("should return success when API call succeeds", async () => {
    const result = await client.addMemory("test", scope);
    expect(result.success).toBe(true);
    expect(result.id).toBeDefined();
  });

  it("should return error when API call fails", async () => {
    // Mock failure
    const result = await client.addMemory("test", scope);
    expect(result.success).toBe(false);
    expect(result.error).toBeDefined();
  });
});
```

### Pattern 2: Type Guard Testing

```typescript
// For type guard functions
describe("isRecord", () => {
  it("should return true for objects", () => {
    expect(isRecord({})).toBe(true);
    expect(isRecord({ key: "value" })).toBe(true);
  });

  it("should return false for non-objects", () => {
    expect(isRecord(null)).toBe(false);
    expect(isRecord("string")).toBe(false);
    expect(isRecord(123)).toBe(false);
  });
});
```

### Pattern 3: Configuration Testing

```typescript
// For configuration validation
describe("readUnitInterval", () => {
  it("should accept valid values", () => {
    expect(readUnitInterval(0, 0.5)).toBe(0);
    expect(readUnitInterval(1, 0.5)).toBe(1);
    expect(readUnitInterval(0.5, 0.5)).toBe(0.5);
  });

  it("should reject out-of-range values", () => {
    expect(readUnitInterval(-0.1, 0.5)).toBe(0.5);
    expect(readUnitInterval(1.1, 0.5)).toBe(0.5);
  });

  it("should use fallback for non-numbers", () => {
    expect(readUnitInterval("0.5", 0.5)).toBe(0.5);
    expect(readUnitInterval(null, 0.5)).toBe(0.5);
  });
});
```

### Pattern 4: Regex Pattern Testing

```typescript
// For keyword detection
describe("detectMemoryKeyword", () => {
  it("should detect built-in keywords", () => {
    expect(detectMemoryKeyword("Please remember this")).toBe(true);
    expect(detectMemoryKeyword("Don't forget to...")).toBe(true);
    expect(detectMemoryKeyword("Save this for later")).toBe(true);
  });

  it("should not detect keywords in code blocks", () => {
    expect(detectMemoryKeyword("```\nremember this\n```")).toBe(false);
  });

  it("should detect custom patterns", () => {
    // With custom pattern configured
    expect(detectMemoryKeyword("Make a note of this")).toBe(true);
  });
});
```

### Pattern 5: Privacy Filter Testing

```typescript
describe("stripPrivateContent", () => {
  it("should remove private tags", () => {
    const input = "Public content <private>secret</private> more public";
    expect(stripPrivateContent(input)).toBe("Public content [REDACTED] more public");
  });

  it("should handle multiline private content", () => {
    const input = "Start <private>\nsecret\ncontent\n</private> End";
    expect(stripPrivateContent(input)).toBe("Start [REDACTED] End");
  });

  it("should detect fully private content", () => {
    expect(isFullyPrivate("<private>secret</private>")).toBe(true);
    expect(isFullyPrivate("<private>secret</private> public")).toBe(false);
  });
});
```

---

## Mocking Strategy

### External Dependencies to Mock

1. **Mem0 API** (`fetch` global):
```typescript
// Mock fetch for API testing
global.fetch = vi.fn().mockResolvedValue({
  ok: true,
  json: () => Promise.resolve({ results: [] }),
});
```

2. **File System** (`node:fs`):
```typescript
// Mock fs for config testing
vi.mock("node:fs", () => ({
  existsSync: vi.fn(),
  readFileSync: vi.fn(),
  writeFileSync: vi.fn(),
}));
```

3. **Git Commands** (`node:child_process`):
```typescript
// Mock execSync for git email extraction
vi.mock("node:child_process", () => ({
  execSync: vi.fn().mockReturnValue("user@example.com"),
}));
```

4. **Environment Variables:**
```typescript
// Mock process.env
const originalEnv = process.env;
beforeEach(() => {
  process.env = { ...originalEnv, MEM0_API_KEY: "test-key" };
});
afterEach(() => {
  process.env = originalEnv;
});
```

---

## Test File Organization

**Recommended structure:**
```
src/
├── __tests__/
│   ├── unit/
│   │   ├── jsonc.test.ts
│   │   ├── privacy.test.ts
│   │   ├── tags.test.ts
│   │   └── config.test.ts
│   ├── integration/
│   │   ├── client.test.ts
│   │   └── plugin.test.ts
│   └── fixtures/
│       └── mem0-responses.json
```

**Alternative (co-located):**
```
src/
├── services/
│   ├── client.ts
│   ├── client.test.ts
│   ├── jsonc.ts
│   ├── jsonc.test.ts
│   └── ...
```

---

## CI/CD Testing

**GitHub Actions workflow** (recommended):
```yaml
name: Test
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: oven-sh/setup-bun@v1
      - run: bun install
      - run: bun run typecheck
      - run: bun test
      - run: bun run build
```

---

## Code Coverage

**Target Coverage Goals:**
- **Critical paths**: 90%+ (API client, plugin hooks, compaction)
- **Utilities**: 80%+ (parsers, formatters, filters)
- **CLI**: 70%+ (installer, config management)

**Coverage tools:**
- Vitest has built-in coverage (via `v8` or `istanbul`)
- Report generation: `bun test --coverage`

---

## Manual Testing Checklist

**Before Release:**

1. **Build verification:**
   - [ ] `bun run build` completes without errors
   - [ ] `dist/index.js` exists and is valid
   - [ ] `dist/cli.js` exists and is executable
   - [ ] Type declarations generated

2. **Plugin functionality:**
   - [ ] Plugin loads without errors
   - [ ] Memory keywords detected
   - [ ] Context injected on first message
   - [ ] Tool commands work

3. **CLI installer:**
   - [ ] Interactive mode works
   - [ ] Non-interactive mode works (`--no-tui`)
   - [ ] Config files created correctly
   - [ ] Plugin registration successful

4. **Edge cases:**
   - [ ] Empty configuration handled
   - [ ] Missing API key handled gracefully
   - [ ] Network errors handled
   - [ ] Invalid regex patterns handled

---

## Risk Assessment

**High Risk (no test coverage):**
1. Mem0 API integration - Network failures, API changes
2. Context compaction - Data loss potential
3. Privacy filtering - Security implications
4. Configuration loading - Wrong defaults, validation failures

**Medium Risk:**
1. JSONC parsing - Edge cases in comment handling
2. Tag generation - Hash collisions, identification issues
3. CLI installer - File permission issues, config corruption

**Low Risk:**
1. Type definitions - Compile-time checked
2. Simple utilities - Type-safe by design

---

## Next Steps for Testing

**Phase 1: Unit Tests (Critical)**
1. Set up Vitest
2. Test `jsonc.ts` parsing functions
3. Test `privacy.ts` filtering
4. Test `config.ts` validation
5. Test `tags.ts` generation

**Phase 2: Integration Tests**
1. Mock Mem0 API client
2. Test plugin initialization
3. Test tool execution
4. Test compaction flow

**Phase 3: E2E Tests**
1. Test CLI installation flow
2. Test full user workflow
3. Test error scenarios

---

*Testing analysis: 2025-04-01*
