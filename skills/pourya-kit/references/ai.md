# AI

`PouryaKit.AI`: the Host's one chat client, `IChatClient` of Microsoft.Extensions.AI, built from validated options for OpenAI, Azure OpenAI or Ollama through the OpenAI client alone; the Gateway tool, which offers a Module's command or query to a model and runs each call as an Operation through the Module's Gateway; and agents of the Microsoft Agent Framework over that client, in plain, streaming and structured runs and in sequential, concurrent and handoff workflows, with sessions and memory over the application's stores, a chat reducer and a guardrail hook; the Host's clients of the MCP servers its options name, over stdio or HTTP, whose tools a run is given beside its Gateway tools; and the Host's embedding generator over the same OpenAI client. It references `PouryaKit.Cqrs` and `PouryaKit.Modularity`. Namespace `PouryaKit.AI`. The optional `PouryaKit.AI.Postgres` (namespace `PouryaKit.AI.Postgres`) stores text by its embedding in PostgreSQL with pgvector and searches it ([Embeddings and semantic search](#embeddings-and-semantic-search)).

The worked example is the Proving Sample's agent: Catalog's `AskAboutProducts` command, whose handler, `AskAboutProductsCommandHandler`, runs an agent over the Host's client with one Gateway tool, in the asking Actor's conversation, its session and its memory kept in Catalog's schema, and the Host's guardrail, `CardNumberGuardrail`. Every snippet below is that code, or a test's where the sample has none.

- [Registering](#registering)
- [A Gateway tool](#a-gateway-tool)
- [MCP tools](#mcp-tools)
- [Agent runs](#agent-runs)
- [Sessions](#sessions)
- [A chat reducer](#a-chat-reducer)
- [Memory](#memory)
- [Workflows](#workflows)
- [Guardrails](#guardrails)
- [Scheduled agent work](#scheduled-agent-work)
- [Embeddings and semantic search](#embeddings-and-semantic-search)
- [What the Kit guarantees](#what-the-kit-guarantees)
- [Limits](#limits)
- [Testing](#testing)

## Registering

On the Host, as `ModularMonolithHost` composes it:

```csharp
using PouryaKit.AI;

builder.Services.AddPouryaKitAI();
builder.Services.Configure<PouryaKitAIOptions>(builder.Configuration.GetSection("PouryaKit:AI"));
// The application's guardrail checks every call to the model, before and after it: no card number goes either way.
builder.Services.AddSingleton<IChatGuardrail, CardNumberGuardrail>();
```

The sample's `appsettings.json`:

```json
"PouryaKit": {
  "AI": {
    "Provider": "Ollama",
    "Endpoint": "http://localhost:11434/v1",
    "Model": "qwen2.5:0.5b",
    "Tools": ["GetProduct"]
  }
}
```

- `AddPouryaKitAI()` registers the Host's one `IChatClient`, a singleton. `AddPouryaKitAI(configure)` and `AddPouryaKitEmbeddings(configure)` set the options in code instead; with both registrations, each call's `configure` applies, in the order of the calls. A Host that registers another unkeyed `IChatClient`, before or after the call, a second call included, fails when its container is built, naming each registration, and so does one that removes it after the call, naming the contract: the registration adds a composition check, since a second client would serve runs without the allow-list and the cap.
- Locally, an Aspire AppHost runs the model beside the Host and sets `PouryaKit__AI__Endpoint` and `PouryaKit__AI__Model` on it ([aspire.md](aspire.md#a-local-agent)).
- `PouryaKitAIOptions.Provider` (`AIProvider`): `OpenAI` (the default; `Endpoint` may stay null for OpenAI's own API), `AzureOpenAI` (`Endpoint` is the resource's v1 endpoint, such as `https://my-resource.openai.azure.com/openai/v1/`, and `Model` the deployment's name) or `Ollama` (`Endpoint` is its OpenAI-compatible API, `http://<host>:11434/v1`, and no key). Each goes through the OpenAI client's chat completions API: no OllamaSharp and no `Azure.AI.OpenAI`.
- `ApiKey` is required for OpenAI and Azure OpenAI. Set it from user secrets or the environment, as `PouryaKit__AI__ApiKey`, never in a committed file; the Kit never logs it.
- `MaximumIterationsPerRun`, 10 by default, caps how many times a run calls the model and invokes the tools it asks for; `Tools` is the allow-list, the names of the tools a run may be given, empty by default. A declaration that is no function, which the client never invokes and hands back to its caller, such as a handoff workflow's `handoff_to_1`, passes unnamed.
- The Host fails when it starts, naming the option, when the provider is not an `AIProvider` value (a number such as `99` binds), `Model` is blank, `Endpoint` is not an absolute HTTP or HTTPS address or is missing for Azure OpenAI or Ollama, `ApiKey` is missing for OpenAI or Azure OpenAI, the cap is below 1, or a tool name is blank.
- The client sends through the HTTP client named `AIServiceCollectionExtensions.HttpClientName` (`PouryaKit.AI`). The package adds no resilience handler and turns the OpenAI client's own retries off, so the client keeps whatever the Host's defaults give every client: with the service defaults ([host-recipes.md](host-recipes.md)), their one pipeline, which retries safe methods only, so a chat completion, a POST, is sent once. That pipeline's attempt timeout is 10 seconds; give the client its own limits by replacing it, as for any client: `builder.Services.AddHttpClient(AIServiceCollectionExtensions.HttpClientName).RemoveAllResilienceHandlers()`, then `AddPouryaKitResilience().Configure(...)` ([web.md](web.md)). Adding a second pipeline instead is refused when the client is created.
- The client's pipeline, outermost first: the allow-list, function invocation with the cap, the application's guardrails when it registers any ([Guardrails](#guardrails)), OpenTelemetry with sensitive data off, and logging. The telemetry step writes a span and duration and token metrics for each call to the model, on the activity source and meter Experimental.Microsoft.Extensions.AI, which the service defaults' tracing and metrics export ([host-recipes.md](host-recipes.md#what-each-recipe-does)); a Host with an exporter of its own adds that name. Function invocation writes an `orchestrate_tools` span on the same source around a run's calls, so each `chat` span sits inside it, and it inside the Operation whose handler ran the agent. The telemetry carries no message content; the logging step records the messages and options when the Host's logger enables Trace, as Microsoft.Extensions.AI documents, which says never to enable Trace in production.

## A Gateway tool

A run is one `GetResponseAsync` or `GetStreamingResponseAsync` call of the Host's `IChatClient`, given its tools in `ChatOptions.Tools`, as each agent's run makes. `Tool` on a Module's `IGatewayProcessor<TModule>` makes a command or query a Gateway tool for one run's Operation Context. The model's arguments are read with the Kit's JSON into an arguments record, whose schema the model is given, and mapped explicitly to the operation. `AskAboutProductsCommandHandler` gives its agent `GetProduct`, with the `IGatewayProcessor<CatalogModule>` and the `OperationContext` it injects:

```csharp
var getProduct = catalog.Tool(
    "GetProduct",
    "Finds one product of the catalog by its id: its id, its category's id and its name.",
    context,
    (ProductToFind product) => new GetProductQuery(product.ProductId)
);

/// <summary>The product the model asks the GetProduct tool for.</summary>
internal sealed record ProductToFind(Guid ProductId);
```

- Three overloads: a `Query<TResponse>`, a `Command` and a `Command<TResponse>`. Name the tool as `Tools` names it, and describe it for the model; a `[Description]` on the arguments record's properties reaches the schema.
- Each call runs through the Gateway in a scope of its own, as an Operation with the run's context continued (`Continue()`): its own Operation id, the run's as its causation, the run's Actor, tenant and correlation, and no request id. Validation, the authorizers, the tenant requirement of a tenant-scoped Module and, for a command, its own Unit of Work apply, which commits that call alone.
- **The run's authority.** A run that executes inside an authorized Operation, as in a command's or query's handler that injects its `OperationContext`, gives each call a trusted continuation: like any work that Operation causes, a declared permission is not checked again for it, so a tool over Stub's `ReadStampLedgerQuery`, which declares a permission Ada does not hold, answers inside her authorized Operation. So give such a run only tools its Operation may run. A run outside one, with a context the code builds, as in a background service, an endpoint that runs the model itself or a test, gives untrusted continuations: each call's permissions are checked for the Actor, and a denied one is a failed tool result, never a bypass.
- A success returns to the model as the response in the Kit's JSON, a `Command` with no value as a completed call; a failed `Result` returns its Errors, `{"errors":[{"code":…,"message":…,"target":…}]}`, never thrown, so the model can recover. An exception, a bug or arguments that do not read as the record, is function invocation's to handle: the model is told the call failed, and when the calls of three iterations in a row have failed, the next failure is thrown to the run's caller.
- In a tenant-scoped Module, the Dapper tenant rule reads every string literal that names a tenant-scoped table beside a SQL word as SQL ([multi-tenancy.md](multi-tenancy.md#reads)), a model's instructions included: `"...the products ... with the GetProduct tool"` names `catalog.products` beside `with`, so word a prompt around the table names, as the sample's instructions and memory do.
- A tool call carries no idempotency of its own: two calls of one command type in a run each run, even when the run's context has a request id.

## MCP tools

`McpServers` in the same section names the MCP servers whose tools a run may be given, each by a name of the Host's: one with a `Command`, its `Arguments`, `WorkingDirectory` and `Environment`, is a program the Host starts and speaks to over its standard input and output (stdio), one with an `Endpoint` is reached over HTTP. The sample's agent uses none; `McpFixture` configures a stdio server, `dotnet` with the test assembly `ModularMonolith.Tests.McpServer` as its argument, and an HTTP one, and names their tools in `Tools`. A handler lists a server's tools with `IMcpServers.ListToolsAsync(server)` and gives them to its agent beside its Gateway tools, as `McpTests.A_gateway_tool_and_an_mcp_tool_run_in_one_agent` does.

- `IMcpServers.ListToolsAsync(server)` gives the tools of the server the configuration names, as `McpClientTool`s of the official MCP C# SDK (`ModelContextProtocol.Core`, which `PouryaKit.AI` brings): functions a run is given in `ChatOptions.Tools`, or an agent in its tools, beside its Gateway tools. A name the configuration does not have throws, naming the servers it has.
- **One allow-list and one cap.** An MCP tool is a function like a Gateway tool, so the allow-list and the iteration cap of the Host's client hold for it: `Tools` names each MCP tool a run may be given as its server names it, and one the list does not name is withheld from the model and its call answered as a function that does not exist, never reaching the server; each iteration that calls MCP tools counts toward `MaximumIterationsPerRun`. A tool of a server and a Gateway tool with one name are one name to the list: rename one, as with the SDK's `WithName`.
- **An MCP tool call is not an Operation.** It runs on its server, with the credentials the server was configured with, its `Environment` or the authentication the Host adds to `PouryaKit.AI.Mcp`: no Actor, tenant, authorizer or Unit of Work of the Kit's applies to it, so the allow-list alone decides which of a server's tools a run may call, and what a call changes is the server's, never part of the run's Operation.
- **The Host owns each client.** `IMcpServers` is a singleton of the Host's services: a server is connected at its first listing, which concurrent listings share, and stays connected for every run after it; a listing that cancels stops waiting and leaves the connection to the others, and a connection that failed is tried again by the next listing. A listing the server does not answer, its process gone or its HTTP server out of reach, throws and drops that client, so the next listing connects again; an error the server answers with keeps it. No run owns a client: disposing the Host's services ends them all, a connection still under way included. The SDK kills a stdio server's process once its shutdown timeout, 5 seconds by default, has passed, without closing the server's input first, and ends an HTTP server's session, where the server keeps one, with its DELETE.
- **A stdio server gets no secret of the Host's.** Its process inherits only the variables a process needs to start, such as `PATH` and `HOME`, and those `Environment` names, set from a secret store like `ApiKey`; `WorkingDirectory` is where it starts, by default the Host's.
- An HTTP server is reached through the HTTP client named `AIServiceCollectionExtensions.McpHttpClientName` (`PouryaKit.AI.Mcp`), apart from the model's, so the Host's defaults apply to it as to every client and the Host adds a server's authentication there, by that name, without sending it to the model's provider. The client opens no standing stream for messages the server sends unasked, so a resilience pipeline's attempt timeout never cuts one; it does apply to each call, as to the model's.
- The Host fails when it starts, naming the option, when a server has neither or both of `Command` and `Endpoint`, or an `Endpoint` that is not an absolute HTTP or HTTPS address.

## Agent runs

An agent is the Agent Framework's `AIAgent` (package `Microsoft.Agents.AI`, which `PouryaKit.AI` brings) that `AsAIAgent` makes over the Host's `IChatClient`. Build it in the handler, for each run, since its Gateway tools are bound to the run's Operation Context. `AskAboutProductsCommandHandler` builds its agent with its instructions, its tool, its history and its memory:

```csharp
using Microsoft.Agents.AI;

var agent = chat.AsAIAgent(
    new ChatClientAgentOptions
    {
        ChatOptions = new() { Instructions = Instructions, Tools = [getProduct] },
        ChatHistoryProvider = new InMemoryChatHistoryProvider(new() { ChatReducer = new MessageCountingChatReducer(HistoryLength) }),
        AIContextProviders = [new ProductsAskedAbout(store, key)],
    }
);
```

- A plain run, `RunAsync`, returns the agent's response, whose text is its answer and whose `FinishReason` says how it ended; a streaming run, `RunStreamingAsync`, yields the answer in parts; a structured run, `RunAsync<T>`, gives the model the JSON schema of the type, as the request's `response_format`, and reads the answer into the typed response's result, which throws when the answer does not read as the type. `AgentRunTests` runs each shape (`A_plain_run_completes_and_each_tool_call_is_an_operation_that_continues_the_run`, `A_streaming_run_streams_its_answer_and_stops_calling_tools_at_the_iteration_cap`, `A_structured_run_asks_for_its_schema_and_returns_the_typed_answer`).
- The agent runs through the Host's client, whose function invocation it finds and keeps, adding none, so the allow-list, the cap, the guardrails and the telemetry apply to every run of every agent.

## Sessions

A conversation over several runs is a session, which the agent restores from the application's store and saves back to it. `AskAboutProductsCommandHandler` keeps one per Actor in each tenant:

```csharp
var store = new ProductConversations(conversations, context);
var key = ProductConversations.KeyOf(context);

var session = await agent.LoadSessionAsync(store, key, cancellationToken);
var answer = await agent.RunAsync(command.Question, session, cancellationToken: cancellationToken);
// A refused turn fails, so the Unit of Work keeps nothing of it, what the memory noted included: a refused question
// kept in the session would be sent again, and refused again, with every later one.
if (answer.FinishReason == ChatFinishReason.ContentFilter)
{
    return ProductErrors.QuestionRefused(answer.Text);
}

await agent.SaveSessionAsync(store, key, session, cancellationToken);
return answer.Text;
```

`ProductConversations`, Catalog's store, over its `ProductConversation` aggregate (`catalog.product_conversations`, a Guid key and a unique index on the conversation's key), through the repository the handler injects and the specification `ProductConversationKeyed`:

```csharp
internal sealed class ProductConversations(IRepository<ProductConversation, ProductConversationId> conversations, OperationContext context)
    : IAgentSessionStore
{
    private readonly Dictionary<string, ProductConversation?> _read = [];

    // The Actor's conversation in the Operation's tenant, each part escaped, so no two Actors or tenants share one.
    public static string KeyOf(OperationContext context) =>
        $"{Uri.EscapeDataString(context.Tenant!)}/{context.Actor.Kind}/{Uri.EscapeDataString(context.Actor.Id)}";

    public async Task<JsonElement?> FindAsync(string key, CancellationToken cancellationToken) =>
        (await ConversationAsync(key, cancellationToken))?.Session is { } session ? JsonElement.Parse(session) : null;

    public async Task SaveAsync(string key, JsonElement session, CancellationToken cancellationToken) =>
        (await OfAsync(key, cancellationToken)).Keep(session.GetRawText());

    // RecallAsync and RememberAsync keep the memory the same way. ConversationAsync reads a key once per turn, into _read;
    // OfAsync returns what it read, or starts the conversation with the Operation's tenant and adds it.
}
```

- `IAgentSessionStore` is the application's port, with `FindAsync` and `SaveAsync` of a session's state, JSON, under a key such as a conversation's id. Implement it over the application's own storage, keyed so that one Actor never reads another's session; the Kit ships no store and keeps no copy, so a session that is not saved ends with its scope.
- `LoadSessionAsync` restores the saved session, the history with it, or starts a new one when the store holds none; `SaveSessionAsync` serializes the session as the agent does, the history its chat history provider kept included.
- **Written by the Unit of Work.** A store over a repository changes an aggregate and saves nothing (ADR 0003): the command's Unit of Work writes the row when the turn succeeds and keeps none of it when the turn fails. So the agent that keeps a session runs in a command, as `AskAboutProducts` is, never a query, where the repository refuses to add. The key names the tenant and the Actor, and the aggregate is `ITenantScoped`, so another Actor or another tenant starts a conversation of its own.
- Keep the session's JSON as text or `json`, not `jsonb`: PostgreSQL's `jsonb` reorders an object's keys, and the Agent Framework reads a session's content back only with its `"$type"` first (`ProductConversationConfiguration`).
- **Two turns at once.** A turn reads its conversation once, as it starts, and writes what it read, so of two turns of one conversation at once the second to commit fails with `UnitOfWorkErrors.ConcurrencyConflict`, from the aggregate's version, or, for two first turns, from the unique index on its key, which `CatalogModule` maps with `MapPouryaKitUniqueConstraint`, and keeps nothing. A store that read the conversation again before its write would find the other turn's committed row and overwrite it (`AgentRunTests.Of_two_first_turns_of_one_conversation_at_once_the_second_to_commit_fails_with_a_conflict_and_keeps_nothing`).

## A chat reducer

An agent's history is its `InMemoryChatHistoryProvider`, kept in the session; a reducer bounds what each run sends the model and what the session saves. `AskAboutProductsCommandHandler` keeps the last 20 messages:

```csharp
// The last messages of the conversation each turn sends the model, its tool calls and their results left out.
private const int HistoryLength = 20;

#pragma warning disable MEAI001 // Microsoft.Extensions.AI 10.11's reducers are experimental.
ChatHistoryProvider = new InMemoryChatHistoryProvider(new() { ChatReducer = new MessageCountingChatReducer(HistoryLength) }),
#pragma warning restore MEAI001
```

- `MessageCountingChatReducer(targetCount)` keeps the first system message and the last messages up to the count, leaving out tool calls and their results; `SummarizingChatReducer(chat, targetCount, threshold)` asks the model for a summary of the older messages and sends it in their place (`AgentRunTests.A_summarizing_reducer_replaces_the_older_history_with_the_models_summary`). Give it the Host's `IChatClient`, never a second client.
- The reducer runs before each run reads the history and keeps it reduced: the run sends it, and the session saves it with the run's new messages.

## Memory

Memory is an `AIContextProvider` of the Agent Framework over the application's store: before each run it gives the agent instructions, messages or tools, and after the run it receives the run's messages to keep. The Kit adds no type of its own. The sample's, `ProductsAskedAbout`, keeps what `GetProduct` found in the same conversation, which outlives the 20 messages the history sends:

```csharp
internal sealed class ProductsAskedAbout(ProductConversations conversations, string key) : AIContextProvider
{
    private const int Kept = 20;

    protected override async ValueTask<AIContext> ProvideAIContextAsync(InvokingContext context, CancellationToken cancellationToken = default) =>
        await conversations.RecallAsync(key, cancellationToken) is [_, ..] remembered
            ? new AIContext { Instructions = $"Earlier the customer asked about: {string.Join("; ", remembered)}." }
            : new AIContext();

    protected override async ValueTask StoreAIContextAsync(InvokedContext context, CancellationToken cancellationToken = default)
    {
        // What each GetProduct call answered: the product, as the Kit's JSON writes it, or a failed call's errors.
        string[] found =
        [
            .. context
                .ResponseMessages!.SelectMany(message => message.Contents.OfType<FunctionResultContent>())
                .Select(result => result.Result)
                .OfType<JsonElement>()
                .Where(product => product.ValueKind == JsonValueKind.Object && product.TryGetProperty("name", out _))
                .Select(product => $"{product.GetProperty("name").GetString()} ({product.GetProperty("id").GetGuid()})")
                .Reverse(),
        ];
        if (found is [])
        {
            return;
        }

        await conversations.RememberAsync(key, [.. found.Concat(await conversations.RecallAsync(key, cancellationToken)).Distinct().Take(Kept)], cancellationToken);
    }
}
```

- Key what it keeps by the run's Actor and tenant, from the Operation Context, so no Actor reads another's memory: the sample's provider writes through the same store and key as the session, so the command's Unit of Work writes both or neither.
- The Agent Framework calls `StoreAIContextAsync` only after a run that did not throw. A run that ends with a refusal did not throw: what the provider noted is undone only when the handler fails the turn, as the sample's does.

## Workflows

The Agent Framework's workflows (package `Microsoft.Agents.AI.Workflows`, which `PouryaKit.AI` brings) compose agents, each built over the Host's client with its tools bound to the run's context. `AgentRunTests` composes each shape inside an authorized Operation, its agents given `GetProduct` bound to that Operation's context:

```csharp
using Microsoft.Agents.AI.Workflows;

var sequential = AgentWorkflowBuilder.BuildSequential(Agent(context, "finder"), Agent(context, "checker"));
var concurrent = AgentWorkflowBuilder.BuildConcurrent(
    [Agent(context, "pricer", "You price products."), Agent(context, "describer", "You describe products.")]
);
var handoff = AgentWorkflowBuilder
    .CreateHandoffBuilderWith(triage)
    .WithHandoff(triage, Agent(context, "expert"), "Answers questions about one product.")
    .Build();

var answer = (await sequential.AsAIAgent().RunAsync($"Find {product}.", cancellationToken: cancellationToken)).Text;
```

- Sequential: each agent is given the conversation so far, with what the agent before it answered. Concurrent: each agent is given the question at once, and their answers are gathered. Handoff: an agent hands the conversation to another by calling a `handoff_to_` declaration, which the allow-list passes unnamed since it runs nothing.
- A workflow runs its agents off the calling thread, and the run's authority goes with them: inside an authorized Operation, each Gateway tool call of each agent is a trusted continuation of the run, as in a single agent's run.

## Guardrails

A guardrail is the application's `IChatGuardrail`, registered as a singleton on the Host, which the client resolves when it is built; a Host that registers one with another lifetime fails when its container is built, naming it. The Kit has no policy and registers none. The sample's Host registers `CardNumberGuardrail` ([Registering](#registering)), which the templates' modular monolith takes too:

```csharp
public sealed partial class CardNumberGuardrail : IChatGuardrail
{
    public const string Refusal = "Card numbers are not taken here: ask again without one.";

    public Task<string?> CheckRequestAsync(IReadOnlyList<ChatMessage> messages, CancellationToken cancellationToken) =>
        Task.FromResult(HoldsCardNumber(messages) ? Refusal : null);

    public Task<string?> CheckResponseAsync(ChatResponse response, CancellationToken cancellationToken) =>
        Task.FromResult(HoldsCardNumber(response.Messages) ? Refusal : null);

    private static bool HoldsCardNumber(IEnumerable<ChatMessage> messages) =>
        messages.SelectMany(message => message.Contents).Any(content => CardNumber().IsMatch(TextOf(content)));

    // A message's Text is its TextContent alone: a tool call's arguments and a tool's answer are read as their JSON.
    private static string TextOf(AIContent content) =>
        content switch
        {
            TextContent text => text.Text,
            FunctionCallContent call => JsonSerializer.Serialize(call.Arguments),
            FunctionResultContent result => JsonSerializer.Serialize(result.Result),
            _ => "",
        };

    [GeneratedRegex(@"(?<![\w-])\d{4}(?:[ -]?\d{4}){3}(?![\w-])")]
    private static partial Regex CardNumber();
}
```

- Each guardrail is called before every call to the model, with the messages it sends, the call's instructions, an agent's and its context providers', first as the system message the model is given, and after it, with the response, before any tool call the response asks for runs: every run of the Host's client, every agent of a workflow and a summarizing reducer's summary included. Null lets the call through. A refusal ends that call as the provider's content filter would, with the refusal as its text and `ChatFinishReason.ContentFilter`: refused before, the model is not called; refused after, the response is replaced and no tool it asked for runs. The first refusal, in registration order, wins.
- **Read every content, not only the text.** `ChatMessage.Text` and `ChatResponse.Text` join a message's `TextContent` alone, so a policy over what reaches the model reads a tool call's arguments (`FunctionCallContent`) and what a tool answered (`FunctionResultContent`) too, as the sample's does: a product named with a card number is refused before its look-up reaches the model (`AgentRunTests.A_card_number_in_a_tool_result_never_reaches_the_model_and_a_later_turn_of_the_conversation_is_answered`). What a provider context gives the agent, such as memory, is checked as the instructions it becomes, so a policy that let a value into memory would refuse every later turn.
- A handler decides what a refused run means to its caller. The sample's agent fails the turn with `Product.QuestionRefused`, the refusal its message, so its conversation keeps nothing of it ([Sessions](#sessions)).
- With a guardrail registered, a streamed call is checked whole: the client takes the model's complete response, checks it, then gives it to the caller as updates.

## Scheduled agent work

Agent work on a cadence is a recurring command whose handler runs the agent ([messaging.md](messaging.md#recurring-commands)): each occurrence is an Operation of its own for the named system Actor, and the agent's Gateway tool calls continue it. The sample has none: Stub's `RunScheduledAgentCommandHandler` runs one, declared with `AddPouryaKitRecurringCommand`, and `AgentRunTests.A_scheduled_run_executes_as_its_named_system_actor_and_its_tool_calls_carry_that_actor` proves each tool call carries the occurrence's system Actor as a trusted continuation.

- Each tool call carries the system Actor, as a trusted continuation of the occurrence, and no tenant: a recurring command's Module is not tenant-scoped, and a tenant-scoped Module's tool refuses a call with no tenant.

## Embeddings and semantic search

The sample stores no embedding; the worked example is the test Module `VectorsModule`, which `VectorFixture` composes alone on a Host, with the embedding generator:

```csharp
using PouryaKit.AI;

builder.AddPouryaKitModules(modules => modules.Add<VectorsModule>());
builder.Services.AddPouryaKitEmbeddings();
builder.Services.Configure<PouryaKitAIOptions>(builder.Configuration.GetSection("PouryaKit:AI"));
```

The Module adds the vector store, and its migration creates its collection:

```csharp
using PouryaKit.AI.Postgres;

module.UsePouryaKitPostgres<VectorsDbContext>(connectionName: "app", schema: "vectors");
module.AddPouryaKitCqrs();
module.AddPouryaKitTenancy();
module.AddPouryaKitVectorStore();

// In the Module's migration, Passages; 8 is the dimensions the tests' model returns.
migrationBuilder.CreateVectorCollection(VectorsModule.Passages, VectorsModule.Dimensions);
```

A command's handler stores, and a query searches, as an agent's Gateway tool may:

```csharp
internal sealed class StorePassageCommandHandler(IVectorStore vectors) : CommandHandler<StorePassageCommand>
{
    public override async Task<Result> Handle(StorePassageCommand command, CancellationToken cancellationToken)
    {
        await vectors.StoreAsync(VectorsModule.Passages, command.Key, command.Text, cancellationToken);
        return command.Fail ? Refused : Result.Success();
    }
}

internal sealed class SearchPassagesQueryHandler(IVectorStore vectors) : QueryHandler<SearchPassagesQuery, IReadOnlyList<VectorMatch>>
{
    public override async Task<Result<IReadOnlyList<VectorMatch>>> Handle(SearchPassagesQuery query, CancellationToken cancellationToken) =>
        Result.Success(await vectors.SearchAsync(VectorsModule.Passages, query.Text, query.Top, cancellationToken));
}
```

- `AddPouryaKitEmbeddings(configure)` registers the Host's `IEmbeddingGenerator<string, Embedding<float>>` (`configure` sets the options in code as `AddPouryaKitAI(configure)` does), a singleton over the same OpenAI client, HTTP client and provider rules as the chat client, wrapped in OpenTelemetry and logging as the chat client is: the telemetry carries no input, and the logging step records the input and options when the Host's logger enables Trace. The Host fails when it starts, naming the option, on the provider rules of [Registering](#registering) and on a blank `EmbeddingModel`; for Azure OpenAI it is the embedding deployment's name. With both registrations the provider is checked once. A Host that registers another unkeyed `IEmbeddingGenerator<string, Embedding<float>>`, before or after the call, fails when its container is built, naming each registration: the registration adds a composition check, as the chat client's does. Each provider serves the model through its embeddings API; an Ollama chat model may not: `qwen2.5:0.5b` on `ollama/ollama:0.40.6` answers 501, so name an embedding model such as `all-minilm:22m`.
- `AddPouryaKitVectorStore()` on a Module stored with `UsePouryaKitPostgres` adds the scoped `IVectorStore`. `CreateVectorCollection(name, dimensions)` creates the pgvector extension where the database has none, the package's schema `ai` and the table `ai.<name>`: tenant, key, the stored text and its `vector(<dimensions>)`. A name is a lower-case ASCII letter, then up to 59 lower-case letters, digits and underscores; any other is refused. A database that has no vector extension and cannot create it fails the migration, naming the fix: install pgvector on the server, or have a role that may create extensions run `create extension vector`. An installed extension is used where it is, and its schema must be on the Host role's `search_path`, as `public`, the default, is: the column's `vector` type and the search's `::vector` cast are unqualified, so with the extension in another schema the migration, or a search, fails with PostgreSQL's `type "vector" does not exist` instead.
- `StoreAsync` embeds the text and writes it under the key, replacing what the key held, on the Unit of Work's transaction: a rolled-back command stores nothing. Like any write it runs only in a command's handler or a Domain Event Handler, and it calls the model inside the transaction, so a slow model holds the Operation's locks. `RemoveAsync` deletes a key the same way.
- `SearchAsync` embeds the question and returns at most `top` `VectorMatch` (`Key`, `Content`, `Distance`), nearest first by cosine distance, from 0 to 2. In a command it reads on the Unit of Work's connection and so sees what the command stored.
- Every row carries the Operation's tenant, `OperationContext.NoTenant` without one, and every call reads and writes only its own Operation's tenant: a caller never passes a tenant.
- The search is exact, over the tenant's rows, with no approximate index: a Host whose collection grows adds an HNSW index with `vector_cosine_ops` in a migration of its own, and pgvector's iterative scan where a tenant's rows are few among many.
- A Host without `PouryaKit.AI.Postgres` takes no pgvector: `PackageGraphTests` keeps the `Pgvector` client and the package out of every other Kit package's closure, and the package itself sends each embedding as a `real[]`, which pgvector casts to its `vector`, so it needs no client of its own.

## What the Kit guarantees

- **A Gateway tool call is an Operation through the Module's Gateway under the run's continued context**, in every run shape and in every agent of a workflow, trusted only when the run executes inside the authorized Operation the context belongs to. An MCP tool call runs on its server, never as an Operation.
- **A failed Result is a failed tool result**, never an exception to the model.
- **The allow-list and the cap hold for every run** of the Host's client, each agent's included, and for every tool, Gateway or MCP: a function `Tools` does not name is withheld from the model, and a call it asks for anyway is answered as a function that does not exist and never runs; a declaration-only tool, such as a handoff's, passes unnamed, since the client never invokes it and hands its call back to the run's caller; the call after the last allowed iteration offers no tool, so the run ends with the model's answer.
- **A guardrail is called before and after every call to the model**, and a refusal stops the call or the tool calls its response asked for; the Kit refuses nothing itself.
- **The Kit keeps no session and no memory**: they live in the application's stores, through `IAgentSessionStore` and its context providers.
- **The Host owns its MCP clients**: created with its services and disposed with them, which ends a stdio server's process and an HTTP server's session; no run ends one.
- **One resilience pipeline**: the provider's HTTP client is the Host's, with nothing added.
- **The options are checked before the Host serves**, not at the first run, and one provider serves the Host: no switch at run time.
- **One chat client and one embedding generator per Host**: a Host that registers another unkeyed `IChatClient` or `IEmbeddingGenerator<string, Embedding<float>>`, before or after the Kit's registration, fails when its container is built, naming each registration, where the container would otherwise serve the last one silently.
- **A stored embedding commits with its Operation and is found within its tenant**: `IVectorStore` writes on the Unit of Work's transaction in the package's own schema, and every read and write filters by the Operation's tenant.

## Limits

- Chat completions and embeddings only.
- Of an MCP server's tools, resources and prompts, the Kit gives a run its tools only. A tool listed from a session that has since ended, as when the server's process crashed, fails its calls as a function that throws does, the model told the call failed: only a new listing reaches the server connected again.
- Each Gateway tool call is its own Operation and commits on its own: a run that fails later, or whose handler then fails, undoes no call that already committed.
- An agent that keeps its session through the Unit of Work runs in a command, which holds its transaction, and its connection, while the model answers, as the sample's agent does: a slow model holds them.
- The reducers are experimental in Microsoft.Extensions.AI 10.11 (`MEAI001`), and a streamed call is not streamed while a guardrail is registered.
- The model decides which tools to call and with what arguments; the Kit decides only what each call may do.

## Testing

- The Proving Sample's `AiTests` and `AgentRunTests` run the application on a database of their own with its chat client pointed at `ChatStub`, an OpenAI-compatible chat completions server in the test process on a loopback port, which answers each request with the replies a test scripts (text or tool calls, streamed when the request asks for a stream, or chosen from each request's body when agents call the model at once) and records every request: the tools offered and what each call returned. That Host, `AiFixture`'s, has none of the application's guardrails, so a streamed run streams; a test that needs one composes a Host of its own. No hosted provider key is used in any test.
- The sample's agent, through its command: `AiTests.A_tool_call_runs_through_the_gateway_as_an_operation_that_continues_the_run_once_per_call` asserts each call's context; in `AgentRunTests`, `The_samples_agent_continues_a_conversation_only_for_the_same_actor_in_the_same_tenant` sends a second turn by the same Actor, another Actor and the same Actor in another tenant, and reads what the model was sent, the history and the memory; `A_turn_of_the_samples_agent_the_applications_guardrail_refuses_fails_and_keeps_no_conversation` counts the conversation's rows after `CardNumberGuardrail` refused an answer; `Each_model_call_of_the_samples_agent_is_checked_before_and_after_and_its_tool_calls_stop_at_the_cap` and `The_allow_list_withholds_the_samples_agents_tool_when_it_does_not_name_it` hold the Host's guardrail, cap and allow-list to its tool calls.
- `AgentRunTests` runs each agent that calls a tool inside an authorized Operation, through a test Module's query that hands the test its context, and asserts each call's context as `AiTests` does; a guardrail registered on a Host of its own, and scheduled work on a Host with a database and a `ChatStub` of its own. `A_gateway_tool_over_a_permission_declared_query_runs_on_the_authority_of_the_run_it_continues` runs a tool over Stub's `ReadStampLedgerQuery` inside and outside an authorized Operation.
- The exported span: `AgentRunTests.The_samples_agents_call_to_the_model_is_exported_as_a_span_under_its_commands_activity` composes the application with the service defaults and an in-process exporter on their tracer provider, `ExportedActivities` (`tests/ModularMonolith.Tests/Fixtures/RecordedTelemetry.cs`), and finds the `chat` span inside `orchestrate_tools`, inside the command's activity. A listener of the source itself would see the span whatever the provider exports.
- The one real probe, `AiTests.An_ollama_model_in_a_container_completes_a_tool_call_with_no_hosted_key`, sends the sample's command to Ollama in a container (`Testcontainers.Ollama`, `ollama/ollama:0.40.6`, the model `qwen2.5:0.5b`) only with `POURYAKIT_OLLAMA_PROBE=1`, and is skipped otherwise. With the same variable, `AppHostTests` starts the sample's AppHost with its local agent, the same image and model through `AddPouryaKitOllama` ([aspire.md](aspire.md#a-local-agent)), and asks the agent over HTTP.
- `McpTests` gives the application a stdio MCP server, the test assembly `ModularMonolith.Tests.McpServer` started with `dotnet`, and an HTTP one in the test process that keeps sessions, and runs their tools beside a Gateway tool against `ChatStub`; disposing a Host of its own shows the stdio process gone and the HTTP server refusing the ended session. On Hosts of their own, with the stdio server recording each start and holding one on request, concurrent first listings share one process, a listing that cancels leaves the connection to the other, a failed connection and a killed process are each connected again by a later listing, and disposal ends a connection still under way without waiting for it. No public MCP server is used in any test.
- `VectorSearchTests` runs a test Module, `VectorsModule`, alone on PostgreSQL with pgvector in a container (`docker.io/pgvector/pgvector:0.8.7-pg18`), its embeddings `ChatStub`'s deterministic ones, and proves the nearest neighbours, the package's schema, the migration's refusal on a server without pgvector, the tenant filter with two tenants, the rollback with a failed command and the search inside a command; every Host `VectorFixture` composes builds with the composition checks and one generator, and `CompositionCheckTests` fails the build of a Host with a second, registered before or after `AddPouryaKitEmbeddings`. Its opt-in probe, with `POURYAKIT_OLLAMA_PROBE=1`, embeds a text through Ollama with `all-minilm:22m`.
- Observe a tool call's context as the sample's tests do: an `IAuthorizer<T>` of the test assembly for the tool's operation records the `OperationContext` it runs with.
