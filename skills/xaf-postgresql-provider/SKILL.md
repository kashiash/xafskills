---
name: xaf-postgresql-provider
description: >
  Configure or review PostgreSQL support in DevExpress XAF applications that use EF Core.
  Use when adding Npgsql, switching database providers, configuring migrations or PostgreSQL
  extensions, or diagnosing PostgreSQL date and time behavior. Covers runtime and design-time
  provider wiring, connection settings, citext, and provider-specific model mapping. Do not use
  as an XPO setup guide; inspect the XPO provider and its project-specific requirements instead.
---

# XAF PostgreSQL Provider Setup

Use this skill when a task enables or changes PostgreSQL support in an XAF application with EF Core. First inspect the target repository. The three reference projects use different provider and date-time policies, so do not copy one project's setup as a universal recipe.

For the observed DataDrive, HIS, and PathQ settings, see [the PostgreSQL setup comparison](../../docs/postgresql-provider-setup.md). Treat it as evidence about those repositories, not as a guarantee that a target project still has the same code.

## Inspect the target

Before editing, find:

- XAF, EF Core, and Npgsql package versions. Keep provider and EF Core major versions compatible.
- Every runtime and design-time `DbContext` configuration, including secured and audited contexts.
- How the app chooses a provider: fixed provider, configuration value, build symbol, or environment variable.
- Every `MigrationsAssembly` setting and the migration project for each provider.
- Connection string sources for app startup, tests, design tools, background jobs, and tenant databases.
- PostgreSQL-specific model types, extensions, raw SQL, indexes, and functions.
- The current `DateTime` and `DateTimeOffset` storage policy, including converters and global Npgsql switches.

Do not print or copy credentials found in configuration files. Use secret storage or environment-specific configuration.

## Wire the provider consistently

Configure `UseNpgsql` for all contexts that use PostgreSQL. Include the same provider-specific options in runtime, XAF design-time, migrations, and test factories. Preserve XAF proxy, audit, and object-space options already used by the project.

If the application supports more than one database, make provider selection explicit and use the matching migration assembly in every creation path. Check Hangfire or other libraries that open their own database connection; they may need a provider-specific connection string or storage adapter.

Keep provider packages compatible with the application's EF Core version. Add or retain `Microsoft.EntityFrameworkCore.SqlServer` only when the target project's tooling or SQL Server path needs it. Do not add it just because another XAF project does.

## Configure connection settings safely

Use the configuration convention already established by the target project. Confirm which values are required in development, release, tests, and design-time commands. Prefer a complete connection string supplied through User Secrets, a secret manager, or environment variables.

For multi-tenant applications, verify both the host database and every tenant database. Provisioning tenant databases and granting permissions to create extensions are separate concerns. Never assume a host connection string is also a valid tenant connection string.

## Provision PostgreSQL extensions

Compare the EF model and migrations with extensions installed in the target server:

- In XAF applications using PostgreSQL, `citext` is **required for every text column**. Map all string properties—including XAF authentication, system, audit, and application properties—to PostgreSQL `citext`; do not leave mapped text properties as ordinary `text`/`varchar`. Declare the extension with `modelBuilder.HasPostgresExtension("citext")` and verify the generated schema actually uses `citext` for every text property.
- Provision `citext` in every database before the application creates or updates its schema. This includes host, tenant, test, and design-time databases. A model declaration or migration does not prove that the extension exists in each database or that the application role can install it. If the role lacks permission, have a DBA run `CREATE EXTENSION IF NOT EXISTS citext;` for each database before startup/migration.
- For an existing database, inventory all text columns and check for case-insensitive collisions before converting them (for example, `Admin` and `admin`). Use reviewed migrations or DBA scripts to convert the columns; installing the extension alone does not change existing column types. Recheck unique constraints and indexes, because `citext` makes their comparisons case-insensitive.
- Verify a fresh database and an upgraded database. Inspect the resulting schema to confirm all text columns use `citext`, and test equality, uniqueness, and XAF login behavior with values that differ only by case.
- `vector` is needed only when the model or queries use pgvector. EF's `UseVector()` registers Npgsql mappings; it does not install the server extension.
- `pg_trgm` is needed only for trigram indexes or queries.

Check whether the application or deployment process creates each extension. If not, arrange database administrator provisioning. Do not silently grant broad privileges to the app account just to make migrations pass.

## Choose and preserve a date-time policy

Npgsql distinguishes UTC, local, and unspecified `DateTime` values. Determine the intended meaning of existing fields and database columns before changing their PostgreSQL type or converter.

- If the project stores UTC instants, configure a deliberate UTC mapping for `DateTime` and `DateTimeOffset`, then check XAF system entities, audit data, seeds, and application entities.
- If the project stores timezone-free local or wall-clock values, map them intentionally to `timestamp without time zone` and preserve that meaning on reads and writes.
- Use `Npgsql.EnableLegacyTimestampBehavior` only when the target application depends on legacy Npgsql semantics and its behavior has been checked. Set it before the first Npgsql initialization on every entry path, including tests and design-time tools. It is process-wide compatibility behavior, not a database setting.
- Do not add the legacy switch as a generic fix for a timestamp exception. It can hide inconsistent data semantics.

## Apply and validate the change

1. Make the smallest provider-specific change that covers every context and entry point.
2. Confirm the connection source without exposing its password.
3. Confirm `citext` exists in every database before schema creation or migrations run, and verify that all text columns map to `citext`. Confirm other required extensions exist in their exact databases too.
4. Generate or inspect migrations with the intended provider and migration assembly.
5. Review generated column types, indexes, defaults, and raw SQL for PostgreSQL compatibility.
6. Check writes and reads for `DateTime` and `DateTimeOffset`, including XAF audit/model tables and seed data.
7. When both providers are supported, confirm the other provider's startup and migrations remain configured separately.

Report which paths you inspected and whether PostgreSQL is a supported runtime option, a test-only option, or leftover code. Do not infer support from a package reference alone.
