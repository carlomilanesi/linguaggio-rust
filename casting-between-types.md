% Conversione fra tipi

Rust, concentrandosi sulla sicurezza, fornisce due diversi modi di convertire
un valore in un valore di tipo diverso. Il primo, `as`, è per conversioni
sicure. Invece, `transmute` consente conversioni arbitrarie, ed è una delle
caratteristiche più pericolose di Rust!

# La forzatura

La forzatura tra tipi è implicita e non ha una sua sintassi, ma può essere
esplicitata usando [`as`](#explicit-coercions).

La forzatura può avvenire nelle istruzioni `let`, `const`, e `static`; negli
argomenti delle chiamate di funzione; nei valori dei campi
nell'inizializzazione di struct; e nei risultati di funzioni.

Il caso più tipico di forzatura è la rimozione della mutabilità
da un riferimento:

 * `&mut T` to `&T`

Un'analoga conversione si ha per rimuovere la mutabilità
da un [puntatore grezzo](raw-pointers.md):

 * `*mut T` to `*const T`

I riferimenti possono anche essere forzati in puntatori grezzi:

 * `&T` to `*const T`

 * `&mut T` to `*mut T`

Si possono definire delle forzature personalizzate usando
[`Deref`](deref-coercions.md).

Le forzature sono transitive.

# `as`

La parola-chiave `as` esegue una conversione sicura:

```rust
let x: i32 = 5;

let y = x as i64;
```

Ci sono tre categorie principali di conversioni sicure: le forzature esplicite,
le conversioni tra tipi numerici, e le conversioni tra puntatori.

Le conversioni non sono transitive: anche se `e as U1 as U2` è un'espressione
valida, `e as U2` non lo è necessariamente (di fatto è valida solamente se 
`U1` può essere forzata a `U2`).

## Forzature esplicite

Una conversione `e as U` è valida se `e` ha tipo `T` e `T` *può essere forzato*
a `U`.

## Conversioni numeriche

Una conversione `e as U` è pure valida in ognuno dei seguenti casi:

 * `e` è di tipo `T`, e `T` e `U` sono tipi numerici qualunque; *numeric-cast*
 * `e` è un enum in stile C (cioò senza dati allegati alle varianti),
    e `U` è un tipo intero; *enum-cast*
 * `e` è di tipo `bool` o `char`, e `U` è un tipo intero; *prim-int-cast*
 * `e` è di tipo `u8`, e `U` è `char`; *u8-char-cast*

Per esempio

```rust
let one = true as u8;
let at_sign = 64 as char;
let two_hundred = -56i8 as u8;
```

La semantica delle conversioni numeriche è la seguente:

* La conversione fra due interi della stessa dimensione (per es. i32 -> u32)
  è una no-op (cioè non fa niente)
* La conversione da un intero più grande a un intero più piccolo (per es.
  u32 -> u8) troncherà
* La conversione da un intero più piccolo a un intero più grande (per es.
  u8 -> u32)
    * estenderà con zeri se l'originale è senza segno
    * estenderà con bit del segno se l'originale è con segno
* La conversione da un numero a virgola mobile a un intero arrotonderà
  un numero a virgola mobile verso lo zero
    * **[NOTA: attualmente ciò avrà comportamento indefinito se il valore
      arrotondato non può essere rappresentato dal tipo intero obiettivo]
      [float-int]**. Sono compresi Inf e NaN. Questo difetto dovrà essere
      corretto.
* La conversione da un intero a un numero a virgola mobile produrrà
  la rappresentazione a virgola mobile dell'intero, arrotondato se necessario
  (la strategia di arrotondamento non è specificata)
* La conversione da un f32 a un f64 è perfetta e senza perdite di precisione
* La conversione da un f64 a un f32 produrrà il valore più vicino possibile
  (la strategia di arrotondamento non è specificata)
    * **[NOTA: attualmente ciò avrà comportamento indefinito se il valore è
      finito ma più grande o più piccolo del valore finito più grande o più
      piccolo rappresentabile da f32][float-float]**. Questo difetto dovrà
      essere corretto.

[float-int]: https://github.com/rust-lang/rust/issues/10184
[float-float]: https://github.com/rust-lang/rust/issues/15536

## Conversione di puntatori

Forse sorprenderà qualcuno, ma è sicuro convertire [puntatori grezzi]
(raw-pointers.md) in interi e interi in puntatori grezzi, e convertire
fra puntatori che puntano a tipi diversi pur di rispettare alcuni vincoli.
È insicuro solamente dereferenziare il puntatore:

```rust
let a = 300 as *const char; // un puntatore alla posizione 300
let b = a as u32;
```

`e as U` è una valida conversione di puntatore in ognuno dei seguenti casi:

* `e` è di tipo `*T`, `U` è `*U_0`, e o vale `U_0: Sized` o
vale `unsize_kind(T) == unsize_kind(U_0)`; è un *ptr-ptr-cast*

* `e` è di tipo `*T` e `U` è un tipo numerico, mentre vale `T: Sized`;
  è un *ptr-addr-cast*

* `e` è un intero e `U` è `*U_0`, mentre vale `U_0: Sized`; è un *addr-ptr-cast*

* `e` è di tipo `&[T; n]` e `U` è `*const T`; è un *array-ptr-cast*

* `e` è un puntatore a funzione e `U` è `*T`,
  mentre `T: Sized`; è un *fptr-ptr-cast*

* `e` è un puntatore a funzione e `U` è un tipo intero; è un *fptr-addr-cast*


# La funzione `transmute`

`as` consente solamente conversioni sicure, e per esempio respngerà
un tentativo di convertire quattro bye in un `u32`:

```rust,ignore
let a = [0u8, 0u8, 0u8, 0u8];

let b = a as u32; // quattro u8 fanno un u32
```

Ciò produrrà questo errore:

```text
error: non-scalar cast: `[u8; 4]` as `u32`
let b = a as u32; // quattro u8 fanno un u32
        ^~~~~~~~
```

Questa è una conversione ’non scalare’, perché qui abbiamo più valori:
i quattro elementi dell'array. Questi generi di conversioni sono molto
pericolose, perché fanno assunzioni sul modo in cui più strutture soggiacenti
sono implementate. Per questo, ci serve qualcosa di più pericoloso.

La funzione `transmute` è fornita da un [intrinseco del compilatore]
[intrinseci], e quello che fa è molto semplice, ma molto pauroso. Dice a Rust
di trattare un valore di un tipo come se fosse di un altro tipo. Lo fa senza
riguardo per il sistema di verifica dei tipi, e si fida completamente
del programmatore.

[intrinseci]: intrinsics.html

Nell'esempio precedente, sappiamo che un array di quattro `u8` rappresenta
appropriatamente un `u32`, e quindi vogliamo fare la conversione. Usando
`transmute` invece di `as`, Rust ce lo lascia fare:

```rust
use std::mem;

fn main() {
    unsafe {
        let a = [0u8, 1u8, 0u8, 0u8];
        let b = mem::transmute::<[u8; 4], u32>(a);
        println!("{}", b); // 256
        // o, più concisamente:
        let c: u32 = mem::transmute(a);
        println!("{}", c); // 256
    }
}
```

Dobbiamo avvolgere l'operazione in un blocco `unsafe` affinché compili
con successo. Tecnicamente, solamente la chiamata `mem::transmute` stessa ha
bisogno di essere nel blocco, ma in questo caso è carino racchiudere tutte
le cose correlate, così da saper dove guardare. In questo caso, anche
i dettagli su `a` sono importanti, e quindi sono nel blocco. Capiterà di vedere
del codice in entrambi gli stili; talvolta il contesto è troppo lontano, e
avvolgere tutto il codice in `unsafe` non è un'ottima idea.

Per quanto `transmute` faccia pochissime verifiche, almeno si assicura che
i tipi siano della stessa dimensione. Questa codice:

```rust,ignore
use std::mem;

unsafe {
    let a = [0u8, 0u8, 0u8, 0u8];

    let b = mem::transmute::<[u8; 4], u64>(a);
}
```

dà il seguente errore:

```text
error: transmute called with differently sized types: [u8; 4] (32 bits) to u64
(64 bits)
```

A parte quello, ci si deve arrangiare!
