---
project: awrit
---
# Testing Patterns

**Analysis Date:** 2026-09-01

## Test Framework

**Runner:**
- Bun built-in test runner (`bun:test`)
- No configuration file required — tests run directly via `bun test`

**Assertion Library:**
- Bun's built-in `expect` (compatible with Jest API)

**Run Commands:**
```bash
bun test                              # Run all tests
bun test src/keybindings.test.ts      # Run single test file
bun test --watch                      # Watch mode (not supported natively; rerun manually)
```

## Test File Organization

**Location:**
- Co-located with source files in the same directory
- Pattern: `src/{module}.test.ts` alongside `src/{module}.ts`

**Naming:**
- Test files use `.test.ts` suffix: `layout.test.ts`, `inputHandler.test.ts`
- Test files excluded from TypeScript compilation in `tsconfig.json:36`: `"src/**/*.test.ts"`

**Structure:**
```
src/
├── layout.ts                    # Source
├── layout.test.ts               # Co-located test
├── keybindings.ts
├── keybindings.test.ts
├── inputHandler.test.ts
├── fake-timers.test.ts          # Shared test utility
└── tty/
    ├── tmux.ts
    ├── tmux.test.ts
    ├── tmuxProtocol.ts
    ├── tmuxProtocol.test.ts
    ├── tmuxRenderer.ts
    ├── tmuxRenderer.test.ts
    ├── imageIds.ts
    └── imageIds.test.ts
```

## Test Structure

**Suite Organization:**
```typescript
import { describe, expect, test } from 'bun:test';
import { functionUnderTest } from './module';

describe('Module Name', () => {
  test('describes specific behavior', () => {
    // Arrange
    const input = createTestData();
    
    // Act
    const result = functionUnderTest(input);
    
    // Assert
    expect(result).toEqual(expectedValue);
  });
});
```

**Patterns:**
- One `describe` block per module or logical grouping
- Descriptive test names that state the expected behavior: `'parses forced tmux coordinate mode override'`
- Tests are isolated — no shared state between tests
- Use `beforeEach` for common setup within a describe block

**Setup/Teardown:**
```typescript
describe('Keybindings System', () => {
  beforeEach(() => {
    loadKeyBindings({ keybindings: {} }); // Reset state
  });

  test('parses simple keybinding', () => {
    // ...
  });
});
```

## Mocking

**Framework:** None — tests use real implementations and filesystem

**Patterns:**
- **Filesystem mocking via temp directories:**
```typescript
import { mkdtempSync, rmSync, writeFileSync } from 'node:fs';
import { join } from 'node:path';
import { tmpdir } from 'node:os';

let fixtureDir = '';

beforeEach(() => {
  fixtureDir = mkdtempSync(join(tmpdir(), 'awrit-test-'));
});

afterEach(() => {
  rmSync(fixtureDir, { recursive: true, force: true });
});
```

- **Environment variable mocking:**
```typescript
const originalPath = process.env.PATH;
const originalPane = process.env.TMUX_PANE;

beforeEach(() => {
  process.env.TMUX_PANE = '%12345';
});

afterEach(() => {
  if (originalPane == null) {
    delete process.env.TMUX_PANE;
  } else {
    process.env.TMUX_PANE = originalPane;
  }
});
```

- **Class internal access via `as any`:**
```typescript
const renderer = new TmuxRenderer() as any;
renderer.outputFd = fd;
renderer.pendingBySlot.set(7, { slotId: 7 });
```

**What to Mock:**
- Environment variables (use `process.env` manipulation with restore)
- Filesystem operations (use temp directories, not mocking libraries)
- External process calls (create fake executables in temp directories)

**What NOT to Mock:**
- Core business logic functions
- Data structures and pure functions
- Internal state management

## Fixtures and Factories

**Test Data:**
- Inline fixture objects, not factory libraries:
```typescript
const termSize = { cols: 100, rows: 40, width: 1000, height: 800 };

const event: TermEvent = {
  eventType: 'key',
  keyEvent: {
    code: 's',
    modifiers: ['ctrl'],
    down: true,
    isCharEvent: false,
  },
};
```

- Helper functions for common patterns:
```typescript
function paneState(status: TmuxPaneStatus, viewers = 'client-a') {
  return { status, viewers: status === 'invisible' ? '' : viewers };
}
```

- Real binary data for graphics tests:
```typescript
const png = Buffer.alloc(6400, 0x41);
```

**Location:**
- Test fixtures defined inline at the top of test files
- Shared utilities in `src/fake-timers.test.ts`

## Fake Timers

**Shared Utility (`src/fake-timers.test.ts`):**
```typescript
import { install } from '@sinonjs/fake-timers';
import { afterAll, beforeEach } from 'bun:test';

export function fakeTimers() {
  const clock = install();

  beforeEach(() => {
    clock.reset();
  });

  afterAll(() => {
    clock.uninstall();
  });

  return clock;
}
```

**Usage:**
```typescript
import { fakeTimers } from './fake-timers.test';

describe('Keybindings', () => {
  const clock = fakeTimers();

  test('handle partial matches with timeout', async () => {
    // ...setup...
    await clock.tickAsync(500);
    expect(triggered).toBe('menu');
  });
});
```

**Clock API:**
- `clock.tickAsync(ms)` — advance time asynchronously
- `clock.reset()` — called automatically in `beforeEach`
- `clock.uninstall()` — called automatically in `afterAll`

## Coverage

**Requirements:** None enforced

**View Coverage:**
```bash
# Bun does not have built-in coverage reporting
# Use external tools if needed:
bun test --coverage  # Not available; use c8 or similar
```

## Test Types

**Unit Tests:**
- Pure function testing with no side effects
- Example: `src/layout.test.ts` tests layout calculation functions
- Example: `src/tty/imageIds.test.ts` tests ID allocator

**Integration Tests:**
- Tests that involve filesystem I/O, environment variables, or child processes
- Example: `src/tty/tmux.test.ts` creates fake `tmux` executable and tests interaction
- Example: `src/tty/tmuxRenderer.test.ts` tests rendering with real file descriptors

**E2E Tests:**
- Not present in this codebase
- Would require full Electron + terminal environment

## Common Patterns

**Async Testing:**
```typescript
test('refreshes pane size after cache TTL expires', async () => {
  process.env.TMUX_PANE = `%${Date.now()}2`;
  
  expect(getPaneSize()).toEqual({ cols: 120, rows: 40 });
  await clock.tickAsync(251);
  expect(getPaneSize()).toEqual({ cols: 120, rows: 40 });
});
```

**Error Testing:**
```typescript
test('rejects invalid TMUX_PANE values before invoking tmux', () => {
  process.env.TMUX_PANE = ';rm -rf /';
  
  expect(() => getPaneSize()).toThrow('Invalid TMUX_PANE value');
});
```

**File System Verification:**
```typescript
test('reads the effective pane-scoped passthrough value', () => {
  process.env.TMUX_PANE = `%${Date.now()}3`;
  
  expect(getAllowPassthrough()).toBe('all');
  expect(readFileSync(logPath, 'utf8')).toContain('show-options -p -A -v -t');
});
```

**State Restoration Pattern:**
```typescript
describe('tmux helpers', () => {
  const originalPath = process.env.PATH;
  const originalPane = process.env.TMUX_PANE;

  afterEach(() => {
    // Restore each env var to its original state (or delete if not set)
    if (originalPath == null) {
      delete process.env.PATH;
    } else {
      process.env.PATH = originalPath;
    }
    // ... repeat for each env var
  });
});
```

**Async Cleanup Pattern:**
```typescript
describe('TmuxRenderer', () => {
  const openedFds: number[] = [];
  const tempDirs: string[] = [];

  afterEach(() => {
    while (openedFds.length > 0) {
      const fd = openedFds.pop();
      if (fd != null) closeSync(fd);
    }
    while (tempDirs.length > 0) {
      const dir = tempDirs.pop();
      if (dir) rmSync(dir, { recursive: true, force: true });
    }
  });
});
```

**Typed Assertion Helpers:**
```typescript
function countLinesContaining(text: string, needle: string) {
  return text.split('\n').filter((line) => line.includes(needle)).length;
}
```

---

*Testing analysis: 2026-09-01*
