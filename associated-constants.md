% Le costanti associate

Con la caratteristica delle `associated_consts`, si possono definire
costanti così:

```rust
#![feature(associated_consts)]

trait Foo {
    const ID: i32;
}

impl Foo for i32 {
    const ID: i32 = 1;
}

fn main() {
    assert_eq!(1, i32::ID);
}
```

Qualunque implementatore di `Foo` dovrà definire `ID`. Senza la definizione:

```rust,ignore
#![feature(associated_consts)]

trait Foo {
    const ID: i32;
}

impl Foo for i32 {
}
```

si ottiene:

```text
error: not all trait items implemented, missing: `ID` [E0046]
     impl Foo for i32 {
     }
```

Si può implementare anche un valore di default:

```rust
#![feature(associated_consts)]

trait Foo {
    const ID: i32 = 1;
}

impl Foo for i32 {
}

impl Foo for i64 {
    const ID: i32 = 5;
}

fn main() {
    assert_eq!(1, i32::ID);
    assert_eq!(5, i64::ID);
}
```

Come si vede, quando si implementa `Foo`, si può lasciare la costante
non implementata, come per `i32`. In tal caso si userà il valore di default.
Ma, come per `i64`, si può anche aggiungere una ridefinizione.

Le costanti associate non devono necessariamente essere associate a un tratto.
Un blocco `impl` funziona bene anche per una `struct` o una `enum`:

```rust
#![feature(associated_consts)]

struct Foo;

impl Foo {
    const FOO: u32 = 3;
}
```
