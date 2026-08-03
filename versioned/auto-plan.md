---
url: https://entgo.io/docs/versioned/auto-plan
title: "Auto Plan"
description: ""
access_date: 2026-08-03T18:12:34.399Z
current_date: 2026-08-03T18:12:34.399Z
---

One of the convenient features of Automatic Migrations is that developers do not need to write the SQL statements to create or modify the database schema. To achieve similar benefits, we will now add a script to our project that will automatically plan migration files for us based on the changes to our schema.

To do this, Ent uses [Atlas](https://atlasgo.io/), an open-source tool for managing database schemas, created by the same people behind Ent.

If you have been following our example repo, we have been using SQLite as our database until this point. To demonstrate a more realistic use case, we will now switch to MySQL. See this change in [PR #3](https://github.com/rotemtam/ent-versioned-migrations-demo/pull/3/files).

## Using the Atlas CLI to plan migrations

In this section, we will demonstrate how to use the Atlas CLI to automatically plan schema migrations for us. In the past, users had to create a custom Go program to do this (as described [here](programmatically.md)). With recent versions of Atlas, this is no longer necessary: Atlas can natively load the desired database schema from an Ent schema.

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

Then, run the following command to automatically generate migration files for your Ent schema:

#### MySQL

```shell
atlas migrate diff migration_name \
  --dir "file://ent/migrate/migrations" \
  --to "ent://ent/schema" \
  --dev-url "docker://mysql/8/ent"
```

#### MariaDB

```shell
atlas migrate diff migration_name \
  --dir "file://ent/migrate/migrations" \
  --to "ent://ent/schema" \
  --dev-url "docker://mariadb/latest/test"
```

#### PostgreSQL

```shell
atlas migrate diff migration_name \
  --dir "file://ent/migrate/migrations" \
  --to "ent://ent/schema" \
  --dev-url "docker://postgres/15/test?search_path=public"
```

#### SQLite

```shell
atlas migrate diff migration_name \
  --dir "file://ent/migrate/migrations" \
  --to "ent://ent/schema" \
  --dev-url "sqlite://file?mode=memory&_fk=1"
```

> **The role of the dev database:**
> 
> Atlas loads the **current state** by executing the SQL files stored in the migration directory onto the provided [dev database](https://atlasgo.io/concepts/dev-database). It then compares this state against the **desired state** defined by the `ent/schema` package and writes a migration plan for moving from the current state to the desired state.

Next, let's see how to upgrade an existing production database to be managed with versioned migrations.
