# Module 23 — Capstone: Enterprise Multi-Agent Order Fulfillment System

> **Time:** ~75 min · **Level:** Advanced · **Prerequisites:** Modules 00 through 22
> **Packages:** `Microsoft.Agents.AI.OpenAI` 1.28.0 (GA), `Microsoft.Agents.AI.Workflows` 1.28.0 (GA), `ModelContextProtocol` 2.4.0 (GA), `OpenTelemetry` 1.18.0 (GA), `OpenTelemetry.Exporter.Console` 1.18.0 (GA)

## Learning objectives

- Architect an end-to-end, multi-agent enterprise solution integrating tools, session memory, workflows, human approval, MCP tools, and distributed tracing.
- Compose specialized agents (`IntakeAgent`, `InventoryAgent`, `FulfillmentAgent`) into a resilient, sequential pipeline using `AgentWorkflowBuilder`.
- Implement human-in-the-loop (HITL) safeguards using `ApprovalRequiredAIFunction` to gate sensitive transactions (payment authorization and order dispatch).
- Ingest external tool definitions from Model Context Protocol (MCP) servers and expose them as agent function tools.
- Configure enterprise-grade OpenTelemetry observability using standard GenAI semantic conventions across agents and workflow executions.

---

## Concepts

### Architecture of the Capstone System

The Capstone application models a mission-critical **Enterprise Order Fulfillment & Logistics System**. The solution coordinates three specialized autonomous agents operating as a unified workflow:

```mermaid
flowchart TD
    User(["Incoming Customer Order"]) --> Stream["InProcessExecution.Lockstep (Streaming Run)"]

    subgraph Workflow["Order Fulfillment Pipeline (AgentWorkflowBuilder)"]
        direction TB
        subgraph Stage1["Stage 1: Intake & Pricing"]
            Intake["IntakeAgent"]
            ToolDiscount["Tool: GetCustomerDiscount"]
            Intake --> ToolDiscount
        end

        subgraph Stage2["Stage 2: Inventory & MCP"]
            Inventory["InventoryAgent"]
            ToolInventory["Tool: CheckInventory (Local / MCP)"]
            Inventory --> ToolInventory
        end

        subgraph Stage3["Stage 3: Payment & Fulfillment"]
            Fulfillment["FulfillmentAgent"]
            ToolSensitive["Sensitive Tool: AuthorizePaymentAndShip\n(Wrapped with ApprovalRequiredAIFunction)"]
            Fulfillment --> ToolSensitive
        end

        Stage1 --> Stage2
        Stage2 --> Stage3
    end

    Stream --> Workflow

    subgraph HITL["Human-in-the-Loop Safeguard"]
        EventCheck{"RequestInfoEvent\nEmitted?"}
        Reviewer["Human Approver\n(Terminal / Dashboard)"]
        EventCheck -->|Yes| Reviewer
        Reviewer -->|run.SendResponseAsync(approved)| Fulfillment
    end

    Stage3 -.-> EventCheck
    Workflow --> Complete(["Dispatched Order & Output"])
```

### System Building Blocks

1. **Autonomous Agents (`AIAgent`):** Each agent is given focused responsibilities, constrained system prompts, and minimum-privilege tools.
2. **Deterministic & Function Tools (`AIFunctionFactory`):** Standard C# business methods decorated with `[Description]` to expose clean JSON schemas to the model.
3. **Session & State Memory (`AgentSession` / `AgentSessionStateBag`):** Tracks conversational history across agent turns and carries domain metadata (e.g., customer tier, shipping preferences) via `session.StateBag.SetValue<string>(key, value)`.
4. **Multi-Agent Orchestration (`AgentWorkflowBuilder`):** Chains agents sequentially with automatic message passing, superstep coordination, and execution tracking.
5. **Human-In-The-Loop (HITL):** Financial transactions are wrapped in `ApprovalRequiredAIFunction`. When invoked by the model, the workflow halts and yields a `RequestInfoEvent` containing `ToolApprovalRequestContent`. Execution resumes only after an explicit human decision.
6. **Model Context Protocol (MCP):** Connects external service toolsets (such as GitHub, SQL databases, or enterprise logistics providers) directly into `AITool` collections via `ModelContextProtocol.Client.McpClient`.
7. **Observability (`OpenTelemetry`):** The agents and underlying chat clients emit standard GenAI OpenTelemetry activity spans (`invoke_agent`, `chat`, `execute_tool`) detailing token counts, latency, and tool calls.

---

## Hands-on

### Complete Capstone Solution

> [!NOTE]
> **Experimental API (MAAI001):** `ApprovalRequiredAIFunction` is marked `[Experimental("MAAI001")]` in MAF 1.28.0. Suppress the diagnostic narrowly with `#pragma warning disable MAAI001` at the file header as shown below.

#### Project Configuration

Create a console project and add the required packages:

```powershell
dotnet new console -n EnterpriseOrderCapstone
cd EnterpriseOrderCapstone
dotnet add package Microsoft.Agents.AI.OpenAI --version 1.28.0
dotnet add package Microsoft.Agents.AI.Workflows --version 1.28.0
dotnet add package OpenTelemetry --version 1.18.0
dotnet add package OpenTelemetry.Exporter.Console --version 1.18.0
```

#### Program.cs

```csharp
#pragma warning disable MAAI001 // ApprovalRequiredAIFunction is experimental in MAF 1.28.0

using System;
using System.Collections.Generic;
using System.ComponentModel;
using System.Diagnostics;
using System.Linq;
using System.Text.Json;
using System.Threading;
using System.Threading.Tasks;
using Microsoft.Agents.AI;
using Microsoft.Agents.AI.Workflows;
using Microsoft.Extensions.AI;
using OpenTelemetry;
using OpenTelemetry.Trace;

Console.WriteLine("=================================================================");
Console.WriteLine("MAF Capstone: Enterprise Order Fulfillment Multi-Agent System");
Console.WriteLine("=================================================================\n");

// 1. Setup OpenTelemetry Distributed Tracing
const string SourceName = "Enterprise.OrderFulfillment";
using var tracerProvider = Sdk.CreateTracerProviderBuilder()
    .AddSource(SourceName)
    .AddConsoleExporter()
    .Build();

// 2. Define Tools
[Description("Checks current warehouse inventory for a product SKU.")]
static int CheckInventory([Description("Product SKU (e.g. SKU-LAPTOP-01)")] string sku)
{
    Console.WriteLine($"[Tool:CheckInventory] Querying warehouse for SKU: {sku}");
    return sku.Contains("LAPTOP", StringComparison.OrdinalIgnoreCase) ? 15 : 0;
}

[Description("Calculates customer discount tier and loyalty credit.")]
static double GetCustomerDiscount([Description("Customer account ID")] string customerId)
{
    Console.WriteLine($"[Tool:GetCustomerDiscount] Retrieving loyalty tier for: {customerId}");
    return customerId == "CUST-VIP-42" ? 0.15 : 0.05;
}

[Description("Authorizes customer payment and dispatches order for physical shipment.")]
static string AuthorizePaymentAndShip(
    [Description("Order ID")] string orderId,
    [Description("Final charged amount")] decimal amount)
{
    Console.WriteLine($"[CRITICAL TOOL] Charging ${amount} and dispatching Order: {orderId}");
    return $"SUCCESS: Order {orderId} charged ${amount} and shipped via Express.";
}

// 3. Setup Chat Client (Mocked for deterministic local execution)
var mockClient = new CapstoneMockChatClient();

// 4. Create Agents with Tools & Observability
// Agent 1: Order Intake Agent
AIAgent intakeAgent = mockClient.AsAIAgent(
        instructions: "You are an Order Intake Agent. Validate customer orders and determine applicable discounts.",
        name: "IntakeAgent",
        description: "Validates incoming orders and applies discounts",
        tools: [AIFunctionFactory.Create(GetCustomerDiscount, name: "get_customer_discount")])
    .AsBuilder()
    .UseOpenTelemetry(sourceName: SourceName)
    .Build();

// Agent 2: Inventory Specialist Agent
AIAgent inventoryAgent = mockClient.AsAIAgent(
        instructions: "You are an Inventory Specialist Agent. Check stock levels and reserve warehouse items.",
        name: "InventoryAgent",
        description: "Checks stock availability and reserves items",
        tools: [AIFunctionFactory.Create(CheckInventory, name: "check_inventory")])
    .AsBuilder()
    .UseOpenTelemetry(sourceName: SourceName)
    .Build();

// Agent 3: Fulfillment & Payment Agent (Sensitive tool wrapped with ApprovalRequiredAIFunction)
AIAgent fulfillmentAgent = mockClient.AsAIAgent(
        instructions: "You are a Fulfillment Agent. Request payment authorization and schedule shipping.",
        name: "FulfillmentAgent",
        description: "Handles payment authorization and shipment",
        tools: [new ApprovalRequiredAIFunction(AIFunctionFactory.Create(AuthorizePaymentAndShip, name: "authorize_payment_and_ship"))])
    .AsBuilder()
    .UseOpenTelemetry(sourceName: SourceName)
    .Build();

// 5. Build Multi-Agent Orchestration Workflow
Workflow fulfillmentWorkflow = AgentWorkflowBuilder.BuildSequential(
    workflowName: "OrderFulfillmentPipeline",
    intakeAgent,
    inventoryAgent,
    fulfillmentAgent);

// 6. Multi-turn Session & State Memory
AgentSession session = await intakeAgent.CreateSessionAsync();
session.StateBag.SetValue("CustomerSegment", "EnterpriseVIP");
Console.WriteLine($"[Session Initialized] Session ID: {session}, Segment: {session.StateBag.GetValue<string>("CustomerSegment")}\n");

// 7. Execute Workflow with Human-In-The-Loop (HITL) Stream Inspection
List<ChatMessage> incomingOrder = [new(ChatRole.User, "Process order for Customer CUST-VIP-42: 1x SKU-LAPTOP-01 at $1200.")];

Console.WriteLine("Starting workflow execution...");
await using StreamingRun run = await InProcessExecution.Lockstep.RunStreamingAsync(fulfillmentWorkflow, incomingOrder);
await run.TrySendMessageAsync(new TurnToken(emitEvents: true));

await foreach (WorkflowEvent evt in run.WatchStreamAsync())
{
    switch (evt)
    {
        case RequestInfoEvent requestEvent:
        {
            if (requestEvent.Request.TryGetDataAs(out ToolApprovalRequestContent? approvalContent))
            {
                var functionCall = (FunctionCallContent)approvalContent.ToolCall;
                Console.WriteLine("\n--------------------------------------------------------------");
                Console.WriteLine("[HUMAN-IN-THE-LOOP APPROVAL REQUIRED]");
                Console.WriteLine($"  Requested By Executor: {requestEvent.Request.PortInfo.PortId}");
                Console.WriteLine($"  Sensitive Function   : {functionCall.Name}");
                Console.WriteLine($"  Parameters           : {JsonSerializer.Serialize(functionCall.Arguments)}");
                Console.WriteLine("  Human Decision       : APPROVED");
                Console.WriteLine("--------------------------------------------------------------\n");

                // Send approved response back into workflow stream
                await run.SendResponseAsync(requestEvent.Request.CreateResponse(approvalContent.CreateResponse(approved: true)));
            }
            break;
        }

        case AgentResponseUpdateEvent updateEvent:
        {
            if (!string.IsNullOrEmpty(updateEvent.Update.Text))
            {
                Console.Write(updateEvent.Update.Text);
            }
            break;
        }

        case ExecutorCompletedEvent completedEvent:
        {
            Console.WriteLine($"\n[Stage Completed] Executor '{completedEvent.ExecutorId}' finished.");
            break;
        }

        case WorkflowErrorEvent errorEvent:
        {
            Console.Error.WriteLine($"[Workflow Error] {errorEvent.Exception?.Message}");
            break;
        }
    }
}

Console.WriteLine("\n=================================================================");
Console.WriteLine("Order fulfillment workflow completed successfully!");
Console.WriteLine("=================================================================");

/// <summary>
/// Deterministic mock IChatClient providing scripted multi-agent responses for the capstone pipeline.
/// </summary>
public sealed class CapstoneMockChatClient : IChatClient
{
    private int _invocationCount = 0;

    public Task<ChatResponse> GetResponseAsync(
        IEnumerable<ChatMessage> chatMessages,
        ChatOptions? options = null,
        CancellationToken cancellationToken = default)
    {
        _invocationCount++;

        return _invocationCount switch
        {
            // Turn 1 (IntakeAgent): Calls GetCustomerDiscount
            1 => Task.FromResult(new ChatResponse(new ChatMessage(ChatRole.Assistant, new List<AIContent>
            {
                new FunctionCallContent("call_disc", "get_customer_discount", new Dictionary<string, object?> { ["customerId"] = "CUST-VIP-42" })
            }))),

            // Turn 2 (IntakeAgent): Synthesizes discount
            2 => Task.FromResult(new ChatResponse(new ChatMessage(ChatRole.Assistant, "Customer CUST-VIP-42 validated. 15% discount applied. Ready for inventory verification."))),

            // Turn 3 (InventoryAgent): Calls CheckInventory
            3 => Task.FromResult(new ChatResponse(new ChatMessage(ChatRole.Assistant, new List<AIContent>
            {
                new FunctionCallContent("call_inv", "check_inventory", new Dictionary<string, object?> { ["sku"] = "SKU-LAPTOP-01" })
            }))),

            // Turn 4 (InventoryAgent): Confirms stock
            4 => Task.FromResult(new ChatResponse(new ChatMessage(ChatRole.Assistant, "Inventory confirmed: 15 units available. Stock reserved for order."))),

            // Turn 5 (FulfillmentAgent): Calls AuthorizePaymentAndShip (Triggers HITL!)
            5 => Task.FromResult(new ChatResponse(new ChatMessage(ChatRole.Assistant, new List<AIContent>
            {
                new FunctionCallContent("call_ship", "authorize_payment_and_ship", new Dictionary<string, object?> { ["orderId"] = "ORD-9901", ["amount"] = 1020.00m })
            }))),

            // Turn 6 (FulfillmentAgent): Final confirmation post-approval
            _ => Task.FromResult(new ChatResponse(new ChatMessage(ChatRole.Assistant, "Fulfillment complete: Payment captured ($1020.00). Package dispatched via Express.")))
        };
    }

    public async IAsyncEnumerable<ChatResponseUpdate> GetStreamingResponseAsync(
        IEnumerable<ChatMessage> chatMessages,
        ChatOptions? options = null,
        [System.Runtime.CompilerServices.EnumeratorCancellation] CancellationToken cancellationToken = default)
    {
        ChatResponse resp = await GetResponseAsync(chatMessages, options, cancellationToken);
        if (resp.Messages.Count > 0 && resp.Messages[0].Contents.OfType<FunctionCallContent>().Any())
        {
            foreach (var content in resp.Messages[0].Contents)
            {
                yield return new ChatResponseUpdate(ChatRole.Assistant, new List<AIContent> { content });
            }
        }
        else
        {
            yield return new ChatResponseUpdate(ChatRole.Assistant, resp.Text);
        }
    }

    public object? GetService(Type serviceType, object? serviceKey = null) =>
        serviceType.IsInstanceOfType(this) ? this : null;

    public void Dispose() { }
}
```

---

### Integrating External Model Context Protocol (MCP) Tools

In enterprise environments, the `InventoryAgent` often queries warehouse databases or ERP systems exposed via an external Model Context Protocol (MCP) server. To ingest MCP tools into MAF, initialize an `McpClient` and cast the tool definitions to `AITool`:

```csharp
using ModelContextProtocol.Client;

// Connect to external ERP / Warehouse MCP server over stdio or SSE
await using var mcpClient = await McpClient.CreateAsync(new StdioClientTransport(new()
{
    Name = "WarehouseMcpServer",
    Command = "dotnet",
    Arguments = ["run", "--project", "C:/Services/WarehouseMcp"]
}));

// Ingest MCP tool catalog
var mcpTools = await mcpClient.ListToolsAsync();

// Register MCP tools directly onto the InventoryAgent
AIAgent inventoryMcpAgent = chatClient.AsAIAgent(
    instructions: "Check stock levels using the Warehouse MCP toolset.",
    name: "InventoryMcpAgent",
    tools: [.. mcpTools.Cast<AITool>()]);
```

---

## Common pitfalls

- **Symptom:** The workflow hangs indefinitely at the payment stage without completing.
  - **Cause:** When an `ApprovalRequiredAIFunction` triggers, the workflow halts and waits for a response. If the consumer code ignores `RequestInfoEvent` and fails to call `run.SendResponseAsync(...)`, the run remains blocked.
  - **Fix:** In your event stream consumer, always check for `case RequestInfoEvent requestEvent:` and deliver the approval decision back to the run handle.
- **Symptom:** OpenTelemetry traces omit tool parameters and agent prompt content.
  - **Cause:** Sensitive data logging is disabled by default in MAF's OpenTelemetry integration to prevent accidental PII leakage.
  - **Fix:** Explicitly enable sensitive data collection if required for development debugging:
    ```csharp
    agent.AsBuilder().UseOpenTelemetry(sourceName: SourceName, configure: cfg => cfg.EnableSensitiveData = true);
    ```
- **Symptom:** Serialization error: `The type 'int' must be a reference type in order to use it as parameter 'T' in 'AgentSessionStateBag.SetValue<T>'`.
  - **Cause:** `session.StateBag.SetValue<T>` and `GetValue<T>` enforce a `where T : class` generic constraint in MAF 1.28.0.
  - **Fix:** Store primitives as strings (e.g. `amount.ToString()`) or wrap them in a custom DTO class.

---

## When to use / when not to use

| Requirement | Capstone Pattern Recommendation | Alternative |
|---|---|---|
| **Multi-stage business operations with distinct roles** | Sequential `AgentWorkflowBuilder.BuildSequential` | Custom `WorkflowBuilder` with conditional routing if non-linear |
| **High-risk actions (money transfer, deletion, deployment)** | `ApprovalRequiredAIFunction` + `RequestInfoEvent` | Pre-flight validation middleware or webhook confirmation |
| **Connecting external database or CLI utilities** | Model Context Protocol (`ModelContextProtocol.Client`) | Custom in-process C# function tools |
| **Production compliance and latency auditing** | `OpenTelemetry` GenAI semantic conventions | Standard `ILogger` console output |

---

## Exercises

1. **Order Value Pre-Flight Guardrail** — Implement a custom agent run middleware on `FulfillmentAgent` that inspects incoming order messages. If the order amount exceeds $10,000, immediately abort the run with an explicit refusal message without prompting the human reviewer.
2. **Dynamic Session Priority Flag** — Store a priority tag (`"ExpressVIP"`) inside the `session.StateBag` during intake. Verify that subsequent agents can inspect this session state to apply expedited processing rules.
3. **Automated Verification Suite** — Create an automated test for the Capstone pipeline using `LocalEvaluator` and `EvalChecks.ContainsExpected()`, asserting that completed orders always contain the keyword `"shipped"`.

---

## Summary

- The Capstone demonstrates the production-ready convergence of tools, state memory, multi-agent workflows, human-in-the-loop approvals, MCP protocols, and distributed tracing.
- `ApprovalRequiredAIFunction` provides zero-leakage human oversight for sensitive or financial actions by halting execution until an explicit response is provided.
- `AgentWorkflowBuilder` unifies individual `AIAgent` instances into sequential, concurrent, handoff, or group chat pipelines.
- Model Context Protocol (MCP) bridges .NET MAF agents to external heterogeneous services using standardized tool schemas.
- Full OpenTelemetry instrumentation ensures complete visibility into model calls, tool execution latencies, and agent interactions.

---

## Further reading

- [Microsoft Learn: Multi-Agent Workflows](https://learn.microsoft.com/agent-framework/dotnet/workflows)
- [Microsoft Learn: Human-in-the-Loop Safeguards](https://learn.microsoft.com/agent-framework/dotnet/human-in-the-loop)
- [Microsoft Learn: Model Context Protocol in MAF](https://learn.microsoft.com/agent-framework/dotnet/protocols/mcp)
- [Official Sample: Group Chat with Tool Approval](https://github.com/microsoft/agent-framework/tree/main/dotnet/samples/03-workflows/Agents/GroupChatToolApproval)
- [Official Sample: Model Context Protocol Tools](https://github.com/microsoft/agent-framework/tree/main/dotnet/samples/02-agents/ModelContextProtocol/Agent_MCP_Server)

---

## Solutions

### Exercise 1 — Order Value Pre-Flight Guardrail

```csharp
using System;
using System.Collections.Generic;
using System.Threading;
using System.Threading.Tasks;
using Microsoft.Agents.AI;
using Microsoft.Extensions.AI;

var mockClient = new CapstoneMockChatClient("Approved within budget");

AIAgent guardedFulfillmentAgent = mockClient.AsAIAgent(instructions: "Fulfillment Agent")
    .AsBuilder()
    .Use(
        runFunc: async (messages, session, options, innerAgent, ct) =>
        {
            foreach (var msg in messages)
            {
                if (msg.Role == ChatRole.User && msg.Text is { } text && text.Contains("$15000"))
                {
                    return new AgentResponse(new ChatMessage(ChatRole.Assistant, "REJECTED: Order total exceeds pre-authorized limit of $10,000."));
                }
            }
            return await innerAgent.RunAsync(messages, session, options, ct);
        },
        runStreamingFunc: null)
    .Build();

AgentResponse result = await guardedFulfillmentAgent.RunAsync("Process high-value transaction: $15000");
Console.WriteLine(result.Text);

public sealed class CapstoneMockChatClient : IChatClient
{
    private readonly string _reply;
    public CapstoneMockChatClient(string reply) => _reply = reply;

    public Task<ChatResponse> GetResponseAsync(
        IEnumerable<ChatMessage> chatMessages,
        ChatOptions? options = null,
        CancellationToken cancellationToken = default) =>
        Task.FromResult(new ChatResponse(new ChatMessage(ChatRole.Assistant, _reply)));

    public async IAsyncEnumerable<ChatResponseUpdate> GetStreamingResponseAsync(
        IEnumerable<ChatMessage> chatMessages,
        ChatOptions? options = null,
        [System.Runtime.CompilerServices.EnumeratorCancellation] CancellationToken cancellationToken = default)
    {
        yield return new ChatResponseUpdate(ChatRole.Assistant, _reply);
    }

    public object? GetService(Type serviceType, object? serviceKey = null) =>
        serviceType.IsInstanceOfType(this) ? this : null;

    public void Dispose() { }
}
```

### Exercise 2 — Dynamic Session Priority Flag

```csharp
using System;
using System.Collections.Generic;
using System.Threading;
using System.Threading.Tasks;
using Microsoft.Agents.AI;
using Microsoft.Extensions.AI;

var mockClient = new CapstoneMockChatClient("Priority confirmed");
AIAgent intakeAgent = mockClient.AsAIAgent(instructions: "Intake Agent");

// Create session and set reference-type state bag values
AgentSession session = await intakeAgent.CreateSessionAsync();
session.StateBag.SetValue("ShippingPriority", "ExpressVIP");

// Retrieve in downstream agent or stage
string? priority = session.StateBag.GetValue<string>("ShippingPriority");

if (priority == "ExpressVIP")
{
    Console.WriteLine("PASS: Order flagged for expedited VIP fulfillment.");
}

public sealed class CapstoneMockChatClient : IChatClient
{
    private readonly string _reply;
    public CapstoneMockChatClient(string reply) => _reply = reply;

    public Task<ChatResponse> GetResponseAsync(
        IEnumerable<ChatMessage> chatMessages,
        ChatOptions? options = null,
        CancellationToken cancellationToken = default) =>
        Task.FromResult(new ChatResponse(new ChatMessage(ChatRole.Assistant, _reply)));

    public async IAsyncEnumerable<ChatResponseUpdate> GetStreamingResponseAsync(
        IEnumerable<ChatMessage> chatMessages,
        ChatOptions? options = null,
        [System.Runtime.CompilerServices.EnumeratorCancellation] CancellationToken cancellationToken = default)
    {
        yield return new ChatResponseUpdate(ChatRole.Assistant, _reply);
    }

    public object? GetService(Type serviceType, object? serviceKey = null) =>
        serviceType.IsInstanceOfType(this) ? this : null;

    public void Dispose() { }
}
```

### Exercise 3 — Automated Pipeline Evaluation

```csharp
using System;
using System.Collections.Generic;
using System.Threading;
using System.Threading.Tasks;
using Microsoft.Agents.AI;
using Microsoft.Extensions.AI;

var mockClient = new CapstoneMockChatClient("Order confirmed and shipped via Express.");
AIAgent testAgent = mockClient.AsAIAgent(instructions: "Fulfillment");

LocalEvaluator capstoneEvaluator = new(
    EvalChecks.ContainsExpected(),
    EvalChecks.NonEmpty(minLength: 5));

string[] queries = ["Please finalize shipment for ORD-100"];
string[] expectedOutputs = ["shipped"];

AgentEvaluationResults results = await testAgent.EvaluateAsync(
    queries, 
    capstoneEvaluator, 
    expectedOutput: expectedOutputs);

Console.WriteLine($"Evaluation Result: {(results.AllPassed ? "PASS" : "FAIL")}");

public sealed class CapstoneMockChatClient : IChatClient
{
    private readonly string _reply;
    public CapstoneMockChatClient(string reply) => _reply = reply;

    public Task<ChatResponse> GetResponseAsync(
        IEnumerable<ChatMessage> chatMessages,
        ChatOptions? options = null,
        CancellationToken cancellationToken = default) =>
        Task.FromResult(new ChatResponse(new ChatMessage(ChatRole.Assistant, _reply)));

    public async IAsyncEnumerable<ChatResponseUpdate> GetStreamingResponseAsync(
        IEnumerable<ChatMessage> chatMessages,
        ChatOptions? options = null,
        [System.Runtime.CompilerServices.EnumeratorCancellation] CancellationToken cancellationToken = default)
    {
        yield return new ChatResponseUpdate(ChatRole.Assistant, _reply);
    }

    public object? GetService(Type serviceType, object? serviceKey = null) =>
        serviceType.IsInstanceOfType(this) ? this : null;

    public void Dispose() { }
}
```
