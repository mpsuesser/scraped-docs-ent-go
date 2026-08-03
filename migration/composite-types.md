---
url: https://entgo.io/docs/migration/composite-types
title: "Composite Types"
description: ""
access_date: 2026-08-03T18:22:54.710Z
current_date: 2026-08-03T18:22:54.710Z
---

In PostgreSQL, a composite type is structured like a row or record, consisting of field names and their corresponding data types. Setting an Ent field as a composite type enables you to store complex and structured data in a single column.

This guide explains how to define a schema field type as a composite type in your Ent schema and configure the schema migration to manage both the composite types and the Ent schema as a single migration unit using Atlas.

> **Atlas Pro Feature:**
> 
> Atlas support for [Composite Types](https://atlasgo.io/atlas-schema/hcl#composite-type) is available exclusively to Pro users. To use this feature, run:
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

An `ent/schema` package is mostly used for defining Ent types (objects), their fields, edges and logic. Composite types, or any other database objects do not have representation in Ent models - A composite type can be defined once, and may be used multiple times in different fields and models.

In order to extend our PostgreSQL schema to include both custom composite types and our Ent types, we configure Atlas to read the state of the schema from a [Composite Schema](https://atlasgo.io/atlas-schema/projects#data-source-composite_schema) data source. Follow the steps below to configure this for your project:

1\. Create a `schema.sql` that defines the necessary composite type. In the same way, you can configure the composite type in [Atlas Schema HCL language](https://atlasgo.io/atlas-schema/hcl-types#composite-type):

#### Using SQL

```sql
CREATE TYPE address AS (
   street text,
   city   text
);
```

#### Using HCL

```markdown
schema "public" {}

composite "address" {
  schema = schema.public
  field "street" {
    type = text
  }
  field "city" {
    type = text
  }
}
```

2\. In your Ent schema, define a field that uses the composite type only in PostgreSQL dialect:

#### Schema

```markdown
// Fields of the User.
func (User) Fields() []ent.Field {
    return []ent.Field{
        field.String("address").
            GoType(&Address{}).
            SchemaType(map[string]string{
                dialect.Postgres: "address",
            }),
    }
}
```

> **Note:**
> 
> In case a schema with custom driver-specific types is used with other databases, Ent falls back to the default type used by the driver (e.g., "varchar").

#### Address Type

```markdown
type Address struct {
    Street, City string
}

var _ field.ValueScanner = (*Address)(nil)

// Scan implements the database/sql.Scanner interface.
func (a *Address) Scan(v interface{}) (err error) {
    switch v := v.(type) {
    case nil:
    case string:
        _, err = fmt.Sscanf(v, "(%q,%q)", &a.Street, &a.City)
    case []byte:
        _, err = fmt.Sscanf(string(v), "(%q,%q)", &a.Street, &a.City)
    }
    return
}

// Value implements the driver.Valuer interface.
func (a *Address) Value() (driver.Value, error) {
    return fmt.Sprintf("(%q,%q)", a.Street, a.City), nil
}
```

3\. Create a simple `atlas.hcl` config file with a `composite_schema` that includes both your custom types defined in `schema.sql` and your Ent schema:

```markdown
data "composite_schema" "app" {
  # Load first custom types first.
  schema "public" {
    url = "file://schema.sql"
  }
  # Second, load the Ent schema.
  schema "public" {
    url = "ent://ent/schema"
  }
}

env "local" {
  src = data.composite_schema.app.url
  dev = "docker://postgres/15/dev?search_path=public"
}
```

## Usage

After setting up our schema, we can get its representation using the `atlas schema inspect` command, generate migrations for it, apply them to a database, and more. Below are a few commands to get you started with Atlas:

#### Inspect the Schema

The `atlas schema inspect` command is commonly used to inspect databases. However, we can also use it to inspect our `composite_schema` and print the SQL representation of it:

```shell
atlas schema inspect \
  --env local \
  --url env://src \
  --format '{{ sql . }}'
```

The command above prints the following SQL. Note, the `address` composite type is defined in the schema before its usage in the `address` field:

```sql
-- Create composite type "address"
CREATE TYPE "address" AS ("street" text, "city" text);
-- Create "users" table
CREATE TABLE "users" ("id" bigint NOT NULL GENERATED BY DEFAULT AS IDENTITY, "address" "address" NOT NULL, PRIMARY KEY ("id"));
```

#### Generate Migrations For the Schema

To generate a migration for the schema, run the following command:

```shell
atlas migrate diff \
  --env local
```

Note that a new migration file is created with the following content:

```sql
-- Create composite type "address"
CREATE TYPE "address" AS ("street" text, "city" text);
-- Create "users" table
CREATE TABLE "users" ("id" bigint NOT NULL GENERATED BY DEFAULT AS IDENTITY, "address" "address" NOT NULL, PRIMARY KEY ("id"));
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

The code for this guide can be found in [GitHub](https://github.com/ent/ent/tree/master/examples/compositetypes).
