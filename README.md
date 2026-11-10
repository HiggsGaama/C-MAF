# Microsoft Agent Framework (MAF) for C# Developers

Welcome to the comprehensive, production-grade guide to building intelligent agentic systems with the **Microsoft Agent Framework (MAF)** and **.NET 10**.

This course is designed specifically for average C# and .NET developers who already understand C#, `async`/`await`, dependency injection, and NuGet. It focuses exclusively on agentic AI concepts, architectural patterns, and verified code without boilerplate fluff or cross-language confusion. **100% C# — no Python, no Go.**

---

## Complete Course Index

| # | Module | Level | Focus & Key Concepts |
|---|---|---|---|
| **00** | [Prerequisites & Environment](00-prerequisites.md) | Beginner | .NET 10 setup, LLM basics (tokens, context windows, tool calling), OpenAI / Azure / Foundry providers, API keys vs Entra ID. |
| **01** | [Introduction to MAF](01-introduction.md) | Beginner | Framework architecture, relationship to Semantic Kernel & AutoGen, `IChatClient`, `AIAgent`, package ecosystem map. |
| **02** | [Your First Agent](02-first-agent.md) | Beginner | `AsAIAgent`, `ChatClientAgent`, instructions, `RunAsync`, streaming `RunStreamingAsync`, structured outputs `RunAsync<T>`. |
| **03** | [Tools & Function Calling](03-tools.md) | Beginner | `AIFunctionFactory.Create`, `[Description]`, agents-as-tools (`AsAIFunction`), dynamic per-run tools, hosted tools. |
| **04** | [Conversations & Sessions](04-conversations-and-threads.md) | Intermediate | Multi-turn memory with `AgentSession`, `CreateSessionAsync`, local vs service-managed history, session serialization, `StateBag`. |
| **05** | [Memory Architecture](05-memory.md) | Intermediate | Short-term `ChatHistoryProvider`, bounded history, vector memory via `ChatHistoryMemoryProvider`, long-term memory with Mem0. |
| **06** | [Agent Middleware](06-middleware.md) | Intermediate | `AsBuilder().Use(...)`, delegate pipelines, function-calling middleware, guardrails, short-circuiting, `CurrentRunContext`. |
| **07** | [Retrieval-Augmented Generation (RAG)](07-rag.md) | Intermediate | Chunking, embeddings, `TextSearchProvider`, in-memory vector stores (`Microsoft.Extensions.VectorData`), grounding, citations. |
| **08** | [Agent Skills](08-agent-skills.md) | Intermediate | Progressive disclosure with `SKILL.md`, file-based and code-defined skills, auto-approval, comparing tools vs skills. |
| **09** | [The Agent Harness](09-agent-harness.md) | Advanced | `HarnessAgent`, plan-and-execute modes, context window compaction (`MaxContextWindowTokens`), standing approvals, `LoopAgent`. |
| **10** | [Background Agents & Responses](10-background-agents.md) | Advanced | Long-running asynchronous tasks, `BackgroundAgentsProvider`, polling, continuation tokens, background response streams. |
| **11** | [Workflows Fundamentals](11-workflows.md) | Intermediate | `Executor<TIn, TOut>`, `[MessageHandler]`, `WorkflowBuilder`, edges, conditional branching, supersteps, `InProcessExecution`. |
| **12** | [Agents in Workflows](12-agents-in-workflows.md) | Intermediate | Agents as executors, `TurnToken`, `AIAgentHostOptions`, writer-critic feedback loops, workflow-as-an-agent (`AsAIAgent`). |
| **13** | [Multi-Agent Orchestrations](13-orchestrations.md) | Advanced | `AgentWorkflowBuilder`: Sequential pipelines, concurrent fan-out/fan-in with aggregators, dynamic handoffs, group chats. |
| **14** | [Human-in-the-Loop (HITL)](14-human-in-the-loop.md) | Advanced | `ApprovalRequiredAIFunction`, handling approvals in streaming runs, workflow `RequestPort`, `RequestInfoEvent`, interactive review. |
| **15** | [State Checkpointing & Persistence](15-checkpointing.md) | Advanced | `CheckpointManager`, saving superstep states, durable execution resumption, surviving process restarts. |
| **16** | [Agent Protocols: MCP & A2A](16-protocols.md) | Advanced | Model Context Protocol (`ModelContextProtocol.Client`), consuming & exposing MCP tools, Agent-to-Agent (A2A) protocol. |
| **17** | [Ecosystem Integrations](17-integrations.md) | Advanced | Microsoft Copilot Studio, GitHub Copilot agents, Azure Functions Durable Task integration (`Microsoft.Agents.AI.DurableTask`). |
| **18** | [Hosting & Deployment](18-hosting-and-deployment.md) | Advanced | ASP.NET Core minimal APIs (`Microsoft.Agents.AI.Hosting.AspNetCore`), OpenAI Responses endpoints, Container Apps, scaling. |
| **19** | [Observability & OpenTelemetry](19-observability.md) | Intermediate | GenAI OpenTelemetry semantic conventions, distributed tracing, metrics, sensitive data flags, Aspire and Azure Monitor. |
| **20** | [Security & Production Readiness](20-security-and-best-practices.md) | Advanced | Prompt injection defenses, least-privilege tools, secret management, tenant data isolation, token budgets, production checklist. |
| **21** | [Testing, Evaluation, & DevUI](21-testing-evaluation-devui.md) | Advanced | Unit testing with fake `IChatClient`, `LocalEvaluator`, `EvalChecks`, Foundry Evals, interactive debugging via `DevUI`. |
| **22** | [Migration: SK, AutoGen, Previews](22-migration.md) | Intermediate | Side-by-side refactoring from Semantic Kernel and AutoGen, API rename tables from early previews to 1.28.0 GA. |
| **23** | [Capstone: Enterprise Fulfillment](23-capstone.md) | Advanced | Complete enterprise application combining tools, session memory, workflows, HITL approvals, MCP tools, and OpenTelemetry. |

---

## Tailored Learning Paths

Choose the curriculum path that matches your role and project goals:

### Path 1: Fast Track for Agentic Developers (~3 hours)
*Goal: Build, tool-enable, and deploy single and multi-turn agents rapidly.*
1. [Module 00: Prerequisites & Environment](00-prerequisites.md)
2. [Module 01: Introduction to MAF](01-introduction.md)
3. [Module 02: Your First Agent](02-first-agent.md)
4. [Module 03: Tools & Function Calling](03-tools.md)
5. [Module 04: Conversations & Sessions](04-conversations-and-threads.md)
6. [Module 06: Agent Middleware](06-middleware.md)
7. [Module 21: Testing, Evaluation, & DevUI](21-testing-evaluation-devui.md)

### Path 2: Workflow & Multi-Agent Specialization (~4 hours)
*Goal: Master complex autonomous pipelines, agent swarms, handoffs, and human gates.*
1. [Module 11: Workflows Fundamentals](11-workflows.md)
2. [Module 12: Agents in Workflows](12-agents-in-workflows.md)
3. [Module 13: Multi-Agent Orchestrations](13-orchestrations.md)
4. [Module 14: Human-in-the-Loop (HITL)](14-human-in-the-loop.md)
5. [Module 15: State Checkpointing & Persistence](15-checkpointing.md)
6. [Module 23: Capstone Solution](23-capstone.md)

### Path 3: Enterprise Architecture & Production Readiness (~4 hours)
*Goal: Scale, secure, host, and monitor enterprise agents under strict compliance.*
1. [Module 16: Agent Protocols: MCP & A2A](16-protocols.md)
2. [Module 17: Ecosystem Integrations](17-integrations.md)
3. [Module 18: Hosting & Deployment](18-hosting-and-deployment.md)
4. [Module 19: Observability & OpenTelemetry](19-observability.md)
5. [Module 20: Security & Best Practices](20-security-and-best-practices.md)
6. [Module 22: Migration Guide](22-migration.md)

---

## NuGet Package Reference Matrix (MAF 1.28.0)

All code throughout this course targets **.NET 10** (`net10.0`) and uses verified NuGet versions:

| Package | Shipped Version | Release Status | Primary Purpose |
|---|---|---|---|
| `Microsoft.Agents.AI` | 1.28.0 | **GA** | Core agent abstractions, `AIAgent`, `ChatClientAgent`, evaluation. |
| `Microsoft.Agents.AI.Abstractions` | 1.28.0 | **GA** | Interfaces, `AgentSession`, session state bag contracts. |
| `Microsoft.Agents.AI.OpenAI` | 1.28.0 | **GA** | OpenAI & Azure OpenAI provider bindings, client extensions. |
| `Microsoft.Agents.AI.Anthropic` | 1.28.0 | **GA** | Claude and Anthropic model bindings. |
| `Microsoft.Agents.AI.Workflows` | 1.28.0 | **GA** | Workflow engine, `Executor<TIn, TOut>`, `AgentWorkflowBuilder`. |
| `Microsoft.Agents.AI.Workflows.Generators` | 1.28.0 | **GA** | Roslyn source generator for `[MessageHandler]` attributes. |
| `Microsoft.Agents.AI.Workflows.Declarative` | 1.28.0 | **GA** | JSON / declarative workflow representations. |
| `Microsoft.Agents.AI.Harness` | 1.28.0 | **GA** *(APIs Experimental)* | Advanced agent loop, plan/execute modes, context compaction. |
| `Microsoft.Agents.AI.GitHub.Copilot` | 1.28.0 | **GA** | GitHub Copilot integration bindings. |
| `Microsoft.Agents.AI.Hosting.AspNetCore` | 1.28.0-preview.261106.1 | **Preview** | ASP.NET Core minimal API hosting, `AddAIAgent`. |
| `Microsoft.Agents.AI.DevUI` | 1.28.0-preview.261106.1 | **Preview** | Interactive local debugging web workbench (`MapDevUI`). |
| `Microsoft.Agents.AI.Foundry` | 1.28.0-preview.261106.1 | **Preview** | Azure AI Foundry agents and cloud evaluations. |
| `Microsoft.Agents.AI.A2A` | 1.28.0-preview.261106.1 | **Preview** | Agent-to-Agent communication protocol. |
| `Microsoft.Agents.AI.Mcp` | 1.28.0-alpha.261106.1 | **Alpha** | Hosted MCP tools and client abstractions. |
| `Microsoft.Agents.AI.DurableTask` | 1.16.0 | **GA** | Azure Functions & Durable Task persistent orchestrations. |
| `Microsoft.Extensions.AI` | 10.11.0 | **GA** | Base `IChatClient`, `AIFunction`, `ChatMessage` standards. |
| `Microsoft.Extensions.AI.Evaluation` | 10.11.0 | **GA** | Evaluation framework and standard metric evaluators. |
| `ModelContextProtocol` | 2.4.0 | **GA** | Standardized MCP client transports (`StdioClientTransport`). |
| `OpenTelemetry` | 1.18.0 | **GA** | Industry-standard distributed tracing and metric collection. |

---

## Course Reference Documents

- **[Cheat Sheet](cheat-sheet.md):** Quick-reference syntax for all namespaces, builder patterns, workflows, and attributes.
- **[Course Glossary](glossary.md):** Definitions of all agentic AI and MAF terminology specifically tailored for C# developers.
