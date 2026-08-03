---
url: https://entgo.io/docs/migration/row-level-security
title: "Row Level Security"
description: ""
access_date: 2026-08-03T19:00:49.651Z
current_date: 2026-08-03T19:00:49.651Z
---

Row-level security (RLS) in PostgreSQL enables tables to implement policies that limit access or modification of rows according to the user's role, enhancing the basic SQL-standard privileges provided by `GRANT`.

Once activated, every standard access to the table has to adhere to these policies. If no policies are defined on the table, it defaults to a deny-all rule, meaning no rows can be seen or mutated. These policies can be tailored to specific commands, roles, or both, allowing for detailed management of who can access or change data.

This guide explains how to attach Row-Level Security (RLS) Policies to your Ent types (objects) and configure the schema migration to manage both the RLS and the Ent schema as a single migration unit using Atlas.

> **Atlas Pro Feature:**
> 
> Atlas support for [Row-Level Security Policies](https://atlasgo.io/atlas-schema/hcl#row-level-security-policy) used in this guide is available exclusively to Pro users. To use this feature, run:
> 
> ```markdown
> atlas login
> ```

## Install Atlas

To install the latest release of Atlas, simply run one of the following commands in your terminal, or check out the [Atlas website](https://atlasgo.io/getting-started#installation):

#### macOS + Linux

```shell
curl -sSf https://atlasgo.sh | sh
```

#### Homebrew

```shell
brew install ariga/tap/atlas
```

#### Docker

```shell
docker pull arigaio/atlas
docker run --rm arigaio/atlas --help
```

If the container needs access to the host network or a local directory, use the `--net=host` flag and mount the desired directory:

```shell
docker run --rm --net=host \
  -v $(pwd)/migrations:/migrations \
  arigaio/atlas migrate apply
  --url "mysql://root:pass@:3306/test"
```

#### Windows

Download the [latest release](https://release.ariga.io/atlas/atlas-windows-amd64-latest.exe) and move the atlas binary to a file location on your system PATH.

## Login to Atlas

```shell
$ atlas login a8m
You are now connected to "a8m" on Atlas Cloud.
```

## Composite Schema

An `ent/schema` package is mostly used for defining Ent types (objects), their fields, edges and logic. Table policies or any other database native objects do not have representation in Ent models.

In order to extend our PostgreSQL schema to include both our Ent types and their policies, we configure Atlas to read the state of the schema from a [Composite Schema](https://atlasgo.io/atlas-schema/projects#data-source-composite_schema) data source. Follow the steps below to configure this for your project:

1\. Let's define a simple schema with two types (tables): `users` and `tenants`:

```markdown
// Tenant holds the schema definition for the Tenant entity.
type Tenant struct {
    ent.Schema
}

// Fields of the Tenant.
func (Tenant) Fields() []ent.Field {
    return []ent.Field{
        field.String("name"),
    }
}

// User holds the schema definition for the User entity.
type User struct {
    ent.Schema
}

// Fields of the User.
func (User) Fields() []ent.Field {
    return []ent.Field{
        field.String("name"),
        field.Int("tenant_id"),
    }
}
```

2\. Now, suppose we want to limit access to the `users` table based on the `tenant_id` field. We can achieve this by defining a Row-Level Security (RLS) policy on the `users` table. Below is the SQL code that defines the RLS policy:

```sql
--- Enable row-level security on the users table.
ALTER TABLE "users" ENABLE ROW LEVEL SECURITY;

-- Create a policy that restricts access to rows in the users table based on the current tenant.
CREATE POLICY tenant_isolation ON "users"
    USING ("tenant_id" = current_setting('app.current_tenant')::integer);
```

3\. Lastly, we create a simple `atlas.hcl` config file with a `composite_schema` that includes both our Ent schema and the custom security policies defined in `schema.sql`:

```markdown
data "composite_schema" "app" {
  # Load the ent schema first with all tables.
  schema "public" {
    url = "ent://ent/schema"
  }
  # Then, load the RLS schema.
  schema "public" {
    url = "file://schema.sql"
  }
}

env "local" {
  src = data.composite_schema.app.url
  dev = "docker://postgres/15/dev?search_path=public"
}
```

## Usage

After setting up our composite schema, we can get its representation using the `atlas schema inspect` command, generate schema migrations for it, apply them to a database, and more. Below are a few commands to get you started with Atlas:

#### Inspect the Schema

The `atlas schema inspect` command is commonly used to inspect databases. However, we can also use it to inspect our `composite_schema` and print the SQL representation of it:

```shell
atlas schema inspect \
  --env local \
  --url env://src \
  --format '{{ sql . }}'
```

The command above prints the following SQL. Note, the `tenant_isolation` policy is defined in the schema after the `users` table:

```sql
-- Create "users" table
CREATE TABLE "users" ("id" bigint NOT NULL GENERATED BY DEFAULT AS IDENTITY, "name" character varying NOT NULL, "tenant_id" bigint NOT NULL, PRIMARY KEY ("id"));
-- Enable row-level security for "users" table
ALTER TABLE "users" ENABLE ROW LEVEL SECURITY;
-- Create policy "tenant_isolation"
CREATE POLICY "tenant_isolation" ON "users" AS PERMISSIVE FOR ALL TO PUBLIC USING (tenant_id = (current_setting('app.current_tenant'::text))::integer);
-- Create "tenants" table
CREATE TABLE "tenants" ("id" bigint NOT NULL GENERATED BY DEFAULT AS IDENTITY, "name" character varying NOT NULL, PRIMARY KEY ("id"));
```

#### Generate Migrations For the Schema

To generate a migration for the schema, run the following command:

```shell
atlas migrate diff \
  --env local
```

Note that a new migration file is created with the following content:

```sql
-- Create "users" table
CREATE TABLE "users" ("id" bigint NOT NULL GENERATED BY DEFAULT AS IDENTITY, "name" character varying NOT NULL, "tenant_id" bigint NOT NULL, PRIMARY KEY ("id"));
-- Enable row-level security for "users" table
ALTER TABLE "users" ENABLE ROW LEVEL SECURITY;
-- Create policy "tenant_isolation"
CREATE POLICY "tenant_isolation" ON "users" AS PERMISSIVE FOR ALL TO PUBLIC USING (tenant_id = (current_setting('app.current_tenant'::text))::integer);
-- Create "tenants" table
CREATE TABLE "tenants" ("id" bigint NOT NULL GENERATED BY DEFAULT AS IDENTITY, "name" character varying NOT NULL, PRIMARY KEY ("id"));
```

#### Apply the Migrations

To apply the migration generated above to a database, run the following command:

```markdown
atlas migrate apply \
  --env local \
  --url "postgres://postgres:pass@localhost:5432/database?search_path=public&sslmode=disable"
```

> **Apply the Schema Directly on the Database:**
> 
> Sometimes, there is a need to apply the schema directly to the database without generating a migration file. For example, when experimenting with schema changes, spinning up a database for testing, etc. In such cases, you can use the command below to apply the schema directly to the database:
> 
> ```shell
> atlas schema apply \
>   --env local \
>   --url "postgres://postgres:pass@localhost:5432/database?search_path=public&sslmode=disable"
> ```
> 
> Or, using the [Atlas Go SDK](https://github.com/ariga/atlas-go-sdk):
> 
> ```markdown
> ac, err := atlasexec.NewClient(".", "atlas")
> if err != nil {
>     log.Fatalf("failed to initialize client: %w", err)
> }
> // Automatically update the database with the desired schema.
> // Another option, is to use 'migrate apply' or 'schema apply' manually.
> if _, err := ac.SchemaApply(ctx, &atlasexec.SchemaApplyParams{
>     Env: "local",
>     URL: "postgres://postgres:pass@localhost:5432/database?search_path=public&sslmode=disable",
>     AutoApprove: true,
> }); err != nil {
>     log.Fatalf("failed to apply schema changes: %w", err)
> }
> ```

## Code Example

After setting up our Ent schema and the RLS policies, we can open an Ent client and pass the different mutations and queries the relevant tenant ID we work on. This ensures that the database upholds our RLS policy:

```markdown
ctx1, ctx2 := sql.WithIntVar(ctx, "app.current_tenant", a8m.ID), sql.WithIntVar(ctx, "app.current_tenant", r3m.ID)
users1 := client.User.Query().AllX(ctx1)
// Users1 can only see users from tenant a8m.
users2 := client.User.Query().AllX(ctx2)
// Users2 can only see users from tenant r3m.
```

> **Real World Example:**
> 
> In real applications, users can utilize [hooks](../hooks.md) and [interceptors](../interceptors.md) to set the `app.current_tenant` variable based on the user's context.

The code for this guide can be found in [GitHub](https://github.com/ent/ent/tree/master/examples/rls).
