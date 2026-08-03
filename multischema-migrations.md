---
url: https://entgo.io/docs/multischema-migrations
title: "Multischema Migrations"
description: ""
access_date: 2026-08-03T19:39:02.137Z
current_date: 2026-08-03T19:39:02.137Z
---

Using the [Atlas](https://atlasgo.io/) migration engine, an Ent schema can be defined and managed across multiple database schemas. This guides show how to achieve this with three simple steps:

> **Atlas Pro Feature:**
> 
> The *multi-schema migration* feature is fully implemented in the Atlas CLI and requires a login to use:
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

## Annotate your Ent schemas

The `entsql` package allows annotating an Ent schema with a database schema name. For example:

```markdown
// Annotations of the User.
func (User) Annotations() []schema.Annotation {
    return []schema.Annotation{
        entsql.Schema("db3"),
    }
}
```

To share the same schema configuration across multiple Ent schemas, you can either use `ent.Mixin` or define and embed a *base* schema:

#### Mixin schema

```markdown
// Mixin holds the default configuration for most schemas in this package.
type Mixin struct {
    mixin.Schema
}

// Annotations of the Mixin.
func (Mixin) Annotations() []schema.Annotation {
    return []schema.Annotation{
        entsql.Schema("db1"),
    }
}
```
```markdown
// User holds the edge schema definition of the User entity.
type User struct {
    ent.Schema
}

// Mixin defines the schemas that mixed into this schema.
func (User) Mixin() []ent.Mixin {
    return []ent.Mixin{
        Mixin{},
    }
}
```

#### Base schema

```markdown
// base holds the default configuration for most schemas in this package.
type base struct {
    ent.Schema
}

// Annotations of the base schema.
func (base) Annotations() []schema.Annotation {
    return []schema.Annotation{
        entsql.Schema("db1"),
    }
}
```
```markdown
// User holds the edge schema definition of the User entity.
type User struct {
    base
}
```

## Generate migrations

To generate a migration, use the `atlas migrate diff` command. For example:

#### MySQL

```shell
atlas migrate diff \
  --to "ent://ent/schema" \
  --dev-url "docker://mysql/8"
```

#### MariaDB

```shell
atlas migrate diff \
  --to "ent://ent/schema" \
  --dev-url "docker://maria/8"
```

#### PostgreSQL

```shell
atlas migrate diff \
  --to "ent://ent/schema" \
  --dev-url "docker://postgres/15/dev"
```

> **Note:**
> 
> The `migrate` diff command generates a list of SQL statements without indentation by default. If you would like to generate the SQL statements with indentation, use the `--format` flag. For example:
> 
> ```shell
> atlas migrate diff \
>   --to "ent://ent/schema" \
>   --dev-url "docker://postgres/15/dev" \
>   --format "{{ sql . \"  \" }}"
> ```

## Controlling the Ent Client

When the `sql/schemaconfig` feature flag is enabled, Ent automatically uses the schema names defined in your `ent/schema` as the default runtime configuration. This means that any `entsql.Schema("db_name")` annotation is applied by default, and you can optionally override it at runtime if needed.

To enable the feature for your project, use the `--feature` flag:

```shell
--feature sql/schemaconfig
```

Once enabled, you can also override the schema configuration at runtime using the `ent.AlternateSchema` option:

```markdown
c, err := ent.Open(dialect, conn, ent.AlternateSchema(ent.SchemaConfig{
    User: "usersdb",
    Car:  "carsdb",
}))
c.User.Query().All(ctx) // SELECT * FROM \`usersdb\`.\`users\`
c.Car.Query().All(ctx)  // SELECT * FROM \`carsdb\`.\`cars\`
```
