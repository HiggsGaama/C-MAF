# Microsoft Agent Framework (MAF) — C# Developer Glossary

A comprehensive glossary of terms, design patterns, and architectural abstractions used in the **Microsoft Agent Framework (MAF)** and modern agentic AI development on **.NET 10**.

---

### A

#### A2A (Agent-to-Agent Protocol)
An interoperability protocol allowing AI agents hosted in separate processes, microservices, or networks to communicate, delegate subtasks, exchange contextual history, and stream execution updates using standardized HTTP/JSON-RPC semantics. In MAF, client connectivity is provided by `Microsoft.Agents.AI.A2A`.

#### Agent (`AIAgent`)
The central entity in MAF responsible for orchestrating interactions between a language model (`IChatClient`), a prompt template/instruction set, and registered function tools. Agents encapsulate system personas, manage session history, execute tool call feedback loops, and emit responses.

#### Agent Harness (`HarnessAgent`)
An advanced runtime wrapper around an `AIAgent` that equips the agent with autonomous execution capabilities, including multi-step planning, file-based memory, standing tool permissions, dynamic loop termination, and context window compaction (`MaxContextWindowTokens`). Shipped in `Microsoft.Agents.AI.Harness`.

#### `AgentResponse`
The strongly-typed return object yielded by an agent invocation (`agent.RunAsync`). Contains the final synthesized text (`response.Text`), the complete list of generated messages (`response.Messages`), and token usage diagnostics (`response.Usage`).

#### `AgentResponseUpdate`
The incremental chunk emitted during a streaming agent invocation (`agent.RunStreamingAsync`). Carries delta text tokens, function call fragments, or role assignments as they arrive from the model provider.

#### `AgentSession`
The abstraction that maintains state and conversational history across multiple interactions with an agent. Replaces early preview concepts like `AgentThread`. Sessions manage short-term memory providers and carry a thread-safe `StateBag` for domain-specific metadata.

#### `AgentWorkflowBuilder`
A high-level orchestration builder in `Microsoft.Agents.AI.Workflows` for constructing standard multi-agent patterns without writing manual message-passing loops:
- `BuildSequential`: Linear pipeline where each agent transforms or appends to prior results.
- `BuildConcurrent`: Parallel execution fan-out with an aggregator fan-in step.
- `CreateHandoffBuilderWith`: Dynamic routing where agents transfer control based on conversation intent.
- `CreateGroupChatBuilderWith`: Multi-agent collaboration coordinated by a `GroupChatManager`.

#### `AIFunction` / `AIFunctionFactory`
The foundational tool abstraction in `Microsoft.Extensions.AI`. `AIFunctionFactory.Create` takes a standard C# method (delegate, static method, or instance method), inspects its parameters and XML/`[Description]` attributes, and generates a valid JSON Schema description passed to the LLM.

#### `AITool`
The abstract base class representing any capability an AI model can interact with during execution, encompassing local C# `AIFunction` instances, external MCP tools, and hosted tools (code interpreter, web search, file search).

#### `ApprovalRequiredAIFunction`
A security wrapper for sensitive function tools (e.g., executing financial transfers, deleting records, or running shell commands). When invoked by the model, execution halts, emitting a `RequestInfoEvent` to request approval from a human reviewer before running.

---

### C

#### Checkpointing (`CheckpointManager`)
A mechanism for persisting the internal execution state of a workflow across supersteps to durable storage (e.g., disk, Azure Blob, Cosmos DB). Enables resilient workflows to pause, withstand machine restarts, and resume execution asynchronously.

#### Compaction
The process of summarizing, truncating, or evicting older messages from an agent's context window when token limits are approached. The agent harness utilizes context compaction pipelines to maintain long-running conversations within model window boundaries.

#### Context Window
The fixed upper bound of tokens (input prompt + output completion) that a language model can process within a single inference call. Exceeding this boundary triggers provider errors.

---

### D

#### `DelegatingAIAgent`
An agent subclass that wraps an inner `AIAgent`, used to create reusable middleware components, telemetry hooks, or context-enrichment filters. Equivalent in concept to `DelegatingHandler` in `System.Net.Http`.

#### Developer UI (`Microsoft.Agents.AI.DevUI`)
An embedded web-based workbench served directly from an ASP.NET Core host during local development (`app.MapDevUI()`). Provides an interactive chat interface, real-time tool parameter inspection, and workflow visualization.

---

### E

#### `EvalCheck` / `EvalItem`
Primitives in MAF's evaluation suite. An `EvalItem` encapsulates a test query, response, conversation history, and expected outputs. An `EvalCheck` is a delegate evaluated by `LocalEvaluator` to score response quality.

#### Executor (`Executor<TIn, TOut>`)
The fundamental processing unit in a MAF workflow. Inherits from `Executor<TIn, TOut>` and overrides `HandleAsync(TIn, IWorkflowContext, CancellationToken)`. Executors can be plain C# computational logic, agent bindings, or external API gateways.

---

### F

#### Fan-out / Fan-in
An architectural workflow pattern where a single triggering event or message is distributed concurrently to multiple worker executors or agents (fan-out), whose outputs are subsequently collected and synthesized by a coordinating aggregator executor (fan-in).

#### Function Calling / Tool Calling
A model capability where the LLM recognizes that an incoming prompt requires external data or side-effects, selects a matching tool schema from its prompt definitions, and generates structured JSON arguments instead of a natural language response.

---

### G

#### Grounding
The practice of supplying verifiable, domain-specific reference facts (via RAG, database queries, or tool invocations) within the model's system prompt to anchor its generation and prevent hallucinations.

#### Group Chat (`GroupChatManager`)
A collaborative multi-agent architecture where multiple agents converse in a shared conversation space. A `GroupChatManager` (such as `RoundRobinGroupChatManager`) inspects conversation state at each turn to select the next agent to speak.

---

### H

#### Handoff
A dynamic routing pattern where an active agent decides to transfer conversation ownership to another specialized agent (e.g., `TriageAgent` handing off to `BillingAgent`). Configured in MAF using `AgentWorkflowBuilder.CreateHandoffBuilderWith`.

#### Human-in-the-Loop (HITL)
An architectural requirement where an automated AI process pauses execution to wait for validation, input, or an approval decision from a human operator before proceeding with sensitive actions.

---

### I

#### `IChatClient`
The foundational interface defined in `Microsoft.Extensions.AI` representing an abstraction over language model providers. Exposes `GetResponseAsync` and `GetStreamingResponseAsync`. Both OpenAI, Azure AI, Anthropic, and local test doubles implement this interface.

---

### L

#### `LocalEvaluator`
An in-memory, deterministic evaluator in MAF that executes rule-based assertions (`EvalChecks.ContainsExpected()`, `EvalChecks.KeywordCheck()`, `FunctionEvaluator.Create(...)`) on agent responses without calling an external LLM judge.

#### `LoopAgent`
An agent harness specialization that repeatedly invokes an inner agent until an explicit termination evaluator (e.g., `AIJudgeLoopEvaluator` or `CompletionMarkerLoopEvaluator`) determines the task has reached completion.

---

### M

#### Model Context Protocol (MCP)
An open standard created by Anthropic that standardizes how applications provide context, tools, and resources to LLMs. MAF agents consume MCP servers using `ModelContextProtocol.Client.McpClient` and expose their tools as `AITool`.

#### Middleware
Software components assembled into an agent invocation pipeline to handle cross-cutting concerns:
- **Agent Run Middleware (`AsBuilder().Use(runFunc)`):** Intercepts input messages, sessions, and final responses.
- **Function Calling Middleware:** Intercepts tool calls before and after local method execution.
- **Chat Client Middleware (`ChatClientBuilder.Use`):** Intercepts low-level HTTP requests, retries, and token logging.

---

### R

#### `RequestInfoEvent` / `RequestPort`
Workflow events emitted when an executor or approval-gated tool requires external or human intervention. The hosting environment handles this event, presents the request, and delivers the response back via `run.SendResponseAsync`.

#### Retrieval-Augmented Generation (RAG)
An architecture that enhances model accuracy by searching an external knowledge base or vector database for relevant text passages and injecting them into the agent's prompt context prior to generation.

---

### S

#### `StateBag` (`AgentSessionStateBag`)
A thread-safe, session-scoped dictionary supporting strongly-typed storage and retrieval of domain metadata (`session.StateBag.SetValue<T>`, `session.StateBag.GetValue<T>`). Automatically serialized during session snapshots.

#### Streaming Run (`StreamingRun`)
A workflow execution handle returned by `InProcessExecution.RunStreamingAsync`. Exposes `WatchStreamAsync()` to asynchronously stream events (`SuperStepStartedEvent`, `AgentResponseUpdateEvent`, `WorkflowOutputEvent`) as they occur.

#### Superstep
A discrete, coordinated execution cycle within a MAF workflow. During a superstep, all executors with pending input messages run concurrently; messages produced during the superstep are buffered and routed to connected executors at the start of the next superstep.

---

### T

#### `TurnToken`
A signaling message passed into a workflow stream (`run.TrySendMessageAsync(new TurnToken())`) instructing agents within an orchestration to advance the conversation by one turn.

---

### V

#### Vector Store
A database optimized for storing high-dimensional vector embeddings and performing semantic similarity searches. In MAF, integrated via `Microsoft.Extensions.VectorData` and `ChatHistoryMemoryProvider`.

---

### W

#### Workflow (`Workflow`)
A directed graph of interconnected computational steps (`Executor`) connected by edges. Workflows provide deterministic control flow, conditional branching, looping, and parallel execution for complex business logic.

---

### Y

#### `YieldOutputAsync`
A method on `IWorkflowContext` called by an executor to publish an intermediate or final output message to the workflow's public event stream (`WorkflowOutputEvent`).
