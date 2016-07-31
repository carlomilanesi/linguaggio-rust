% Tipi non dimensionati

La maggior parte dei tipi hanno una dimensione particolare, in byte, che è
conoscibile in fase di compilazione.
Per esempio, un `i32` è grande trentadue bit, ossia quattro byte. Però, si sono
alcuni tipi che sono utili da esprimere, ma che non hanno una dimensione
definita in fase di compilazione. Questi tipi sono detti ‘non dimensionati’
o ‘dimensionati dinamicamente’. Un esempio è `[T]`. Questo tipo rappresenta
un certo numero di `T` in sequenza. Ma non sappiamo quanti ce ne sono, e quindi
la dimensione dell'oggetto non è nota.

Rust capisce alcuni di questi tipi, ma hanno alcune restrizioni.
Ce ne sono tre:

1. Un'istanza di un tipo non dipensionato può essere manipolata solamente
   tramite un puntatore. Un `&[T]` funziona bene, ma un `[T]` no.
2. Le variabili e gli argomenti di funzione non possono avere tipi dimensionati
   dinamicamente.
3. Solamente l'ultimo campo di una `struct` può avere un tipo dimensionato
   dinamicamente; gli altri campi no. Le varianti delle enumerazioni non
   possono avere come dati dei tipi dimensionati dinamicamente.

Quindi che fastidio danno? Beh, siccome `[T]` può essere usato dietro
un puntatore, se non avessimo un supporto linguistico ai tipi non dimensionati,
sarebbe impossibile scrivere questo:

```rust,ignore
impl Foo for str {
```

o

```rust,ignore
impl<T> Foo for [T] {
```

Invece, dovresti scrivere:

```rust,ignore
impl Foo for &str {
```

Intendendo che questa implementazione funzionerebbe solamente
per dei [riferimenti][ref], e non per altri tipi di puntatori.
Con l'`impl for str`, tutti i puntatori, compresi (in futuro, per adesso
ci sono dei difetti da correggere) gli smart pointer personalizzati definiti
dall'utente, possono usare questa `impl`.

[ref]: references-and-borrowing.html

# ?Sized

Se si vuole scrivere una funzione che accetta un tipo dimensionato
dinamicamente, si può usare la spaciale sintassi legata, `?Sized`:

```rust
struct Foo<T: ?Sized> {
    f: T,
}
```

Questo `?Sized` va letto come “T può essere oppure no `Sized`”, il che ci
consente di accettare sia tipi dimensionati che non dimensionati. Tutti
i parametri generici di tipo implicitamente sono legati a `Sized`, e quindi
`?Sized` può essere usato per uscire dal legame implicito.
