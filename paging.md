---
url: https://entgo.io/docs/paging
title: "Paging"
description: ""
access_date: 2026-08-03T18:12:34.399Z
current_date: 2026-08-03T18:12:34.399Z
---

## Limit

`Limit` limits the query result to `n` entities.

```markdown
users, err := client.User.
    Query().
    Limit(n).
    All(ctx)
```

## Offset

`Offset` sets the first node to return from the query.

```markdown
users, err := client.User.
    Query().
    Offset(10).
    All(ctx)
```

## Ordering

`Order` returns the entities sorted by the values of one or more fields. Note that, an error is returned if the given fields are not valid columns or foreign-keys.

```markdown
users, err := client.User.Query().
    Order(ent.Asc(user.FieldName)).
    All(ctx)
```

Starting with version `v0.12.0`, Ent generates type-safe ordering functions for fields and edges. The following example demonstrates how to use these generated functions:

```markdown
// Get all users sorted by their name (and nickname) in ascending order.
users, err := client.User.Query().
    Order(
        user.ByName(),
        user.ByNickname(),
    ).
    All(ctx)

// Get all users sorted by their nickname in descending order.
users, err := client.User.Query().
    Order(
        user.ByNickname(
            sql.OrderDesc(),
        ),
    ).
    All(ctx)
```

## Order By Edge Count

`Order` can also be used to sort entities based on the number of edges they have. For example, the following query returns all users sorted by the number of posts they have:

```markdown
users, err := client.User.Query().
    Order(
        // Users without posts are sorted first.
        user.ByPostsCount(),
    ).
    All(ctx)

users, err := client.User.Query().
    Order(
        // Users without posts are sorted last.
        user.ByPostsCount(
            sql.OrderDesc(),
        ),
    ).
    All(ctx)
```

## Order By Edge Field

Entities can also be sorted by the value of an edge field. For example, the following query returns all posts sorted by their author's name:

```markdown
// Posts are sorted by their author's name in ascending
// order with NULLs first unless otherwise specified.
posts, err := client.Post.Query().
    Order(
        post.ByAuthorField(user.FieldName),
    ).
    All(ctx)

posts, err := client.Post.Query().
    Order(
        post.ByAuthorField(
            user.FieldName,
            sql.OrderDesc(),
            sql.OrderNullsFirst(),
        ),
    ).
    All(ctx)
```

## Custom Edge Terms

The generated edge ordering functions support custom terms. For example, the following query returns all users sorted by the sum of their posts' likes and views:

```markdown
// Ascending order.
posts, err := client.User.Query().
    Order(
        user.ByPosts(
            sql.OrderBySum(post.FieldNumLikes),
            sql.OrderBySum(post.FieldNumViews),
        ),
    ).
    All(ctx)

// Descending order.
posts, err := client.User.Query().
    Order(
        user.ByPosts(
            sql.OrderBySum(
                post.FieldNumLikes,
                sql.OrderDesc(),
            ),
            sql.OrderBySum(
                post.FieldNumViews,
                sql.OrderDesc(),
            ),
        ),
    ).
    All(ctx)
```

## Select Order Terms

Ordered terms like `SUM()` and `COUNT()` are not defined in the schema and thus do not exist on the generated entities. However, sometimes there is a need to retrieve their information in order to either display it to the user or implement cursor-based pagination. The `Value` method, defined on each entity, allows you to obtain the order value if it was selected in the query:

```markdown
// Define the alias for the order term.
const as = "pets_count"

// Query users sorted by the number of pets
// they have and select the order term.
users := client.User.Query().
    Order(
        user.ByPetsCount(
            sql.OrderDesc(),
            sql.OrderSelectAs(as),
        ),
        user.ByID(),
    ).
    AllX(ctx)

// Retrieve the order term value.
for _, u := range users {
    fmt.Println(u.Value(as))
}
```

## Custom Ordering

Custom ordering functions can be useful if you want to write your own storage-specific logic.

```markdown
names, err := client.Pet.Query().
    Order(func(s *sql.Selector) {
        // Logic goes here.
    }).
    Select(pet.FieldName).
    Strings(ctx)
```

#### Order by JSON fields

The [`sqljson`](https://pkg.go.dev/entgo.io/ent/dialect/sql/sqljson) package allows to easily sort data based on the value of a JSON object:

#### By Value

```markdown
users := client.User.Query().
    Order(
        sqljson.OrderValue(user.FieldData, sqljson.Path("key1", "key2")),
    ).
    AllX(ctx)
```

#### By Length

```markdown
users := client.User.Query().
    Order(
        sqljson.OrderLen(user.FieldData, sqljson.Path("key1", "key2")),
    ).
    AllX(ctx)
```

#### Descending

```markdown
users := client.User.Query().
    Order(
        sqljson.OrderValueDesc(user.FieldData, sqljson.Path("key1", "key2")),
    ).
    AllX(ctx)

pets := client.Pet.Query().
    Order(
        sqljson.OrderLenDesc(pet.FieldData, sqljson.Path("key1", "key2")),
    ).
    AllX(ctx)
```

> [!-info] -info
> PostgreSQL limitation on `ORDER BY` expressions with `SELECT DISTINCT`
> 
> PostgreSQL does not support `ORDER BY` expressions with `SELECT DISTINCT`. Thus, the `Unique` modifier should be set to `false`. However, keep in mind that this may result in duplicate results when performing graph traversals.
> 
> ```markdown
> users := client.User.Query().
>     Order(
>         sqljson.OrderValue(user.FieldData, sqljson.Path("key1", "key2")),
>     ).
> +   Unique(false).
>     AllX(ctx)
> ```
