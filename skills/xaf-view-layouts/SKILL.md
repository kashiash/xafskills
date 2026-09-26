---
name: xaf-view-layouts
description: Customize and persist user-specific XAF ListView and DetailView layouts, including column order, visibility, grouping, and form arrangement. Use when implementing layout customization or diagnosing why a user's saved layout does not appear.
---

# XAF view layouts

Use XAF's built-in layout customization and model-difference storage for personal view layouts. Do not create a second layout persistence system for ordinary per-user customization.

## User flow

- Enable the platform's native layout customization UI on the target view. In XAF Blazor, users open the view's context menu and choose **Customize Layout**.
- Let the user change the layout and close or apply the customization using the platform's own editor. XAF saves the result to that user's model differences.
- Keep a clear way to restore the application's default layout using the native **Reset Layout** command.
- Treat DetailView arrangement, ListView columns, sorting, grouping, and bands as view model differences. Do not mix them with saved business filters or appearance rules.
- Do not add a separate named-layout menu or management screen unless the product explicitly requires multiple named layouts per user.

## Persistence contract

Use the standard XAF model-difference entities and store. For EF Core, include these exact persistent types in the DbContext and register them with the XAF module:

```csharp
public DbSet<ModelDifference> ModelDifferences { get; set; }
public DbSet<ModelDifferenceAspect> ModelDifferenceAspects { get; set; }
```

The persistent contract is:

| Entity | Required persistent members | Meaning |
|---|---|---|
| `ModelDifference` | inherited `Id` key; `UserId: string`; `ContextId: string`; `Version: int`; `Aspects` one-to-many | One user's model-difference layer for one application context. |
| `ModelDifferenceAspect` | inherited `Id` key; `Owner: ModelDifference` many-to-one; `Name: string`; `Xml: string` | One culture aspect and its XAFML differences. Empty `Name` represents culture-neutral values. |

Use the XAF persistent implementations of `IModelDifference` and `IModelDifferenceAspect` (EF Core: `ModelDifference` and `ModelDifferenceAspect`; XPO: the corresponding XAF model-difference persistent objects). Preserve the `Owner` relationship with cascade deletion. Do not replace this schema with a custom `SavedLayout` table for the standard layout editor.

Scope the difference store with the same `ContextId` used by the application. The user store identifies the owner with the current user's XAF identifier. Keep the store secured and grant users only the read/write/create permissions the application requires for their own difference and aspect rows. A user layout is not public merely because it is in the database.

## Layer behavior and diagnostics

- XAF merges model layers. The user's saved layout overrides lower-level defaults for the same nodes.
- Treat code-generated or module-provided layouts as defaults. Do not write a competing runtime layout store; verify how the generated layer and user differences merge in the target XAF version.
- A reset removes that user's relevant differences; it does not rewrite the generated model or another user's layout.
- If changes do not appear, inspect the active model-difference store, `ContextId`, current user's `UserId`, and matching aspect XML before changing the layout code.
- When the application changes or removes model nodes, check existing user differences for obsolete paths. Confirm how the configured XAF store handles those differences on its next save.
- Before replacing a view's default layout generator or node tree, inspect stored `ModelDifference` records and module XAFML for that view. Differences targeting removed node paths may stop applying and can be removed by a later XAF save; preserve or migrate user customizations deliberately.
- Do not clear all model differences to fix one view. Reset only the target view through XAF's supported operation or perform a narrowly scoped, reviewed data repair.
- Layout persistence stores XAF model differences, not business data. Keep normal object-level permissions and tenant scoping intact.

## When named layouts are explicitly required

Multiple named layouts are a different feature from XAF's personal model layer. Before implementing one, specify who can see each layout, its user or role scope, target object type and `ViewId`, default-selection rules, tenant scope, and migration behavior. Do not silently repurpose `ModelDifference` aspects as named layouts: XAF uses those aspects for model and culture differences, not as a user-facing layout catalog.

## Completion checks

- A user can customize the required ListView or DetailView and the result survives a new session.
- A second user does not inherit the first user's personal layout.
- Reset restores the lower model layer for the selected view.
- The store uses the expected application `ContextId` and current user's identity.
- No custom persistence table or filter criterion is used for ordinary layout customization.
