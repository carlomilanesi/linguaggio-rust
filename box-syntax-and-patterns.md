%  La sintassi e i pattern di Box

Attualmente, l'unico modo stabile di creare un `Box` è tramite il metodo
`Box::new`. Inoltre, nella versione stabile di Rust, non è possibile
destrutturare un `Box` in un pattern di `match`. La parola-chiave instabile
`box` si può usare sia per creare che per destrutturare un `Box`.
Un esempio di utilizzo sarebbe:

```rust
#![feature(box_syntax, box_patterns)]

fn main() {
    let b = Some(box 5);
    match b {
        Some(box n) if n < 0 => {
            println!("Il box contiene il numero negativo {}", n);
        },
        Some(box n) if n >= 0 => {
            println!("Il box contiene il numero non negativo {}", n);
        },
        None => {
            println!("Nessun box");
        },
        _ => unreachable!()
    }
}
```

Si noti che queste caratteristiche sono attualmente nascoste dietro i gate
`box_syntax` (per la creazione dei box) e `box_patterns`
(per la destrutturazione e il pattern matching), perché la sintassi
può ancora cambiare in futuro.

# Restituire puntatori

In molti linguaggi dotati di puntatori, si restituirebbe un puntatore
da una funzione per evitare di copiare una grande struttura dati. Per esempio:

```rust
struct GrossaStruct {
    uno: i32,
    due: i32,
    // ecc
    cento: i32,
}

fn foo(x: Box<GrossaStruct>) -> Box<GrossaStruct> {
    Box::new(*x)
}

fn main() {
    let x = Box::new(GrossaStruct {
        uno: 1,
        due: 2,
        cento: 100,
    });

    let y = foo(x);
}
```

L'idea è che passando un box, sia sta solo copiando un puntatore, invece
dei cento `i32` che costituiscono la `GrossaStruct`.

Questo in Rust è un antipattern. Invece, si scrive così:

```rust
#![feature(box_syntax)]

struct GrossaStruct {
    uno: i32,
    due: i32,
    // ecc
    cento: i32,
}

fn foo(x: Box<GrossaStruct>) -> GrossaStruct {
    *x
}

fn main() {
    let x = Box::new(GrossaStruct {
        uno: 1,
        due: 2,
        cento: 100,
    });

    let y: Box<GrossaStruct> = box foo(x);
}
```

Ciò fornisce flessibilità senza sacrificare le prestazioni.

Si potrebbe pensare che così si otterrebbero prestazioni terribili:
restituire un valore e poi inscatolarlo immediatamente?! Questo pattern
non è il peggio di entrambi i mondi? Ma Rust è più scaltro.
In questo codice non si fanno copie. `main` alloca abbastanza spazio
per il `box`, passa a `foo`, col nome di `x`, l'indirizzo di tale spazio,
e poi `foo` scrive il valore direttamente nel `Box<T>`.

Ciò è abbastanza importante che vale la pena ripeterlo: i puntatori
non sono da usare per ottimizzare la restituzione di valori
dal proprio codice. Si consenta al chiamante di scegliere come vuole usare
l'output della funzione.
