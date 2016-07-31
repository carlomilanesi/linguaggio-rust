% Mutabilità

La mutabilità, ossia la capacità di cambiare qualcosa, funziona in Rust un po'
diversamente che in altri linguaggi. Il primo aspetto della mutabilità in Rust
è il fatto di non esserci per default:

```rust,ignore
let x = 5;
x = 6; // errore!
```

Si può introdurre la mutabilità con la parola-chiave `mut`:

```rust
let mut x = 5;
x = 6; // non c'è problema!
```

Questo è un [legame di variabile][vb] mutabile. Quando un legame è mutabile,
significa che si può cambiare ciò a cui quel legame punta. Perciò nell'esempio
sopra, non è tanto che il valore alla posizione `x` cambia, quanto che
il legame passa dal puntare un `i32` al puntarne un altro.

[vb]: variable-bindings.html

Si può anche creare un [riferimento][ref] ad esso, usando `&x`, ma se si vuole
usare il riferimento per cambiarlo, ci vorrà un riferimento mutabile:

```rust
let mut x = 5;
let y = &mut x;
```

[ref]: references-and-borrowing.html

`y` è un legame immutabile a un riferimento mutabile, il che significa che non
si può legare 'y' a qualcos'altro (`y = &mut z`), ma `y` può essere usato
per legare `x` a qualcos'altro (`*y = 5`). Una distinzione sottile.

Naturalmente, se servono entrambi:

```rust
let mut x = 5;
let mut y = &mut x;
```

Adesso `y` può essere legato a un altro valore, e inoltre il valore che sta
referenziando può essere cambiato.

È importante notare che `mut` fa parte di un [pattern][pattern], e perciò
si possono fare cose come questa:

```rust
let (mut x, y) = (5, 6);

fn foo(mut x: i32) {
# }
```

Si noti che qui la `x` è mutabile, mentre la `y` non lo è.

[pattern]: patterns.html

# Mutabilità interiore contro mutabilità esteriore

Però, quando diciamo che qualcosa è ‘immutabile’ in Rust, non intendiamo che
è impossibile cambiarla: ci stiamo riferendo alla sua ‘immutabilità esteriore’.
Si consideri, per esempio, [`Arc<T>`][arc]:

```rust
use std::sync::Arc;

let x = Arc::new(5);
let y = x.clone();
```

[arc]: ../std/sync/struct.Arc.html

Quando chiamiamo `clone()`, l'oggetto di tipo `Arc<T>` deve aggiornare
il conteggio dei riferimenti. Però qui non abbiamo usato nessun `mut`, `x` è
un legame immutabile, e non abbiamo preso il valore `&mut 5` né altri valori.
E allora?

Per capirlo, dobbiamo tornare al nucleo della filosofia guida di Rust, che è
la sicurezza di memoria, e al meccanismo col quale Rust la garantisce, che è
il sistema di [possesso][possesso], e più specificamente,
il [prestito][prestito]:

> Si possono avere l'uno o l'altro di questi due tipi di prestiti, ma non
> entrambi allo stesso tempo:
>
> * uno o più riferimenti immutabili (`&T`) a una risorsa,
> * esattamente un riferimento mutabile (`&mut T`).

[ownership]: ownership.html
[borrowing]: references-and-borrowing.html#borrowing

Perciò, questa è la vera definizione di ‘immutabilità’: è sicuro che ci siano
due puntatori a questo oggetto? Nel caso di `Arc<T>`, sì: la mutazione è
contenuta interamente dentro la struttura stessa. Non si presenta all'utente.
Per questa ragione, passa un `&T` a `clone()`. Se gli avesse passato
un `&mut T`, però, sarebbe un errore.

Altri tipi, come quelli nel modulo [`std::cell`][stdcell], sono
nella situazione opposta: mutabilità interiore. Per esempio:

```rust
use std::cell::RefCell;

let x = RefCell::new(42);

let y = x.borrow_mut();
```

[stdcell]: ../std/cell/index.html

RefCell passa dei riferimenti `&mut` a ciò che c'è al suo interno usando
il metodo `borrow_mut()`. Non è pericoloso? Che succede se facciamo:

```rust,ignore
use std::cell::RefCell;

let x = RefCell::new(42);

let y = x.borrow_mut();
let z = x.borrow_mut();
# (y, z);
```

Di fatto questo andrà in panico, in fase di esecuzione. Questo è quello che fa
`RefCell`: forza le regole di prestito di Rust in fase di esecuzione, e va in
`panic!` se sono violate. Questo ci consente di aggirare un altro aspetto
delle regole di mutabilità di Rust. Prima parliamone.

## Mutabilità a livello di campo

La mutabilità è una proprietà o di un prestito (`&mut`) o di un legame
(`let mut`). Ciò significa che, per esempio, non si può avere
una [`struct`][struct] con alcuni campi mutabili e altri immutabili:

```rust,ignore
struct Punto {
    x: i32,
    mut y: i32, // non si può
}
```

La mutabilità di una struct sta nel suo legame:

```rust,ignore
struct Punto {
    x: i32,
    y: i32,
}

let mut a = Punto { x: 5, y: 6 };

a.x = 10;

let b = Punto { x: 5, y: 6};

b.x = 10; // errore: non si può assegnare al campo immutabile `b.x`
```

[struct]: structs.html

Però, usando [`Cell<T>`][cell], si può emulare la mutabilità a livello
di campo:

```rust
use std::cell::Cell;

struct Punto {
    x: i32,
    y: Cell<i32>,
}

let punto = Punto { x: 5, y: Cell::new(6) };

punto.y.set(7);

println!("y: {:?}", punto.y);
```

[cell]: ../std/cell/struct.Cell.html

Questo stamperà `y: Cell { value: 7 }`. Abbiamo aggiornato `y` con successo.
