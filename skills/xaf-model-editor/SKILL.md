---
name: xaf-model-editor
description: Build or extend an XAF Model Editor for runtime inspection and editing of Application Model nodes and values. Use when adding model-tree actions, editing model properties, localization aspects, reset operations, or saving user or shared model differences.
---

# XAF Model Editor

Keep Application Model editing separate from view layout customization. A Model Editor changes model nodes and values; a layout editor changes the arrangement and presentation of a particular view. Both may persist through XAF model differences, but they need distinct actions, scope, and user guidance.

## Actions and scope

- Provide an **Edit Model** action for editing the current user's model layer.
- Provide **View in Model** when an action should open the editor focused on the current view's model node.
- Offer a separate **Edit Shared Model** action only when the application configures a shared difference store and authorizes administrators to write to it.
- Gate model editing with XAF's `ModelOperationPermissionRequest`. Do not treat an administrative role flag alone as proof that a user may edit the model.
- Make edits pending until the user saves. Warn before closing with pending changes; offer a clear discard path.
- Support reset of a value or node by removing the override from the active difference layer, so the lower layer supplies the value again.
- For localized values, edit the selected XAF aspect and keep the culture-neutral aspect distinct from named culture aspects.
- Validate required values and references before saving newly created or changed nodes. Report invalid node paths and property names to the user.

## Persistence contract

Use XAF's standard persistent model-difference types. Do not invent a generic `ModelSetting`, `ModelNode`, or JSON document table for data XAF already represents as model differences.

The EF Core schema contract is exactly:

| Entity | Required persistent members | Meaning |
|---|---|---|
| `ModelDifference` | inherited `Id` key; `UserId: string`; `ContextId: string`; `Version: int`; `Aspects` one-to-many | A model layer owned by one user in one application context. |
| `ModelDifferenceAspect` | inherited `Id` key; `Owner: ModelDifference` many-to-one; `Name: string`; `Xml: string` | XML differences for one culture aspect; empty `Name` means culture-neutral. |

Use `DevExpress.Persistent.BaseImpl.EF.ModelDifference` and `ModelDifferenceAspect`, or the corresponding XPO persistent implementations of `IModelDifference` and `IModelDifferenceAspect`. In EF Core, expose both types as DbSets and configure `ModelDifference.Aspects` to `ModelDifferenceAspect.Owner` with cascade deletion. Keep the store's optimistic version check; do not overwrite a newer difference layer silently.

The user store uses the signed-in user's XAF `UserId`. A shared administrator layer uses the same entities and context with an empty `UserId`, separated by its configured store. Treat this as a distinct privileged layer, not a public user preference. Authorize shared writes on the difference and aspect objects and ensure the shared store is loaded beneath each user's layer.

## Save and discard behavior

- Save the user's model differences through XAF's configured `ModelDifferenceDbStore` or the application's existing model-difference store.
- Validate pending edits before committing. If validation fails, keep the editor open and identify the missing or invalid values.
- Apply add, clone, move, delete, and reset operations in the target model layer. Do not persist incomplete newly created nodes.
- XAF may save a user's live model when a circuit closes or a user logs in again. Ensure pending editor operations cannot leak into that deferred save.
- If the editor applies changes to a warmed-up model whose values cannot be rolled back safely, discard by rebuilding the user's model from persisted differences; suppress any deferred save of the discarded in-memory state.
- After save or discard, refresh/rebuild the active application model when required by the platform. Do not claim a model change is visible until the model has reloaded.
- When resetting the last value for a culture aspect, verify that the persisted aspect no longer brings the old value back on the next model load. Clean only the emptied aspect for the current owner, context, and version.

## Security and concurrency

- Check model-edit permission before showing edit actions and recheck authorization at save time.
- Scope user differences by the current XAF user identity and application `ContextId`.
- Shared editing needs separate authorization for the shared `ModelDifference` and its `ModelDifferenceAspect` records.
- Use secured ObjectSpaces for normal reads and writes. Use a non-secured ObjectSpace only for a narrowly justified lookup that the security system cannot perform; keep all mutation permission checks explicit.
- Preserve XAF's version/concurrency guard. If another editor saved a newer version, report the conflict and reload before retrying.
- A model-difference XML document is configuration data, not permission to bypass business-object security.

## Separation from layout customization

- Route requests to arrange columns, groups, bands, editors, or tabs to the view-layout workflow.
- Route requests to change model node values, add or remove model nodes, localize captions, inspect model metadata, or edit shared application defaults here.
- Do not add layout arrangement controls to the model-tree editor merely because both features use the same persistence entities.

## Completion checks

- User edits remain isolated by `UserId` and `ContextId`.
- Shared edits use the configured empty-user shared layer and require explicit write permission.
- Save validates required values and rejects stale-version writes.
- Reset removes only the intended override and remains reset after a fresh model load.
- Discarded or unsaved edits are absent after closing and rebuilding the model.
- The editor does not change business data or use layout customization as a substitute for model editing.
