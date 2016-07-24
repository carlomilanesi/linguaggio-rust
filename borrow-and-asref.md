% Borrow e AsRef

I tratti [`Borrow`][borrow] e [`AsRef`][asref] sono molto simili, ma diversi.
Ecco un rapido ripasso di quello che questi due tratti significano.

[borrow]: ../std/borrow/trait.Borrow.html
[asref]: ../std/convert/trait.AsRef.html

# Borrow

Il tratto `Borrow` si usa quando si sta scrivendo una struttura dati,
e si vuole usare un tipo o posseduto o preso in prestito come sinonimo,
per qualche scopo.

Per esempio, [`HashMap`][hashmap] ha un metodo [`get`][get] che usa `Borrow`:

```rust,ignore
fn get<Q: ?Sized>(&self, k: &Q) -> Option<&V>
    where K: Borrow<Q>,
          Q: Hash + Eq
```

[hashmap]: ../std/collections/struct.HashMap.html
[get]: ../std/collections/struct.HashMap.html#method.get

Questa firma è parecchio complicata. Quello che ci interessa qui è
il parametro `K`. Si riferisce a un parametro di `HashMap` stesso:

```rust,ignore
struct HashMap<K, V, S = RandomState> {
```

The `K` parameter is the type of _key_ the `HashMap` uses. So, looking at
the signature of `get()` again, we can use `get()` when the key implements
`Borrow<Q>`. That way, we can make a `HashMap` which uses `String` keys,
but use `&str`s when we’re searching:

```rust
use std::collections::HashMap;

let mut map = HashMap::new();
map.insert("Foo".to_string(), 42);

assert_eq!(map.get("Foo"), Some(&42));
```

This is because the standard library has `impl Borrow<str> for String`.

For most types, when you want to take an owned or borrowed type, a `&T` is
enough. But one area where `Borrow` is effective is when there’s more than one
kind of borrowed value. This is especially true of references and slices: you
can have both an `&T` or a `&mut T`. If we wanted to accept both of these types,
`Borrow` is up for it:

```rust
use std::borrow::Borrow;
use std::fmt::Display;

fn foo<T: Borrow<i32> + Display>(a: T) {
    println!("a is borrowed: {}", a);
}

let mut i = 5;

foo(&i);
foo(&mut i);
```

This will print out `a is borrowed: 5` twice.

# AsRef

The `AsRef` trait is a conversion trait. It’s used for converting some value to
a reference in generic code. Like this:

```rust
let s = "Hello".to_string();

fn foo<T: AsRef<str>>(s: T) {
    let slice = s.as_ref();
}
```

# Which should I use?

We can see how they’re kind of the same: they both deal with owned and borrowed
versions of some type. However, they’re a bit different.

Choose `Borrow` when you want to abstract over different kinds of borrowing, or
when you’re building a datastructure that treats owned and borrowed values in
equivalent ways, such as hashing and comparison.

Choose `AsRef` when you want to convert something to a reference directly, and
you’re writing generic code.
