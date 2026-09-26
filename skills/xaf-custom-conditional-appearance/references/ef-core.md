# EF Core appearance rules

Use this path only after confirming that the target XAF module uses EF Core. DataDrive is the most complete observed implementation. HIS and PathQ have a simpler EF implementation with different lifecycle and rendering behavior.

## Persistent rule entity

Follow the application's EF conventions: its `BaseObject`, virtual mapped properties, `DbSet` or model registration, key type, and schema-update workflow. Do not introduce a second entity base or assume the feature can avoid a schema change.

The DataDrive entity is `AdditionalAppearanceRule` and currently includes a name, `DataTypeName`, criteria, target items, colors, border color, font style, visibility, context, priority, `IsDisabled`, and `ViewId`. Its `DataType` UI property is not mapped and resolves the stored name through `XafTypesInfo.Instance.FindTypeInfo`. Color values are persisted as strings and exposed as non-mapped `Color?` properties.

HIS and PathQ use `CustomApperance`, also derived from EF `BaseObject`, and persist `ObjectTypeFullName` plus `ObjectTypeName`. Their entity directly implements `IAppearanceRuleProperties`; DataDrive instead converts the entity to `AdditionalAppearanceRuleSnapshot`, which itself implements the interface.

Choose one consistent storage contract for a new project. Avoid copying both mappings or their legacy field names into a new entity.

## ObjectSpace and cache

- Query through the provider's XAF `IObjectSpace` and existing secured/tenant-aware patterns. Keep entity instances inside their ObjectSpace lifetime.
- If the application needs a process cache, project records to immutable snapshots. DataDrive's `AdditionalAppearanceRuleStorage` is keyed by a resolved tenant key and holds snapshots by rule ID.
- DataDrive schedules cache upsert/removal on `ObjectSpace.Committed` from `AdditionalAppearanceRule.OnSaving`. Preserve the important transaction boundary: do not publish a new rule before persistence succeeds.
- HIS/PathQ's `CustomApperance.OnSaving` updates a static collection of live entities before commit. This can make memory disagree with the database after rollback and retains provider objects outside the ObjectSpace lifecycle. Treat it as legacy behavior to replace, not a recommended pattern.
- In DataDrive, `AdditionalAppearanceRuleViewController.currentRules` is also cached per view. Updating shared storage does not by itself prove every already-open view has reselected rules. Explicitly invalidate affected view snapshots, reset and refresh XAF's appearance controller, and account for other application instances.

## Type and view selection

DataDrive's `AppearanceTypeMatching` gathers `FullName` for the view type and its base types, stopping before EF `BaseObject` and `object`. It then uses exact membership matching. `AdditionalAppearanceRuleSelector` applies disabled-state and `ViewId` checks and orders by `Priority`.

This allows a rule for a domain base class to apply to its derived views without allowing a rule for the framework's root entity class to color every view. Keep this policy only if it matches the target application's intended semantics; test both sides of the inheritance boundary.

HIS and PathQ compare `ObjectTypeName` with a name derived from `View.ObjectTypeInfo.Name` after removing a project-specific proxy prefix. That is a project workaround, not a general type identity strategy. Prefer exact canonical type names and XAF type metadata when designing a new implementation.

## Keep platform extensions separate

The standard rule is injected through `CollectAppearanceRules` in the shared Module. DataDrive's extra grid border rendering lives in a Blazor-specific controller and builds a criteria evaluator once per painter rather than once for every row. HIS also has separate Blazor grid rendering based on `BorderColorString`. Preserve this separation: standard appearance works through Conditional Appearance; border rendering depends on the editor/platform API.

## Files to inspect

- DataDrive entity, projection, selector, storage, type matching, and tests under `CS/DataDrive.Module/Features/Appearance/` and `CS/DataDrive.Module.Tests/Features/Appearance/`.
- HIS entity/controller/storage under `HIS.Module/BusinessObjects/Helpers/`, `HIS.Module/Controllers/HelperControllers/`, and `HIS.Module/Storages/`.
- HIS platform border handling in `HIS.Blazor.Server/XafControllers/DataGridListViewController.cs`.
- PathQ counterparts under `HIS.Module/` in the PathQ repository.
