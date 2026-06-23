+++
title = "Create a new type"
template = "documentation.html"
weight = 1
+++

To add a new type conversion, you need to implement two traits:
`elephantry::ToSql` and `elephantry::FromSql` which convert a rust value to its
postgresql representation and vis versa.

```rust
pub trait FromSql: Sized {
    fn from_text(ty: &elephantry::pq::Type, raw: Option<&str>) -> elephantry::Result<Self>;
    fn from_binary(ty: &elephantry::pq::Type, raw: Option<&[u8]>) -> elephantry::Result<Self>;
}

pub trait ToSql {
    fn ty(&self) -> elephantry::pq::Type;
    fn to_text(&self) -> elephantry::Result<Option<String>>;
    fn to_binary(&self) -> elephantry::Result<Option<Vec<u8>>>;
}
```

Both traits have text and binary versions[^1].

For this tutorial, we’ll implement step by step these traits to convert the
`ltree` type. According to the [postgresql
documentation](https://www.postgresql.org/docs/current/ltree.html), `ltree` is:

> A label path is a sequence of zero or more labels separated by dots, for
> example L1.L2.L3, representing a path from the root of a hierarchical tree to
> a particular node. The length of a label path cannot exceed 65535 labels.
>
> Example: Top.Countries.Europe.Russia

The first step is to determine the best rust representation for this type. Here
I choose a custom type that contains a `Vec<String>`:

```rust
#[derive(Default)]
struct Ltree(Vec<String>);
```

Why? First, `Vec<T>` already implements these traits where `T` implements its
(and `String` does). Second, the `Vec` type has similar operators to `ltree`.

Now we need to implement traits.

A good place to start is with the server-side C implementation, of these
functions. For the core types, you can find code in `src/backend/utils/adt`, but
here it’s an extension and code can be found in
[`contrib/ltree/ltree_io.c`](https://github.com/postgres/postgres/blob/REL_14_STABLE/contrib/ltree/ltree_io.c).

On the postgres side, we also have 4 functions:

- `*_in` → `to_text`;
- `*_out` → `from_text`;
- `*_recv` → `to_binary`;
- `*_send` → `from_binary`.

The text version is the easier to implement, as long as you understand C…

# Type information

Before implementing the conversion, we define your type information, the
`ToSql::ty()` function.

This information is used for query parameters and error messages.

All built-in types have this information in [postgresql
sources](https://github.com/postgres/postgres/blob/REL_14_STABLE/src/include/catalog/pg_type.dat).

Your type comes with an extension and we need to create this information. By
default, `elephantry::pq::types::UNKNOWN` is a good placeholder:

```rust
fn ty(&self) -> elephantry::pq::Type {
    elephantry::pq::Type {
        descr: "LQUERY - data type for hierarchical tree-like structures",
        name: "lquery",

        ..elephantry::pq::types::UNKNOWN
    }
}
```

# Text conversion

Converting a `Ltree` to the postgresql text representation consists of
concatenating all elements with dots:

```sql
select 'Top.Countries.Europe.Russia'::ltree;
```

```rust
fn to_text(&self) -> elephantry::Result<Option<String>> {
    Ok(Some(self.0.join(".")))
}
```

This function returns an `Option` for null value, we always returns `Some`, this
case is threatened by the `Option` conversion implementations.

If an error occurs during the conversion, you can use the `ToSql::error()`
function to return an `elephantry::Result`.

To reduce the boitelplate, you can delegate the `String` conversion to the
appropriate trait implementations:

```rust
fn to_text(&self) -> elephantry::Result<Option<String>> {
    self.0.join(".").to_text()
}
```

Well, as you can imagine the `from_text` is the opposite:

```rust
fn from_text(ty: &elephantry::pq::Type, raw: Option<&str>) -> elephantry::Result<Self> {
    // 1
    let s = String::from_text(ty, raw)?;

    // 2
    let ltree = if s.is_empty() {
        Self::default()
    } else {
        // 3
        Self(s.split('.').map(ToString::to_string).collect())
    };

    Ok(ltree)
}
```

1. First, we need to convert the `raw` value into string. This deals with null
   value, so if `raw` is `null` here, it’s an error (the user should use the
   `Option<Ltree>` type;
2. Second, we need to deal with empty string here, because splitting its made an
   one array element;
3. Finally, just split the string and convert it to `String`.

# Binary conversion

Well, if you arrive here, you know all you need to know about implementing
conversion. The second part is to do the same thing for the binary
format and the hardest part is to understanding the C code, again…

Here the [function
comment](https://github.com/postgres/postgres/blob/REL_14_STABLE/contrib/ltree/ltree_io.c#L188-L195)
is useful and clearly explains how the value is sent:

```C
/*
 * ltree type send function
 *
 * The type is sent as text in binary mode, so this is almost the same
 * as the output function, but it's prefixed with a version number so we
 * can change the binary format sent in future if necessary. For now,
 * only version 1 is supported.
 */
```

If there is no comment, you can look for `pg_send*` calls in the code. Here:

```C
pq_sendint8(&buf, version);
pq_sendtext(&buf, res, strlen(res));
```

Postgresql sends:

- an `i8`: the binary format version (`1`);
- a `String`: the deparse version of the ltree (like the text version).

```rust
fn to_binary(&self) -> elephantry::Result<Option<Vec<u8>>> {
    let mut buf = vec![1];
    buf.extend_from_slice(&self.0.join(".").into_bytes());

    Ok(Some(buf))
}
```

Finally, to convert a value from postgresql to rust, it’s similar. Just skip
the version number and the leading string can be parsed as the text version:

```rust
fn from_binary(ty: &elephantry::pq::Type, raw: Option<&[u8]>) -> elephantry::Result<Self> {
    let mut buf = elephantry::from_sql::not_null(raw)?;

    let _version = elephantry::from_sql::read_i8(&mut buf)?;
    let s = String::from_binary(ty, Some(buf))?;
    Self::from_text(ty, Some(&s))
}
```

You can retreive the complete code
[here](https://github.com/elephantry/elephantry/blob/4.0.0/core/src/sql/ltree/mod.rs).

# Entity

Well, now your type implements both `elephantry::FromSql` and
`elephantry::ToSql` traits, we can implement the empty trait
`elephantry::entity::Simple`:

```rust
impl elephantry::entity::Simple for Ltree {}
```

This trait implements the `elephantry::Entity` for you and allow using your type
as a member to another entity.

# FromText/ToText

If your type have the same representation as a string, in text **and** binary
formats, you can use the `elephantry::FromText` and `elephantry::ToText` to
simplify the implementation.

# Tests

Ideally, you should start by writing test for your new type, but it’s easier to
understand now we saw the different parts of the implementation.

With the `testing` features, you get the `convertion!` macro to write test:

```rust
#[cfg(test)]
mod test {
    elephantry::testing::convertion! {
        fixture: "test",
        sql_type: ltree,
        rust_type: crate::Ltree,
        tests: [
            ("''", crate::Ltree::default()),
            (
                "'Top.Countries.Europe.Russia'",
                crate::Ltree(vec![
                    "Top".to_string(),
                    "Countries".to_string(),
                    "Europe".to_string(),
                    "Russia".to_string()
                ])
            ),
        ],
    }
}
```

You can optionally add a fixture, by specified its path or just a name and the
file should be located at `fixtures/{name}.sql`.

I hope `sql_type` and `rust_type` parameters are transparent.

`tests` is an array of tuples with the SQL value and the rust version.

This macro execute these queries and check if the result is the same after
converting in rust:

```rust
// from_text
connection.execute::<{rust_type}>("select {value}::{sql_type}");

// from_binary
connection.query::<{rust_type}>("select {value}::{sql_type}");

// to_text
connection.mode = elephantry::pq::Type::Text;
connection.query::<{rust_type}>("select $1::{sql_type}", &[value]);

// to_binary
connection.mode = elephantry::pq::Type::Binary;
connection.query::<{rust_type}>("select $1::{sql_type}", &[value]);
```

Tests are overlapped, it could be a little tricky if you don’t know which
function to start with.

I recommend following this order:

- `from_text`;
- `from_binary`;
- `to_text`;
- `to_binary`.

You can valid tests one after another, and text format is easier to implement,
before writing the binary one.

You can retrieve the complete code of this tutorial in the
[14-new-type.rs](https://github.com/elephantry/elephantry/blob/5.0.0/core/examples/14-new-type.rs) example.

---

[^1]: `ToSql::to_binary` is used for `Connection::copy` function,
    `ToSql::to_text` for all other query params. `FromSql::from_text` is used
    for `Connection::execute` and `FromSql::from_binary` for
    `Connection::query`.
