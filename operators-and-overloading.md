% Operatori e sovraccaricamento ["overloading"]

Rust consente una forma limitata di sovraccaricamento degli operatori.
Ci sono certi operatori che possono essere sovraccaricati. Per supportare
un particolare operatore fra tipi, c'è uno specifico tratto che si può
implementare, il quale si occupa di sovraccaricare l'operatore.

Per esempio, l'operatore `+` può essere sovraccaricato con il tratto `Add`:

```rust
use std::ops::Add;

#[derive(Debug)]
struct Punto {
    x: i32,
    y: i32,
}

impl Add for Punto {
    type Output = Punto;

    fn add(self, other: Punto) -> Punto {
        Punto { x: self.x + other.x, y: self.y + other.y }
    }
}

fn main() {
    let p1 = Punto { x: 1, y: 0 };
    let p2 = Punto { x: 2, y: 3 };

    let p3 = p1 + p2;

    println!("{:?}", p3);
}
```

In `main`, possiamo usare `+` sui nostri due oggetti di tipo `Punto`, dato che
abbiamo implementato `Add<Output=Punto>` per `Punto`.

Ci sono vari operatori che possono essere sovraccaricati in questo modo,
e tutti i loro tratti associati si trovano nel modulo [`std::ops`][stdops].
Si veda la sua documentazione per avere l'elenco completo.

[stdops]: ../std/ops/index.html

L'implementazione di questi tratti segue un pattern. Guardiamo [`Add`][add]
più in dettaglio:

```rust
# mod foo {
pub trait Add<RHS = Self> {
    type Output;

    fn add(self, rhs: RHS) -> Self::Output;
}
# }
```

[add]: ../std/ops/trait.Add.html

Qui ci sono tre tipi coinvolti: il tipo per cui si implementa `Add`, `RHS`,
che di default è ancora `Self`, e `Output`. Nel caso di dell'espressione
`let z = x + y`, `x` è di tipo `Self`, `y` è di tipo RHS, e `z` è di tipo
`Self::Output`.

```rust
# struct Punto;
# use std::ops::Add;
impl Add<i32> for Punto {
    type Output = f64;

    fn add(self, rhs: i32) -> f64 {
        // somma un i32 a un Punto e ottieni un f64
# 1.0
    }
}
```

consente di fare questo:

```rust,ignore
let p: Punto = // ...
let x: f64 = p + 2i32;
```

# Utilizzo di tratti di operator in strutture generiche

Adesso che sappiamo come sono definiti i tratti degli operatori, possiamo
definire in modo più generico il nostro tratto `HaArea` e la nostra struttura
`Quadrato` di cui si è parlato nel [capitolo sui tratti][tratti]:

[tratti]: traits.html

```rust
use std::ops::Mul;

trait HaArea<T> {
    fn area(&self) -> T;
}

struct Quadrato<T> {
    x: T,
    y: T,
    lato: T,
}

impl<T> HaArea<T> for Quadrato<T>
        where T: Mul<Output=T> + Copy {
    fn area(&self) -> T {
        self.lato * self.lato
    }
}

fn main() {
    let s = Quadrato {
        x: 0.0f64,
        y: 0.0f64,
        lato: 12.0f64,
    };

    println!("Area di s: {}", s.area());
}
```

Per `HaArea` e `Quadrato`, dichiariamo un parametro di tipo `T` e sostituiamo
`f64` con tale parametro. L'`impl` ha bisogno di modifiche più involute:

```rust,ignore
impl<T> HasArea<T> for Square<T>
        where T: Mul<Output=T> + Copy { ... }
```

Il metodo `area` richiede che possiamo moltiplicare i lati, e quindi
dichiariamo che il tipo `T` deve implementare `std::ops::Mul`. Come `Add`,
citato prima, `Mul` stesso prende un parametro `Output`: dato che sappiamo
che i numeri non cambiano tipo quando vengono moltiplicati, impostiamo
anch'esso a `T`. `T` deve anche supportare la copia, cosicché Rust non provi
a spostare `self.side` nel valore reso.
