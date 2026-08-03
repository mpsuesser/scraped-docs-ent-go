---
url: https://entgo.io/docs/testing
title: "Testing"
description: ""
access_date: 2026-08-03T18:12:34.399Z
current_date: 2026-08-03T18:12:34.399Z
---

If you're using `ent.Client` in your unit-tests, you can use the generated `enttest` package for creating a client and auto-running the schema migration as follows:

```markdown
package main

import (
    "testing"

    "<project>/ent/enttest"

    _ "github.com/mattn/go-sqlite3"
)

func TestXXX(t *testing.T) {
    client := enttest.Open(t, "sqlite3", "file:ent?mode=memory&_fk=1")
    defer client.Close()
    // ...
}
```

In order to pass functional options to `Open`, use `enttest.Option`:

```markdown
func TestXXX(t *testing.T) {
    opts := []enttest.Option{
        enttest.WithOptions(ent.Log(t.Log)),
        enttest.WithMigrateOptions(migrate.WithGlobalUniqueID(true)),
    }
    client := enttest.Open(t, "sqlite3", "file:ent?mode=memory&_fk=1", opts...)
    defer client.Close()
    // ...
}
```
