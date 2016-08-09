% Gli attributi

In Rust le dichiarazioni possono essere annotate con ‘attributi’.
Si presentano così:

```rust
#[test]
# fn foo() {}
```

o così:

```rust
# mod foo {
#![test]
# }
```

La differenza fra i due è il `!`, che cambia ciò a cui l'attributo si applica:

```rust,ignore
#[foo]
struct Foo;

mod bar {
    #![bar]
}
```

L'attributo `#[foo]` si applica all'elemento che lo segue, che è
la dichiarazione di `struct`. Invece l'attributo `#![bar]` si applica
all'elemento che lo racchiude, che è la dichiarazione di `mod`. Per gli altri
aspetti, sono equivalenti. Entrambi modificano in qualche modo il significato
dell'elemento a cui si applicano.

Per esempio, si consideri una funzione come questa:

```rust
#[test]
fn check() {
    assert_eq!(2, 1 + 1);
}
```

È marcata da `#[test]`. Ciò significa che è speciale: quando si eseguono i
[test][test], questa funzione verrà eseguita. Quando si compila come al solito,
non verrà nemmeno inclusa. Questa funzione adesso è una funzione di collaudo.

[test]: testing.html

Gli attributi possono anche avere dati aggiuntivi:

```rust
#[inline(always)]
fn super_fast_fn() {
# }
```

O perfino chiavi e valori:

```rust
#[cfg(target_os = "macos")]
mod macos_only {
# }
```

Gli attributi di Rust servono a varie cose diverse. C'è una lista completa
degli attributi [nel riferimento][riferimento]. Attualmente, non è consentito
creare i propri attributi; li definisce solamente il compilatore Rust.

[riferimento]: ../reference.html#attributes
