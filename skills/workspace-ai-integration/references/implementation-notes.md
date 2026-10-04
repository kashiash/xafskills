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

The current Kierat A2UI code still assembles its system prompt in `AIChat.razor`; treat this as a
known maintenance gap, not a completed pattern. When evolving that integration, move system
instructions and named examples into explicit Markdown resources loaded by a small provider. For a
single assembly, embedded resources are deployable; if operators need post-deployment edits, use a
separately managed, validated configuration store. Fail clearly for missing resources/examples,
keep allowed components synchronized with the server contract, and add focused checks for resource
loading and variable substitution when this extraction is implemented.

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
working point-selection callback. The host therefore renders Workspace pie output with an explicit
`SfAccumulationChart` implementation and point selection; the rest of the app’s legacy chat chart
remains a separate path. Similarly, spreadsheet and Gantt adapters have their own
model/configuration requirements. Verify the actual renderer, its data source, and its license at
runtime.

Do not make the model responsible for wiring callbacks that are not part of the supported A2UI
schema. In the current Kierat Workspace, a pie request selects Priority or Status from the prompt,
then renders one host-owned pie. Clicking a slice filters the server-owned issue DTOs in the A2UI
data model; “Pokaż wszystkie zgłoszenia” clears the filter. The default legend position is Bottom;
an explicit request for a legend beside the chart uses Right. Tooltip content includes category,
count and percentage. Syncfusion's `SelectionMode.Point` applies selection to a pie point instead of
the generic chart adapter's component rectangle. If multiple category charts are added later, model
each selection separately and combine the filters deliberately; do not imply that one selection
state supports multiple charts.

For a Gantt view, query the actual authorized schedule entity, not business records with dates that
can be repurposed as tasks. In Kierat, use the secured XAF ObjectSpace for `Project`, `ScheduleTask`
and `ScheduleDependency`; bind task ID/name/start/end/duration/progress/parent/dependency fields.
The Gantt adapter needs a valid `EndDate` as well as `StartDate` and `Duration` for reliable taskbar
rendering in the host renderer. Keep the result bounded, resolve ambiguous project names against
authorized projects, and test a seeded known project end to end, asserting task rows and visible
taskbars rather than the component shell alone. See Syncfusion's [Gantt task field mapping](https://blazor.syncfusion.com/documentation/gantt-chart/connecting-to-database/mysql-server)
and [Gantt WBS/dependency mapping](https://blazor.syncfusion.com/documentation/gantt-chart/wbs-column).

For full-height XAF editors, avoid subtracting a guessed fixed number from `100vh` inside a nested
view item. Use the host's available height and a flex chain with `min-height: 0`; verify the actual
workspace bounding box against the viewport and check that embedded renderers (including Gantt and
Spreadsheet) stretch within that chain. Scope changes to a shared editor container only after
measuring the other editor types that use it.

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
- Pie selection changes only a chart point, and the tooltip shows its category plus the active
  measure (count or percentage); default and explicitly requested side legend positions are checked.
- Known schedule prompts render real secured task rows and taskbars, including hierarchy and
  dependencies; the editor reaches the available bottom edge of the XAF view.
- Clicking one category filters rows; clicking a second intersects it; deselection and clear-all
  restore expected rows. Keyboard activation works.
- Spreadsheet output includes actual authorized rows and readable requested columns.
- Browser testing logs in, reaches the workspace, exercises tools/actions, and records failures.
- Any deployment follows the target project’s own approval, isolation and rollback procedure.

## Case study: extracting the Workspace into reusable layers (Kierat, October 2026)

A second pass on the same integration, driven by test-environment findings. Failure log and what
fixed it, so the next agent does not rediscover them:

- **One 1,800-line component mixed everything.** Splitting it into a core library (no XAF), an AI
  turn library, a Blazor library, an XAF library and application tools was possible only after the
  sanitizer, action validation and prompt composition were pulled out of private Razor methods and
  given unit tests. Do this extraction first; moving files without tests hides regressions.
- **Namespace clash.** The app had an entity named `Workspace`; library namespaces beginning
  `Workspace.` produced CS0118 in the entity module. Use a different root (`WorkspaceA2UI.*`) while
  keeping the assembly name.
- **The older branch was not the deployed one.** A working tree on `main` was a stale parallel line;
  the test environment ran a much larger feature branch. Compare the intended change with the
  deployed branch (`git diff --stat origin/<deployed>`) before building; port the work onto the
  deployed line instead of deploying the working tree.
- **Silent empty Gantt.** The adapter maps rows to Gantt tasks by field name; `TaskID`/`ParentID`/
  `Predecessors` rendered zero rows and raised nothing. Use `TaskId`, `ParentId`, `Dependency`, and
  assert visible taskbars.
- **Children as `BlazorNodeRef`.** A custom `Row` renderer that stringified its children produced
  "Child 'BlazorNodeRef { Id = ... }' not found". Handle the reference type.
- **Chart beside Gantt looked empty** in one branch (no SVG, no console error). Not reproduced on the
  deployed line; the real defects were the layout being replaced by a Gantt-only layout and the issue
  list being zeroed. Reproduce on the deployed line before chasing adapter theories.
- **Components that end the circuit.** A coverage pass over all catalog entries found `3DChart`
  (inline data), `ListView` (`fields.groupBy`) and `PdfViewer` (missing host script) throwing. Block
  them in the catalog and sanitizer until fixed.
- **Authorization via search.** Full-text list filters walk related types (`Issue.Cycle.Name`,
  `Issue.EodProcess.Title`); a deny-by-default role threw `NullReferenceException` and showed an error
  page. Fix generally: drop search paths that cross types or members the user cannot read, and keep
  the role narrow. A blanket type-level read permission leaks other users' records. Also check that
  related-collection permissions granted automatically by XAF (for example `Team.Cycles`) do not
  expose more than the explicit rule.
- **Unbounded reads.** Tools that `ToArray()` all projects and all schedule tasks and then `Take(100)`
  in memory are a service-killer on large data. See the completeness contract in the skill.
- **Stub registration order.** With a real provider in user secrets, a stub registered first lost to
  it. Register the stub last and prove no external traffic from the host log.
- **Dev-only noise.** `NullReferenceException` in `MainPageBase.get_Application` (DevExpress) appears
  when a page is reloaded within a few hundred milliseconds of the previous load; reproduced
  locally, no user-visible effect; do not treat as a regression from your change.
- **Test data on the shared test environment.** E2E runs leave `E2E ...` records; an interrupted run
  can block the next one through a unique index that ignores soft-delete. Do not interrupt full
  runs; clean up only your own records, with a hard delete where a unique index requires it.

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
- [Syncfusion Gantt task field mapping](https://blazor.syncfusion.com/documentation/gantt-chart/connecting-to-database/mysql-server)
- [Syncfusion Gantt dependencies and WBS](https://blazor.syncfusion.com/documentation/gantt-chart/wbs-column)
- [Syncfusion accumulation chart tooltips](https://blazor.syncfusion.com/documentation/accumulation-chart/tool-tip)
- [Syncfusion accumulation chart adaptive legend layout](https://blazor.syncfusion.com/documentation/accumulation-chart/adaptive-layout)
- [DevExpress NavigationItemNodeGenerator](https://docs.devexpress.com/eXpressAppFramework/DevExpress.ExpressApp.SystemModule.NavigationItemNodeGenerator)
- [DevExpress built-in node generators](https://docs.devexpress.com/eXpressAppFramework/113316/ui-construction/application-model-ui-settings-storage/how-application-model-works/built-in-nodes-generators)
- [DevExpress two-tier security](https://docs.devexpress.com/eXpressAppFramework/113436/data-security-and-safety/security-system/security-tiers/2-tier-security-integrated-mode-and-ui-level)
