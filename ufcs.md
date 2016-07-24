% Sintassi universale di chiamata di funzione ["Universal Function Call Syntax" o "UFCS"]

Talvolta, più funzioni possono avere lo stesso nome. Si consideri
questo codice:

```rust
trait Foo {
    fn f(&self);
}

trait Bar {
    fn f(&self);
}

struct Baz;

impl Foo for Baz {
    fn f(&self) {
        println!("implementazione del tratto Foo per la struttura Baz");
    }
}

impl Bar for Baz {
    fn f(&self) {
        println!("implementazione del tratto Bar per la struttura Baz");
    }
}

let b = Baz;
```

Se provassimo a chiamare `b.f()`, otterremmo un errore:

```text
error: multiple applicable methods in scope [E0034]
b.f();
  ^~~
note: candidate #1 is defined in an impl of the trait `main::Foo` for the type
`main::Baz`
    fn f(&self) {
    ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
note: candidate #2 is defined in an impl of the trait `main::Bar` for the type
`main::Baz`
    fn f(&self) {
    ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

```

Ci serve un modo per disambiguare quale metodo scegliere. Questa caratteristica
è chiamata ‘sintassi universale di chiamata di funzione’, e si presenta così:

```rust
# trait Foo {
#     fn f(&self);
# }
# trait Bar {
#     fn f(&self);
# }
# struct Baz;
# impl Foo for Baz {
#     fn f(&self) {
#         println!("implementazione del tratto Foo per la struttura Baz");
#     }
# }
# impl Bar for Baz {
#     fn f(&self) { println!("Baz’s impl of Bar"); }
#         println!("implementazione del tratto Bar per la struttura Baz");
#     }
# }
# let b = Baz;
Foo::f(&b);
Bar::f(&b);
```

Scomponiamola.

```rust,ignore
Foo::
Bar::
```

Queste metà dell'invocazione sono i tipi dei due tratti: `Foo` e `Bar`.
Questo è quello che disambigua effettivamente fra le due funzioni: Rust chiama
quella del tratto specificato.

```rust,ignore
f(&b)
```

Quando si chiama un metodo come `b.f()` usando la [sintassi di metodo]
[sintassi di metodo], Rust automaticamente prenderà in prestito `b` se `f()`
prende l'argomento `&self`. In questo caso Rust non lo fa, e quindi
dobbiamo passare un esplicito `&b`.

[sintassi di metodo]: method-syntax.html

# Forma a parentesi angolari

La forma di UFCS di cui abbiamo appena parlato:

```rust,ignore
Tratto::metodo(argomenti);
```

È un'abbreviazione. Ne esiste una forma espansa che serve in alcune situazioni:

```rust,ignore
<Tipo as Tratto>::metodo(argomenti);
```

La sintassi `<>::` è un mezzo di fornire un suggerimento di tipo. Il tipo va
all'interno delle `<>`. In questo caso, il tipo è `Tipo as Tratto`, che
indica che qui vogliamo che sia chiamata la versione di `Tratto` del `metodo`.
La parte `as Tratto` è opzionale se non c'è ambiguità. È lo stesso con
le parentesi angolari, da cui la forma più breve.

Ecco un esempio di utilizzo della forma più lunga.

```rust
trait Foo {
    fn foo() -> i32;
}

struct Bar;

impl Bar {
    fn foo() -> i32 {
        20
    }
}

impl Foo for Bar {
    fn foo() -> i32 {
        10
    }
}

fn main() {
    assert_eq!(10, <Bar as Foo>::foo());
    assert_eq!(20, Bar::foo());
}
```

Usare la sintassi con le parentesi angolari consente di chiamate il metodo
del tratto, invece di quello definito per la struttura.
