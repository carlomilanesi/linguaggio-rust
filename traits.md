% Tratti ["trait"]

Un tratto è una caratteristica del linguaggio che dice al compilatore Rust
quali funzionalità un tipo deve fornire.

Ricordiamo la parola-chiave `impl`, usata per chiamare una funzione con
la [sintassi dei metodi][methodsyntax]:

```rust
struct Cerchio {
    x: f64,
    y: f64,
    raggio: f64,
}

impl Cerchio {
    fn area(&self) -> f64 {
        std::f64::consts::PI * (self.raggio * self.raggio)
    }
}
```

[methodsyntax]: method-syntax.html

I tratti sono simili, eccetto che dapprima si definisce un tratto con
una firma di metodo, e poi si implementa il tratto per un tipo. In questo
esempio, implementiamo il tratto `HaArea` per `Cerchio`:

```rust
struct Cerchio {
    x: f64,
    y: f64,
    raggio: f64,
}

trait HaArea {
    fn area(&self) -> f64;
}

impl HaArea for Cerchio {
    fn area(&self) -> f64 {
        std::f64::consts::PI * (self.raggio * self.raggio)
    }
}
```

Come si può vedere, il blocco `trait` appare molto simile al blocco `impl`,
ma non contiene il corpo della funzione, solamente una firma di tipo.
Quando si implementa un tratto, si usa la formula `impl Trait for Item`,
invece della più semplice `impl Item`.

## Legami del tratto sulle funzioni generiche

I tratti sono utili perché consentono a un tipo di fare certe promesse sul suo
comportamento. Le funzioni generiche possono sfruttare questo per vincolare, o
[legare][legami], i tipi che accettano. Si consideri questa funzione,
che non compila:

[legami]: glossary.html#bounds

```rust,ignore
fn stampa_area<T>(figura: T) {
    println!("Questa figura ha un'area di {}", figura.area());
}
```

Rust si lamenta:

```text
error: no method named `area` found for type `T` in the current scope
```

Siccome `T` può essere qualunque tipo, non possiamo essere sicuri
che implementi il metodo `area`. Ma possiamo aggiungere un tratto legato
al nostro `T` generico, assicurando che lo faccia:

```rust
# trait HaArea {
#     fn area(&self) -> f64;
# }
fn stampa_area<T: HaArea>(figura: T) {
    println!("Questa figura ha un'area di {}", figura.area());
}
```

La sintassi `<T: HaArea>` significa “qualunque tipo che implementa il tratto
`HaArea`.” Siccome i tratti definiscono delle firme di tipo di funzione,
possiamo star sicuri che qualunque tipo che implementa `HaArea` avrà un metodo
`.area()`.

Ecco un esempio esteso di come funziona questa cosa:

```rust
trait HaArea {
    fn area(&self) -> f64;
}

struct Cerchio {
    x: f64,
    y: f64,
    raggio: f64,
}

impl HaArea for Cerchio {
    fn area(&self) -> f64 {
        std::f64::consts::PI * (self.raggio * self.raggio)
    }
}

struct Quadrato {
    x: f64,
    y: f64,
    lato: f64,
}

impl HaArea for Quadrato {
    fn area(&self) -> f64 {
        self.lato * self.lato
    }
}

fn stampa_area<T: HaArea>(figura: T) {
    println!("Questa figura ha un'area di {}", figura.area());
}

fn main() {
    let c = Cerchio {
        x: 0.0f64,
        y: 0.0f64,
        raggio: 1.0f64,
    };

    let q = Quadrato {
        x: 0.0f64,
        y: 0.0f64,
        lato: 1.0f64,
    };

    stampa_area(c);
    stampa_area(q);
}
```

Questo programma emette:

```text
Questa figura ha un'area di 3.141593
Questa figura ha un'area di 1
```

Da come si vede, `stampa_area` adesso è generica, ma assicura anche che le
abbiamo passato i tipi corretti. Se le passiamo un tipo scorretto:

```rust,ignore
stampa_area(5);
```

Otteniamo un errore in fase di compilazione:

```text
error: the trait bound `_ : HasArea` is not satisfied [E0277]
```

## Legami di tratto su struct generiche

Anche le proprie struct generiche possono trarre beneficio dai legami
dei tratti. L'unica cosa da fare è attaccare il legame quando si dichiarano
i parametri di tipo. Ecco un nuovo tipo `Rettangolo<T>` e la sua operazione
`e_quadrato()`:

```rust
struct Rettangolo<T> {
    x: T,
    y: T,
    larghezza: T,
    altezza: T,
}

impl<T: PartialEq> Rettangolo<T> {
    fn e_quadrato(&self) -> bool {
        self.larghezza == self.altezza
    }
}

fn main() {
    let mut r = Rettangolo {
        x: 0,
        y: 0,
        larghezza: 47,
        altezza: 47,
    };

    assert!(r.e_quadrato());

    r.altezza = 42;
    assert!(!r.e_quadrato());
}
```

`e_quadrato()` ha bisogno di verificare che i lati siano uguali, perciò i lati
devono essere di un tipo che implementa il tratto
[`core::cmp::PartialEq`][PartialEq]:

```rust,ignore
impl<T: PartialEq> Rettangolo<T> { ... }
```

Adesso, un rettangolo può essere definito in termini di ogni tipo che può
essere confrontato per l'uguaglianza.

[PartialEq]: ../core/cmp/trait.PartialEq.html

Qui abbiamo definito una nuova struct `Rettangolo` che accetta dei numeri
di qualunque precisione — in realtà, oggetti quasi di qualunque tipo — purché
possano essere confrontati per l'uguaglianza. Potremmo fare lo stesso
per le nostre struct `HaArea`, cioè `Quadrato` e `Cerchio`? Sì, ma hanno
bisogno della moltiplicazione, e per lavorarci ci serve saperne di più
riguardo ai [tratti di operatore][operators-and-overloading].

[operators-and-overloading]: operators-and-overloading.html

# Regole per implementare i tratti

Finora, abbiamo aggiunto implementazioni di tratti solamente a delle struct,
ma un tratto può essere implementato per qualunque tipo. Perciò tecnicamente,
_potremmo_ implementare `HaArea` anche per il tipo `i32`:

```rust
trait HaArea {
    fn area(&self) -> f64;
}

impl HaArea for i32 {
    fn area(&self) -> f64 {
        println!("questo è sciocco");

        *self as f64
    }
}

5.area();
```

È considerato stile scadente implementare dei metodi su tali tipi primitivi,
anche se è possibile.

Questo può sembrare come il Far West, ma ci sono due restrizioni riguardo
l'implementazione dei tratti che prevengono che la cosa ci sfugga di mano.
La prima è che se il tratto non è definito nel nostro ambito, non si applica.
Ecco un esempio: la libreria standard fornisce un tratto [`Write`][write]
che aggiunge delle funzionalità ai `File`, per fare I/O su file. Di default,
un `File` non avrà i suoi metodi:

[write]: ../std/io/trait.Write.html

```rust,ignore
let mut f = std::fs::File::open("foo.txt").expect("Fallita apertura di foo.txt");
let buf = b"qualcosa"; // letterale di stringa di byte. buf: &[u8; 8]
let risultato = f.write(buf);
# risultato.unwrap(); // ignora l'errore
```

Ecco l'errore:

```text
error: type `std::fs::File` does not implement any method in scope named `write`
let result = f.write(buf);
               ^~~~~~~~~~
```

Dapprima dobbiamo importare il tratto `Write` con `use`:

```rust,ignore
use std::io::Write;

let mut f = std::fs::File::open("foo.txt").expect("Fallita apertura di foo.txt");
let buf = b"qualcosa"; // letterale di stringa di byte. buf: &[u8; 8]
let risultato = f.write(buf);
# result.unwrap(); // ignore the error
```

Questo compilerà senza errori.

Ciò significa che anche se qualcuno fa qualcosa di male come implementare
un tratto per `i32`, questo non ci toccherà, a meno che importiamo quel tratto.

C'è un'altra restrizione sull'implementare i tratti: o il tratto
o il tipo per cui lo stiamo implementando, devono essere definiti da noi.
O per meglio dire, almeno uno di essi deve essere definito nello stesso crate
in cui si trova l'`impl` che stiamo scrivendo. Per saperne di più sul sistema
dei moduli e dei pacchetti di Rust, si veda la sezione su [crate e moduli][cm].

Perciò, potremmo implementare il tratto `HasArea` per `i32`, dato che abbiamo
definito `HaArea` nel nostro codice. Ma se provassimo a implementare
`ToString`, un tratto fornito da Rust, per `i32`, non potremmo, perché né
il tratto né il tipo sono definiti nel nostro crate.

Un'ultima cosa sui tratti: le funzioni generiche con un legame di tratto usano
la ‘monomorfizzazione’ (dal greco "mono"="uno" e "morfo"="forma"), e quindi
sono smistati staticamente.
Che significa? Si guardi la sezione sugli [oggetti-tratto][to] per avere
maggiori dettagli.

[cm]: crates-and-modules.html
[to]: trait-objects.html

# Legami di tratto multipli

Abbiamo visto che si può legare un parametro generico di tipo a un tratto:

```rust
fn foo<T: Clone>(x: T) {
    x.clone();
}
```

Se serve più di un legame, si può usare `+`:

```rust
use std::fmt::Debug;

fn foo<T: Clone + Debug>(x: T) {
    x.clone();
    println!("{:?}", x);
}
```

`T` adesso ha bisogno di essere sia `Clone` che `Debug`.

# La clausola Where

Scrivere funzioni con solamente alcuni tipi generici e un piccolo numero
di legami di tratto non è malaccio, ma man mano che il loro numero si accresce,
la sintassi divenga sempre più goffa:

```rust
use std::fmt::Debug;

fn foo<T: Clone, K: Clone + Debug>(x: T, y: K) {
    x.clone();
    y.clone();
    println!("{:?}", y);
}
```

Il nome della funzione è all'estrema sinistra, e la lista degli argomenti è
all'estrema destra. I legami stanno diventando d'intralcio.

Rust ha una soluzione, e si chiama ‘clausola `where`’:

```rust
use std::fmt::Debug;

fn foo<T: Clone, K: Clone + Debug>(x: T, y: K) {
    x.clone();
    y.clone();
    println!("{:?}", y);
}

fn bar<T, K>(x: T, y: K) where T: Clone, K: Clone + Debug {
    x.clone();
    y.clone();
    println!("{:?}", y);
}

fn main() {
    foo("Ciao", "mondo");
    bar("Ciao", "mondo");
}
```

`foo()` usa la sintassi che abbiamo mostrato prima, e `bar()` usa una clausola
`where`. Si devono solo omettere i vincoli quando si definiscono i propri
parametri di tipo, e poi aggiungere `where` dopo l'elenco degli argomenti.
Per liste più lunghe, si possono aggiungere spaziature:

```rust
use std::fmt::Debug;

fn bar<T, K>(x: T, y: K)
    where T: Clone,
          K: Clone + Debug {

    x.clone();
    y.clone();
    println!("{:?}", y);
}
```

Questa flessibilità può aggiungere chiarezza un situazioni complesse.

`where` è anche più potente della sintassi più semplice. Per esempio:

```rust
trait ConvertTo<Output> {
    fn convert(&self) -> Output;
}

impl ConvertTo<i64> for i32 {
    fn convert(&self) -> i64 { *self as i64 }
}

// can be called with T == i32
fn normal<T: ConvertTo<i64>>(x: &T) -> i64 {
    x.convert()
}

// can be called with T == i64
fn inverse<T>(x: i32) -> T
        // this is using ConvertTo as if it were "ConvertTo<i64>"
        where i32: ConvertTo<T> {
    x.convert()
}
```

This shows off the additional feature of `where` clauses: they allow bounds
on the left-hand side not only of type parameters `T`, but also of types (`i32` in this case). In this example, `i32` must implement
`ConvertTo<T>`. Rather than defining what `i32` is (since that's obvious), the
`where` clause here constrains `T`.

# Default methods

A default method can be added to a trait definition if it is already known how a typical implementor will define a method. For example, `is_invalid()` is defined as the opposite of `is_valid()`:

```rust
trait Foo {
    fn is_valid(&self) -> bool;

    fn is_invalid(&self) -> bool { !self.is_valid() }
}
```

Implementors of the `Foo` trait need to implement `is_valid()` but not `is_invalid()` due to the added default behavior. This default behavior can still be overridden as in:

```rust
# trait Foo {
#     fn is_valid(&self) -> bool;
#
#     fn is_invalid(&self) -> bool { !self.is_valid() }
# }
struct UseDefault;

impl Foo for UseDefault {
    fn is_valid(&self) -> bool {
        println!("Called UseDefault.is_valid.");
        true
    }
}

struct OverrideDefault;

impl Foo for OverrideDefault {
    fn is_valid(&self) -> bool {
        println!("Called OverrideDefault.is_valid.");
        true
    }

    fn is_invalid(&self) -> bool {
        println!("Called OverrideDefault.is_invalid!");
        true // overrides the expected value of is_invalid()
    }
}

let default = UseDefault;
assert!(!default.is_invalid()); // prints "Called UseDefault.is_valid."

let over = OverrideDefault;
assert!(over.is_invalid()); // prints "Called OverrideDefault.is_invalid!"
```

# Inheritance

Sometimes, implementing a trait requires implementing another trait:

```rust
trait Foo {
    fn foo(&self);
}

trait FooBar : Foo {
    fn foobar(&self);
}
```

Implementors of `FooBar` must also implement `Foo`, like this:

```rust
# trait Foo {
#     fn foo(&self);
# }
# trait FooBar : Foo {
#     fn foobar(&self);
# }
struct Baz;

impl Foo for Baz {
    fn foo(&self) { println!("foo"); }
}

impl FooBar for Baz {
    fn foobar(&self) { println!("foobar"); }
}
```

If we forget to implement `Foo`, Rust will tell us:

```text
error: the trait bound `main::Baz : main::Foo` is not satisfied [E0277]
```

# Deriving

Implementing traits like `Debug` and `Default` repeatedly can become
quite tedious. For that reason, Rust provides an [attribute][attributes] that
allows you to let Rust automatically implement traits for you:

```rust
#[derive(Debug)]
struct Foo;

fn main() {
    println!("{:?}", Foo);
}
```

[attributes]: attributes.html

However, deriving is limited to a certain set of traits:

- [`Clone`](../core/clone/trait.Clone.html)
- [`Copy`](../core/marker/trait.Copy.html)
- [`Debug`](../core/fmt/trait.Debug.html)
- [`Default`](../core/default/trait.Default.html)
- [`Eq`](../core/cmp/trait.Eq.html)
- [`Hash`](../core/hash/trait.Hash.html)
- [`Ord`](../core/cmp/trait.Ord.html)
- [`PartialEq`](../core/cmp/trait.PartialEq.html)
- [`PartialOrd`](../core/cmp/trait.PartialOrd.html)
