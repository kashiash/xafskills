---
name: workspace-ai-integration
description: >
  Integrate Syncfusion A2UI Workspace AI into an application, especially XAF Blazor Server.
  Use when adding an A2UI chat/workspace, server-side data tools, OnAction handling, component
  adapters, chart/list interactions, Excel output, or when model-generated UI does not match
  the request. Covers prompt resources, authorization-aware data, action validation, mapping
  A2UI components to host renderers, and real-browser verification. Triggers: Syncfusion A2UI,
  Workspace AI, A2UI OnAction, A2UI tools, Syncfusion Composer, A2UI Blazor adapter.
---

# Workspace AI integration

Use this skill to connect Syncfusion A2UI to real application data and behavior. Treat model
output as a request for a view: the server owns data access, authorization, action handling,
component mapping and export. A component appearing in the Composer catalog does not prove that
the target Blazor adapter renders it or that it is wired to application data.

Before coding, inspect the installed Syncfusion package versions, the application’s Composer
configuration, generated/available component catalog, Blazor adapters, authentication model,
and existing UI test setup. Read [implementation notes](references/implementation-notes.md) for
the reusable patterns and the Kierat case study’s failure log. For an XAF Blazor Server host,
also use the [workspace window, model-tool, and secured ObjectSpace snippets](references/xaf-blazor-workspace.md).
They are self-contained examples: the next agent does not need access to the Kierat repository.
Read the current Syncfusion and DevExpress XAF documentation for the APIs in use before implementation. The references below record checks that changed this integration. Verify API details against the
official [A2UI overview](https://blazor.syncfusion.com/documentation/syncfusion-a2ui/overview),
[getting started](https://blazor.syncfusion.com/documentation/syncfusion-a2ui/getting-started),
[AI integration](https://blazor.syncfusion.com/documentation/syncfusion-a2ui/ai-integration),
[supported components](https://blazor.syncfusion.com/documentation/syncfusion-a2ui/supported-components),
and the [A2UI v0.9 specification](https://a2ui.org/specification/v0.9-a2ui/). For XAF model/navigation behavior, consult DevExpress [NavigationItemNodeGenerator](https://docs.devexpress.com/eXpressAppFramework/DevExpress.ExpressApp.SystemModule.NavigationItemNodeGenerator), [built-in node generators](https://docs.devexpress.com/eXpressAppFramework/113316/ui-construction/application-model-ui-settings-storage/how-application-model-works/built-in-nodes-generators), and [two-tier security](https://docs.devexpress.com/eXpressAppFramework/113436/data-security-and-safety/security-system/security-tiers/2-tier-security-integrated-mode-and-ui-level). Package APIs and the specification can change; check the docs and the installed package version instead of copying an old sample.

Keep system instructions and user-facing prompt examples in separate editable Markdown resources,
not in a Razor component or a long C# literal. Load them through an explicit resource provider
(embedded resources are suitable for a single assembly); fail clearly when a resource or named
example is missing. Keep data field names and allowed component names synchronized with the server
contract.

Expose narrow, read-only model tools for fetching data. Execute every query through the current
user’s authorized application context (for XAF, the secured `IObjectSpace`/tenant scope); bound
result counts and fields, and never treat a model-supplied tenant, object id, filter, or serialized
record as authorization. For actions that mutate application state, validate the action name,
surface, source component and arguments against an explicit allowlist, then perform the operation
through application services and authorization checks. Ignore unknown and stale actions.

Implement the requested behavior in the host instead of expecting a generated event property to
execute. Subscribe to the A2UI action stream at the correct component/circuit lifetime, validate
the surface and source, marshal UI changes onto Blazor’s dispatcher, unsubscribe on disposal, and
verify the actual event payload from the installed package. Keep user-facing actions such as
Refresh as explicit, understandable actions. Model tools and UI actions are separate paths: tools
are invoked by the model; `OnAction` carries user interaction from rendered components.

For every advertised component, trace the complete path from model JSON to validated component,
server data, and concrete Blazor renderer/adapter. Build a compatibility table from the installed
Composer schema and actual host renderers. Normalize mismatches deliberately (for example a
generic chart schema may need a dedicated pie renderer); do not claim support based only on a
Composer catalog entry. Choose chart type from the user’s words and provide usable labels and
series values. For linked charts and tables, keep filter state in the host and combine selected
categories explicitly; provide a clear-all action and keyboard-accessible chart selection.

For pie charts, Syncfusion's generic chart adapter may select the component rectangle instead of a
slice. Prefer an explicit `SfAccumulationChart` with `SelectionMode.Point` when the host needs point
selection, and test the rendered SVG/callback. Enable a tooltip that includes category and the active
measure (count, value, or percentage). The default legend position is Bottom; move it to a side only
when requested. For Gantt, use the actual secured schedule/task objects, bind `StartDate`, `EndDate`,
`Duration`, progress, parent and dependency fields, and verify visible taskbars (a grid with task
names is insufficient). Read the [Gantt task mapping](https://blazor.syncfusion.com/documentation/gantt-chart/connecting-to-database/mysql-server), [accumulation tooltip](https://blazor.syncfusion.com/documentation/accumulation-chart/tool-tip), and [legend layout](https://blazor.syncfusion.com/documentation/accumulation-chart/adaptive-layout) docs alongside the installed package XML.

For full-height editors nested in XAF, avoid a guessed `100vh - Npx` offset. Use the available
height through each flex parent (`min-height: 0`) and verify the actual editor bounds in a browser.
Measure other editor types before changing shared host layout CSS.

For Excel or spreadsheet requests, export real authorized rows and readable domain values. Verify
the rendered sheet contains data, headers and the requested fields; a blank spreadsheet shell or
an unrelated download is not success. Do not create demo business records to make a test pass
unless the user explicitly authorizes that data setup.

Build the integration as reusable layers, not as one large component. A proven split (Kierat, October
2026): a core library with no UI-framework or XAF dependency (tool ports, server-owned action handlers,
layout sanitizer, component catalog, prompt composer, session/history), a small AI library that runs
the turn through `IChatClient` and function invocation, a Blazor library (workspace shell, example
tiles, preview, inspector, native renderers for what the adapter lacks, and a resizable
assistant/preview splitter), an XAF library (non-persistent workspace object, view item, navigation,
a secured full-text search controller) and the application, which supplies only the tools that know
its entities and registers them with one call. Add an architecture test that fails if the core or
Blazor library references XAF or DevExpress, and ship a sample app without XAF to prove the core is
replaceable. Do not duplicate Czat-style code into the library; reproduce its behavior.

The model returns layout only. The server supplies data and attaches user-interaction events
(row selection, refresh, open) to the components it renders; a validated layout may not contain
model-defined `event`/`on*` keys, model-supplied inline data or data paths outside an allowlist, or
external addresses (`urlTemplate`, `shapeData`, absolute `url`/`src`/`href`). Keep a component
catalog with one descriptor per allowed component and a verification status; mark components that
end the user's circuit as `Blocked` (excluded from the allowlist and the prompt, and rejected by the
sanitizer with a readable message), and test each one. Measure every catalog entry in a real
`SyncfusionA2UIProvider` (a development-only diagnostic page behind an explicit flag and sign-in)
and generate a coverage matrix; do not infer support from the Composer catalog.

Every tool result must say whether it is complete. Bound reads in the query (`Take(limit + 1)`,
bounded count), never `ToArray()` a whole table and then trim in memory, and return
`Complete`/`LimitExceeded`. On `LimitExceeded` give the model metadata, the bounded count and
suggested groupings/filters (with database-side `GROUP BY ... LIMIT` counts) instead of rows, and
make the system prompt require the model to say it cannot display that much data and to propose a
grouping. Compute chart aggregates in the database over the full set, never from already-truncated
rows. Do not send large row sets to the model; the host binds rows for the UI. A fixed row cap such
as 100 is a presentation choice, not a safety limit: the safety limit is a configurable ceiling
plus a query timeout and a payload-size cap for the browser circuit.

Workspace UX that users expect: example tiles only on an empty history; after the first command
(tile or typed) they disappear and a visible command history appears above the prompt box; clicking a
tile sends immediately and starts a fresh conversation (clear the model history and the current
layout, keep the visible command log), while a typed follow-up amends the current view; “Clear
view” resets history, filters and tiles. Provide a keyboard-accessible, persisted (guard
`localStorage`) splitter between the assistant and the preview with a hide/show button.

Known Syncfusion A2UI 35.1.37 adapter behavior to verify in the installed version before relying on
it: children of `Row` arrive as `BlazorNodeRef` (handle the type explicitly, not via `ToString()`);
`SyncfusionCard` honors `children`, not `child`; Gantt fields must be `TaskId`, `ParentId`,
`Dependency` (other casings render zero rows silently) and Gantt, TreeGrid and Scheduler data must be
inline in the component; `SyncfusionChart` has no pie; the toolbar emits no action; media reject
`data:` URLs; the spreadsheet reads `byte[]`; `SyncfusionQRCode` is the registered name; use
`InvariantCulture` for splitter pane sizes (a comma breaks `33,3%`). `3DChart` with inline data and
`ListView` with `fields.groupBy` throw and end the circuit; `PdfViewer` needs its script in the host
page.

When a development machine has a real model configured in user secrets, register the test
`IChatClient` stub after the configured providers (scoped registrations: the last one wins) and prove
from the host log that no external model traffic occurred. Verify real-model behavior on the
deployed test environment and say so explicitly; a stub proves the host path, not model behavior.

Test policy and mapping logic with focused tests. Then use the project’s actual test environment
and Playwright/browser setup to sign in, submit representative prompts, inspect rendered controls,
exercise actions, and check the resulting data/export. Capture evidence for failures. Report which
steps ran and which did not; a successful build or HTTP 200 alone is not runtime verification.

For XAF, see [XAF security](../xaf-security/SKILL.md) for permission patterns and
[XAF Playwright testing](../xaf-playwright-testing/SKILL.md) when those skills are present in the
same collection. For map-like output, inspect the installed adapter implementation: a model-controlled `shapeData` URL can cause server-side fetching. Reject remote resource properties unless the app has a fixed, allowlisted source. For tile maps, use the documented `UrlTemplate` only with a fixed provider; keep provider keys server-side and bound tile coordinates. See the [Syncfusion MapsLayer API](https://help.syncfusion.com/cr/blazor/Syncfusion.Blazor.Maps.MapsLayer-1.html).

Ship substantial integrations as small, isolated pull requests. Each PR should contain one coherent
reviewable slice, pass its scoped build/checks, and receive a review before starting or publishing
the next slice. Start from the intended base branch and stage only that slice; do not include the
rest of a dirty workspace.

Deployment is a separate authorized operation and must follow the target
project’s deployment procedure.
