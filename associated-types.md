% I tipi associati

I tipi associati sono una parte potente del sistema di tipi di Rust. Sono
correlati all'idea di una ‘famiglia di tipi’, in altre parole, raggruppando
insieme più tipi. Questa descrizione è un po' astratta, e quindi tuffiamoci
subito in un esempio. Se si vuole scrivere un tratto `Grafo`, ci sono due tipi
sui quali essere generici: il tipo dei nodi e il tipo degli archi. Perciò
si potrebbe scrivere un tratto, `Grafo<N, A>`, così:

```rust
trait Grafo<N, A> {
    fn ha_arco(&self, &N, &N) -> bool;
    fn archi(&self, &N) -> Vec<A>;
    // ecc
}
```

Mentre questo in qualche modo funziona, finisce per essere goffo. Per esempio,
ogni funione che vuole prendere un `Grafo` come parametro adesso ha bisogno
_anche_ di essere generica sui tipi `N`odo e `A`rco:

```rust,ignore
fn distanza<N, A, G: Grafo<N, A>>(grafo: &G, inizio: &N, fine: &N) -> u32 { ... }
```

Il nostro calcolo di distanza funziona indipendentemente dal nostro tipo
`Arco`, e quindi citare la `A` in questa firma è una distrazione.

Ciò che vogliamo realmente dire è che un certo tipo `A`rco e un certo tipo
`N`odo si mettono insieme per formare ogni tipo di `Grafo`. Lo possiamo fare
con i tipi associati:

```rust
trait Grafo {
    type N;
    type A;

    fn ha_arco(&self, &Self::N, &Self::N) -> bool;
    fn archi(&self, &Self::N) -> Vec<Self::A>;
    // ecc
}
```

Adesso, i nostri clienti possono usare tutta l'astrazione di un dato `Graph`:

```rust,ignore
fn distanza<G: Graph>(grafo: &G, inizio: &G::N, fine: &G::N) -> u32 { ... }
```

Qui non c'è bisogno di trattare con il tipo `A`rco!

Analizziamolo in maggiore dettaglio.

## Definire tipi associati

Costruiamo quel tratto `Grafo`. Ecco la definizione:

```rust
trait Grafo {
    type N;
    type A;

    fn ha_arco(&self, &Self::N, &Self::N) -> bool;
    fn archi(&self, &Self::N) -> Vec<Self::A>;
}
```

Abbastanza semplice. I tipi associati usano la parola-chiave `type`, e vanno
dentro il corpo del tratto, insieme alle funzioni.

Queste dichiarazioni `type` possono avere tutte le cose che hanno le funzioni.
Per esempio, se volessimo che il nostro tipo `N` implementi `Display`, così da
poter stampare i nodi, potremmo fare così:

```rust
use std::fmt;

trait Grafo {
    type N: fmt::Display;
    type A;

    fn ha_arco(&self, &Self::N, &Self::N) -> bool;
    fn archi(&self, &Self::N) -> Vec<Self::A>;
}
```

## Implementare i tipi associati

Proprio come ogni tratto, i tratti che usano tipi associati usano
la parola-chiave `impl` per fornire implementazioni. Ecco una semplice
implementazione di Grafo:

```rust
# trait Grafo {
#     type N;
#     type A;
#     fn ha_arco(&self, &Self::N, &Self::N) -> bool;
#     fn archi(&self, &Self::N) -> Vec<Self::A>;
# }
struct Nodo;

struct Arco;

struct IlMioGrafo;

impl Grafo for IlMioGrafo {
    type N = Nodo;
    type A = Arco;

    fn ha_arco(&self, n1: &Nodo, n2: &Nodo) -> bool {
        true
    }

    fn archi(&self, n: &Nodo) -> Vec<Arco> {
        Vec::new()
    }
}
```

Questa implementazione sciocca restituisce sempre `true` e un `Vec<Arco>`
vuoto, ma dà un'idea di come implementare questo tipo di cose.
Dapprima ci servono tre
`struct`, una per il grafo, una per il nodo, e una per l'arco. Se avesse
più senso usare un altro tipo, quello funzionerebbe altrettanto, qui useremo
le `struct` per tutti e tre.

Poi c'è la riga `impl`, che è un'implementazione come per qualunque
altro tratto.

Da qui, usiamo `=` per definire i nostri tipi associati. Il nome usato
dal tratto va sulla sinistra dell'`=`, e il tipo concreto per cui stiamo
implementando questo va sulla destra. Infine, usiamo i tipi concreti
nelle nostre dichiarazioni di funzione.

## Gli oggetti-tratto con tipi associati

C'è un altro pezzo di sintassi di cui dovremmo parlare: gli oggetti-tratto. Se
si prova a creare un oggetto-tratto da un tratto con un tipo associato, così:

```rust,ignore
# trait Grafo {
#     type N;
#     type A;
#     fn ha_arco(&self, &Self::N, &Self::N) -> bool;
#     fn archi(&self, &Self::N) -> Vec<Self::A>;
# }
# struct Nodo;
# struct Arco;
# struct IlMioGrafo;
# impl Grafo for IlMioGrafo {
#     type N = Nodo;
#     type A = Arco;
#     fn ha_arco(&self, n1: &Nodo, n2: &Nodo) -> bool {
#         true
#     }
#     fn archi(&self, n: &Nodo) -> Vec<Arco> {
#         Vec::new()
#     }
# }
let grafo = IlMioGrafo;
let ogg = Box::new(grafo) as Box<Grafo>;
```

Otterremo due errori:

```text
error: the value of the associated type `A` (from the trait `main::Grafo`) must
be specified [E0191]
let ogg = Box::new(grafo) as Box<Grafo>;
          ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~
24:44 error: the value of the associated type `N` (from the trait
`main::Grafo`) must be specified [E0191]
let ogg = Box::new(grafo) as Box<Grafo>;
          ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~
```

Non possiamo creare un oggetto-tratto così, perché non conosciamo
i tipi associati. Invece, possiamo scrivere così:

```rust
# trait Grafo {
#     type N;
#     type A;
#     fn ha_arco(&self, &Self::N, &Self::N) -> bool;
#     fn archi(&self, &Self::N) -> Vec<Self::A>;
# }
# struct Nodo;
# struct Arco;
# struct IlMioGrafo;
# impl Grafo for IlMioGrafo {
#     type N = Nodo;
#     type A = Arco;
#     fn ha_arco(&self, n1: &Nodo, n2: &Nodo) -> bool {
#         true
#     }
#     fn archi(&self, n: &Nodo) -> Vec<Arco> {
#         Vec::new()
#     }
# }
let grafo = IlMioGrafo;
let ogg = Box::new(grafo) as Box<Grafo<N=Nodo, A=Arco>>;
```

La sintassi `N=Nodo` ci permette di fornire un tipo concreto, `Nodo`, per
il parametro di tipo `N`. Lo stesso con `A=Arco`. Se non fornissimo questo
vincolo, non potremmo essere sicuri di quale `impl` corrisponda a questo
oggetto-tratto.
