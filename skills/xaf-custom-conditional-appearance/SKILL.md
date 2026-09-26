---
name: xaf-custom-conditional-appearance
description: >
  Design, implement, or review custom conditional appearance rules in DevExpress XAF.
  Choose among built-in attributes/model rules, rules generated in code, and administrator-editable
  rules persisted with EF Core or XPO. Covers CollectAppearanceRules, IAppearanceRuleProperties,
  type/view selection, security, caching, and provider-specific persistence. Use for additional,
  runtime-configurable appearance rules; ordinary one-off [Appearance] usage can stay with XAF's
  built-in mechanism.
---

# XAF Custom Conditional Appearance

Use this skill for rules that extend XAF Conditional Appearance, especially rules managed at runtime. First decide whether the rule belongs in code, the Application Model, or persistent configuration. Do not turn every `[Appearance]` attribute into a database record.

## Choose the rule source

| Requirement | Prefer |
|---|---|
| Fixed business behavior that changes with application code | `[Appearance]` attribute or Application Model rule, following the project's existing pattern. |
| Runtime rule generated from current view or non-persistent data | An in-memory object implementing `IAppearanceRuleProperties`, added through `AppearanceController.CollectAppearanceRules`. |
| Authorized administrators must create or edit rules without deploying code | A persistent rule entity and `CollectAppearanceRules` integration. Choose its persistence implementation from the project's ORM. |

Keep built-in and custom rules additive unless the product explicitly asks to replace a rule. Keep custom grid rendering (for example, cell borders or CSS customization) in a platform-specific controller; standard Conditional Appearance and custom rendering are separate presentation paths.

## Inspect before implementation

Establish these facts from the target repository before choosing a design:

1. ORM and XAF version for the business module. Do not infer the ORM from transitive package references.
2. Persistent base class, entity registration, key conventions, and schema-update process.
3. Supported UI platforms and whether the rule affects standard XAF view items, list rows, actions, or a custom grid renderer.
4. Tenant isolation, XAF permissions for rule records, and who may create, edit, disable, or delete them.
5. Existing appearance attributes, controllers, cache/storage helpers, non-persistent views, and rule configuration screens.
6. How criteria are authored, validated, stored, and evaluated for the selected type.

Read the relevant provider reference and project examples before copying a pattern:

- [EF Core projects](references/ef-core.md)
- [XPO projects](references/xpo.md)
- [Observed project differences](references/project-patterns.md)

## Canonical persistent contract

For new EF Core and XPO projects, use the same domain name and persisted field names:

- Entity and table: `AdditionalAppearanceRule`.
- Shared fields: `Name`, `ObjectTypeFullName`, `Criterion`, `TargetItems`, `Context`, `IsDisabled`, `ViewId`, and `Priority`.
- Keep appearance-specific values (such as colors, font style, visibility, and enabled state) as additional fields when required.
- Do not create provider-specific synonyms such as `ObjectTypeName`, `Criteria`, `IsActive`, or a typo-based table name. Provider-specific base classes, key conventions, and mappings may differ, but the public domain contract stays identical.
- When adopting this contract in an existing application, preserve existing rows and migrate old property/table names explicitly; do not silently create an empty replacement table.

## Create a rule from a list filter

When users need to turn the current list filter into an appearance rule, expose a ListView action named “Zapisz filtr jako regułę wyglądu”. Read the active filter from the list editor (for a grid, use its filter criteria when the editor filter is empty), and create a new `AdditionalAppearanceRule` prefilled with `Name`, the current view's `ObjectTypeFullName`, `Criterion`, `Context = ListView`, and `ViewId`. Open its DetailView in a modal dialog so the user can choose appearance properties and target items before saving. Do not save if the filter is empty or cannot be validated for the view's object type. After a successful commit, invalidate appearance-rule caches and refresh the current list. The action does not grant create permission: keep it unavailable to users who lack permission to create rules.

## Shared XAF integration

For database-backed rules, the reusable shape is:

```text
provider-specific persistent rule
        -> immutable rule snapshot / IAppearanceRuleProperties adapter
        -> selector for type, view, tenant, state, and priority
        -> ObjectView controller subscribes to CollectAppearanceRules
        -> AppearanceController evaluates the selected rules
```

The persistence model and ObjectSpace queries are provider-specific. The rule-selection policy, runtime XAF contract, and cache invalidation goals can be shared.

### Rule data and type selection

- Store a stable type identifier, normally `Type.FullName`, unless the project's existing type system requires another canonical key. Resolve it through `XafTypesInfo` and handle renamed or missing types explicitly.
- Do not use `StartsWith` as general type matching. Compare canonical type names exactly. If base-type rules should apply to derived views, collect that view type's intended business hierarchy and match each name exactly. Stop at framework-wide roots such as `BaseObject` or `object` when including them would apply a rule to every entity.
- Include an optional `ViewId` only if administrators need view-level scope. Define whether an empty value means every view of the selected type.
- Store criteria, target items, context, priority, and only the appearance properties the feature supports. Nullable appearance values mean “leave this aspect unchanged.” Use the target-type criteria editor where the provider and XAF model support it.
- When the target type changes, clear or revalidate dependent criteria and target items. Otherwise stale member names can remain attached to the new type.
- Model enabled/disabled state explicitly when administrators need reversible deactivation. Keep ordering direction and collision behavior consistent with the project's tested XAF/version behavior; do not assume that larger priority numbers always win.

### Controller and lifetime

- Guard `Frame.GetController<AppearanceController>()`; some views do not expose it.
- Subscribe to `CollectAppearanceRules` only for the relevant ObjectView lifetime and always unsubscribe during deactivation. Reset/refresh the XAF controller cache at the points required by the installed XAF version and project behavior.
- Filter rules for the current view before adding them. Do not query or add unrelated rules from all types or tenants.
- Do not retain persistent entities after their ObjectSpace ends. If caching is needed, cache immutable snapshots, not EF or XPO objects.
- A cached rule set has at least two invalidation layers: the shared/provider cache and each controller's selected-rule cache. After a rule changes, ensure affected open views reload both layers and refresh. Resetting `AppearanceController` alone does not refresh a separate list already cached by a custom controller.
- Update shared cache only after a successful commit. For multiple application instances, add cross-instance invalidation or a bounded freshness strategy. A process-local static dictionary cannot notify other nodes.

### Criteria and failure handling

- Validate the criterion against the selected object type when saving. Revalidate when applying it because schema/member names can change after the rule was saved.
- Verify target item names and the supported `AppearanceItemType`/context combinations. A database rule should not be allowed to refer to arbitrary UI actions or members unless the feature explicitly supports them.
- A malformed or stale rule must not take down the view. Skip that rule, log its identifier and failure reason, and make the issue visible to an administrator. Do not use an empty catch that hides rule corruption.
- Keep XAF security responsible for access to business objects. A visual rule is not an authorization boundary. Protect the rule entity itself with type/object/member permissions and tenant scoping.

## Provider-specific implementation

Do not copy a persistent entity between providers. For EF Core, use the project's EF base type, virtual persistent properties, DbContext registration, and migration conventions. For XPO, use the project's XPO persistent base and session/property conventions. Keep provider-specific code in the Module layer if all supported platforms need the rule; put editor/grid-specific rendering in the appropriate platform project.

Read only the relevant provider reference for implementation details. The XPO reference describes a porting pattern: Fleetman demonstrates XPO and code-generated appearance rules, but it does not currently provide the same administrator-editable database rule system as DataDrive.

## Keep these designs separate

- **Persistent configuration:** rule records that admins edit. They need permissions, validation, persistence lifecycle handling, and cache invalidation.
- **Generated in-memory rules:** rules derived from an active workflow or non-persistent view. They need correct controller lifecycle and refresh when inputs change, but no persistent rule entity.
- **Custom rendering:** styling unsupported by or outside standard Conditional Appearance. It belongs in platform-specific grid/editor code and should consume a validated rule snapshot if it shares configuration.

Do not merge these paths into one controller merely because they all affect appearance.

## Verification checklist

- Confirm both a built-in rule and a custom rule can coexist where intended.
- Check exact type matching, derived types, framework roots, and each supported `ViewId` behavior.
- Check authorized and unauthorized users, tenants, disabled rules, and create/update/delete behavior.
- Check valid criteria, invalid criteria, deleted/renamed members, and target items that are no longer present.
- Check failed commit leaves the previous cached rules intact; successful commit updates the active view as required.
- Check navigation/deactivation detaches event handlers and does not retain disposed ObjectSpace entities.
- If deployed on multiple instances, verify how a committed update reaches the other instances.
- Check each UI platform and separately verify any custom border/CSS/grid rendering.
- Use existing project tests and validation rules. Do not run builds or tests when repository instructions prohibit them.

## Related material

- [Comparison and recommendation](../../docs/appearance-rules-comparison.md) — Fleetman, DataDrive, HIS, and PathQ.
- DataDrive implementation: `CS/DataDrive.Module/BusinessObjects/AdditionalAppearanceRule.cs` and `CS/DataDrive.Module/Features/Appearance/`.
- Standard XAF entry point: `AppearanceController.CollectAppearanceRules` and `IAppearanceRuleProperties` in the installed DevExpress version.
