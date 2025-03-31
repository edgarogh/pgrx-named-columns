# `pgrx-named-columns`

The **`pgrx`** Rust crate ([github](https://github.com/pgcentralfoundation/pgrx) · [crates](https://crates.io/crates/pgrx) · [docs](https://docs.rs/pgrx/latest/pgrx/)) is a really nice library to develop PostgreSQL extensions in Rust. Given the following Rust code:

```rust
const ALPHABET: &str = "ABCDEFGHIJKLMNOPQRSTUVWXYZ";

#[pg_extern]
fn alphabet(length: i8) -> impl Iterator<Item=(
    name!(idx, i8),
    name!(letter, char),
)> {
    ALPHABET
        .chars()
        .take(length.clamp(0, 25) as usize)
        .enumerate()
        .map(|(i, l)| (i as i8, l))
}
```

...you can use the `alphabet` function inside your database.

```sql
select alphabet(8);
```

Note how the column names are defined in Rust, line 3 and 4, using an inert declarative macro. There is currently no other way to define them. This is a problem for 2 reasons :

* Column names cannot easily be reused accross two different function that are expected to return the same value. i.e. you can't easily create an `alphabet_reverse` function that returns the exact same columns without copy-pasting code. This become a big problem when you don't have 2, but 50 columns.
* It isn't clear at first glance which value of the returned tuples corresponds to which column name. If your tuple contains a lot of columns of the same type, it's incredibly easy to mix them up.

These problems could easily be solved by using a `struct` as the `impl Iterator`'s item, but due to the way procedural macros work, they cannot access type-level information when they run. The only proper way to solve this would be a complete redesign of `pgrx`, which I cannot do. I opened [pgrx/issues#451](https://github.com/pgcentralfoundation/pgrx/issues/451).

Hence, the creation of this library : using filthy procedural macro hacks, including having the macro open a Rust file twice to read data outside the item it is applied to, it makes it possible to use a structure as returned rows. There are millions of way this can fail due to how badly implemented it is, but it should work for the general use-case. Here's how it looks :

```rust
const ALPHABET: &str = "ABCDEFGHIJKLMNOPQRSTUVWXYZ";

struct IndexedLetter {
    idx: i8,
    letter: char,
}

#[pg_extern_columns("path/to/current/file.rs")]
fn alphabet(length: i8) -> ::pgrx::iter::TableIterator<'static, IndexedLetter> {
    ALPHABET
        .chars()
        .take(length.clamp(0, 25) as usize)
        .enumerate()
        .map(|(idx, letter)| IndexedLetter {
            idx: idx as _,
            letter,
        })
}
```

_The path in the attribute parameters is probably the ugliest aspect of the macro, it is used to find the definition of `IndexedLetter`._
