# Vitest 3.2 Production Rules

## 1. Configuration & Architecture

### Project Structure (CRITICAL - Breaking Change in 3.2)
- **MUST** migrate from `workspace` to `projects` configuration - workspace is deprecated and will be removed in v4.
- **MUST** use `defineConfig()` and `defineProject()` from TypeScript for type safety and autocomplete.
- **MUST** assign unique names to all projects - required for `--project` CLI filtering and execution control.
- **MUST** use `mergeConfig()` when sharing configuration across projects - projects can't extend root config directly as they'd inherit the projects array itself.
- **MUST** use glob patterns (`packages/*`) for monorepo auto-discovery to reduce maintenance burden.

### Configuration Best Practices
- **MUST** create `vitest.shared.ts` for common configuration and compose with `mergeConfig()` in individual project configs.
- **MUST** use specific config file patterns (`vitest.config.{e2e,unit}.ts`) to separate test types within packages.
- **MUST** control project execution order with `sequence.groupOrder` - projects with same group number run in parallel; groups execute lowest to highest.
- **MUST** use `extends: true` in inline project configs to inherit root-level settings.
- **MUST** write configs in TypeScript (`vitest.config.ts`) for type safety and IDE support.

---

## 2. Test Organization & Structure

### File Naming & Location
- **MUST** use `.test.{js,ts,jsx,tsx}` or `.spec.{js,ts,jsx,tsx}` extensions exclusively.
- **MUST** choose one consistent location strategy project-wide: collocated (next to source) for libraries, mirrored structure (`tests/`) for larger apps.
- **SHOULD** use type-based suffixes (`Button.unit.test.tsx`, `Button.integration.test.tsx`) when using projects configuration for precise filtering.

### Suite Organization
- **MUST** use nested `describe` blocks to create hierarchical, self-documenting test structure.
- **MUST** use descriptive "should" or "when/then" test naming patterns.
- **MUST** use `describe` modifiers strategically: `.skip` for temporary disabling, `.only` for focus during development, `.concurrent` for async-heavy tests, `.sequential` when tests have shared state.

### Hooks & Lifecycle
- **MUST** use `beforeEach`/`afterEach` for test isolation setup; `beforeAll`/`afterAll` for expensive one-time setup only.
- **MUST** always clean up resources in `afterEach` - prevents test pollution and flaky tests.
- **SHOULD** return cleanup functions from `beforeEach`/`beforeAll` instead of separate `afterEach` hooks for better locality.
- **MUST** configure `mockReset: true` and `unstubEnvs: true` in config for automatic cleanup between tests.

---

## 3. Assertions & Matchers

### Async Assertions (CRITICAL)
- **MUST** always `await` promise assertions (`expect(...).resolves`, `expect(...).rejects`) - unawaited assertions warn in v3 and will fail in v4.
- **MUST** pass promises directly to `.rejects`, not wrapped in arrow functions.
- **MUST** wrap synchronous throwing functions in arrow functions for `toThrow()`.

### Assertion Selection
- **MUST** use `toBeCloseTo()` for floating-point comparisons - strict equality fails due to precision issues.
- **MUST** use `toStrictEqual()` over `toEqual()` when type safety matters - catches undefined properties and type mismatches.
- **MUST** use `expect.soft()` for collecting multiple independent assertion failures in one test run.
- **MUST** use semantic matchers (`toContain`, `toHaveProperty`, `toMatch`) over manual comparisons for better error messages.

### Polling & Eventually-Consistent Assertions
- **MUST** use `expect.poll()` for eventually-consistent values (DOM updates, async operations with unpredictable timing).
- **MUST** always `await` `expect.poll()` calls.
- **MUST** configure appropriate `interval` and `timeout` for `expect.poll()` based on operation characteristics.
- **NEVER** use snapshot matchers, `.resolves/.rejects`, or `toThrow` with `expect.poll()` - not supported.

### Custom Matchers (Vitest 3.2+)
- **MUST** extend unified `Matchers` interface in TypeScript for type-safe custom matchers:
```typescript
declare module 'vitest' {
  interface Matchers<T = any> extends CustomMatchers<T> {}
}
```
- **NEVER** mutate `pass` based on `this.isNot` in custom matchers - Vitest handles negation automatically.

### Type Testing
- **MUST** use `expectTypeOf()` or `assertType()` for type-level assertions, not runtime `expect()`.
- **MUST** use `expectTypeOf` for detailed type checks; `assertType` for simple type verification.

### Concurrent Test Context
- **MUST** destructure `expect` from test context when using `test.concurrent` to ensure correct snapshot association:
```typescript
test.concurrent('test', async ({ expect }) => { ... })
```

### Assertion Verification
- **MUST** use `expect.assertions(n)` or `expect.hasAssertions()` in tests with conditional assertions or callbacks to prevent false positives.

---

## 4. Mocking & Test Isolation

### Module Mocking Strategy
- **MUST** use `vi.mock()` for complete module replacement - hoisted and affects all imports consistently.
- **MUST** use `vi.hoisted()` when referencing variables in `vi.mock()` factories - `vi.mock()` is hoisted before variable declarations.
- **MUST** use `vi.importActual()` with spread for partial module mocking - preserves real implementations:
```typescript
vi.mock('./module', async () => ({
  ...(await vi.importActual<typeof import('./module')>('./module')),
  specificExport: vi.fn()
}))
```
- **MUST** use `vi.doMock()` with dynamic imports when runtime/per-test mock control is needed - not hoisted like `vi.mock()`.
- **NEVER** use `vi.spyOn()` for partial module mocks - creates confusing behavior.

### Spy & Mock Function Usage
- **MUST** use `vi.spyOn()` for observable side effects without changing behavior - preserves real implementation by default.
- **MUST** use `vi.fn()` for pure mock functions with full control over implementation.
- **MUST** chain return values with `mockReturnValueOnce()`, `mockResolvedValueOnce()`, `mockRejectedValueOnce()` for sequential behavior.
- **MUST** use `vi.mocked()` helper in TypeScript for proper type inference on mocked imports.

### Timer Mocking
- **MUST** wrap fake timers in `beforeEach(vi.useFakeTimers)` and `afterEach(vi.useRealTimers)` to prevent state bleeding.
- **MUST** use `vi.advanceTimersByTime(ms)` over `vi.runAllTimers()` for predictable control - `runAllTimers` can cause infinite loops with recursive timers.
- **MUST** use async timer methods (`vi.advanceTimersByTimeAsync()`, `vi.runAllTimersAsync()`) when timer callbacks contain promises.
- **MUST** use `vi.setSystemTime(date)` to control `Date.now()` and `new Date()` independently of timers.
- **MUST** explicitly enable `nextTick` and `queueMicrotask` mocking with `toFake` option when testing microtasks:
```typescript
vi.useFakeTimers({ toFake: ['nextTick', 'queueMicrotask'] })
```

### Mock Cleanup (CRITICAL)
- **MUST** use `vi.clearAllMocks()` or `clearMocks: true` in config between tests - clears call history only, safest cleanup.
- **MUST** use `vi.restoreAllMocks()` or `restoreMocks: true` when using `vi.spyOn()` - restores original implementations.
- **NEVER** use `vi.resetAllMocks()` or `mockReset: true` unless explicitly needing to remove implementations - resets mocks to `vi.fn()` returning `undefined`, breaking most tests.
- **MUST** understand cleanup hierarchy: `clear` < `reset` < `restore` in destructiveness.

### Network Mocking
- **MUST** use Mock Service Worker (MSW) for HTTP/API mocking instead of mocking fetch/axios directly.
- **MUST** configure MSW with three-phase lifecycle: `beforeAll(server.listen)`, `afterEach(server.resetHandlers)`, `afterAll(server.close)`.
- **MUST** set `onUnhandledRequest: 'error'` in MSW configuration to catch missed mocks early.

### Test Isolation Configuration
- **MUST** keep `test.isolate: true` (default) unless code is side-effect-free and profiling confirms isolation is the bottleneck.
- **MUST** use worker-scoped fixtures with `isolate: false` for expensive setup (database connections) to benefit from shared resources.
- **NEVER** disable isolation without thorough testing - prevents state pollution between tests.

### Dependency Injection
- **MUST** prefer dependency injection (passing dependencies as parameters) over module mocking when possible.
- **MUST** use factory functions for complex test dependencies with sensible defaults.
- **MUST** leverage Vitest's test context API for per-test or per-suite fixtures instead of module-level variables.

---

## 5. Async Testing

### Async/Await Patterns
- **MUST** always use async functions for async tests - cleaner than returning promises.
- **MUST** always `await` promise assertions (`expect(...).resolves`, `expect(...).rejects`).
- **MUST** use `expect.poll()` for eventually-consistent conditions with retry logic.
- **MUST** use `vi.waitFor()` for complex async conditions not suitable for `expect.poll()`.

### Timeout Configuration
- **MUST** configure timeouts at appropriate levels: global in config, per-test as third argument.
- **MUST** set realistic timeouts for flaky tests - too short causes false negatives, too long slows CI.
- **MUST** use different timeouts for CI vs local: `process.env.CI ? 10000 : 5000`.
- **MUST** use AbortSignal for timeout-aware resource cleanup (Vitest 3.2+):
```typescript
test('request', async ({ signal }) => {
  await fetch('/endpoint', { signal })
}, 2000)
```

### Retry Logic
- **MUST** use retry sparingly and only in CI: `retry: process.env.CI ? 3 : 0`.
- **MUST** understand retry limitations - doesn't fix shared state, global mocks, or event listeners.
- **MUST** fix root causes instead of masking with retries.
- **MUST** disable retries for accurate flaky test detection.

### Concurrent Execution
- **MUST** use test context `expect` and `onTestFinished` for concurrent tests to prevent test interference:
```typescript
test.concurrent('test', async ({ expect, onTestFinished }) => { ... })
```
- **MUST** only use `.concurrent` for async-heavy tests (I/O, API calls, database queries) - no benefit for CPU-bound tests.
- **MUST** configure `maxConcurrency` based on resource limits (database connections, API rate limits).
- **MUST** use `test.sequential` within concurrent suites for tests that can't run in parallel.

### Fake Timers with Async
- **MUST** restore real timers after tests to prevent cross-test contamination.
- **MUST** use async timer methods (`*Async`) for promise-based timer callbacks.
- **MUST** call `vi.runOnlyPendingTimers()` before `vi.useRealTimers()` to flush pending timers.

---

## 6. Performance Optimization

### Test Execution Speed
- **MUST** disable test isolation for pure functions: `test.isolate: false` (only if verified safe).
- **MUST** use `pool: 'threads'` instead of default `forks` for better performance.
- **MUST** disable file parallelism when startup overhead dominates: `fileParallelism: false`.
- **MUST** enable dependency optimization for large test suites: `deps.optimizer.web.enabled: true`.
- **MUST** use V8 coverage provider (default in 3.2) - combines V8 speed with Istanbul accuracy via AST remapping.
- **MUST** define `coverage.include` patterns to limit file scanning.
- **NEVER** import from barrel files in tests - import directly from specific modules to avoid unnecessary parsing.

### Parallel Execution
- **MUST** use test sharding (`--shard=X/Y`) when total test time exceeds 5-10 minutes in CI.
- **MUST** limit threads in CI to `cpus / 4` to avoid memory pressure and context switching:
```typescript
const osThreads = process.env.CI ? cpus().length / 4 : undefined
```
- **MUST** use `test.concurrent` for I/O-heavy tests only.

### Watch Mode
- **MUST** use `watchTriggerPatterns` (Vitest 3.2) for file-based test reruns on non-import dependencies:
```typescript
test: {
  watchTriggerPatterns: ['config/*.json', 'fixtures/**/*.yml']
}
```
- **MUST** use `vitest related <files>` to test only affected code during development.
- **MUST** use `--standalone` for background test runners to keep process warm.

---

## 7. CI/CD Integration

### Pipeline Configuration
- **MUST** use blob reporter with merge strategy for sharded runs:
```bash
vitest run --reporter=blob --shard=1/4
# After all shards: vitest --merge-reports
```
- **MUST** use `github-actions` reporter in GitHub Actions for inline PR annotations:
```typescript
reporters: process.env.GITHUB_ACTIONS ? ['dot', 'github-actions'] : ['dot']
```
- **MUST** use `--bail=1` for fail-fast behavior during development branches.
- **MUST** always use `npm ci` instead of `npm install` in CI pipelines.

### Coverage in CI
- **MUST** configure appropriate format for CI platform: `cobertura` for Azure DevOps, `json-summary` for most others.
- **MUST** merge coverage from sharded runs: `vitest --merge-reports --coverage`.
- **MUST** set coverage thresholds as quality gates:
```typescript
coverage: {
  thresholds: { lines: 80, functions: 80, branches: 75, statements: 80 }
}
```
- **MUST** set `coverage.reportOnFailure: true` in CI to generate reports even when tests fail.

### CI Best Practices
- **MUST** set headless mode explicitly in browser configs for CI: `headless: process.env.CI === 'true'`.
- **MUST** configure realistic timeouts for CI environments with variable latency.
- **MUST** use `npm ci` for deterministic, faster installs.

---

## 8. Browser Mode & Component Testing

### Browser Mode Setup (CRITICAL - Changed in 3.2)
- **MUST** explicitly enable browser mode in config: `browser.enabled: true` - `--browser` flag alone fails without config since 3.2.
- **MUST** use Playwright provider over WebdriverIO for parallel execution and better performance:
```typescript
browser: {
  enabled: true,
  provider: 'playwright',
  instances: [{ browser: 'chromium' }]
}
```
- **MUST** use workspace configuration to separate browser and Node.js tests with different settings.
- **MUST** use `vitest init browser` for proper automated setup.

### DOM Testing
- **MUST** always use `expect.element()` with page locators for built-in retry-ability:
```typescript
await expect.element(page.getByText('Alice')).toBeInTheDocument()
```
- **MUST** use Vitest's native locators (`page.getBy*()`) from `@vitest/browser/context`, not external query libraries.
- **MUST** always await user interactions - they're asynchronous via CDP/WebDriver.
- **MUST** use `@vitest/browser/context` userEvent, not `@testing-library/user-event` for actual browser APIs instead of fake events.

### Component Testing
- **MUST** use `fill()` over `type()` for input performance unless special keys needed (`{shift}`, `{selectall}`).
- **MUST** choose browser mode over JSDOM for layout calculations, CSS behavior, and advanced APIs (IntersectionObserver, ResizeObserver).
- **MUST** mock API calls, routing, and external services for isolated, deterministic browser tests.

### Visual Regression Testing (Vitest 4.0.0-beta.4+)
- **MUST** use native `toMatchScreenshot()` for visual regression:
```typescript
await expect.element(page.getByRole('main')).toMatchScreenshot('homepage', {
  threshold: 0.2,
  allowedMismatchedPixelRatio: 0.01
})
```
- **MUST** configure defaults in config for consistent behavior across suite.
- **MUST** generate reference screenshots in CI environment, not locally, to avoid font rendering differences.

### Accessibility Testing
- **MUST** use `vitest-axe` (not `jest-axe`) for Vitest-specific accessibility testing.
- **MUST** use with `jsdom` or browser mode, not `happy-dom` (known compatibility issue).
- **MUST** understand axe's 30-50% detection rate limitation - combine with manual testing.

---

## 9. Advanced Patterns & Enterprise Features

### Test Fixtures (Vitest 3.2+)
- **MUST** use worker-scoped fixtures for expensive resources (database connections) with `scope: 'worker'`.
- **MUST** disable isolation (`isolate: false`) to benefit from worker-scoped fixtures.
- **MUST** use file-scoped auto fixtures for `beforeAll`/`afterAll` behavior that only runs if used:
```typescript
const test = baseTest.extend({
  setup: [async ({}, use) => { await use() }, { scope: 'file', auto: true }]
})
```
- **MUST** type fixtures with TypeScript generics for IDE autocomplete and compile-time safety.
- **MUST** compose fixtures through extension for layered test abstractions.

### Global Setup & Teardown
- **MUST** use global setup for cross-test resources (test database, mock servers).
- **MUST** return teardown function from global setup for proper cleanup.
- **MUST** pass serializable data via `provide/inject` API - global setup runs in separate scope:
```typescript
export async function setup({ provide }) {
  provide('apiPort', String(port))
}
```
- **MUST** type-check injection with TypeScript `ProvidedContext` augmentation.

### Test Context & Injection
- **MUST** use test context over global variables for explicit dependency declaration.
- **MUST** use scoped context overrides (Vitest 3.1+) for environment-specific configurations:
```typescript
describe('staging', () => {
  test.scoped({ apiUrl: 'https://staging.api.com' })
})
```
- **MUST** mark fixtures with `{ injected: true }` for project-level overrides in monorepos.

### Tool Integration
- **MUST** use MSW at setup file level with three-phase lifecycle for network mocking.
- **MUST** use browser mode for Playwright integration, not dual test runners.
- **MUST** use workspace configuration for multi-environment tests in single command.

### Custom Test Environments
- **MUST** create custom environments for specialized setup (polyfills, domain-specific globals).
- **MUST** use per-file environment selection with `@vitest-environment` comments when mixing Node.js and browser tests.

### Vitest 3.2 Enterprise Features
- **MUST** use `watchTriggerPatterns` for monorepo optimization - automatically rerun dependent package tests when shared code changes.
- **MUST** use AbortSignal for resource cleanup on timeout, failure, or Ctrl+C:
```typescript
test('cleanup', async ({ signal }) => {
  await fetch('/endpoint', { signal })
})
```
- **MUST** use test annotations for debugging context in CI:
```typescript
task.meta.annotation = 'Known flaky - investigating'
```

### Performance at Scale
- **MUST** shard tests by files with `--shard=X/Y` for large suites in CI.
- **MUST** disable isolation with `pool: 'threads'` and worker-scoped fixtures for integration suites without mutable state.
- **MUST** disable file parallelism for startup-heavy projects after profiling with `--reporter=verbose`.

### Custom Matchers & Assertions
- **MUST** use unified `Matchers` interface (Vitest 3.2) for type-safe custom matchers.
- **MUST** leverage asymmetric matchers (`expect.any()`, `expect.objectContaining()`) for partial matching.

### Snapshot Testing
- **MUST** use inline snapshots (`toMatchInlineSnapshot()`) for small assertions (<10 lines).
- **MUST** use file snapshots with custom extensions (`.sql`, `.graphql`) for better IDE integration.
- **MUST** use custom serializers to normalize dynamic data (dates, UUIDs, paths) and prevent snapshot churn.

### Browser Mode Component Testing
- **MUST** use browser mode for real DOM APIs that JSDOM can't polyfill.
- **MUST** use framework-specific render utilities (`vitest-browser-react`, `vitest-browser-vue`) for integrated cleanup.

---

## 10. Critical Migration & Compatibility Notes

### Breaking Changes in Vitest 3.2
- **Workspace → Projects:** Mandatory migration. Workspace files deprecated, will be removed in v4.
- **Browser flag requires config:** `--browser` flag fails without `browser.enabled: true` in config.
- **AST-based V8 coverage:** Automatic in 3.2, experimental flag deprecated.
- **Unawaited promise assertions:** Warnings in v3, will fail in v4.

### New Features in Vitest 3.2
- **`sequence.groupOrder`:** Control multi-project execution order.
- **`watchTriggerPatterns`:** Selective test reruns on file changes.
- **AbortSignal API:** Test signal for timeout-aware resource cleanup.
- **Unified Matchers interface:** Single interface for all matcher contexts.
- **Scoped fixtures:** Worker and file scope for performance optimization.
- **Test annotations:** Debugging context in reporters.

### Known Limitations
- Browser mode locators only work with single elements - use `.all()` for multiple.
- Browser mode is maturing - expect edge cases, maintain JSDOM fallback for critical paths.
- Worker-scoped fixtures require `isolate: false` to benefit from shared resources.
- AbortSignal with `pool: 'threads'` and Node.js fetch may leak processes - use `pool: 'forks'` as workaround.

---

## Summary: Non-Negotiable Production Rules

1. **Migrate to `projects` configuration immediately** - workspace is deprecated.
2. **Always `await` promise assertions** - will fail in v4 if not awaited.
3. **Always clean up mocks and timers** - use `clearMocks: true`, `restoreMocks: true`, and proper timer lifecycle.
4. **Use test context for concurrent tests** - destructure `expect` and `onTestFinished`.
5. **Enable browser mode explicitly in config** - `--browser` flag alone fails since 3.2.
6. **Use MSW for network mocking** - mocks at network layer, not module layer.
7. **Prefer dependency injection over module mocking** - improves architecture and testability.
8. **Use V8 coverage provider** - AST remapping provides accuracy with performance.
9. **Shard tests in CI for large suites** - reduces wall-clock time dramatically.
10. **Use worker-scoped fixtures with `isolate: false`** for expensive setup - maximizes performance.

All rules sourced from official Vitest 3.2 documentation (vitest.dev), MSW documentation, and reputable testing resources (LogRocket, Epic Web Dev, Testing Library) as of mid-2025.