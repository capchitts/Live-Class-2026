# Senior Engineer interview preparation — Agentic AI

Prepared from this repository on 17 October 2026. Assumption: you are targeting a Senior Software/AI Engineer role building LLM applications, rather than a model-training or research role.

This guide is based on the repository index, class-summary topics, selected implementation files, and later multi-agent notebooks. It is a targeted review, not an exhaustive audit of every notebook, PDF, or dependency. Existing application code has not been changed or executed against external services.

## How to use this guide

1. Explain each question aloud before reading its answer outline.
2. Use the linked implementation to show what the code actually does.
3. Add failure handling, a tradeoff, and a verification strategy to every design answer.
4. Complete the exercises and the timed mock interview.

A useful answer structure is: **requirement → simplest viable design → failure modes → tradeoffs → evidence**. Define terminology briefly, then spend most of your answer on decisions.

## 1. Repository map and priorities

Links below are relative to this guide.

| Priority | Area | Repository starting point | Senior-level expectation |
|---|---|---|---|
| P0 | Python, validation, API boundaries | [Pydantic summary](<../classes_summary/03 - 4 Jul - Pydantic Deep Dive.md>) | Distinguish types, validation, business rules, and authorization |
| P0 | Agent execution | [Project Zero](<../Weekend 02 - 4-5 Jul/04-05 - 5 Jul & 11 Jul - Anatomy of an Agent & The Agentic Loop/Project 0/>) | Explain the loop without a framework; bound execution and effects |
| P0 | Structured output and tools | [CineBot](<../Weekend 05 - 25-26 Jul/09-10 - 25-26 Jul - Structured Output & Tools (CineBot)/>) | Separate a proposed action from a committed action |
| P0 | Resilience and middleware | [Middleware summary](<../classes_summary/12 - 08 Aug - Mastering Middleware.md>) | Retry safely, enforce budgets, handle fallback and ordering |
| P0 | Memory, runtime, human review | [Runtime and HITL](<../classes_summary/15 - 22 Aug - Runtime & Human-in-the-Loop.md>) | Persist execution, isolate tenants, resume safely |
| P0 | MCP | [TimeTrack server and clients](<../Complete MCP/TimeTrackProject/mcp-client-and-advanced/>) | Explain protocol boundaries, security, cancellation, compatibility |
| P1 | Multi-agent orchestration | [Pattern notebook](<../Weekend 14 - Oct 3 & Oct 4/Multi Agents in Langchain/LangChain_MultiAgent_Patterns_VSCode.ipynb>) | Justify decomposition, context transfer, and state merging |
| P1 | RAG and service design | [Meridian backend](<../Weekend 14 - Oct 3 & Oct 4/meridian-ai-learner-main/backend/>) | Evaluate retrieval separately from generation; design ingestion lifecycle |
| P0 | Evaluation and operations | Extend the projects above | Define success, release gates, tracing, incidents, and ownership |

The top-level README stops its class index at Class 18. Later September and October folders contain additional material worth studying, particularly MCP integration, multi-agent patterns, and the Meridian application.

## 2. Python and Pydantic

### Q1. Do Python type hints validate external input?

No. Type annotations guide tooling; runtime validation needs explicit logic or a validation library. A dataclass does not automatically enforce annotated field types. Pydantic parses and validates according to its schema and configuration; coercion may occur unless strict rules are chosen.

For a booking request, validate shape and constraints at the boundary, then enforce customer ownership, showtime availability, pricing, and inventory in the application/database layer. A structurally valid request can still be unauthorized or impossible.

**Follow-up:** A positive `float` is insufficient for money or hours. Consider finite-number checks, precision, permitted range, and domain representation. Use decimal or integer minor units for monetary calculations.

### Q2. How would you design an async tool that calls an external API?

Use an async client with explicit connect/read/overall deadlines, bounded concurrency, resource cleanup, and cancellation propagation. A synchronous network call inside `async def` can block the event loop. A thread pool can bridge a blocking library, but cancelling the await does not necessarily stop work in that thread.

**Follow-up:** Async improves overlap of I/O. It does not make CPU-heavy work parallel; use a process or worker service where appropriate.

### Q3. Where should request-specific counters live?

In run/thread state or a shared counter system with appropriate atomicity, depending on scope. A mutable attribute on a reused middleware object can mix concurrent requests and users. A distributed rate limit cannot rely on one process's memory.

## 3. Agent loop, tools, and structured output

### Q4. Walk through an agent without saying “LangChain handles it.”

1. Construct messages and expose allowed tool schemas.
2. Call the model within a time/token/iteration budget.
3. If it requests tools, validate names and arguments.
4. Derive identity from trusted application context and authorize each operation.
5. Execute permitted calls, retaining their call IDs and typed outcomes.
6. Append the assistant tool request and matching tool results.
7. Repeat until a final response, interruption, error, or budget limit.

Tool binding exposes a contract; it does not execute the function. The application performs execution. A model can request several tools in one response. Only parallelize calls whose dependencies and side effects permit it.

**Repository exercise:** Trace [04_agent_loop.py](<../Complete MCP/TimeTrackProject/mcp-client-and-advanced/client/04_agent_loop.py>) from discovery through the second model call. Identify the unbounded loop, synchronous model client, sequential tool execution, and string serialization of tool results.

### Q5. Why does valid structured output not prove a booking succeeded?

A schema validates the response's shape. It does not establish factual truth, authorization, or a committed database transaction. Treat model output as a proposal. Return a booking confirmation only from a successful service operation with a durable booking ID.

**Follow-up:** Provider-enforced structure and application validation solve different parts of the problem. Still handle refusal, truncation, unsupported schemas, and business-rule failures. Bound any repair attempts.

### Q6. What makes a good tool interface?

A narrow operation, unambiguous description, constrained input schema, explicit result/status, and documented side-effect semantics. Inject credentials and caller identity from trusted context. Return concise model-facing content and keep bulky artifacts separately when possible.

Avoid tools like `execute_anything(query)` when a smaller typed interface works. Tool descriptions influence model selection but do not enforce permission.

### Q7. When would you use a workflow instead of an autonomous agent?

Use a workflow when the steps, required approvals, and invariants are known. Use model decisions where ambiguity requires interpretation or flexible tool selection. A hybrid often works: model extracts intent; deterministic code validates, authorizes, prices, and commits.

**Strong tradeoff:** Agent flexibility costs predictability, latency, and evaluation effort. Introduce it where a measured benefit justifies those costs.

## 4. Memory, runtime, middleware, and human review

### Q8. Distinguish state, checkpointing, context, store, and database.

| Concept | Purpose | Example |
|---|---|---|
| Conversation/execution state | Current workflow data | Messages, selected movie, pending operation |
| Checkpoint | Persist a recoverable execution snapshot | Resume a paused graph after review |
| Runtime context | Inject application-owned dependencies and caller facts | Authenticated user/tenant ID, configuration |
| Long-term store | Recall selected information across sessions | User preferences with retention rules |
| Business database | Authoritative committed records | Bookings, payments, time entries |

A checkpoint is not automatically an atomic transaction with an external booking service. A store is not automatically an authorization boundary. An in-memory checkpointer demonstrates semantics but does not survive process loss.

### Q9. How do you prevent memory leakage across tenants?

Derive tenant/user identity from verified authentication. Scope queries, namespaces, checkpoints, caches, and retrieval to that identity. Authorize access to thread IDs; an arbitrary supplied thread ID must not grant access. Test cross-tenant reads and writes, deletion, retention, and log redaction.

### Q10. What does middleware add, and where can it go wrong?

Middleware provides lifecycle interception for policy checks, summarization, telemetry, retries, and model/tool configuration. Node hooks can return state updates; wrap hooks control calls to the underlying handler. The class summary's claim that `before_model` is read-only is too broad: official documentation explicitly supports returned state updates. See [custom middleware documentation](https://docs.langchain.com/oss/python/langchain/middleware/custom).

Ordering matters: placing authorization inside a retry can repeat checks; placing redaction after a provider call is too late. Draw entry/exit order for your configured stack and verify it with a trace. Use context managers or `try/finally` for critical cleanup; do not assume an end-of-agent hook covers every failure path.

### Q11. Design a safe retry policy.

Classify failures first. Retry selected transient failures with exponential backoff, jitter, bounded attempts, and a shared deadline. Respect rate-limit guidance. Do not retry authorization errors or invalid inputs as transient infrastructure failures.

For writes, assume a timeout may occur after the remote commit. Use an idempotency key bound to the authenticated caller and canonical operation payload; persist the outcome atomically with the write or use the downstream service's idempotency contract. A repeated key with different arguments must fail.

**Follow-up:** Nesting three model retries inside three workflow retries can multiply calls. Assign retry ownership and track the total budget.

### Q12. How should a refund approval survive a restart?

Persist workflow state and a pending approval record. Show the approver the exact operation, amount, target, and relevant evidence. Bind approval to an operation ID and immutable payload/version. Verify reviewer permissions, expiry, and current business preconditions on resume. Execute with idempotency and keep an audit trail.

Edits must pass validation and policy checks again. Handle rejection, no response, duplicate approval, and two reviewers racing. Approval authorizes a specific action; it does not prove that action completed.

**Version note:** The available decision types and conditional interrupt APIs depend on the installed version. The [current HITL reference](https://reference.langchain.com/python/langchain/agents/middleware/human_in_the_loop) lists approve, edit, reject, and respond. Do not memorize one notebook's API shape as universal.

### Q13. Does hiding a tool from the model enforce security?

It reduces exposure and accidental selection, but the server must still authorize every call. A user may invoke another client, a stale request may resume, or a tool may be reached through a different path. Enforce permissions where the protected data or side effect is accessed.

Prompt injection should be handled as a trust-boundary problem: retrieved documents and tool results are untrusted data. Restrict tools and egress, preserve provenance, validate actions, and require policy checks outside the model. A stronger system prompt alone is insufficient.

## 5. MCP deep dive

### Q14. Explain MCP's host, client, and server.

The host is the application coordinating the user experience and model interaction. A client provides the host's protocol connection to a server. The server exposes capabilities and integrates with a domain system. MCP provides a standard integration boundary; it does not itself choose the application's agent strategy or make tool implementations safe.

### Q15. Tools versus resources versus prompts?

A tool exposes an invocable operation; a resource exposes addressable context; a prompt exposes a reusable interaction template. “Tools act, resources inform, prompts guide” is a useful mnemonic, but tools can also be read-only.

**Repository anchor:** TimeTrack exposes `log_time`, the `timesheet://projects` resource, and `generate_weekly_report`. Explain why exposing project data through both a tool and a resource may serve different host interaction models.

### Q16. Describe an MCP request lifecycle and its version boundary.

First establish the transport and the protocol/version/capability contract applicable to the implementation. Discover available operations, issue requests, correlate results, handle errors, and release resources. In the repository's legacy examples, explain `initialize`, the initialized notification, capability negotiation, and subsequent calls. Distinguish JSON-RPC request IDs from model tool-call IDs.

Do not apply a legacy handshake or capability layout indiscriminately to a newer protocol revision. Inspect the SDK lockfile and negotiated protocol before discussing exact wire messages. Compare [legacy lifecycle documentation](https://github.com/modelcontextprotocol/modelcontextprotocol/blob/main/docs/specification/2025-03-26/basic/lifecycle.mdx) with the version being used.

### Q17. How would you deploy a local versus remote MCP server?

For local stdio, explain subprocess ownership, environment, lifecycle, and keeping protocol stdout free of debug logging. For remote HTTP, explain authenticated connections, TLS termination, connection limits, timeouts, proxy behavior, and any session state required by the selected transport/version. Verify that the hosting platform supports the needed connection lifetime and persistence model.

### Q18. Sampling versus elicitation versus application approval?

Historically, sampling allowed a server to request model generation through the client. Elicitation requests additional user input through a supporting client. Application approval is a business authorization workflow. A yes/no elicitation demo does not provide durable approval records or a reviewer authorization system by itself.

**Version-sensitive correction:** The official sampling page marks sampling deprecated as of protocol version `2026-07-28`, recommends direct provider integration for new implementations, and distinguishes deprecation from immediate removal. TimeTrack's README describes `ctx.sample()`, while the inspected server calls Anthropic directly. Explain the code and revision you actually have. See [official sampling specification](https://modelcontextprotocol.io/specification/draft/client/sampling).

### Q19. Does cancellation undo a write?

No. Cancellation requests stopping work; it does not prove a remote operation stopped before committing. Define commit boundaries and provide an operation-status lookup. Retry with the same idempotency key or reconcile uncertain outcomes. Progress notifications are user feedback, not proof of durable completion.

### Q20. What must a multi-server MCP integration handle?

Tool-name collisions, different authentication scopes, schema/version changes, server availability, timeouts, and provenance. Namespace discovered tools, maintain a controlled registry, validate returned data, and keep credentials isolated per server. Avoid indiscriminately forwarding all conversation history to every server.

## 6. Multi-agent design

### Q21. Which orchestration pattern would you choose?

| Pattern | Good fit | Cost or failure mode |
|---|---|---|
| Single agent | Small toolset, coherent task | Tool/context growth |
| Supervisor with subagents | Isolated specialist work | More calls; summaries lose evidence |
| Handoff | Multi-turn ownership or workflow stage changes | Invalid transitions and broken history |
| Router | Classify then dispatch to relevant domains | Misrouting; fan-out amplification |
| Skills | Load specialist instructions on demand | Instructions do not grant isolation or permissions |
| Custom workflow | Explicit dependencies and fixed guarantees | More orchestration code |

Use the repository's [multi-agent notebooks](<../Weekend 14 - Oct 3 & Oct 4/Multi Agents in Langchain/>) to demonstrate these choices. The [official pattern overview](https://docs.langchain.com/oss/python/langchain/multi-agent) provides the framework vocabulary; the decision should come from your workload.

### Q22. How do parallel graph branches merge state?

Define an explicit reducer for keys multiple branches update, or use separate branch-owned keys and a join step. The router notebook demonstrates `Annotated[list[str], operator.add]`. Concatenation merges values but does not establish business ordering, deduplicate replayed results, or resolve conflicting decisions. For richer workflows, collect results by task ID with explicit merge rules.

### Q23. Should Meridian's specialists run in parallel?

The inspected `ProcurementSupervisor.run_audit()` calls risk, tax, and control specialists sequentially, then performs synthesis. Because each receives the original request rather than a previous specialist's result, they are candidates for bounded concurrent execution.

Approximate sequential latency: `risk + tax + control + synthesis`.
Approximate concurrent latency: `max(risk, tax, control) + synthesis + overhead`.

These are design estimates, not measured benchmarks. Check service quotas and tool side effects first. Define what happens if one specialist fails: block final approval, return an explicitly incomplete report, or retry within the deadline. Never silently synthesize a clean approval from missing checks.

## 7. RAG and evaluation

### Q24. Explain the actual Meridian RAG pipeline.

PDF pages → recursive character chunks → embeddings → Vertex AI Vector Search → retrieval → context plus question → model answer.

The inspected ingestion uses 1,000-character chunks and 100-character overlap. These are characters, not tokens. Retrieval supports similarity (`k=3`), multi-query rewriting, and contextual compression over a larger initial candidate set. The response path returns answer text rather than exposing retrieved evidence to the API caller.

### Q25. How would you improve retrieval quality?

Build a representative, labeled query set before tuning. Segment by document type, query intent, language, freshness, and authorization. Measure recall@k and ranking quality, then inspect failures. Tune parsing, chunking, metadata, lexical/vector hybrid retrieval, and reranking based on the errors found.

Multi-query can improve coverage but adds generation and search cost. Compression can reduce context size while discarding important qualifiers. Larger `k` can improve recall while increasing distraction and latency. Compare alternatives on the same dataset and budget.

### Q26. How would you make ingestion production-ready?

Use stable document and chunk IDs, content hashes, versioned metadata, idempotent upserts, deletion propagation, and a persistent ingestion manifest. Apply tenant/ACL metadata during ingestion and enforce it during retrieval. Use background jobs with bounded batch sizes, retry policy, and visible status. Define whether partially indexed documents are searchable.

The current `ingest_pdf()` calls `add_documents(chunks)` without an explicit stable-ID scheme. Repeated ingestion therefore raises a deduplication/lifecycle concern that should be verified against the vector store's behavior.

### Q27. What does a useful evaluation suite contain?

| Layer | Cases | Measures |
|---|---|---|
| Contracts | Missing fields, invalid dates, malformed tool requests | Validation and error correctness |
| Workflow | Approval, rejection, resume, branch failure | Legal transitions and completion |
| Retrieval | Known evidence, missing evidence, ACL restrictions | Recall@k, ranking, isolation |
| Generation | Answerable, ambiguous, unanswerable | Correctness, groundedness, citation support |
| Security | Tool injection, forged identity, cross-tenant IDs | Unauthorized effects or disclosures |
| Reliability | Commit then timeout, duplicate delivery, restart | Duplicate-write rate and recovery |
| Operations | Load, rate limits, provider outage | p50/p95 latency, cost per successful task |

Scripted models test orchestration deterministically. They do not establish that a live model chooses the right tool or retrieves useful evidence. Use a small live-model evaluation set with versioned prompts/models alongside deterministic tests. Calibrate model-based judging against human review.

## 8. Code-review drills from this repository

These findings come from static inspection of the linked files; proposed improvements have not been implemented or benchmarked.

| File | Observation | What to explain in an interview |
|---|---|---|
| [TimeTrack main.py](<../Complete MCP/TimeTrackProject/mcp-client-and-advanced/main.py>) | `summarize_week` filters only by start date | Parse the date and enforce a seven-day window; clarify inclusive/exclusive bounds |
| Same file | Async summarization calls a synchronous Anthropic client | Event-loop blocking; async client or carefully bounded thread bridge |
| [TimeTrack database.py](<../Complete MCP/TimeTrackProject/mcp-client-and-advanced/database.py>) | Inserts have no idempotency key; identity is an employee-name argument | Duplicate delivery and impersonation; trusted caller mapping and atomic deduplication |
| Same file | Parameterized SQL is used, but date/domain checks are limited | SQL injection protection does not prove business correctness |
| [Raw agent loop](<../Complete MCP/TimeTrackProject/mcp-client-and-advanced/client/04_agent_loop.py>) | `while True` and no overall deadline | Iteration/token/time budgets and typed termination |
| [Auth demo](<../Complete MCP/McpWarmup/server_with_auth.py>) | Static tokens are explicitly development-only | Real token validation, expiry, audience, scope, rotation, tenant boundaries |
| [Runtime notes](<../classes_summary/15 - 22 Aug - Runtime & Human-in-the-Loop.md>) | Example “auth gate” checks server metadata | Presence/absence of metadata does not authenticate a caller |
| [Procurement agents](<../Weekend 14 - Oct 3 & Oct 4/meridian-ai-learner-main/backend/agent/agents.py>) | Specialists run sequentially | Conditional concurrency, quotas, deadlines, incomplete-result policy |
| [RAG retrieval](<../Weekend 14 - Oct 3 & Oct 4/meridian-ai-learner-main/backend/rag/retrieval.py>) | `ask_question` returns only answer text | Return supported source/page citations and evidence metadata |
| [API endpoints](<../Weekend 14 - Oct 3 & Oct 4/meridian-ai-learner-main/backend/api/endpoints.py>) | Upload reads full file; GCS upload is synchronous inside async route | Size limits, streaming, blocking I/O, worker jobs |
| Same file | Upload history is process-local; errors expose `str(e)` | Persistent status, sanitized public errors, correlated internal diagnostics |

Be fair to teaching code: explain its learning purpose before proposing production changes. Do not claim that a concern is an exploited vulnerability or confirmed outage without evidence.

## 9. Two system-design rehearsals

### A. Design CineBot for reliable booking

**Clarify:** expected traffic, inventory owner, payment scope, hold expiry, cancellation rules, review requirements, and latency targets.

**Design:** authenticated API → agent/workflow → policy and validation → booking service → transactional inventory database. Use a durable workflow/checkpoint store for pauses, separate from authoritative booking records. The model interprets intent; the service enforces seat and payment invariants.

**Failure drill:** Payment succeeds, but the booking call times out. Explain idempotency, reservation/hold state, status lookup, webhook reconciliation, and compensating refunds where required. Do not promise a distributed transaction across independent providers without a concrete protocol.

**Concurrency drill:** Two users request the last seat. Enforce uniqueness or conditional inventory updates in the database. A prompt or cache lock is insufficient as the final invariant.

**Measure:** successful confirmed bookings, duplicate charges, inventory conflicts, stuck workflows, escalation rate, and p95 completion latency. State target numbers as assumptions until actual requirements exist.

### B. Design Meridian for enterprise document Q&A and audits

**Clarify:** tenancy, document ACLs, evidence requirements, refresh frequency, audit completeness, retention, scale, and human sign-off.

**Design:** authenticated API → ingestion queue → parsing/chunking/embedding workers → versioned document/index storage; query path applies ACLs before evidence reaches generation. Return citations. Run independent audit specialists with deadlines and join typed evidence before synthesis.

**Failure drill:** One specialist is unavailable. Preserve explicit incomplete status and prevent a final approved audit until policy requirements are met.

**Data drill:** A confidential document is deleted. Remove its searchable chunks, invalidate relevant caches, and apply retention policy to histories and traces. Track deletion completion.

**Release:** fixed evaluation set, adversarial/ACL checks, shadow comparison, limited rollout, and rollback tied to prompt/model/index versions.

## 10. Hands-on exercises

1. **Bound the agent loop — 45 minutes.** Add iteration, wall-clock, and per-call limits in a separate practice copy. Test final response, repeated tool requests, tool failure, and budget exhaustion. Produce a typed termination reason.
2. **Fix weekly reporting — 30 minutes.** Validate the start date and apply a seven-day range. Test entries before, at both boundaries, and after the week. Explain timezone semantics if timestamps replace dates.
3. **Make time-entry writes retry-safe — 60 minutes.** Add a caller-scoped idempotency key and atomic persistence. Simulate a successful commit followed by a lost response; retry must return the original result. Reject key reuse with changed input.
4. **Persist an approval — 60 minutes.** Pause, restart the process, and resume the exact operation. Test rejection, expiry, changed arguments, duplicate approval, and wrong reviewer.
5. **Parallelize procurement safely — 45 minutes.** Use bounded concurrency only after verifying independence. Inject one slow/failing specialist and report incomplete status. Compare measured latency under equivalent workloads.
6. **Evaluate RAG — 90 minutes.** Create 20 questions with expected source passages, including unanswerable and restricted-document cases. Compare similarity, multi-query, and compression using retrieval quality, groundedness, latency, and cost.

Deliver each exercise with a short design note: invariant, tradeoff, tests, remaining limitation. Do not call an exercise production-ready solely because its happy path works.

## 11. Fourteen-day study plan

Allow roughly two focused hours per day; extend the schedule if needed.

| Day | Focus | Evidence of readiness |
|---|---|---|
| 1 | Python, validation, async | Explain Q1–3 and trace blocking I/O |
| 2 | Raw agent loop | Draw and implement the loop from memory |
| 3 | Tool contracts and structured output | Explain why valid JSON is not a successful action |
| 4 | Memory and isolation | Separate state/context/checkpoint/store/database |
| 5 | Middleware and budgets | Trace ordering and retry ownership |
| 6 | Idempotency and transactions | Complete duplicate-write drill |
| 7 | HITL | Demonstrate restart-and-resume approval |
| 8 | MCP primitives and lifecycle | Explain one real request trace and its protocol version |
| 9 | MCP auth, errors, cancellation | Handle timeout-after-commit scenario |
| 10 | Multi-agent patterns | Defend one pattern and reject unnecessary complexity |
| 11 | RAG | Build the small evaluation set |
| 12 | Production system design | Rehearse CineBot and Meridian, 30 minutes each |
| 13 | Project and leadership stories | Prepare three honest ownership stories |
| 14 | Mock interview and weak spots | Run the rubric below and repeat low-scoring areas |

If only three days remain: day 1 agent loop/validation/idempotency; day 2 MCP/memory/HITL; day 3 system design/evaluation/mock interview.

## 12. Senior-level ownership stories

Prepare three stories from your actual experience: a difficult design tradeoff, an incident or failure, and improving delivery across a team. Include your personal decision, alternatives, coordination, measurable result, and what you changed afterward.

For these course projects, say “I studied/built/extended this example” accurately. Do not claim course material as a production deployment you owned. If you lack production metrics, describe your evaluation setup and measured local results instead of inventing scale.

Useful follow-ups to rehearse:

- What did you simplify, and why?
- Which failure did you discover before users did?
- How did you resolve disagreement with another engineer?
- What did you delegate or document so others could maintain it?
- What would trigger changing the architecture?

## 13. Sixty-minute mock interview

- **0–5 min:** Explain one repository project and your contribution.
- **5–15 min:** Draw the agent loop; explain structured output versus execution.
- **15–25 min:** Review TimeTrack's weekly summary and duplicate-write scenario.
- **25–35 min:** Explain MCP integration, versioning, authorization, and cancellation.
- **35–50 min:** Design restart-safe CineBot booking with approval.
- **50–57 min:** Design the evaluation and rollout plan.
- **57–60 min:** Explain one tradeoff you would revisit with real traffic.

Score each area from 0–3: correctness, failure analysis, tradeoff reasoning, security/data boundaries, verification/metrics, and communication/ownership.

- **0:** Cannot explain or makes unsafe assumptions.
- **1:** Can define the concept and describe the happy path.
- **2:** Gives a concrete design with failures and tests.
- **3:** Defends tradeoffs, preserves invariants, and connects decisions to evidence.

A practice target is at least 2 in every area, with 3 in reliability and system design. This is a self-assessment rubric, not an employer's hiring threshold.

## 14. What to add beyond this repository

For a broad Senior Software Engineer interview, also prepare data structures/algorithms, SQL/indexing and query plans, transaction isolation, caching/invalidation, queues and delivery semantics, HTTP/networking, distributed consistency, service deployment, and leadership. For an ML-focused role, add model training, experimental design, and ML system fundamentals. These are role-dependent additions; the repository alone is not a complete senior interview syllabus.

## Start here: diagnostic question

**An agent calls `log_time`. The database commits, but the network connection drops before the result reaches the agent. The agent retries. How do you prevent duplicate entries, recover the original outcome, and prove the design works under concurrent retries?**

Aim for a two-minute answer covering trusted identity, an operation key, atomic persistence, payload mismatch, response replay, and a commit-then-timeout test. Answer aloud before rereading Q11 and exercise 3.
