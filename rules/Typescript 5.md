# TypeScript 5 Production Rules (2025)

## 1. Compiler Configuration

### Strict Mode (Non-Negotiable)
**MUST** enable `strict: true` - activates all strict type-checking options, reduces runtime errors by 60%

**MUST** enable `noUncheckedIndexedAccess: true` - adds `undefined` to array/object indexed access, prevents out-of-bounds errors (included in TS 5.9+ defaults)

**MUST** enable `exactOptionalPropertyTypes: true` - enforces distinction between `undefined` values and missing properties

**MUST** enable `noPropertyAccessFromIndexSignature: true` - requires bracket notation for dynamically-defined properties, distinguishes known properties from dynamic access

**MUST** set `allowUnreachableCode: false` and `allowUnusedLabels: false` - prevents dead code reaching production

**MUST** enable `noFallthroughCasesInSwitch: true` - catches accidental switch fall-through bugs

**MUST** enable `noImplicitReturns: true` - ensures all code paths in functions return values

**MUST** enable `noImplicitOverride: true` - requires explicit `override` keyword, prevents typos when overriding base class methods

**MUST** enable `forceConsistentCasingInFileNames: true` - catches cross-platform filesystem issues between Unix and Windows (default in TS 5.0+)

### Module Configuration
**MUST** use `module: "NodeNext"` and `moduleResolution: "NodeNext"` for Node.js projects - handles ESM/CommonJS interop and respects package.json exports/imports

**MUST** use `module: "ESNext"` and `moduleResolution: "Bundler"` for browser applications - optimized for bundler workflows (Vite, Webpack, Rollup)

**MUST** enable `verbatimModuleSyntax: true` - replaces deprecated `importsNotUsedAsValues`/`preserveValueImports` with simpler behavior; requires `import type` for type-only imports

**MUST** enable `isolatedModules: true` - ensures code can be transpiled by single-file transpilers (esbuild, swc, Babel)

**MUST** enable `moduleDetection: "force"` - treats all files as modules, prevents global scope pollution

**MUST** enable `esModuleInterop: true` - fixes CommonJS/ES Module compatibility issues

**MUST** enable `resolveJsonModule: true` - allows importing JSON files with automatic typing

### Performance Optimization
**ALWAYS** enable `skipLibCheck: true` - skips type checking node_modules .d.ts files, improves build time by up to 50%

**MUST** enable `incremental: true` for large codebases - caches previous builds, reduces build time up to 80%

**MUST** enable `noEmitOnError: true` for production builds - prevents emitting JavaScript with type errors in CI/CD

**MUST** set `target: "ES2020"` for browser apps (broad compatibility), `ESNext` for modern Node.js (avoids unnecessary transpilation)

**MUST** set `removeComments: true` for production - reduces bundle size

### Library Publishing
**MUST** enable `declaration: true` and `declarationMap: true` for libraries - generates .d.ts files and enables IDE navigation to source

**MUST** enable `isolatedDeclarations: true` for library authors (TS 5.5+) - requires explicit type annotations on public APIs, enables 10x faster declaration generation

**NEVER** enable `isolatedDeclarations` for application code - trade-off only makes sense for libraries or large monorepos

### Project Structure
**MUST** set `rootDir: "src"` and `outDir: "dist"` explicitly - controls output directory structure

**MUST** use `include: ["src/**/*"]` - explicitly defines input files

**MUST** exclude test files from production builds - use `exclude: ["**/*.test.ts", "**/*.spec.ts", "**/__mocks__"]`

### Monorepo Configuration
**MUST** enable `composite: true` for monorepo packages - enables project references, automatically enables `incremental` and `declaration`

**MUST** use project references with `references: [{ "path": "../other-package" }]` for monorepo dependencies - enables per-project caching and incremental compilation

**MUST** configure `tsBuildInfoFile` with composite projects - specifies location for incremental build cache

---

## 2. Type System & Type Safety

### Type Safety Fundamentals
**NEVER** use `any` - completely disables type checking, defeats TypeScript's purpose

**ALWAYS** use `unknown` instead of `any` for truly dynamic values - type-safe, requires explicit narrowing before use

**MUST** narrow `unknown` types before use with type guards (`typeof`, `instanceof`, `in`) or runtime validation

**ALWAYS** use `never` for exhaustiveness checking - ensures all union cases are handled at compile-time

**MUST** understand `void` vs `undefined` - `void` is for functions with no meaningful return; `undefined` is an actual value type

**ALWAYS** prefer type inference over explicit annotations where clarity permits - TypeScript's inference is sophisticated

**MUST** provide explicit return types for public API functions - prevents unintended breaking changes

### Type Narrowing & Guards
**ALWAYS** leverage control flow analysis - TypeScript automatically tracks narrowing through `typeof`, `instanceof`, `in`, truthiness, and equality checks

**MUST** use user-defined type guards with `is` predicates for complex narrowing - format: `function isType(value: unknown): value is Type`

**ALWAYS** use assertion functions for validation that throws - format: `function assert(condition: any): asserts condition`

**MUST** leverage TypeScript 5.5+ inferred type predicates - simple boolean-returning functions automatically infer type narrowing

**ALWAYS** use discriminated unions over complex conditionals - add literal type discriminant field (e.g., `type: 'success' | 'error'`) for automatic narrowing

**MUST** implement exhaustiveness checking with `never` - add default case assigning to `never` in discriminated union handlers

**ALWAYS** create `assertUnreachable(value: never): never` helper - standard pattern for exhaustive checking

### Advanced Type Patterns

#### Conditional Types
**ALWAYS** use conditional types for type-level branching - format: `T extends U ? X : Y`

**MUST** understand distributive vs non-distributive conditionals - naked type parameters distribute over unions; wrap in brackets `[T]` to prevent

**ALWAYS** use `infer` for pattern matching in conditional types - extracts and captures types from generic structures

**NEVER** nest conditional types excessively - extract to named type aliases for readability and performance

#### Mapped Types
**ALWAYS** use mapped types to transform object structures - format: `{ [K in keyof T]: Transform<T[K]> }`

**ALWAYS** use key remapping with `as` clause (TS 4.1+) - format: `[K in keyof T as NewType<K>]: T[K]`

**ALWAYS** use `never` in mapped types to filter keys - remapping to `never` removes keys

**MUST** understand modifiers: `+readonly` adds, `-readonly` removes; `+?` adds optionality, `-?` makes required

**ALWAYS** leverage built-in utility types before custom ones - `Partial`, `Required`, `Readonly`, `Pick`, `Omit`, `Record` are optimized

#### Template Literal Types
**ALWAYS** use template literal types for string pattern validation - enforce patterns like `${string}@${string}.${string}` at compile time

**MUST** combine template literals with unions for all combinations - `${Direction}${Side}` creates cartesian product

**ALWAYS** use built-in string manipulation types - `Uppercase`, `Lowercase`, `Capitalize`, `Uncapitalize` are compiler-optimized

**NEVER** overuse template literals for complex string parsing - has recursion limits and performance costs

### Generic Constraints & Variance
**ALWAYS** constrain generics with `extends` when operations assume structure - improves IntelliSense and prevents incompatible types

**MUST** use multiple type parameters when different constraints apply - `<T extends A, U extends B>` clearer than forcing both into one

**ALWAYS** use variance annotations (`in`/`out` modifiers, TS 4.7+) during debugging only - remove after fixing unless profiling shows measurable benefit

**ALWAYS** constrain generic functions to most specific acceptable type - balance between type safety and reusability

**MUST** minimize type parameters - use as few as possible; only add when relating multiple values

**NEVER** use type parameters that appear only once - indicates they're not relating anything

### Union & Intersection Types
**ALWAYS** prefer discriminated unions over complex conditionals - add consistent discriminant property like `type`, `kind`, or `status`

**NEVER** create large unions without good reason - dozens of members cause slow compilation

**ALWAYS** use intersection types for composing behaviors - `type Combined = A & B` merges properties

**ALWAYS** prefer unions of objects over objects with optional properties for distinct states - enforces related properties appear together

### Immutability & Constants
**ALWAYS** use `as const` for literal types and deep immutability - prevents widening, makes properties readonly, converts arrays to tuples

**MUST** combine `as const` with `satisfies` for validated immutable objects - gets both compile-time validation and preserved literals

**ALWAYS** use `readonly` modifier for properties that shouldn't change - documents intent and prevents mutations

**NEVER** rely on `as const`/`readonly` for runtime immutability - compile-time only; use `Object.freeze()` for runtime protection

**ALWAYS** use `readonly` arrays when functions don't mutate - `readonly T[]` or `ReadonlyArray<T>` prevents `.push()`, `.pop()`

### Modern TypeScript Features

#### Satisfies Operator (TS 4.9+)
**ALWAYS** use `satisfies` over type assertions for validation - checks compatibility without widening inferred type

**MUST** combine `satisfies` with `as const` for validated literal types - pattern: `{...} as const satisfies Type`

**NEVER** use `as` assertions when `satisfies` suffices - `as` disables checking; `satisfies` enforces it while preserving inference

**NEVER** use `satisfies` gratuitously - only when you want exact type inference (not wider type) and need validation

#### Override Keyword (TS 4.3+)
**ALWAYS** use `override` keyword when overriding base class methods - catches typos and base method renames/removals

**MUST** enable `noImplicitOverride` in strict codebases - forces explicit `override` keyword

#### Const Type Parameters (TS 5.0+)
**MUST** use `const` modifier on type parameters to preserve literal types - format: `function parse<const T>(data: T)`

**NEVER** use `const` type parameter with arrow functions - parsing ambiguity, only works with function declarations

#### NoInfer Utility (TS 5.4+)
**MUST** use `NoInfer<T>` to block unwanted type inference - prevents TypeScript from inferring from specific positions

**ALWAYS** use `NoInfer<T>` for default values that shouldn't influence inference

### Branded Types
**ALWAYS** use branded types for semantically distinct primitives - prevents mixing `UserId` and `OrderId` even though both are `string`

**MUST** use intersection with unique symbol for branding - format: `type UserId = string & { readonly __brand: unique symbol }`

**NEVER** export the brand symbol - keep internal to prevent invalid branded value creation

### Performance Best Practices
**ALWAYS** use `interface` with `extends` for object inheritance instead of type intersections with `&` - interfaces are cached by name, intersections recomputed

**ALWAYS** add explicit type annotations, especially for function return types - reduces compiler inference work

**ALWAYS** name and extract complex types into separate aliases - enables compiler caching

**MUST** avoid deep recursive type definitions - multiplies type-checking time

**NEVER** export `const enum` from libraries - blocks downstream single-file transpilation

**MUST** profile type-checking with `tsc --diagnostics` or `--extendedDiagnostics` in large codebases

**ALWAYS** use `tsc --generateTrace` for deep performance analysis - generates Chrome-compatible trace files

---

## 3. Architecture & Project Structure

### Project Organization
**MUST** organize by domain/business capability (auth, billing) not technical layers (utils, helpers) - clearer boundaries, easier ownership

**MUST** use `apps/` and `packages/` structure for monorepos - separates deployable applications from shared libraries

**NEVER** use barrel files (index.ts re-exports) in application code - causes 68% more module loading, creates circular dependencies, slows bundlers

**MUST** use barrel files only for library entry points in package.json - sole valid use case

**MUST** implement project references for large codebases - dramatically improves build times, enforces logical separation

**MUST** use consistent file naming (kebab-case or snake_case) - choose one and stick with it

**NEVER** use generic names like helpers.ts or utils.ts - use descriptive names (date-formatter.ts, user-validator.ts)

**MUST** use file suffixes indicating purpose (.model.ts, .service.ts, .repository.ts) - faster navigation

### Layered Architecture
**MUST** enforce unidirectional dependency flow: Presentation → Application/Service → Domain → Infrastructure

**NEVER** allow inner layers to import from outer layers - use dependency inversion with abstract interfaces

**ALWAYS** separate Commands (writes) from Queries (reads) - commands use rich domain entities, queries can use optimized DTOs

**ALWAYS** keep controllers/routers thin with zero business logic - only handle HTTP concerns

**ALWAYS** use constructor injection over framework-specific DI for core business logic - reserve framework DI for HTTP layer

**MUST** implement singleton pattern for shared resources using lifespan events - database pools, cache clients initialize once at startup

### Module Design
**MUST** use path aliases (@components, @services) for cleaner imports - improve readability in large projects

**NEVER** create path aliases clashing with node_modules packages - causes resolution conflicts

**MUST** configure build tools to recognize TypeScript aliases - essential for production builds

**MUST** detect and prevent circular dependencies - use madge, dpdm, or eslint-plugin-import

**MUST** automate circular dependency checks in CI/CD - prevent them from entering codebase

**MUST** use workspace protocol (workspace:*) in monorepos - makes local package links explicit

**MUST** pick one package manager and stick with it - pnpm recommended for speed and strictness

### Domain-Driven Design
**MUST** use static typing to model domain logic explicitly - TypeScript's type system enforces domain rules at compile time

**MUST** use interfaces for contracts between bounded contexts - clear boundaries

**MUST** implement value objects as readonly types - immutable by definition

**MUST** separate domain layer from infrastructure layer - domain logic has zero infrastructure dependencies

### Testing Architecture
**MUST** implement both unit tests and integration tests - verify individual components and component interactions

**MUST** use modular architecture to enable isolated unit testing - clean separation enables testing individual units

**MUST** use architecture testing with ts-arch - enforce dependency rules, check for cycles, validate layer boundaries

**MUST** ensure testability through dependency injection - inject dependencies rather than hard-coding

---

## 4. Error Handling & Runtime Safety

### Runtime Validation
**MUST** validate all external data at runtime (API responses, user input, file contents) - TypeScript types vanish at runtime

**ALWAYS** use runtime validation libraries (Zod, io-ts) for external data - manual validation is error-prone

**MUST** use `.safeParse()` over `.parse()` for Zod validation - returns discriminated union instead of throwing

**ALWAYS** infer types from schemas using `z.infer<typeof schema>` - single source of truth

**MUST** validate API responses even if you control backend - services change, deployments desync

**NEVER** trust process.env without validation - always strings, may be missing/malformed

### Error Handling Patterns
**MUST** use Result/Either types for operations that can fail predictably - makes error cases explicit in signatures

**ALWAYS** return Result types from parsing/validation/IO operations - forces callers to handle errors explicitly

**NEVER** throw exceptions for expected error conditions - use Result types; exceptions for unexpected/unrecoverable failures

**MUST** use try/catch for all async/await operations that can reject - unhandled rejections crash Node.js processes

**ALWAYS** reject promises with Error objects, never strings - preserves stack traces

**MUST** implement global unhandledRejection handler in Node.js - prevents silent failures and crashes

**NEVER** mix `.then()/.catch()` with `async/await` - pick one pattern for consistency

**ALWAYS** annotate async function return types explicitly - prevents accidental promise wrapping

### Null & Undefined Handling
**ALWAYS** use optional chaining (`?.`) for potentially null/undefined access - stops execution without throwing

**MUST** use nullish coalescing (`??`) for default values - only triggers on null/undefined, unlike `||` (all falsy)

**NEVER** use `== null` - use `=== null` or `=== undefined` explicitly, or `??` operator

**ALWAYS** define optional properties with `?:` instead of `| undefined` - better semantics and type checking

**MUST** handle undefined array access with optional chaining or validation - array indexing returns `T | undefined` with strict checks

### Defensive Programming
**MUST** validate function inputs at boundaries (API handlers, public methods) - don't trust data crossing architectural boundaries

**ALWAYS** check array.length before accessing elements - arrays can be empty

**NEVER** mutate parameters without explicit intent - leads to action-at-a-distance bugs

**ALWAYS** freeze configuration objects with `as const` or `Object.freeze()` - prevents accidental mutation

### Type Assertions
**NEVER** use type assertions (`as Type`) to bypass compiler errors - fix the actual type mismatch

**ONLY** use assertions when you have runtime guarantees compiler can't verify - external libraries with incorrect types

**MUST** validate data immediately after using type assertions - assertion doesn't make data safe

**ALWAYS** document why assertions are necessary with comments - future maintainers need context

**NEVER** chain type assertions (`value as A as B`) - extremely unsafe

### Production Error Handling
**MUST** implement error boundaries in React applications - catches render errors preventing entire app crash

**MUST** log errors with full context (user ID, request ID, timestamp, stack trace) - essential for debugging

**NEVER** expose internal error details to users - security risk

**ALWAYS** differentiate operational errors (retryable) from programmer errors (bugs) - different handling strategies

**MUST** implement circuit breakers for external service calls - prevents cascade failures

---

## 5. Performance Optimization

### Build Performance
**MUST** split large codebases into project references - enables incremental builds and parallel compilation

**NEVER** let `@types` packages auto-include unconstrained - explicitly specify needed types

**MUST** split large files (>1000 lines) into smaller modules - smaller files enable better incremental builds

**NEVER** include TypeScript compilation in hot-reload dev server - type checking is slow; run separately

**MUST** use `--extendedDiagnostics` to identify performance bottlenecks - shows compilation time breakdown

**ALWAYS** generate `--generateTrace` profiles for deep analysis - detailed timing information

**ALWAYS** separate type checking from transpilation in development - use fast transpilers (swc, esbuild), run `tsc --noEmit` separately

### Bundle Size
**ALWAYS** enable `importHelpers: true` and install `tslib` - reduces bundle size 25% in small projects, 500KB+ in large codebases

**MUST** use `type` modifier on type-only imports/exports - enables bundlers to remove these imports completely

**ALWAYS** use named exports instead of default exports - better tree shaking and bundler behavior

**NEVER** set `preserveConstEnums: true` - emits enum objects at runtime, increasing bundle size

**ALWAYS** leverage Next.js optimizePackageImports for libraries with barrel exports - automatically transforms to direct imports

**MUST** use direct imports in application code - import from actual file paths, not index.ts

### Module Structure
**NEVER** allow circular dependencies - causes half-loaded modules, compilation slowdowns, runtime bugs

**MUST** maintain unidirectional dependency graphs - extract shared code to break circular references

---

## 6. Testing Strategies

### Type Testing
**MUST** use `*.test-d.ts` files for type tests - separates type tests from runtime tests

**MUST** use `expectTypeOf` over `assertType` for complex assertions - better error messages

**MUST** test type equality with `.toEqualTypeOf<T>()` - ensures exact matching, not just assignability

**ALWAYS** use `@ts-expect-error` for negative type tests - validates invalid operations fail at compile time

**MUST** run type tests through `tsc` or tools that type-check - type-level assertions require separate checking

**NEVER** test types in application code - critical for library development only

### Unit Testing
**MUST** use dependency injection over `jest.mock()` - provides type safety and test isolation

**NEVER** rely on module-level mocks - breaks type checking, creates flaky tests

**ALWAYS** inject dependencies through constructors or parameters - enables proper type checking

**NEVER** test private methods directly - indicates SRP violation; test through public API

**ALWAYS** use factory pattern or test data builders for complex fixtures - reduces duplication

**ALWAYS** use const assertions (`as const`) for test data - compile-time immutability and literal types

### Test Organization
**MUST** colocate tests with source files using `.test.ts` or `.spec.ts` - improves discoverability

**NEVER** hide tests in separate `__tests__` folders - harder to find and maintain

**ALWAYS** organize tests by feature, not technical layer - supports vertical slice architecture

**MUST** use descriptive test names with Given-When-Then or AAA structure - improves readability

**NEVER** mix unit tests and integration tests in same file - different isolation requirements

### Mocking Strategies
**MUST** clear/reset mocks in `beforeEach` hooks - ensures test isolation

**ALWAYS** use `jest.resetAllMocks()` and `jest.restoreAllMocks()` in teardown - prevents mock pollution

**NEVER** share mock instances across test cases - creates flaky tests

**MUST** use `jest-mock-extended` for TypeScript interface mocking - full type safety

**NEVER** mock what you don't own - create adapters/wrappers, mock those

### Integration Testing
**MUST** use Docker containers for database integration tests - consistent, isolated environments

**ALWAYS** test against real database engines, not in-memory alternatives - catches database-specific behavior

**MUST** implement proper setup/teardown - database state must be clean between tests

**NEVER** share database connections across parallel test runs - causes race conditions

### Test Isolation
**MUST** ensure each test can run independently in any order - no dependencies on other test outcomes

**ALWAYS** use `beforeEach` for setup, not `before` - each test gets fresh state

**NEVER** share state via module-level variables - creates hidden dependencies

### Async Testing
**MUST** use `async/await` for asynchronous test code - clearer than promise chains

**ALWAYS** return or await promises in tests - test completes before promise resolves otherwise

**NEVER** use `done` callback with async/await - mixing patterns causes confusion

**ALWAYS** use Testing Library's `waitFor` for UI state changes - prevents race conditions

### Test Coverage
**MUST** set minimum coverage thresholds in CI/CD - prevents regression (60% minimum for enterprise)

**NEVER** obsess over 100% coverage - diminishing returns; focus on critical paths

**MUST** exclude generated code and types from coverage - use `coveragePathIgnorePatterns`

**NEVER** let coverage be the only quality metric - code quality and maintainability matter equally

### Performance
**MUST** separate type checking from test execution - use Babel/SWC for transpilation, `tsc` for type checking

**ALWAYS** use `swc` or `esbuild` for test transpilation - 20-70x faster than tsc

**ALWAYS** run tests in parallel with multiple workers - reduces total execution time

**NEVER** include slow integration tests in fast unit test suite - separates fast feedback from comprehensive validation

---

## 7. Tooling Integration

### ESLint Configuration
**MUST** use ESLint flat config format (eslint.config.js) - standard as of 2025, better type safety

**MUST** use typescript-eslint v8+ with recommended-type-checked - catches bugs through type-aware rules

**NEVER** enable all type-checked rules on large codebases without testing - can increase execution time 30-fold

**MUST** use separate configurations for .js and .ts files - apply `tseslint.configs.disableTypeChecked` for JavaScript

**ALWAYS** extend from typescript-eslint's strict config for new projects - more opinionated, catches additional bugs

**MUST** install eslint-config-prettier to prevent conflicts - ESLint handles errors, Prettier handles formatting

**NEVER** rely on global test function exposure - explicitly import describe, it, expect

### Prettier Integration
**MUST** configure Prettier: single quotes, 2-space indentation, 100-character line width, semicolons

**ALWAYS** install eslint-plugin-prettier and eslint-config-prettier together - formatting through ESLint without conflicts

**MUST** run Prettier before linting in pre-commit hooks - format first, then catch errors

### Build Tools
**MUST** use Vite or esbuild for TypeScript 5 projects - 10-20% faster than TS 4.9, native modern feature support

**ALWAYS** enable tree-shaking in production - removes unused code automatically

**MUST** configure minification for production - removes whitespace and dead code

**ALWAYS** enable code splitting and lazy loading - reduces initial bundle size

**MUST** generate source maps for production with separate files - enables debugging without bloating bundles

**ALWAYS** run `tsc --noEmit` alongside build tools - esbuild/Vite transpile but don't type-check

### Pre-commit Hooks
**MUST** use Husky + lint-staged for pre-commit checks - runs linting only on staged files

**ALWAYS** run TypeScript type checking in pre-commit - use `tsc --noEmit` or tsc-files

**MUST** configure lint-staged to run ESLint with --fix and Prettier on staged files

**NEVER** run type checking on entire codebase in hooks - use tsc-files for staged files only

**ALWAYS** format before linting - Prettier first, then ESLint --fix

### CI/CD Pipeline
**MUST** include these steps in order: install deps, type check, lint, test, build

**ALWAYS** use Node.js 20.x or later - required for modern features

**MUST** cache dependencies and build artifacts - significantly reduces pipeline time

**MUST** validate TypeScript compilation before deployment - catch type errors before production

**NEVER** skip linting in CI even if pre-commit hooks exist - ensures all code meets standards

**MUST** use separate staging/production environments with environment variables for secrets

### Documentation
**MUST** use TypeDoc for API documentation - official tool, converts TypeScript comments to HTML/JSON

**ALWAYS** follow TSDoc standard for comment syntax - maintained by TypeScript team

**MUST** configure TypeDoc to find tsconfig.json automatically - ensures correct compiler options

**NEVER** manually write API documentation - TypeDoc auto-generates from exports and annotations

### Monorepo Tools
**MUST** use pnpm + Turborepo for TypeScript monorepos - faster than npm/yarn, better caching than Nx

**ALWAYS** give each package its own tsconfig.json - enables better Turborepo caching

**MUST** leverage Turborepo's incremental builds and parallel execution - 30s initial build becomes 0.2s from cache

**NEVER** reference root tsconfig.json from packages - breaks Turborepo task caching

---

## 8. Code Quality & Standards

### Naming Conventions
**MUST** use camelCase for variables and functions - standard JavaScript convention

**MUST** use PascalCase for classes, interfaces, and types - distinguishes type-level constructs

**MUST** use UPPER_SNAKE_CASE for constants - indicates immutable values

### Access Control
**MUST** use access modifiers (private, public, protected) appropriately - encapsulation critical for maintainability

**NEVER** leave everything public - explicit access control prevents misuse

**ALWAYS** use `#` private fields for runtime privacy - compile-time AND runtime privacy, cannot be "hacked"

**MUST** use `private` keyword only for compile-time checking when runtime privacy not needed - erased during compilation

### Documentation
**MUST** use TSDoc for documenting public APIs - standardized format

**MUST** use JSDoc comments for complex logic requiring explanation - document the "why" not "what"

**NEVER** merely restate property/parameter names in documentation - add value with context

**MUST** document all exported properties/methods whose purpose isn't immediately obvious

### Code Quality Enforcement
**MUST** fail CI builds on type errors, linting errors, test failures - no exceptions

**ALWAYS** enforce consistent naming via ESLint - prevents confusion

**MUST** set up Git hooks to prevent commits with errors - catches issues before CI

**NEVER** allow TODO comments without tracking - use ESLint to enforce format with issue numbers

**ALWAYS** require code review approval before merging - automated checks catch bugs, humans catch logic errors

---

## 9. Modern Features (TypeScript 5.x)

### Resource Management (TS 5.2+)
**MUST** use `using` declarations for explicit resource management - automatic cleanup when objects go out of scope

**ALWAYS** use `await using` for asynchronous resource disposal - supports async cleanup with `SuppressedError`

**MUST** implement `Symbol.dispose` or `Symbol.asyncDispose` on resources - required for `using`/`await using`

### Import Attributes (TS 5.8+)
**MUST** use `import ... with { type: "json" }` instead of deprecated `assert` - aligns with Node.js 22, Stage 3 ECMAScript

### Enums vs Const Objects
**ALWAYS** prefer `as const` objects over enums for new code - zero runtime cost, better tree-shaking

**MUST** use `--erasableSyntaxOnly` flag to prevent enum usage - flags enums/namespaces as errors, keeps code aligned with evolving JavaScript

**NEVER** create traditional enums in modern TypeScript - community consensus 2025: enums effectively deprecated

### Decorators
**MUST** use TypeScript 5.0+ decorators without `--experimentalDecorators` for new code - Stage 3 decorators now standard

**NEVER** mutate decorator targets directly - return new values or wrappers

**ALWAYS** understand Angular still requires `--experimentalDecorators` - parameter decorators missing from Stage 3 spec

### Inferred Type Predicates (TS 5.5+)
**MUST** rely on automatic inference for straightforward predicates - simple boolean checks infer `value is Type` automatically

**ALWAYS** use explicit type predicates for complex narrowing - inference works only for simple cases

---

## 10. Anti-Patterns to Avoid

**NEVER** use type assertions (`as`) to bypass type errors - fix underlying issue

**NEVER** use empty object type `{}` - matches almost everything except null/undefined; use `Record<string, unknown>` or `object`

**NEVER** use `Function` type - too broad; define specific signatures or use `(...args: unknown[]) => unknown`

**NEVER** use `Object` type - use `object` (lowercase) for non-primitives or be more specific

**NEVER** duplicate type definitions across files - create shared type modules

**NEVER** use `any` in type definitions you publish - poisons consumer type safety

**NEVER** ignore TypeScript errors with `@ts-ignore` without investigation - use `@ts-expect-error` with explanation if absolutely necessary

**NEVER** use namespaces in modern code - use ES modules instead

**NEVER** use `var` keyword - use `let` or `const` exclusively

**NEVER** create god objects or god interfaces - violates SRP and ISP

**NEVER** mix business logic with infrastructure code - violates Clean Architecture

**NEVER** create circular dependencies between modules - causes initialization issues

---

## Summary: The Critical Few

If remembering everything is impossible, these are non-negotiable:

1. **MUST** enable `strict: true` + `noUncheckedIndexedAccess` + `exactOptionalPropertyTypes` + `verbatimModuleSyntax`
2. **NEVER** use `any` - use `unknown` instead
3. **ALWAYS** validate external data at runtime (Zod, io-ts)
4. **ALWAYS** use discriminated unions for state modeling with exhaustiveness checking
5. **ALWAYS** use `as const` + `satisfies` for configuration objects
6. **MUST** provide explicit return types for public APIs
7. **ALWAYS** use `interface` with `extends` for object inheritance (not `&` intersections)
8. **NEVER** use barrel files in application code
9. **MUST** enable `skipLibCheck` and `incremental` for build performance
10. **ALWAYS** use Result/Either types for operations that can fail predictably

These rules represent battle-tested practices from Microsoft's TypeScript team, established TypeScript experts (Matt Pocock, Stefan Baumgartner, Total TypeScript), major tech company engineering blogs (Google, Meta, Netflix), and production deployments as of mid-2025. Following these guidelines ensures TOP QUALITY, PRODUCTION READY TypeScript code.
