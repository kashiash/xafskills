---
name: xaf-tags
description: >
  Design or maintain persistent tags for DevExpress XAF business objects. Use for tag models,
  EF Core or XPO assignments, bulk tag actions, tag filters, tag badges, or automatic tag rules.
  Covers ORM-specific mapping, type and tenant scope, assignment provenance, and safe reconciliation.
  Do not use for HTML tags, ordinary string labels, or fixed business statuses unless they are
  explicitly implemented as persistent XAF tags.
---

# XAF Tags

Use this skill when working with persistent tags on XAF business objects. Tags have three separate parts: tag definitions, assignments to records, and optional rules that manage assignments.

Inspect the target application's tag model, ORM, security rules, and assignment lifecycle before choosing an implementation. Use only patterns supported by that application.

When adding reversible automatic rules or provenance to assignments, also read [the automatic tagging design](../../docs/xaf-auto-tags-design.md). It defines safe ownership, removal, disable, and retry behavior.

## Identify what the project means by “tag”

Search for the tag entity, tag interfaces, their implementations, ORM relationships, controllers, property editors, background workers, and tests. Distinguish these cases:

- A persistent `Tag` entity linked to business records.
- A comma-separated or otherwise textual property rendered as badge-like labels. A string property editor does not prove that it uses the `Tag` entity.
- A fixed status, type, or enum. Keep it in the domain model unless users need a genuinely ad hoc label.

Report which business types actually support persistent tags. Do not infer support from the existence of `Tag`, a tag-shaped icon, or an editor named `TagStringPropertyEditor`.

## Model the tag catalog and scope

Follow the target project's XAF base object, naming, audit, localization, and tenant conventions. A catalog commonly needs a name and description; add archive state, type scope, global visibility, warning text, or badge color only when the product uses them.

If tags are limited by business-object type, store a stable type identity and resolve it through XAF type metadata. Apply the same type policy when listing choices, validating assignments, filtering records, and evaluating rules. Decide explicitly whether a tag for a base type also applies to derived types.

Keep archived tags out of new assignments by default. Decide separately whether existing records can continue showing archived tags and whether administrators may reassign them.

Tenant scope and XAF security must be enforced by the actual ObjectSpace and query path. A `Global` tag generally means “available across supported object types”; it must not bypass tenant or object permissions.

## Map assignments for the target ORM

### EF Core

Follow the project's relationship rules. Where the project requires explicit many-to-many entities, add a join entity for each taggable business type, map it in `OnModelCreating`, and expose the required `DbSet`. Review generated migrations and indexes. Do not replace a project-standard explicit join with an implicit skip navigation just to reduce code.

Add a deduplication rule at the assignment boundary. A unique database constraint can protect against duplicates as well as a collection check in application code.

### XPO

Use persistent `XPCollection<Tag>` properties and matching `[Association]` declarations on both sides. Keep association names consistent. Use the project's `Session` and XAF ObjectSpace lifecycle; do not pass persistent objects between sessions.

Do not copy relationship code from one ORM to the other. Keep shared behavior in small provider-neutral services only when both code paths can use them safely.

## Add safe manual actions

Put list actions on the appropriate root ListViews and keep selection behavior explicit. A typical controller offers add, filter, and remove actions. Use the current secured ObjectSpace to resolve popup selections before changing business objects.

Before committing:

- Recheck that each tag is valid for the target type and tenant.
- Avoid duplicate links.
- Define whether filtering by several selected tags means “any selected tag” or “all selected tags.”
- Commit once per user operation, then refresh the affected view through the target XAF lifecycle.
- Store criteria under a controller-owned key. Clear only that key when removing this controller's filter.

Do not rely on hidden actions as security. Keep type and object permissions on the tag catalog and tagged business objects.

## Design automatic rules deliberately

Keep criteria parsing and assignment logic outside UI controllers. A rule should identify a supported taggable type, a valid XAF criteria expression, and a tag valid for that type. Treat stale type names, invalid criteria, and deleted tags as recoverable rule errors; one bad rule should not break unrelated rules.

Make repeated evaluation idempotent. Scope background work to the correct tenant and create its ObjectSpace for that tenant. Commit changes to the assignment and its source together. If multiple worker instances can process the same tenant, use unique constraints and an appropriate lock or lease.

Choose the removal contract explicitly:

- For add-only rules, say that tags remain until a user removes them or another workflow does.
- For reversible rules, keep one visible object–tag assignment with separate source records for manual and rule-based ownership. Remove only the source owned by the rule; remove the visible assignment only when no source remains.
- If add and removal criteria are both configured, define overlap behavior. Prefer removal to win for that rule, and document that the criteria express independent business conditions.

Disabling a rule should pause it without silently deleting its prior assignments. Define explicitly what happens to its source records when the rule is deleted or its target type or tag changes. Keep event-triggered assignments separate from periodic criteria rules unless they share a clear source and lifecycle.

## Keep display separate from data

Use the `Tag` relationship as the source of truth. Avoid copying tag warnings into another delimited string when the display can be derived from the relationship. If a denormalized summary is required for list performance, define one canonical update service and cover add, remove, archive, rename, and duplicate-warning cases.

Treat badge color and row appearance as different features. Add a color field only if an editor or renderer actually consumes it. Use XAF Appearance rules or the project's established appearance mechanism for row styling; keep tag semantics independent from colors.

## Review data changes and behavior

Before finishing, confirm:

- The taggable types and their relationships match the requested scope.
- EF Core migrations or XPO schema updates follow the target project's process.
- Type, tenant, security, and archive filters are consistent across UI and automation.
- Add, remove, duplicate, and multi-tag filter semantics are defined.
- Automatic assignment is idempotent and cannot remove another source's tag.
- Any text badge editor is correctly distinguished from persistent tag relations.
- Tests cover model persistence, manual operations, criteria, tenant boundaries, repeated rule runs, and assignment ownership where applicable.

Keep provider-specific persistence details separate from the shared tag behavior. Follow the target application's established EF Core or XPO schema-update process.
