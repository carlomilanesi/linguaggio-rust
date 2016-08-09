% I generici

Talvolta, quando si scrive una funzione o un tipo di dati, potremmo volere che
funzioni per più tipi di argomenti. In Rust, lo possiamo fare usando
i generici. Nella teoria dei tipi, i generici sono chiamati ‘polimorfismo
parametrico’, che significa che sono tipi o funzioni che hanno più forme
(in greco ‘poli’ significa ‘plurimo‘, e ‘morfo’ significa ‘forma‘) in base a
un dato parametro (da cui ‘parametrico’).

Comunque, basta così con la teoria dei tipi, guardiamo del codice generico.
La libreria standard di Rust fornisce un tipo, `Option<T>`, che è generico:

```rust
enum Option<T> {
    Some(T),
    None,
}
```

La parte `<T>`, che abbiamo già visto alcune volte, indica che questo è un tipo
di dati generico. Ogni volta che nel nostro codice usiamo questo `enum`,
specifichiamo un tipo che sostituisce il parametro `T` ogni volta che compare
nella dichiarazione generica. Ecco un esempio di uso di `Option<T>`,
con un'annotazione di tipo aggiuntiva:

```rust
let x: Option<i32> = Some(5);
```

Nella dichiarazione di tipo, diciamo `Option<i32>`. Si noti quanto simile
appaia a `Option<T>`. Quindi, in questa particolare `Option`, `T` ha il valore
di `i32`. Sul lato destro del legame, costruiamo un `Some(T)`, dove l'oggetto
di tipo `T` è `5`. Dato che si tratta di un `i32`, i due lati combaciano, e
Rust è contento. Se non combaciassero, otterremmo un errore:

```rust,ignore
let x: Option<f64> = Some(5);
// error: mismatched types: expected `core::option::Option<f64>`,
// found `core::option::Option<_>` (expected f64 but found integral variable)
```

Ciò non significa che non si possono costruire degli `Option<T>` che tengono
`f64`! Solamente i due lati dell'assegnamento devono avere lo stesso tipo:

```rust
let x: Option<i32> = Some(5);
let y: Option<f64> = Some(5.0f64);
```

Così va bene. Una sola definizione, utilizzi multipli.

I generici non sono limitati ad essere parametrizzati da un solo tipo.
Si consideri un altro tipo simile, fornito dalla liberia standard di Rust,
`Result<T, E>`:

```rust
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

Questo tipo è generico relativamente a _due_ tipi: `T` ed `E`. Tra l'altro,
le lettere maiuscole possono essere qualunque lettera si gradisca. Si sarebbe
potuto definire `Result<T, E>` come:

```rust
enum Result<A, Z> {
    Ok(A),
    Err(Z),
}
```

se si avesse voluto. Per convenzione il primo parametro generico dovrebbe
essere `T`, per ‘tipo’, e si dovrebbe usare `E` per ‘errore’. Però a Rust
non importa.

Il tipo `Result<T, E>` è pensato per essere usato come risultato
di un'elaborazione, dando la possibilità di rendere un errore nel caso
non si riuscisse a completare correttamente l'elaborazione.

## Funzioni generiche

Si possono scrivere funzioni che prendono tipi generici, usando una sintassi
simile:

```rust
fn prende_qualunque_cosa<T>(x: T) {
    // fai qualcosa con x
}
```

La sintassi ha due parti: la `<T>` dice “questa funzione è generica rispetto
a un tipo, `T`”, e la `x: T` dice “x è di tipo `T`.”

Più argomenti possono essere dello stesso tipo generico:

```rust
fn prende_due_oggetti_del_medesimo_tipo<T>(x: T, y: T) {
    // ...
}
```

Si può anche scrivere una versione che prende più tipi:

```rust
fn prende_due_oggetti<T, U>(x: T, y: U) {
    // ...
}
```

## Struct generiche

Si può usare un tipo generico anche per i campi di una `struct`:

```rust
struct Punto<T> {
    x: T,
    y: T,
}

let origine_intera = Punto { x: 0, y: 0 };
let origine_a_virgola_mobile = Punto { x: 0.0, y: 0.0 };
```

Analogamente alle funzioni, la `<T>` è dove si dichiarano i parametri generici,
che poi vengono usati nelle dichiarazioni dei campi `x: T` e `y: T`.

Quando si vuole aggiungere un'implementazione per una `struct` generica, si
dichiara il parametro di tipo subito dopo la `impl`:

```rust
# struct Punto<T> {
#     x: T,
#     y: T,
# }
#
impl<T> Punto<T> {
    fn swap(&mut self) {
        std::mem::swap(&mut self.x, &mut self.y);
    }
}
```

Finora abbiamo visto dei generici che prendono assolutamente qualunque tipo.
Questi servono in molti casi: abbiamo già visto `Option<T>`, e poi incontreremo
i tipi contenitore universali, come [`Vec<T>`][Vec]. D'altra parte, spesso si
vuole rinunciare a quella flessibilità per avere un maggior poter espressivo.
Si legga la sezione sui [legami di tratto][tratti] per vedere come e perché.

[tratti]: traits.html
[Vec]: ../std/vec/struct.Vec.html
