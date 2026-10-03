# XAF Blazor Server Workspace: reusable code patterns

These examples show the host-side pieces: a Razor workspace window, an authorized XAF query
exposed as a model tool, and the A2UI surface/action lifecycle. They are adapted from a working
Syncfusion A2UI + XAF Blazor Server integration. They are not a drop-in project: replace
the example domain type, DTO fields, component catalog id, allowed component schema and service
registration with the values from the target application and its pinned package versions.
The snippets deliberately contain no Kierat-only types or project paths.

## 1. Razor window

Add the provider to the preview area. Keep prompt entry, progress/errors and the rendered surface
in the same Blazor component so they share one circuit and one surface lifetime:

```razor
@using Syncfusion.Blazor.A2UI
@using Syncfusion.A2UI.Core.Common
@implements IDisposable
@inject ILogger<WorkspaceAiWindow> Log
@inject IWorkspaceAiService WorkspaceAi
@inject MessageProcessor processor
@inject WorkspaceRecords records

<section class="workspace-ai" aria-label="AI workspace">
    <header>
        <h2>Build a data view</h2>
        <form @onsubmit="SubmitAsync" @onsubmit:preventDefault>
            <textarea @bind="prompt" @bind:event="oninput"
                      aria-label="Workspace prompt" disabled="@busy"></textarea>
            <button type="submit" disabled="@(busy || string.IsNullOrWhiteSpace(prompt))">
                @(busy ? "Building…" : "Build view")
            </button>
        </form>
    </header>

    <main class="workspace-ai-preview" aria-live="polite">
        @if (busy)
        {
            <p role="status">The assistant is building the view…</p>
        }
        else if (!string.IsNullOrWhiteSpace(error))
        {
            <p role="alert">@error</p>
        }
        else if (surface is not null)
        {
            <SyncfusionA2UIProvider Surface="@surface" />
        }
        else
        {
            <p>Describe the view you want to build.</p>
        }
    </main>
</section>

@code {
    private readonly string surfaceId = $"workspace-{Guid.NewGuid():N}";
    private string prompt = string.Empty;
    private bool busy;
    private string? error;
    private SurfaceModel? surface;
    private CancellationTokenSource? request;

    private async Task SubmitAsync()
    {
        var submittedPrompt = prompt.Trim();
        if (submittedPrompt.Length == 0 || busy) return;

        prompt = string.Empty;
        busy = true;
        error = null;
        var currentRequest = new CancellationTokenSource(TimeSpan.FromSeconds(120));
        request = currentRequest;
        try
        {
            surface = await WorkspaceAi.BuildSurfaceAsync(submittedPrompt, surfaceId, currentRequest.Token);
        }
        catch (OperationCanceledException) when (currentRequest.IsCancellationRequested)
        {
            error = "The request was cancelled or timed out.";
        }
        catch (Exception exception)
        {
            Log.LogError(exception, "Workspace view generation failed.");
            error = "The view could not be built. Try again or contact support.";
        }
        finally
        {
            currentRequest.Dispose();
            if (ReferenceEquals(request, currentRequest)) request = null;
            busy = false;
        }
    }

    public void Dispose()
    {
        request?.Cancel();
    }
}
```

Keep generated component JSON out of the markup. The service should return a validated view model
or a validated A2UI surface; do not render arbitrary model-provided HTML.

The referenced service has the contract `Task<SurfaceModel> BuildSurfaceAsync(string prompt,
string surfaceId, CancellationToken cancellationToken)`. Its implementation connects the model
tool and surface-processing examples below. The component above owns prompt submission and
request cancellation; the service owns model/tool execution and returns only a validated surface.
In Blazor Server, register the `MessageProcessor` as
scoped if the Syncfusion extension registered it as a singleton; every circuit must own its own
surface/action state. In the tested Kierat setup, `CatalogRegistry` is shared but
`MessageProcessor` is scoped:

```csharp
services.AddA2UIWithSyncfusionComponents();
services.AddSingleton(_ =>
{
    var catalog = new CatalogRegistry();
    catalog.Register(BlazorSyncfusionCatalog.Combine(
        SyncfusionComponentFactory.AllSchemas, BasicFunctions.All));
    return catalog;
});
services.RemoveAll<MessageProcessor>();
services.AddScoped(provider => new MessageProcessor(
    provider.GetRequiredService<CatalogRegistry>()));
```

Do not blindly copy these registrations: inspect the target extension’s service lifetimes and
catalog setup first. Kierat pins `Syncfusion.Blazor.A2UI` 35.1.37 and
`Microsoft.Extensions.AI` 10.8.0; use the versions pinned by the target app.

The Razor component code snippets below use these imports and injected services (shown together
here to make the examples usable without the source application):

```razor
@using System.Text.Json
@using System.Text.Json.Nodes
@using Syncfusion.A2UI.Core.Common
@using Syncfusion.A2UI.Core.Schema
@using Syncfusion.A2UI.Core.Serialization
@using Syncfusion.A2UI.Core.Processing
@using Syncfusion.Blazor.A2UI
@inject MessageProcessor processor
@inject WorkspaceRecords records
```

Namespace locations can differ between Syncfusion releases; use the namespaces from the target
package and the registered DI service.

## 2. Fetch rows through the current XAF security context

Inject `IXafApplicationProvider` into a scoped Blazor service or component. Create and dispose an
`IObjectSpace` for each operation, then project only approved fields into a bounded DTO before
passing data to the model or a component:

```csharp
using DevExpress.ExpressApp;
using DevExpress.ExpressApp.Blazor.Services;

public sealed class WorkspaceRecords(IXafApplicationProvider applications)
{
    public IReadOnlyList<RecordDto> ListVisibleRecords(int limit = 100, string? priority = null)
    {
        limit = Math.Clamp(limit, 1, 100);

        using IObjectSpace objectSpace = applications.GetApplication()
            .CreateObjectSpace(typeof(WorkItem));

        // XAF applies the configured security strategy to this ObjectSpace. Tenant isolation is
        // application-specific; verify that the configured strategy enforces it for this type.
        var query = objectSpace.GetObjects<WorkItem>()
            .Where(item => !item.IsArchived);
        if (Enum.TryParse<WorkItemPriority>(priority, ignoreCase: true, out var selectedPriority))
            query = query.Where(item => item.Priority == selectedPriority);

        // Take bounds the DTO count. GetObjects can materialize/load all matching objects first,
        // so this is not a database-side limit. For large tables, use a secured provider-side
        // query supported by the target XAF provider (for example GetObjectsQuery<T> when available)
        // and verify that the query still applies the current user's security criteria.
        return query
            .OrderBy(item => item.DueDate ?? DateTime.MaxValue)
            .Take(limit)
            .Select(item => new RecordDto(
                item.Number,
                item.Title,
                item.Priority.ToString(),
                item.DueDate,
                item.Status.Name))
            .ToArray();
    }
}

public sealed record RecordDto(
    string Number,
    string Title,
    string Priority,
    DateTime? DueDate,
    string Status);
```

`WorkItem`, `WorkItemPriority`, `IsArchived`, `Status.Name`, and the DTO fields are placeholders.
Map the actual domain model and display labels. `IXafApplicationProvider.GetApplication()
.CreateObjectSpace(type)` obtains the XAF application's ObjectSpace; row-level security comes from
the configured XAF security strategy. Tenant isolation is not automatic in every application:
confirm its tenant strategy/provider is applied to this ObjectSpace and test two tenants. If the
application requires a tenant-aware `IObjectSpaceFactory` or another scoped provider, inject that
provider instead. Do not replace the secured ObjectSpace with an unscoped `DbContext`, trust a
tenant/id supplied by the model, or keep an ObjectSpace alive for the lifetime of a chat component.

## 3. Expose a narrow read-only model tool

This example uses `Microsoft.Extensions.AI` 10.8.0, pinned by the tested Kierat worktree. Kierat
uses the same `AIFunctionFactory.Create`, `ChatOptions.Tools`, and `IChatClient.GetResponseAsync`
pattern. The tool takes only a bounded business filter; authorization and tenant scope are resolved
inside the server method, never from tool arguments. Check the API against the target app's pinned
package version before copying:

```csharp
using Microsoft.Extensions.AI;
using System.ComponentModel;
using System.Text.Json;

public sealed class WorkspaceModelTools(WorkspaceRecords records, IChatClient chatClient)
{
    public async Task<string?> AskAsync(
        string prompt,
        IReadOnlyList<ChatMessage> messages,
        CancellationToken cancellationToken)
    {
        [Description("Lists records visible to the signed-in user. Optional priority is a business filter.")]
        string ListRecords([Description("Optional priority: urgent, high, medium, or low.")] string? priority = null)
        {
            var rows = records.ListVisibleRecords(limit: 100, priority: priority);
            return JsonSerializer.Serialize(rows);
        }

        var listRecords = AIFunctionFactory.Create((Delegate)ListRecords, "list_records");
        var options = new ChatOptions { Tools = [listRecords] };
        var requestMessages = messages.Append(new ChatMessage(ChatRole.User, prompt)).ToArray();
        var response = await chatClient.GetResponseAsync(requestMessages, options, cancellationToken);
        return response.Text;
    }
}
```

Keep the tool read-only and return DTOs, not XAF entities or serialized object graphs. Validate
model output independently before using its component names, data paths, columns or actions.

## 4. Build an A2UI surface and render it

The installed Syncfusion package owns the exact schema types. This v0.9 message shape follows the
working Kierat sequence: create a surface, populate data paths, validate the component tree,
process messages, then pass the resulting `SurfaceModel` to the provider. The mapping below shows
how a grid reads `/records`; do not accept a model-provided data path or field list as trusted.

```csharp
using System.Text.Json;
using System.Text.Json.Nodes;
using Syncfusion.A2UI.Core.Common;
using Syncfusion.A2UI.Core.Serialization;

private readonly string surfaceId = $"workspace-{Guid.NewGuid():N}";
private SurfaceModel? surface;

private void PublishSurface(string answer, IReadOnlyList<RecordDto> rows,
    JsonObject[] validatedComponents)
{
    // The validator rejects unknown components/ids, unsafe properties and paths. The host owns
    // the actual rows and columns: discard any model-provided grid bindings and inject these.
    var grid = validatedComponents.FirstOrDefault(component =>
        component["component"]?.GetValue<string>() == "SyncfusionDataGrid");
    if (grid is not null)
    {
        grid.Remove("dataSource");
        grid.Remove("columns");
        grid["dataSource"] = new JsonObject { ["path"] = "/records" };
        grid["columns"] = JsonSerializer.SerializeToNode(new object[]
        {
            new { field = "Number", headerText = "Number", width = "120" },
            new { field = "Title", headerText = "Title", width = "280" },
            new { field = "Priority", headerText = "Priority", width = "120" },
            new { field = "Status", headerText = "Status", width = "160" }
        });
    }

    var components = new List<JsonObject>
    {
        new() { ["id"] = "root", ["component"] = "Column",
            ["children"] = JsonSerializer.SerializeToNode(
                validatedComponents.Select(x => x["id"]!.GetValue<string>()).ToArray()) }
    };
    components.AddRange(validatedComponents);

    var messages = new List<object>();
    if (!processor.Model.HasSurface(surfaceId))
    {
        // Surface id and catalog are fixed for its lifetime. Duplicate createSurface is invalid.
        messages.Add(new { createSurface = new { surfaceId, catalogId = "<configured-catalog-id>", sendDataModel = false } });
    }
    messages.Add(new { updateDataModel = new { surfaceId, path = "/answer", value = answer } });
    messages.Add(new { updateDataModel = new { surfaceId, path = "/records", value = rows } });
    messages.Add(new { updateComponents = new { surfaceId, components } });
    var payload = JsonSerializer.Serialize(new { version = "v0.9", messages });

    using var json = JsonDocument.Parse(payload);
    processor.ProcessMessages(A2uiJson.ParseMessages(json.RootElement));
    surface = processor.Model.GetSurface(surfaceId);
}
```

`validatedComponents` must be produced by server validation. Reject unknown component names,
duplicate/invalid ids, unsafe properties and data paths outside an explicit allowlist such as
`/answer` and `/records`. The example maps a `SyncfusionDataGrid`'s `dataSource.path` to the
`/records` value supplied by `updateDataModel`, then supplies the host-approved columns matching
`RecordDto`. Remove model-supplied `dataSource`, `columns`, `series` and action callbacks when the
host owns those values; inject the approved server mappings afterward. Replace
`<configured-catalog-id>` with the catalog actually registered in the target app. The message
shape and catalog configuration must match the installed Syncfusion package.

## 5. Handle user actions and dispose the surface

AI tools run when the model asks for data. `OnAction` runs after a user interacts with a rendered
component. Subscribe for the Razor component’s lifetime and allow only the action attached to the
current surface and expected source component:

```csharp
private ISubscription? actionSubscription;

protected override void OnInitialized()
{
    actionSubscription = processor.Model.OnAction.Subscribe(
        new EventListener<A2uiClientAction>(HandleActionAsync));
}

private async ValueTask HandleActionAsync(A2uiClientAction action)
{
    if (action.SurfaceId != surfaceId
        || action.SourceComponentId != "refresh-records"
        || action.Name != "refresh_records"
        || busy)
        return;

    await InvokeAsync(async () =>
    {
        busy = true;
        try
        {
            var rows = records.ListVisibleRecords(limit: 100);
            PublishSurface(answer, rows, lastValidatedComponents);
        }
        catch (Exception exception)
        {
            Log.LogError(exception, "Workspace data refresh failed.");
            error = "The records could not be refreshed.";
        }
        finally
        {
            busy = false;
            StateHasChanged();
        }
    });
}

public void Dispose()
{
    actionSubscription?.Unsubscribe();
    if (processor.Model.HasSurface(surfaceId))
        processor.Model.DeleteSurface(surfaceId);
}
```

This event shape (`EventListener<A2uiClientAction>`, `ISubscription`, `SurfaceId`,
`SourceComponentId`, and `Name`) is verified against Syncfusion A2UI 35.1.37 in Kierat. Other
package versions may change it: compile against the pinned version and inspect a real action
payload. The example uses the same scoped `MessageProcessor` instance that processed the surface.
If a button/action is not part of the supported schema, render a host-owned Blazor action instead
of assuming a Composer property will invoke server code.

## 6. What to replace in another XAF application

- `WorkItem`, filters, DTO properties and readable enum/lookup labels.
- The secured ObjectSpace acquisition for the application’s XAF and tenant setup.
- `catalogId`, component allowlist, component-to-data-path mapping and renderer support.
- The prompt resource and tool descriptions for the actual domain.
- Action source ids/names and the server operation each action is allowed to call.
- Limits, authorization tests, component adapter tests and Playwright selectors.

The sample is a starting implementation, not proof that every Composer component is supported by
the target Blazor adapter. Test the actual component, data and interaction in the target app.
