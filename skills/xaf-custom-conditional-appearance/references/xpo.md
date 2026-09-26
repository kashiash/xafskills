# XPO appearance rules

Use this path only after confirming that the XAF module uses XPO. Fleetman is the observed XPO project. It has many static `[Appearance]` rules and uses an in-memory `AppearanceModel` for wizard behavior, but the inspected source did not contain the same persistent administrator-editable rule system as DataDrive.

## Port the behavior, not the EF entity

When adding administrator-editable rules to an XPO application:

1. Inspect the project's persistent base classes, session constructor patterns, mapped property conventions, class-info registration, and schema-update process.
2. Define the rule as an XPO persistent business object using those conventions. Do not copy an EF `BaseObject` class with virtual mapped properties, `DbSet`, EF annotations, or EF migrations.
3. Expose the target type and property chooser using XAF metadata, but persist a stable type key. Resolve it through XAF type metadata and handle renamed or missing types.
4. Reuse the shared XAF contract: project the persistent object to an immutable snapshot/adapter implementing `IAppearanceRuleProperties`, then select and add the snapshots through `CollectAppearanceRules`.
5. Query and save rules through XAF `IObjectSpace` using the project's secured ObjectSpace and tenant/security rules. Do not retain XPO objects after their ObjectSpace/session lifetime.
6. Publish cache changes only after commit succeeds. Use the appropriate XAF `ObjectSpace.Committed` or provider lifecycle hook after confirming its behavior for the installed XAF/XPO version. Do not mutate a global cache in an object's pre-commit `OnSaving` callback.
7. Invalidate both shared rule storage and per-view selected-rule caches. If Fleetman or the target app runs more than one server instance, define how rule changes reach each process.

The exact XPO base type and constructors vary by project. Choose the class actually used by neighboring business objects; Fleetman has both `XPObject` and its own `BaseObject(Session)` patterns. Do not copy an arbitrary example without checking the target entity family.

## What Fleetman already demonstrates

- Many business classes use XAF `[Appearance]` attributes for fixed rules. Keep them for behavior owned by the application and domain logic.
- `Fleetman.Module/Utils/AppearanceModel.cs` is a mutable in-memory implementation of `IAppearanceRuleProperties`.
- `Fleetman.Module/Controllers/IWizardDetailViewController.cs` creates layout rules from active wizard data, subscribes to `CollectAppearanceRules`, and removes the handler when deactivated. This is a code-generated rule path, not persistence of administrator-created rules.
- `Fleetman.Blazor.Server/XafControllers/OperationalListAppearanceController.cs` customizes a specific DxGrid view. It is not the generic persistence or rule-selection layer.

Use these as XPO/XAF integration examples, not as evidence that Fleetman already has a full database-backed rule editor.

## Verification specific to XPO

- Confirm new rules are discoverable by the application's XPO type/model registration and use the expected schema update procedure.
- Check object creation, edits, and deletes through the secured ObjectSpace; verify rollback does not publish changes to memory.
- Check that snapshot/adapter values remain usable after disposing the ObjectSpace.
- Check sessions and tenant scope under the application's real XPO configuration.
- Check Blazor and WinForms separately if both are supported; do not put DxGrid APIs in the shared XPO Module.
