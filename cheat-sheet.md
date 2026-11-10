# Microsoft Agent Framework (MAF) — C# Developer Cheat Sheet

A concise reference of core APIs, builder extensions, workflow topologies, and architectural patterns in **MAF 1.28.0** and **.NET 10**.

---

## 1. Key Namespaces Quick Reference

```csharp
using Microsoft.Agents.AI;                    // AIAgent, ChatClientAgent, AgentSession, LocalEvaluator
using Microsoft.Agents.AI.Workflows;          // Workflow, WorkflowBuilder, AgentWorkflowBuilder, Executor<TIn, TOut>
using Microsoft.Agents.AI.Hosting;            // AddAIAgent, AddWorkflow, MapOpenAIResponses
using Microsoft.Agents.AI.DevUI;              // AddDevUI, MapDevUI (Preview)
using Microsoft.Agents.AI.Foundry;            // AIProjectClient extensions, FoundryEvals (Preview)
using Microsoft.Extensions.AI;                 // IChatClient, AIFunctionFactory, ChatMessage, ChatRole
using Microsoft.Extensions.AI.Evaluation;      // EvaluationResult, IEvaluator
using ModelContextProtocol.Client;            // McpClient, StdioClientTransport
```

---

## 2. Agent Initialization Patterns

### Direct from OpenAI ChatClient (GA)
```csharp
using OpenAI;
using OpenAI.Chat;
using Microsoft.Agents.AI;

AIAgent agent = new OpenAIClient(apiKey)
    .GetChatClient("gpt-5.4-mini")
    .AsAIAgent(
        instructions: "You are a helpful assistant.",
        name: "AssistantAgent",
        description: "General customer support bot",
        tools: [AIFunctionFactory.Create(MyTool)]);
```

### Direct from Raw `IChatClient` (MEAI)
```csharp
IChatClient chatClient = new OpenAIClient(apiKey).GetChatClient("gpt-5.4-mini").AsIChatClient();

AIAgent agent = chatClient.AsAIAgent(
    instructions: "You are a code review assistant.",
    name: "ReviewerAgent");
```

### From Azure AI Foundry (Preview)
```csharp
using Azure.AI.Projects;
using Azure.Identity;
using Microsoft.Agents.AI;

AIProjectClient projectClient = new(new Uri(endpoint), new DefaultAzureCredential());

AIAgent agent = projectClient.AsAIAgent(
    model: "gpt-5.4-mini",
    instructions: "Enterprise compliance agent.",
    name: "ComplianceAgent");
```

---

## 3. Invocation & Streaming

### Batch Invocations
```csharp
// Simple string input
AgentResponse response = await agent.RunAsync("Calculate quarterly budget.");
Console.WriteLine(response.Text);

// List of chat messages
List<ChatMessage> conversation = [new(ChatRole.User, "Hello!"), new(ChatRole.Assistant, "Hi!"), new(ChatRole.User, "Next question")];
AgentResponse multiTurnResponse = await agent.RunAsync(conversation);

// Structured Output
AgentResponse<OrderSummary> structured = await agent.RunAsync<OrderSummary>("Summarize order #100");
OrderSummary summary = structured.Result;
```

### Real-time Streaming
```csharp
await foreach (AgentResponseUpdate chunk in agent.RunStreamingAsync("Generate a technical report"))
{
    Console.Write(chunk.Text);
}
```

---

## 4. Tools & Function Calling

### Defining Functions with `[Description]`
```csharp
[Description("Fetches weather forecast for a specified city.")]
static string GetWeather(
    [Description("Name of the target city")] string city,
    [Description("Temperature unit: Celsius or Fahrenheit")] string unit = "Celsius")
{
    return $"Forecast for {city}: 22° {unit}";
}

// Convert to AITool
AITool tool = AIFunctionFactory.Create(GetWeather, name: "get_weather");
```

### Gating Sensitive Tools with Human-In-The-Loop (HITL)
```csharp
#pragma warning disable MAAI001 // ApprovalRequiredAIFunction is experimental

AITool sensitiveTool = new ApprovalRequiredAIFunction(
    AIFunctionFactory.Create(ExecuteWireTransfer, name: "wire_transfer"));
```

### Agents Exposing Themselves as Tools
```csharp
AIAgent specialist = chatClient.AsAIAgent(instructions: "SQL Generator", name: "SqlAgent");
AITool agentTool = specialist.AsAIFunction(); // Registerable in parent agent
```

---

## 5. Conversations, Sessions, & StateBag

### Multi-turn Session Management
```csharp
// Create a new provider session
AgentSession session = await agent.CreateSessionAsync();

// Turn 1
await agent.RunAsync("My name is Alice.", session);

// Turn 2 (agent remembers context automatically)
AgentResponse reply = await agent.RunAsync("What is my name?", session);
```

### Storing Domain State in `StateBag`
```csharp
// Set typed reference values
session.StateBag.SetValue<string>("UserTier", "VIP");
session.StateBag.SetValue<CustomerContext>("Context", new CustomerContext { AccountId = 42 });

// Retrieve typed values
string? tier = session.StateBag.GetValue<string>("UserTier");
CustomerContext? ctx = session.StateBag.GetValue<CustomerContext>("Context");
```

### Session Serialization
```csharp
// Serialize session to JSON element
JsonElement json = await agent.SerializeSessionAsync(session);

// Restore session state
AgentSession restoredSession = await agent.DeserializeSessionAsync(json);
```

---

## 6. Middleware Pipelines

### Agent Run Middleware
```csharp
AIAgent pipedAgent = agent.AsBuilder()
    .Use(
        runFunc: async (messages, session, options, innerAgent, ct) =>
        {
            Console.WriteLine($"[Middleware] Intercepting {messages.Count()} messages");
            var response = await innerAgent.RunAsync(messages, session, options, ct);
            Console.WriteLine($"[Middleware] Finished with {response.Text.Length} chars");
            return response;
        },
        runStreamingFunc: null)
    .Build();
```

---

## 7. Multi-Agent Workflows (`AgentWorkflowBuilder`)

### Sequential Pipeline
```csharp
Workflow sequential = AgentWorkflowBuilder.BuildSequential(
    workflowName: "ResearchAndPublish",
    researchAgent,
    writerAgent,
    editorAgent);
```

### Concurrent Fan-out / Fan-in
```csharp
Workflow concurrent = AgentWorkflowBuilder.BuildConcurrent(
    agents: [securityReviewer, performanceReviewer, accessibilityReviewer]);
```

### Agent Handoffs (Swarm Routing)
```csharp
Workflow handoff = AgentWorkflowBuilder.CreateHandoffBuilderWith(triageAgent)
    .WithHandoff(triageAgent, billingAgent)
    .WithHandoff(triageAgent, techSupportAgent)
    .WithHandoff(billingAgent, triageAgent)
    .WithHandoff(techSupportAgent, triageAgent)
    .Build();
```

### Group Chat (Round-Robin Orchestration)
```csharp
Workflow groupChat = AgentWorkflowBuilder.CreateGroupChatBuilderWith(agents =>
        new RoundRobinGroupChatManager(agents) { MaximumIterationCount = 5 })
    .AddParticipants([agentA, agentB, agentC])
    .Build();
```

---

## 8. Custom Workflows (`WorkflowBuilder`)

```csharp
public sealed class DataFetchExecutor() : Executor<string, string>("data_fetcher")
{
    public override ValueTask<string> HandleAsync(string input, IWorkflowContext context, CancellationToken ct) =>
        ValueTask.FromResult($"Fetched: {input}");
}

public sealed class ProcessExecutor() : Executor<string, string>("processor")
{
    public override async ValueTask<string> HandleAsync(string input, IWorkflowContext context, CancellationToken ct)
    {
        string output = input.ToUpperInvariant();
        await context.YieldOutputAsync(output, ct);
        return output;
    }
}

// Assemble workflow
var fetcher = new DataFetchExecutor();
var processor = new ProcessExecutor();

Workflow workflow = new WorkflowBuilder(fetcher)
    .AddEdge(fetcher, processor)
    .WithOutputFrom(processor)
    .Build();

// Run In-Process
await using Run run = await InProcessExecution.RunAsync(workflow, "raw_data");
```

---

## 9. Automated Evaluation

```csharp
// 1. Built-in local checks
LocalEvaluator evaluator = new(
    EvalChecks.ContainsExpected(),
    EvalChecks.NonEmpty(minLength: 5),
    EvalChecks.KeywordCheck("approved", "verified"),
    EvalChecks.ToolCallsPresent());

// 2. Custom lambda checks
EvalCheck noRefusal = FunctionEvaluator.Create("no_refusal", (string resp) =>
    !resp.Contains("I cannot help", StringComparison.OrdinalIgnoreCase));

// 3. Run evaluation
AgentEvaluationResults results = await agent.EvaluateAsync(
    queries: ["What is the account balance?"],
    evaluator: evaluator,
    expectedOutput: ["$150.00"]);

Console.WriteLine($"Passed: {results.Passed}/{results.Total}");
```

---

## 10. ASP.NET Core Hosting & DevUI

```csharp
var builder = WebApplication.CreateBuilder(args);

// Register Chat Client & Agents
builder.Services.AddChatClient(chatClient);
builder.AddAIAgent("support", "Support instructions.")
    .WithAITool(AIFunctionFactory.Create(GetWeather));

// Register Protocol Endpoints
builder.Services.AddOpenAIResponses();
builder.Services.AddOpenAIConversations();

// Add DevUI in Development
if (builder.Environment.IsDevelopment())
{
    builder.AddDevUI();
}

var app = builder.Build();

app.MapOpenAIResponses();
app.MapOpenAIConversations();

if (app.Environment.IsDevelopment())
{
    app.MapDevUI(); // Open /devui in browser
}

app.Run();
```

---

## 11. OpenTelemetry Distributed Tracing

```csharp
const string SourceName = "Enterprise.Agents";

using var tracerProvider = Sdk.CreateTracerProviderBuilder()
    .AddSource(SourceName)
    .AddConsoleExporter()
    .Build();

AIAgent tracedAgent = agent.AsBuilder()
    .UseOpenTelemetry(sourceName: SourceName, configure: cfg => cfg.EnableSensitiveData = false)
    .Build();
```
