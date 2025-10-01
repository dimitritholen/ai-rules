# LangGraph 0.6.8 Production Rules

*Synthesized from official documentation, OWASP guidelines, and enterprise production patterns as of mid-2025.*

---

## 1. Architecture & Graph Design

### Graph Structure
- **MUST** choose appropriate agent architecture based on task requirements: ReAct for tool use + multi-step reasoning; multi-agent patterns (Pipeline/Hub-and-Spoke) for complex orchestration.
- **MUST** use subgraphs for modular design in multi-agent systems; communicate with parent graph through overlapping state schema keys.
- **MUST** design nodes as independent, clearly defined units with single responsibilities.
- **NEVER** rely on `recursion_limit` as primary halt mechanism—hitting it indicates design flaws; implement explicit exit conditions based on state.
- **MUST** ensure graphs terminate gracefully through explicit conditional routing, not timeout limits.

### Control Flow
- **MUST** define clear routing logic with explicit conditionals in edges; avoid implicit assumptions about execution order.
- **MUST** use static breakpoints (`interrupt_before`/`interrupt_after`) for predictable HITL patterns; reserve dynamic `interrupt()` for runtime decisions.
- **NEVER** dynamically modify node structure containing multiple `interrupt()` calls—introduces state consistency and resume value errors.

---

## 2. State Management

### State Schema Design
- **MUST** keep state schemas minimal—include only necessary information for graph execution.
- **MUST** define explicit, typed state structures using user-defined schemas (e.g., TypedDict, Pydantic models).
- **NEVER** include extraneous data in state; bloat degrades performance and complicates debugging.

### Reducers (CRITICAL)
- **MUST** define reducer functions for ALL state keys modified by parallel branches—reducers are mandatory, not optional, to prevent `InvalidUpdateError`.
- **MUST** implement fast, optimized reducers—they execute frequently and impact performance.
- **MUST** test reducer logic for commutativity and associativity when operations may occur in any order.

### Persistence
- **MUST** configure checkpointer when compiling graph for production use—enables HITL, error recovery, session memory, time-travel debugging.
- **MUST** use production-grade database adapters (Redis, Postgres, SQLite) for checkpointer backend; avoid in-memory for production.
- **MUST** implement Store interface for cross-thread persistent state (long-term memory); checkpointers alone cannot share data across threads.
- **MUST** use threads to organize conversation sessions; each thread contains sequential checkpoints.

---

## 3. Error Handling & Fault Tolerance

### Retry Policies
- **MUST** set `retry_policy` on nodes with flaky operations (LLM calls, API requests); configure `retry_on` exceptions explicitly.
- **MUST** define fallback behavior when all retry attempts fail—graph execution stops immediately on retry exhaustion.
- **MUST** wrap error-prone code in try-except blocks within nodes for granular control; log errors with context (node name, input state, stack trace).

### Tool Calling Errors
- **MUST** leverage ToolNode's automatic error capture—returns `ToolMessage` with error field on failure.
- **MUST** route failed tool calls to recovery nodes that either: remove failed messages and retry with better model, or implement fallback logic.
- **MUST** validate tool inputs before execution; sanitize outputs to prevent downstream errors.

### Fault Tolerance Architecture
- **MUST** use transactional queue patterns (e.g., Redis BLPOP) for atomic task handoffs in distributed deployments.
- **MUST** implement idempotent node operations where possible—enables safe retries without side effects.

---

## 4. Testing & Debugging

### Testing Strategy
- **MUST** write unit tests for each node function independently; mock LLM/API responses to control test behavior.
- **MUST** test graph routing logic separately by replacing nodes with mocks while preserving routing conditions.
- **MUST** implement integration tests that verify state flow through entire graph; use cached LLM outputs to reduce cost and improve speed.
- **MUST** test reducer functions in isolation with parallel state updates; verify correctness under concurrent modifications.

### Debugging Tools
- **MUST** enable LangSmith tracing for production deployments—provides visual traces, node timings, state transitions, error tracking.
- **MUST** implement comprehensive logging within nodes: entry/exit, input/output state, intermediate variables, errors, external calls.
- **MUST** use structured logging with levels (DEBUG, INFO, WARNING, ERROR) for verbosity control.
- **MUST** leverage time-travel debugging to replay executions up to checkpoint and fork to explore alternatives.
- **MUST** use LangGraph Studio for visual prototyping and debugging of complex flows.

---

## 5. Performance & Optimization

### Performance Patterns
- **MUST** profile graph execution to identify bottlenecks; optimize most time-consuming nodes first.
- **MUST** divide complex tasks into smaller subtasks; assign most suitable LLM to each (optimize cost and performance).
- **MUST** minimize state size with optimized updates; avoid storing large objects (embeddings, full documents) directly in state.
- **MUST** implement caching for frequently accessed data or repeated LLM calls with identical inputs.

### Scaling for Production
- **MUST** implement horizontal scaling with task queues and multiple worker processes for concurrent requests.
- **MUST** use auto-scaling based on metrics (CPU utilization, request rate, queue depth).
- **MUST** conduct load testing at multiple concurrency levels (10, 20, 30, 40, 50+ users); forecast hardware needs from collected data.
- **MUST** monitor application performance with metrics collection (Prometheus/Grafana or equivalent).

---

## 6. Security

### Prompt Injection Prevention
- **MUST** provide specific system prompt instructions about model role, capabilities, limitations; enforce strict context adherence.
- **MUST** instruct model to ignore attempts to modify core instructions or access restricted data.
- **MUST** validate and sanitize ALL user inputs before passing to LLM; strip embedded commands, hidden instructions, suspicious metadata.
- **MUST** validate LLM outputs against expected formats; use deterministic code to check adherence to schemas.
- **MUST** implement content filtering with semantic filters and pattern detection for disallowed content; continuously update rules.

### Access Control & Data Protection
- **MUST** apply principle of least privilege—grant LLM apps only necessary data access and minimum required permissions.
- **MUST** implement human-in-the-loop approval for sensitive operations (data access, external API calls, state modifications).
- **MUST** add moderation and quality controls to prevent agents from veering off course.
- **MUST** use defense-in-depth strategy—combine multiple security tactics rather than relying on single measure.

### Input/Output Validation
- **MUST** use allowlists to block malicious prompts or restrict output formats to predefined templates.
- **MUST** request detailed reasoning and source citations from LLM; verify claims against known facts.
- **MUST** implement schema validation for all tool inputs and outputs using type checkers or validation libraries.

---

## 7. Streaming & Checkpointing

### Streaming Modes
- **MUST** select appropriate stream mode for use case: `values` (full state snapshots), `updates` (state changes), `messages` (chatbot interactions), `tasks` (pending operations), `checkpoints` (persistence events), `custom` (user-defined).
- **MUST** use `messages` mode for chatbots with token-by-token streaming; use `updates` mode for long-running agents to show intermediate steps.
- **MUST** implement streaming for better UX—show agent reasoning and actions in real-time.

### Checkpoint Management
- **MUST** understand checkpoint lifecycle—saved at every super-step (after all parallel nodes complete).
- **MUST** use checkpoints for session memory, error recovery, HITL approval, time-travel debugging.
- **MUST** configure checkpoint retention policies to avoid unbounded storage growth; archive or delete old checkpoints.
- **MUST** implement checkpoint migration strategy when changing graph structure or state schema in production.

### Interruption Patterns
- **MUST** use static breakpoints for compile-time HITL points; use dynamic `interrupt()` for runtime decisions based on state.
- **MUST** handle interrupts gracefully—provide context to user about why execution paused and what input is needed.
- **MUST** validate user input after interruption before resuming; ensure input meets expected format and constraints.

---

## 8. Deployment & Operations

### CI/CD Pipeline
- **MUST** implement robust CI/CD with automated testing (unit, integration, end-to-end) on every commit.
- **MUST** run regression tests to ensure changes don't degrade performance or introduce bugs.
- **MUST** use versioned deployments with rollback capability for quick recovery from failures.

### Monitoring & Observability
- **MUST** implement distributed tracing across all nodes and external calls.
- **MUST** collect metrics: latency per node, error rates, retry counts, checkpoint sizes, LLM token usage, cost.
- **MUST** set up alerting for critical failures: graph execution errors, retry exhaustion, checkpoint failures, security violations.
- **MUST** regularly review logs and metrics to identify optimization opportunities and emerging patterns.

### Production Deployment Options
- **MUST** choose deployment model based on requirements: Cloud (fully managed), Hybrid (SaaS control plane + self-hosted data plane), Self-Hosted (full control).
- **MUST** use LangGraph Platform features in production: built-in persistence, horizontal scalability, intelligent caching, automated retries.

---

## 9. Common Pitfalls (AVOID)

- **NEVER** skip reducer definitions for state keys updated by parallel branches—causes `InvalidUpdateError`.
- **NEVER** use recursion_limit as primary termination mechanism—signals design problems.
- **NEVER** dynamically modify complex HITL node structures—introduces state consistency bugs.
- **NEVER** store large objects directly in state—causes performance degradation.
- **NEVER** skip input/output validation—opens security vulnerabilities and error propagation.
- **NEVER** deploy without checkpointer configuration—loses fault tolerance and HITL capabilities.
- **NEVER** ignore LangSmith tracing in production—blind to performance bottlenecks and errors.
- **NEVER** trust LLM outputs without validation—implement schema checking and sanity tests.

---

## 10. Documentation & Maintainability

### Code Documentation
- **MUST** document state schema with field descriptions and types; include example values.
- **MUST** document each node's purpose, inputs, outputs, side effects, failure modes.
- **MUST** document routing conditions and their business logic rationale.
- **MUST** maintain architecture decision records (ADRs) for significant design choices.

### Dependency Management
- **MUST** pin LangGraph and LangChain versions in production; test upgrades in staging first.
- **MUST** track breaking changes in release notes when upgrading; plan migration strategy.
- **MUST** monitor LangGraph GitHub issues and discussions for known bugs and workarounds.

---

**Version**: LangGraph 0.6.8
**Last Updated**: October 2025
**Sources**: Official LangGraph Documentation, OWASP GenAI Security Guidelines, AWS/NVIDIA/IBM Technical Blogs, Production Case Studies
