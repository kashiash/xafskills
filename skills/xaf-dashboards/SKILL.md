---
name: xaf-dashboards
description: >
  Build, seed, secure and verify DevExpress Dashboards in an XAF Blazor application with EF Core.
  Use when adding a dashboard that mirrors an existing analysis or report, when a dashboard item
  returns HTTP 500 or shows the wrong number format, when pie charts or legends must filter the
  rest of the dashboard, when a seeded dashboard definition must be updated on deployed tenants,
  or when the Dashboards menu item is missing. Covers module setup, navigation, non-persistent
  data-source classes, building definitions in code, master filtering, number formats, filters,
  calculated fields, seed-and-update by definition hash, tests, and live verification against a
  deployed environment. Triggers: XAF Dashboards, DashboardData, DashboardObjectDataSource,
  DashboardItemGetAction, master filter, Pie dashboard item, HiddenDimensions, Callback request
  failed due to an internal server error.
---

# XAF Dashboards (Blazor, EF Core)

Use this skill to add or fix DevExpress Dashboards in an XAF Blazor Server application. The
guidance comes from building eight dashboards in DataDrive that mirror existing HTML analyses.
Each rule is marked **Verified** (observed on a deployed environment), **Reasoned** (derived from
the DevExpress model, not yet observed) or **Hypothesis** (best explanation of a pattern; confirm
before relying on it). A case study with the evidence and a verification script is in
`docs/xaf-dashboards-datadrive.md`.

Do not use this skill for Reports (`ReportDataV2`, see `xaf-reporting`) or for hand-written HTML
analysis pages.

## Before changing code

Inspect the target project:

- Is `AddDashboards(...)` registered in the Blazor host (not only in WinForms), and is
  `endpoints.MapXafDashboards()` mapped? Is `DashboardData` in the tenant `DbContext`?
- Does a model updater rebuild the navigation (see Navigation)? Are there existing dashboards, and
  who may edit them (security roles, `DashboardData` permissions)?
- How are tenant databases isolated and how does a non-persistent object space reach the tenant
  object space (the `ObjectsGetting` pattern in `xaf-blazor-startup`)?
- Whether an analysis already exists whose numbers the dashboard must reproduce.

## Module setup

```csharp
.AddDashboards(options => {
    options.DashboardDataType = typeof(DashboardData);
    options.HideDirectDataSourceConnections = true;   // users must not create raw SQL sources
})
// ...
endpoints.MapXafDashboards();
```

Dashboard definitions are stored in `DashboardData.Content` (XML). A dashboard built in code is
serialized with `dashboard.SaveToXDocument().ToString()`.

## Navigation

**Verified.** The Dashboards menu item can be missing even when the module is registered. If a
`NavigationItemsModelUpdater` (or similar) removes generated navigation nodes and re-adds only a
curated set, the module's own item is removed with the rest. Add it explicitly:

```csharp
if (node.Application.Views["DashboardData_ListView"] is { } view) {
    var item = reportsNode.Items.AddNode<IModelNavigationItem>("Dashboards");
    item.View = view; item.Caption = "Dashboardy"; item.ImageName = "dashboard";
}
```

- Use the same node Id in the role seeder's `AllowedNavigation` path (for example
  `Reports/Items/Dashboards`). With `DenyAllByDefault`, a missing entry means an invisible item.
- Localization may rename the caption. In DataDrive the item shows as "Pulpity nawigacyjne", so a
  menu filter for "Dashboard" finds nothing. Search by the localized caption.
- A dashboard opens at `/DashboardViewer_DetailView/<guid>`; the list is `/DashboardData_ListView`.

## Data sources: non-persistent row classes

Give each dashboard a flat, read-only row class and feed it from the tenant object space.

- Class: `[DomainComponent, DefaultClassOptions, NavigationItem(false)]`, a `[Key]` Guid `ID`,
  `[XafDisplayName]` on user-visible members, `[Browsable(false), VisibleInDashboards(true)]` on
  helper fields the dashboard needs but the field list should hide.
- Loading: handle `NonPersistentObjectSpace.ObjectsGetting` for the type, find the tenant space with
  `AdditionalObjectSpaces.FirstOrDefault(s => s.IsKnownType(typeof(SomeTenantEntity)))`, call the
  **same loader the analysis uses**, then project to rows. Throw a clear error when no tenant space
  is found.
- Register every row type with `DevExpress.Utils.DeserializationSettings.RegisterTrustedClass` and
  grant read-only `TypePermission` to each role that opens the dashboard. A missing permission
  fails with a generic EF error.
- **Verified:** every dashboard item issues its own data request. **Reasoned:** the loader therefore
  runs once per item. Cache per `NonPersistentObjectSpace` when several sources share one analysis,
  and plan for heavy loaders (see the materialization discussion in the case study).
- Prefer the analysis loader over fresh queries. Equivalence is then true by construction and
  testable: assert that projected sums equal the loader's totals.
- Duplicate display names merge different vehicles or customers. Disambiguate repeated labels with
  a short key suffix in the projection.

## Number formats and empty cells

- **Verified.** The Blazor viewer ignores `FormatType = Custom` with `CustomFormatString`. The
  stored definition keeps the string, but the item data (`MeasureDescriptors[].Format`) still says
  `Currency/Auto`, so users see "1,95M zł" and "0 zł". Use
  `FormatType = Number`, `Unit = Ones`, `IncludeGroupSeparator = true`, and `Precision` as needed.
  Custom text such as "Tak/Nie" also does not appear.
- **Verified.** Show zero as an empty cell by sending `null`, not `0`: declare amounts as
  `decimal?` and project with a `NullIfZero` helper. Make calculated fields return `null` instead of
  `0` in the false branch (`Iif(cond, 1, null)`).
- Chart axes can still abbreviate large numbers ("40K") independently of the measure format.

## Building definitions in code

Build the dashboard with the `DevExpress.DashboardCommon` API and keep the build deterministic:

- Give every item a stable `ComponentName`, put every item in `LayoutRoot`
  (`DashboardLayoutGroup` / `DashboardLayoutItem`), and give every measure an explicit format.
  A measure without a format gets default abbreviations.
- Calculated fields: write aggregate expressions over real row fields (`Sum([ServiceAmount])`).
  **Reasoned:** avoid calculated fields that reference other calculated fields; precompute
  per-row fields in the projection so every aggregate is a plain sum.
- Per-entity values repeated on each row (a vehicle's value) must use `Max`, or
  `Aggr(Max([Value]), [VehicleKey])`, never `Sum`.
- Sort a grid by value: set `dimension.SortByMeasure` and `SortOrder = Descending` on the first
  dimension column; otherwise the first screen shows alphabetical rows with empty values.
- Give card items enough layout weight. **Verified:** a weight of 15 (of about 150) left cards showing
  titles and no values, and 26 showed the values. Treat about 20 as untested.

## Interactivity: legends, pies and master filters

- **Verified.** Master filtering works on dimensions only. If cost classes are six measure
  columns, a pie or legend cannot filter the rest of the dashboard. Model the class as a
  dimension: one row per vehicle × month × class, a `CostClass` argument or series dimension and a
  single amount measure. Then a click on a pie slice or a stacked-bar segment filters every item
  bound to that source.
- Pie item: `Arguments` = class, `Values` = amount, `LabelContentType = PieValueType.ArgumentAndPercent`,
  `LegendPosition = Bottom`, `InteractivityOptions.IsMasterFilter = true`.
- Stacked chart: put the class in `SeriesDimensions`, set `Legend.Visible = true`,
  `Legend.IsInsideDiagram = false`, `OutsidePosition = BottomCenterHorizontal`, and
  `InteractivityOptions.TargetDimensions = Series` to filter by clicking a segment.
- **Verified.** `MasterFilterMode = Single` forces one element to be selected, so the dashboard
  starts already filtered (the selected value shows in the title). Use `Multiple`: a click selects,
  another click clears.
- **Reasoned.** Filtering across different data sources needs a shared dimension name; keep one
  source per dashboard when items must filter each other, and use `IgnoreMasterFilters` on items
  that must stay global.
- Rows with no class (an entity without cost in a month, kept for its valuation) create an empty
  series. Exclude them on pie and charts with `FilterString = "Not IsNull([CostClass])"` while
  keeping them for KPI and grids.

## Item filters: `FilterString`

- **Verified.** `VisibleDataFilterString` on a hidden measure used together with a Top N on that
  hidden measure returned HTTP 500. Sort and Top N by a **visible** measure and drop the visible
  data filter.
- **Hypothesis (fix deployed, confirm).** An item whose `FilterString` references a field that the
  item does not use as a dimension returns HTTP 500. Seven items failed with this pattern while every
  item whose filter fields appeared as dimensions worked. Add the filter fields as hidden
  dimensions:

```csharp
static void FilterBy(DataDashboardItem item, string filter, params string[] fields) {
    item.FilterString = filter;
    foreach (var f in fields) item.HiddenDimensions.Add(new Dimension(f));
}
```

  Open question to verify on a deployed environment: whether hidden dimensions split card
  aggregates into groups.
- Keep a contract test that every item with a filter has its filter fields among its
  `DataItems/Dimension` entries, plus a negative test on an item that lacks them. Parse only the
  item's own `DataItems`; `DefaultId` values such as `DataItem0` repeat across items.
- Filters that return no rows are a real case with sparse data. Do not assume an empty result is
  safe; check it on the deployed data.

## Seeding and updating definitions

`DashboardData` rows are user data. A seeder that only creates missing rows never delivers fixes,
and one that overwrites silently destroys user edits.

- **Verified.** Create a missing row by title. For an existing row, replace the content only when
  its SHA-256 equals one of the **known earlier seeded versions** (`KnownSeededSha256`). A
  definition edited in the designer has an unknown hash and stays untouched.
- Hash the content after normalizing CRLF to LF. Compute the hash of a new version **before**
  changing `Build()`: run `Sha256(Build().SaveToXDocument().ToString())` on the old code and add it to
  the list. Keep `ContentToWrite(existing, current, known)` a pure function and unit-test it.
- Seed at tenant start in the same transaction that commits other seed data, for every tenant.
- When a user has edited the stored definition, new versions will never reach it. Ask the owner:
  delete the row (the seeder recreates it) or seed the new version under a different title.
  Never delete user content without consent.
- Splitting a dashboard in two is a title-keyed change: the existing title keeps the slimmed
  definition, and the new title is created.

## Tests

Unit tests are necessary and not sufficient. In DataDrive they stayed green while the viewer ignored
number formats and seven items returned 500.

Write tests for:

- Projection equals the analysis: sums by period and by key match the loader's own totals.
- Zero amounts become `null`; classification rules (fuel groups, planned versus overdue) behave as
  in the analysis; planned items without a date are not overdue.
- Definition structure: item names, every item placed in the layout, formats on every measure
  definition (count `Measure` elements that have `DataMember`), pie and legend interactivity,
  filter fields present as dimensions.
- Seed update logic by hash, and permissions for every row type.
- Optional: `new DashboardExporter().ExportDashboardItemToPdf(dashboard, name, stream, ...)` with the
  data source replaced by an in-memory list runs the data engine headlessly. **Verified:** it did
  not reproduce the HTTP 500 cases, so it is a smoke test only.

## Live verification (required before reporting success)

After each deployment, open the dashboard in a real browser session and read the server responses.
Use Playwright with its own Chromium, log in with a role that has the permissions end users have,
open the dashboard from `DashboardData_ListView`, and record every response whose URL contains
`/api/dashboard/data/DashboardItemGetAction` (status and body) and
`/api/dashboard/dashboards/<guid>` (the stored definition). See the case study for a script.

Check:

- HTTP status of every `itemId`; a 500 body is `{"Message":"Callback request failed due to an
  internal server error."}` and the application log shows only an `ERR ... responded 500` line.
- `ItemData.MetaData.MeasureDescriptors[].Format` shows `Number/Ones` (not `Currency/Auto`).
- `ItemData.DataStorageDTO.Slices[].Data` holds the values; compare the totals with the analysis.
- The stored definition contains the new item names. If it does not, the row was edited by a user
  and the seeder skipped it.
- A screenshot for what the user sees. KPI cards that show titles and no numbers mean the item is
  too short, not that the data is missing.

Change one thing per deployment cycle when the cause is unclear, and record unconfirmed
explanations as hypotheses in the pull request.

## Troubleshooting

| Symptom | Likely cause | Action |
|---|---|---|
| Dashboards menu item missing | Navigation updater removed generated nodes | Add the item explicitly; match the Id in `AllowedNavigation` |
| Numbers show "1,95M zł" and "0 zł" | `Custom` format ignored by the Blazor viewer | `Number` + `Ones`, `null` instead of zero |
| One item returns 500, others fine | Filter on a non-dimension field, visible filter on a hidden measure, or empty filtered set | Hidden dimensions for filter fields; Top N by a visible measure; check the data |
| Dashboard starts already filtered | Pie or chart master filter in `Single` mode | Use `Multiple` |
| Pie cannot filter other items | Classes are measure columns | One row per class with a class dimension |
| New definition not visible after deploy | Stored row edited by a user, or hash list lacks the previous version | Read the stored definition; add the missing hash or ask the owner |
| Cards show titles but no values | Card item too short in the layout | Raise the layout weight |
| Columns show "54 4…" | Narrow grid columns | Give the grid more width; auto-fit column width is untested |
| Two vehicles appear as one | Duplicate display label | Add a key suffix to repeated labels |
