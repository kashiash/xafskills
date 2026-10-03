# Implementation notes and case study

These notes record patterns verified while building the Kierat Workspace AI integration. They are
examples to adapt, not a framework that every A2UI host must copy. Check each API against the
Syncfusion and Microsoft AI packages pinned by the target project.

## A2UI protocol lifecycle and trust boundary

A2UI messages are a structured protocol, not trusted markup. Validate component ids and references,
allowed catalog names, properties and data paths after generation and before `ProcessMessages`. Keep
the data model host-owned; never let model output choose authorization scope or remote URLs. In
particular, inspect map adapter code: Syncfusion A2UI 35.1.37 maps its `shapeData` input into a
server-side shape fetch, so a model-supplied URL can create an SSRF path. Do not expose that adapter
unless its remote resource is fixed and allowlisted. Syncfusion Maps uses `UrlTemplate` for tile
providers; when the provider needs a key, proxy tiles through a fixed server endpoint, keep the key
server-side, require authorization as appropriate, and validate zoom/x/y bounds. See the official
[MapsLayer API](https://help.syncfusion.com/cr/blazor/Syncfusion.Blazor.Maps.MapsLayer-1.html).

Create each surface once with a unique id and fixed catalog/configuration. A `createSurface` for an
already-existing id is a protocol error; for later renders send `updateDataModel` and
`updateComponents`. Delete and recreate only when the catalog/config changes or the owning component
is disposed. Include the required `root` component and validate every child reference. See the
[A2UI v0.9 protocol specification](https://a2ui.org/specification/v0.9-a2ui/).

## Keep prompts editable

Kierat initially kept system instructions and example prompts in `AIChat.razor`. That made prompt
changes require editing a large UI file and made tests difficult to focus. The implementation now
stores `WorkspaceAiSystemPrompt.md` and `WorkspaceAiExamples.md` as explicit embedded resources,
loaded by a small `WorkspaceAiPrompts` class. Named examples use stable headings such as
`## priority-and-status-pies`; tests verify resource loading and variable substitution. Keep
allowed components dynamic so a catalog change does not silently leave stale prompt instructions.

For a standalone Razor class library, embedded Markdown is simple and deployable. If operators need
to edit prompts after deployment, use a separately managed, validated configuration store instead;
do not write runtime changes back into the assembly.

## Separate model tools from UI actions

An AI function/tool is called by the model. A UI action is emitted by a rendered A2UI component
after a user gesture. Implement each path and test each independently.

The minimal model-tool shape with `Microsoft.Extensions.AI` is:

```csharp
IssueRow[] ListIssues(string? priority = null)
{
    // Query the current user's secured XAF ObjectSpace; apply field/row limits.
    return query.ToArray();
}

var listIssues = AIFunctionFactory.Create((Delegate)ListIssues, "list_issues");
var chatOptions = new ChatOptions { Tools = [listIssues] };
```

The exact ObjectSpace acquisition depends on the XAF View and its lifetime. Do not capture a
disposed ObjectSpace in a long-running chat or reuse it across circuits. Resolve the current user
and tenant from trusted server context. Return DTOs with only fields needed by the model.

For UI events, the Syncfusion integration exposes the processor model’s action observable. The
Kierat pattern subscribes while its component is active and disposes the subscription when the
component is disposed:

```csharp
// Use the exact types exposed by the installed Syncfusion package.
private ISubscription? actionSubscription;

protected override void OnInitialized()
{
    actionSubscription = processor.Model.OnAction.Subscribe(
        new EventListener<A2uiClientAction>(HandleAction));
}

private void HandleAction(A2uiClientAction action)
{
    if (!IsAllowedAction(action.SurfaceId, action.SourceComponentId, action.Name)) return;
    _ = InvokeAsync(async () =>
    {
        await RefreshAuthorizedRowsAsync();
        StateHasChanged();
    });
}

public void Dispose() => actionSubscription?.Unsubscribe();
```

The Kierat integration uses `ISubscription.Unsubscribe()`; some package versions or observables may
use a different subscription type. Use the actual SDK names and event fields from the installed
Syncfusion package; this is not a version-independent drop-in. Bind the action to an explicit policy
over surface id, source component id, action name, and any required arguments. Keep the policy
independently testable.
Do not assume every Composer button variant emits the same action contract; observe and test the
payload for each component and adapter.

## Component compatibility and host rendering

Maintain a table with at least these columns:

| Component requested | Composer schema name | Host renderer/adapter | Data mapping | Interaction tested |
|---|---|---|---|---|
| Example: pie chart | Catalog component name | Actual Blazor chart | XAF DTO fields | Slice click / keyboard |

Kierat found that its Composer component list and available Syncfusion Blazor adapters were not
identical. A generic `SyncfusionChart` declaration did not by itself guarantee a pie chart or a
working point-selection callback. The host therefore renders its Workspace pie output with an
explicit accumulation-chart implementation; the rest of the app’s legacy chat chart remains a
separate path. Similarly, spreadsheet and Gantt adapters have their own model/configuration
requirements. Verify the actual renderer, its data source, and its license at runtime.

Do not make the model responsible for wiring callbacks that are not part of the supported A2UI
schema. In Kierat, two Workspace pies (Priority and Status) share a server-owned filtered issue
list. Clicking a slice toggles that selection; selections from both pies intersect; “Show all”
clears both. The table contains the readable status name. Focused policy tests check the AND behavior
and clearing one selection without dropping the other.

## Excel and data truth

When the user requests a spreadsheet, fetch authorized rows through the model tool or the host’s
equivalent data function. Map UI fields to readable values before binding: a displayable status
name is useful in Excel, while a raw foreign-key id usually is not. Assert that output cells contain
the returned records; checking only that an Excel component appeared misses the original failure
mode (an empty sheet).

Do not invent a “type” field because the prompt mentions kinds of issues. Inspect the actual domain
model. In Kierat the issue has Priority and Status; labels are a separate relation and must be
mapped deliberately if the product later wants that dimension.

## Failures seen and corrections

- A request for a pie was rendered as columns because prompt wording, generic chart schema and the
  host adapter did not agree. State chart type explicitly in the prompt contract and verify the
  rendered chart, not the model JSON alone.
- A chart appeared but selecting a point did not filter the table. The generic chart callback path
  was not reliable in the actual adapter. Keep interaction state in the host and render a tested
  host chart with clickable, keyboard-accessible points/legend controls.
- One filter worked, but a second category dimension was needed. Treat each selection as explicit
  state, combine filters with AND semantics, display active selections, and include a clear-all
  control. Test combinations and deselection.
- Excel opened without issue rows. The sheet component existed, but its data mapping did not
  contain values. Verify real rows and readable field mappings end-to-end.
- Long system prompts and examples in Razor were difficult to adjust. Move them into named Markdown
  resources, make resource loading testable, and keep the C# wrapper small.
- Full test runs may expose unrelated XAF global static initialization order failures when tests
  initialize `ValueManager` as `SimpleValueManager` and later as `AsyncValueManager`. Confirm that
  the failing test also fails on the base revision and run the feature tests independently before
  attributing it to A2UI changes. Do not hide or “fix” global test harness failures inside an A2UI
  change without evidence.
- A green build and a landing page response did not prove deployment worked. Sign in, verify the
  protected workspace, exercise actions, inspect data, and retain browser evidence.

## Verification checklist

- Prompt resources are loaded from the published assembly, and a missing named example fails
  clearly.
- Only the actual allowed component catalog is advertised to the model.
- Data tools use the current user’s tenant and permissions, return bounded DTOs, and do not accept
  authority from model arguments.
- Unsupported components are rejected, mapped explicitly, or rendered by a tested host adapter.
- User actions validate surface, source and action; subscriptions have the same lifetime as the
  component and are disposed.
- Representative chart prompts render the requested chart type and labels.
- Clicking one category filters rows; clicking a second intersects it; deselection and clear-all
  restore expected rows. Keyboard activation works.
- Spreadsheet output includes actual authorized rows and readable requested columns.
- Browser testing logs in, reaches the workspace, exercises tools/actions, and records failures.
- Any deployment follows the target project’s own approval, isolation and rollback procedure.

## XAF navigation and security documentation

For generated navigation, use the XAF model-node generator extension point documented by
DevExpress: implement `ModelNodesGeneratorUpdater<NavigationItemNodeGenerator>` and register it in
`ModuleBase.AddGeneratorUpdaters`; do not assume an item decorated with `[NavigationItem]` alone
produces the desired grouped runtime navigation. For a standalone Blazor page, use a synthetic model
view/controller redirect when that matches the host architecture. Read and follow the DevExpress
[NavigationItemNodeGenerator](https://docs.devexpress.com/eXpressAppFramework/DevExpress.ExpressApp.SystemModule.NavigationItemNodeGenerator)
and [built-in node generators](https://docs.devexpress.com/eXpressAppFramework/113316/ui-construction/application-model-ui-settings-storage/how-application-model-works/built-in-nodes-generators)
docs. Queries must use the secured XAF `ObjectSpace`; its permission criteria apply to data reads.
Verify tenant scoping in the actual app, since tenant isolation is application-specific. See
[DevExpress two-tier security](https://docs.devexpress.com/eXpressAppFramework/113436/data-security-and-safety/security-system/security-tiers/2-tier-security-integrated-mode-and-ui-level).

## References

- [Syncfusion A2UI overview](https://blazor.syncfusion.com/documentation/syncfusion-a2ui/overview)
- [Getting started](https://blazor.syncfusion.com/documentation/syncfusion-a2ui/getting-started)
- [AI integration](https://blazor.syncfusion.com/documentation/syncfusion-a2ui/ai-integration)
- [Supported components](https://blazor.syncfusion.com/documentation/syncfusion-a2ui/supported-components)
- [A2UI specification v0.9](https://a2ui.org/specification/v0.9-a2ui/)
- [Syncfusion MapsLayer API](https://help.syncfusion.com/cr/blazor/Syncfusion.Blazor.Maps.MapsLayer-1.html)
- [DevExpress NavigationItemNodeGenerator](https://docs.devexpress.com/eXpressAppFramework/DevExpress.ExpressApp.SystemModule.NavigationItemNodeGenerator)
- [DevExpress built-in node generators](https://docs.devexpress.com/eXpressAppFramework/113316/ui-construction/application-model-ui-settings-storage/how-application-model-works/built-in-nodes-generators)
- [DevExpress two-tier security](https://docs.devexpress.com/eXpressAppFramework/113436/data-security-and-safety/security-system/security-tiers/2-tier-security-integrated-mode-and-ui-level)
