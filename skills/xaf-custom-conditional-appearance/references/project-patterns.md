# Observed XAF appearance patterns

These are repository observations, not requirements to copy unchanged. Verify current code, ORM, XAF version, hosting topology, and security in the target application.

| Project | ORM | Existing pattern | Reuse / avoid |
|---|---|---|---|
| **DataDrive** | EF Core | `AdditionalAppearanceRule` is an editable EF entity. `AdditionalAppearanceRuleSnapshot` implements `IAppearanceRuleProperties`. Rules are selected by exact type hierarchy, view, disabled state, and priority. A process-local concurrent storage holds snapshots by tenant. | Best observed EF starting point. Reuse the snapshots, exact type matching, tenant key discipline, and post-commit shared-storage update. Improve invalidation for already-open views and other app instances; cache the parsed criteria/selected snapshots deliberately. The Blazor border renderer is separate from standard Conditional Appearance. |
| **HIS** | EF Core | `CustomApperance` is an EF entity and directly implements `IAppearanceRuleProperties`. `CustomApperanceStorage` holds live entity objects in a static list; `OnSaving` updates the list before commit. Separate Blazor code renders configured borders. Another controller generates in-memory rules for non-persistent registration slots. | Good evidence for editable rule fields, property selection, border-color extension, and code-generated non-persistent rules. Do not preserve live entity references outside ObjectSpace or update storage before commit. Type resolution by `Type.GetType` and matching by proxy-stripped short name are project-specific legacy choices. |
| **PathQ** | EF Core | Forked `CustomApperance`/controller/storage pattern from HIS. The generic controller excludes `NonPersistentObjectSpace`. Current entity does not include `BorderColor` or `CloneFrom`. | Check the fork's actual needs rather than assuming feature parity with HIS. Avoid its static live-entity cache and short-name-only matching in a new implementation. |
| **Fleetman** | XPO | Fixed rules mostly use `[Appearance]`. `AppearanceModel` implements `IAppearanceRuleProperties` for wizard-generated in-memory rules. The list filter controller can create a persistent `CustomApperance` from the active filter. `OperationalListAppearanceController` customizes one Blazor grid view. | Use the XPO entity and controller conventions only for provider-specific persistence details; the shared new-project contract is `AdditionalAppearanceRule`. Fleetman also has a list action that opens a new appearance rule prefilled from the current list filter. Keep XPO persistence and platform grid rendering separate. |

## Material design differences

- **Persistent entities:** DataDrive/HIS/PathQ use EF `BaseObject`; Fleetman uses XPO patterns. Their persistent classes and schema update paths are not interchangeable.
- **XAF integration:** All can feed `IAppearanceRuleProperties` into `CollectAppearanceRules`, but Fleetman/HIS also create rules in code. Keep the in-memory and persisted configurations distinct.
- **ObjectSpace lifetime:** DataDrive emits snapshots. HIS/PathQ store persistent instances globally. The latter pattern can retain disposed-session objects and can diverge from the database if a transaction fails.
- **Scope:** DataDrive resolves a tenant key. The inspected HIS/PathQ/Fleetman rule paths do not show the same tenant-keyed appearance cache. Confirm the target system's isolation model; don't assume a missing key means cross-tenant access is valid.
- **Appearance properties:** DataDrive and HIS currently include border color, with separate Blazor rendering. PathQ's fork does not. A property in the persistent model does not make its rendering portable across UI platforms.
- **Non-persistent views:** HIS has an explicit code-generated appearance path for registration slots. PathQ skips non-persistent ObjectSpaces in its database-rule controller. Choose and test this behavior explicitly.
- **Rule refresh:** Updating storage after commit is only one part. A controller may also cache its selected rules for the life of a view. Invalidate that selection, refresh the XAF AppearanceController, and define cross-process invalidation when needed.

## Source locations

- DataDrive: `CS/DataDrive.Module/BusinessObjects/AdditionalAppearanceRule.cs`; `CS/DataDrive.Module/Features/Appearance/`; `CS/DataDrive.Blazor.Server/Controllers/AdditionalAppearanceRuleGridController.cs`; `CS/DataDrive.Module.Tests/Features/Appearance/`.
- HIS: `HIS.Module/BusinessObjects/Helpers/CustomApperance.cs`; `HIS.Module/Controllers/HelperControllers/CustomApperanceViewControler.cs`; `HIS.Module/Storages/CustomApperanceStorage.cs`; `HIS.Module/Controllers/RegistrationFlow/RegistrationTimeSlotNPListViewController.cs`; `HIS.Blazor.Server/XafControllers/DataGridListViewController.cs`.
- PathQ: `HIS.Module/BusinessObjects/Helpers/CustomApperance.cs`; `HIS.Module/Controllers/HelperControllers/CustomApperanceViewControler.cs`; `HIS.Module/Storages/CustomApperanceStorage.cs`.
- Fleetman: `Fleetman.Module/Utils/AppearanceModel.cs`; `Fleetman.Module/Controllers/IWizardDetailViewController.cs`; `Fleetman.Blazor.Server/XafControllers/OperationalListAppearanceController.cs`; `[Appearance]`-decorated business objects under `Fleetman.Module/BusinessObjects/`.
