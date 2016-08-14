% Compilazione condizionale

Rust ha un attributo speciale, `#[cfg]`, che permette di compilare del codice
a seconda di un'opzione passata al compilatore. Ha due forme:

```rust
#[cfg(foo)]
# fn foo() {}

#[cfg(bar = "baz")]
# fn bar() {}
```

Ci sono anche alcune funzioni d'aiuto:

```rust
#[cfg(any(unix, windows))]
# fn foo() {}

#[cfg(all(unix, target_pointer_width = "32"))]
# fn bar() {}

#[cfg(not(foo))]
# fn not_foo() {}
```

Tali funzioni possono annidarsi arbitrariamente:

```rust
#[cfg(any(not(unix), all(target_os="macos", target_arch = "powerpc")))]
# fn foo() {}
```

Per abilitare o disabilitare questi interruttori, se si usa Cargo,
li si imposta nella [sezione `[features]`][features] del file `Cargo.toml`:

[features]: http://doc.crates.io/manifest.html#the-features-section

```toml
[features]
# Di default, nessuna feature
default = []

# Qui si aggiunge la feature "foo", per poterla usare dopo.
# La nostra feature "foo" non dipende da nient'altro.
foo = []
```

Quando lo si fa, Cargo passa un'opzione a `rustc`:

```text
--cfg feature="${feature_name}"
```

La somma di queste opzioni `cfg` determinerà quali vengono attivate, e quindi,
quale codice viene compilato. Prendiamo questo codice:

```rust
#[cfg(feature = "foo")]
mod foo {
}
```

Se lo compiliamo con `cargo build --features "foo"`, Cargo manderà l'opzione
`--cfg feature="foo"` a `rustc`, e l'output conterrà il modulo `foo`.
Se lo compiliamo con un normale `cargo build`, nessun'altra opzione verrà
passata, e quindi, il modulo `foo` non esisterà.

# cfg_attr

Se può anche impostare un altro attributo basato su una variabile `cfg`
usando `cfg_attr`:

```rust
#[cfg_attr(a, b)]
# fn foo() {}
```

Sarò lo stesso di `#[b]` se `a` è impostato dall'attributo `cfg`, e niente
altrimenti.

# cfg!

L'[estensione sintattica][compilerplugins] `cfg!` permette di usare
questo genere di opzioni anche altrove nel codice:

```rust
if cfg!(target_os = "macos") || cfg!(target_os = "ios") {
    println!("Think Different!");
}
```

[compilerplugins]: compiler-plugins.html

Questi saranno sostituiti da un `true` o `false` in fase di compilazione,
a seconda delle impostazioni di configurazione.
