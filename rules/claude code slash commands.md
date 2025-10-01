# Claude Code Slash Commands Production Rules

## 1. Command Architecture & Design

### File Structure & Location
- **MUST** store project-specific commands in `.claude/commands/` (shared with team, marked "project").
- **MUST** store personal commands in `~/.claude/commands/` (available across projects, marked "user").
- **MUST** use Markdown format with frontmatter metadata: `allowed-tools`, `argument-hint`, `description`, `model`.
- **MUST** use subdirectories for namespacing—`.claude/frontend/component.md` becomes `/frontend:component`.
- **MUST** check commands into git for team availability.

### Command Syntax
- **MUST** use `$ARGUMENTS` keyword for parameter injection into prompt templates.
- **MUST** prefix bash commands with `!` for pre-execution (requires Bash in `allowed-tools`).
- **MUST** prefix file references with `@` to include file contents in command context.

### Orchestration Patterns
- **ALWAYS** use slash commands for context injection—prompts inject into main thread, sharing context window.
- **ALWAYS** use subagents for task isolation—subagents get separate context windows, isolated tool access.
- **MUST** design slash commands as orchestrators; subagents as execution units.
- **MUST** use Opus 4 for orchestration, Sonnet 4/4.5 for worker tasks—reduces costs 40-60%.

---

## 2. Prompt Engineering for Claude 4

### Prompt Caching Optimization
- **MUST** cache static content 1024+ tokens minimum (Opus 4, Sonnet 4/4.5); 2048+ tokens for Haiku models.
- **MUST** stabilize prompt prefix—single-token differences invalidate cache from that point forward.
- **MUST** place static content first: system instructions → tool definitions → context → examples → user input.
- **ALWAYS** accept 25% write premium for 90% read savings—cache writes cost 25% more; reads cost 10% of base.
- **MUST** leverage 5-minute TTL—cache persists 5 minutes, resetting with each hit; structure conversations to reuse cached content.
- **ALWAYS** use cache reads to bypass ITPM limits—Sonnet 3.7+ cache reads don't count toward Input Tokens Per Minute.

### Chain of Thought (CoT)
- **MUST** externalize thinking—CoT only works when thinking is output; silent reasoning produces zero benefit.
- **ALWAYS** use XML tags for structured thinking: `<thinking>`, `<answer>`, `<context>` separate reasoning phases.
- **MUST** use three CoT patterns based on complexity: Basic ("Think step-by-step"); Guided (outline specific steps); Structured (XML tags separating thinking from answer).
- **ALWAYS** guide initial thinking for complex tasks—provide first reasoning step or framework to direct Claude's approach.
- **MUST** enable CoT for complex reasoning; disable (`enable_cot=false`) for concise outputs (legal briefs, executive summaries).

### Context Management
- **MUST** prune outdated information—remove conflicting/stale context as new information arrives; never accumulate indefinitely.
- **ALWAYS** tailor context to task—different tasks need different context types; don't reuse identical context across dissimilar operations.
- **MUST** avoid extremes: insufficient context causes hallucinations (nonexistent APIs, wrong configs); overflow causes unfocused responses.
- **ALWAYS** start with PRD, divide into tasks—begin with comprehensive requirements; break into focused tasks executed one at a time with fresh context.
- **MUST** truncate command outputs intelligently—useful information appears in prefix/suffix more than middle; truncate middle, not suffix.
- **NEVER** place variable user input in cacheable prefix—invalidates cache on every request.

### Instruction Design
- **MUST** demand autonomous completion: "Complete tasks from start to finish. If you make a plan, immediately follow it. Do not wait for user confirmation."
- **MUST** forbid premature help-seeking: "Gather information using available tools before asking for help. Verify assumptions before proceeding."
- **ALWAYS** request parallel execution: "Execute multiple independent tasks in parallel to reduce wait time."
- **MUST** front-load specific instructions—Claude 4 success rate improves significantly with upfront specificity.
- **MUST** eliminate ambiguity—most failures come from ambiguity, not model limitations.
- **ALWAYS** define output format explicitly—specify structure, schema, style; use strict JSON enforcement when applicable.
- **MUST** request behaviors explicitly—Claude 4 requires explicit requests for "above and beyond" behaviors previous models inferred.

### Few-Shot Examples
- **MUST** provide 3-5 diverse examples minimum for complex tasks requiring pattern recognition.
- **ALWAYS** align examples with desired behavior—Claude 4 pays close attention to example details.
- **MUST** maintain formatting consistency—examples must match exact formatting, field order, structure expected in outputs.
- **NEVER** rely on examples alone—accompany with explicit instructions; models may pick up unintended patterns.
- **ALWAYS** place examples before instructions in cached prompts—static examples cached early; instructions reference without re-transmitting.

### Tool Use Patterns
- **MUST** document tools with headers, delimiters, markdown—clarity prevents excessive/incorrect tool use.
- **MUST** define structured tool-use format with clear schema examples—Claude 4 excels at parallel tool execution.
- **ALWAYS** design tools for concurrent use—enable parallel execution where operations are independent.
- **MUST** implement step confirmation process—prevent tool spam by requiring confirmation between multi-step sequences.
- **ALWAYS** return structured, JSON-serializable outputs—design functions to return parseable structures, not formatted strings.

### Token Efficiency
- **MUST** strip formatting from cached sections—Markdown, extra whitespace count as tokens; use minimal formatting.
- **ALWAYS** use structured formats over natural language—JSON, YAML require fewer tokens than verbose text for structured data.
- **MUST** reference, don't repeat—once content is cached, reference it ("as shown in examples") rather than repeating specifics.
- **ALWAYS** batch similar requests—group requests using same cached prefix to maximize cache hit rate during 5-minute window.

---

## 3. Parameter Handling & Validation

### Parameter Design
- **NEVER** require more than 2 arguments—one is acceptable, two questionable, three prohibited.
- **ALWAYS** prioritize flags over positional arguments for clarity.
- **NEVER** require prompts—always allow override for automation.
- **MUST** use standard flag names: `-h`/`--help` (help only), `-n`/`--dry-run` (dry runs), `-v`/`--verbose` (verbosity).

### Security Validation
- **MUST** validate server-side only—client-side validation can be circumvented.
- **ALWAYS** validate at point of entry—check input immediately upon receipt from external sources.
- **MUST** treat all input as untrusted—regardless of authentication status.
- **ALWAYS** use allowlists, never blacklists—define explicitly what's permitted.
- **MUST** validate length, type, syntax, business rules—check every dimension before processing.
- **ALWAYS** normalize before validation—handle encoding tricks (URL encoding, unicode variants).
- **MUST** reject unexpected parameters—only accept known parameters; treat unrecognized input as error.

### Type Safety
- **MUST** enforce strict type checking—if expecting number, reject all non-numeric input.
- **ALWAYS** build parameters using arrays—construct command arguments programmatically, not string concatenation.
- **MUST** use end-of-options marker `--` to prevent inputs being misinterpreted as command flags.
- **NEVER** pass passwords as CLI arguments—visible via process listings; use env vars or config files.

---

## 4. Error Handling & User Experience

### Error Message Structure
- **MUST** use Title + Body + Action format: [What went wrong] + [Why it happened, if relevant] + [How to fix it] + [Clear next step].
- **ALWAYS** answer two questions: "What went wrong?" and "How does the user fix it?"
- **MUST** include in error output: error code, title, description, resolution steps, URL for assistance (when applicable).
- **ALWAYS** translate technical errors into human guidance—catch system errors and rewrite for users.
- **MUST** be specific about invalid inputs—name exactly what failed validation.

### Output Streams
- **MUST** write useful information to stdout; warnings/errors to stderr.
- **MUST** exit with zero on success, nonzero on error—consistent exit statuses enable script embedding.
- **NEVER** expose stack traces to end users by default—reserve for debug mode (`--debug`, `--verbose`).

### Progressive Disclosure
- **MUST** provide concise help by default—when no args provided, show brief usage.
- **ALWAYS** provide comprehensive help on demand—`-h`/`--help` shows full documentation.
- **MUST** lead with examples—show common invocations first, explanations second.
- **ALWAYS** suggest next commands after completion—hint at common follow-up actions.

### Responsiveness
- **MUST** print something within 100ms—responsive feels better than fast.
- **ALWAYS** use spinners for 2-10 second tasks; progress bars for 10+ second tasks.
- **MUST** show action before network requests—don't hang silently.
- **ALWAYS** confirm every user action—acknowledge what just happened.

---

## 5. Testing & Reliability

### Test Strategy
- **MUST** write tests based on expected input/output pairs before implementing (TDD).
- **ALWAYS** confirm tests fail first, then write code to pass them.
- **MUST** test real-world scenarios—edge cases, error conditions, unexpected inputs, not just happy paths.
- **ALWAYS** use unit tests for edge cases/boundary conditions; integration tests for user-observable behavior.

### Integration Testing
- **MUST** run real commands—execute actual program with various parameters, examine outputs; don't mock implementations.
- **ALWAYS** separate integration tests in dedicated `tests/` directory (following Rust/Cargo conventions).
- **MUST** test both success and failure paths—verify error handling, unexpected inputs, edge cases.

### Idempotency & Retries
- **MUST** design operations as idempotent—same request won't cause unintended effects if executed multiple times.
- **ALWAYS** implement exponential backoff—wait time grows exponentially (1s, 2s, 4s, 8s, 16s) with specified multiplier.
- **MUST** add jitter to exponential backoff—prevents thundering herd problems.
- **ALWAYS** choose 2-3 retries as optimal; avoid retrying indefinitely.
- **MUST** distinguish error types—retry only transient errors (network timeouts, 5xx); never retry permanent errors (4xx).
- **ALWAYS** combine retries with circuit breakers—after failure threshold, stop trying for recovery period.

### Monitoring
- **MUST** use OpenTelemetry for instrumentation—production-ready standard for metrics, logs, traces.
- **ALWAYS** collect and correlate three pillars: logs, metrics, traces.
- **MUST** track three metrics: latency, throughput, user feedback—optimize based on production data.
- **ALWAYS** monitor KV-cache hit rate—single most important production metric for Claude commands.

---

## 6. Performance & Execution

### Parallel Execution
- **MUST** run independent tasks concurrently—never execute sequentially what can run in parallel.
- **ALWAYS** use stability-based hashing for task distribution—maintains consistent shard assignments when adding/removing files.
- **MUST** implement automatic result merging when running tasks in parallel—consolidate outputs into single summary.

### Resource Management
- **MUST** use lazy initialization for non-essential components—delay creation until first use; cuts startup overhead 50-70%.
- **ALWAYS** move non-critical initialization to background execution—reduces CLI startup time.
- **MUST** cache dependency resolution results—avoid redundant computations on subsequent runs.

### Streaming & Output
- **MUST** stream output token-by-token or chunk-by-chunk—start displaying as soon as first data arrives, don't buffer entire results.
- **ALWAYS** target sub-millisecond to millisecond-level latency for interactive CLI tools through incremental output.
- **MUST** display output progressively for long-running operations—improves perceived responsiveness even when actual processing time unchanged.

---

## 7. Documentation & Discovery

### Help System
- **MUST** provide `--help` flag for every command—list all commands, subcommands, short-name equivalents with descriptions.
- **ALWAYS** allow `--help` after specific commands to see full syntax.
- **MUST** include `--version` flag—always provide version info; show in help text and error output.

### Inline Documentation
- **MUST** explain why, not what—comments should explain rationale code cannot express.
- **ALWAYS** include usage examples and working code snippets for functions, classes, modules.
- **NEVER** document self-evident code—focus on constraints, pre/post-conditions, non-intuitive sections.
- **MUST** update comments whenever referenced code changes—maintain synchronization.

### Discovery Mechanisms
- **MUST** enable auto-discovery—commands discoverable via typed prefix (e.g., `/` triggers command menu).
- **ALWAYS** nudge users toward commonly-used commands—don't dump full documentation.
- **MUST** provide progressive help—start simple, allow drill-down for detailed syntax.

### Versioning & Change Management
- **MUST** use semantic versioning for API version management—increment only when contract changes, not implementation.
- **ALWAYS** maintain distinct documentation for each version.
- **MUST** announce deprecations 6-12 months in advance—allow gradual migration.
- **ALWAYS** prefer additive changes—add new fields/endpoints over modifying existing ones.

---

## 8. Extensibility & Composition

### Plugin Architecture
- **MUST** separate core from extensions—microkernel/plugin architecture with minimal core, independent plugin modules.
- **ALWAYS** define clear interfaces for plugins to interact with core without modifying it.
- **MUST** support dynamic plugin discovery—allow plugins added without recompiling core.
- **ALWAYS** handle initialization order—execute BeforeInitialization events, unload existing, instantiate enabled sequentially.

### Hook Mechanisms
- **MUST** define clear hook points where extension code should be callable.
- **ALWAYS** use consistent naming: pre/post/on conventions (`preExecute`, `onComplete`, `postProcess`).
- **MUST** pass sufficient context—hooks receive all data needed to make decisions.
- **ALWAYS** support async hooks—allow hooks to return promises for async operations.
- **MUST** isolate failures—one hook failure shouldn't break others unless critical.

### Configuration Systems
- **MUST** layer configuration sources with clear precedence: command-line args > env vars > config files > defaults.
- **ALWAYS** provide sensible defaults—most common path should work without configuration.
- **MUST** support user and system levels: `~/.tool-name` for user, `/etc` for system-wide.
- **NEVER** store credentials in command line—use env vars or config files for security.

---

**Sources**: Anthropic official documentation, OWASP ASVS, Google/Microsoft CLI design guides, OpenTelemetry standards, industry best practices from Stripe, AWS Builder's Library, Rust CLI patterns, modern async frameworks (Tokio), enterprise security standards, and 2025-specific AI engineering research.