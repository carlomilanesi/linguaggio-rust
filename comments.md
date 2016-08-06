% Commenti

Adesso che abbiamo un po' di funzioni, è una buona idea imparare i commenti.
I commenti sono annotazioni che si lasciano per gli altri programmatori,
per aiutarli a spiegare il proprio codice. Il compilatore per lo più li ignora.

Rust ha due tipi di commenti a cui si dovrebbe essere interessati:
i *commenti di riga* e i *commenti di documentazione* ["doc comment"].

```rust
// I commenti di riga sono i caratteri tra la coppia di caratteri ‘//’ e la fine della riga.

let x = 5; // anche questo è un commento di riga

// Se si ha una lunga spiegazione da scrivere, si possono mettere più
// commenti di riga, uno dopo l'altro. Mettere uno spazio tra // e il testo
// rende più leggibile il commento.
```

L'altro genere di commenti è il commento di documentazione. I commenti
di documentazione usano `///` invece di `//`, e supportano la notazione
Markdown al loro interno:

```rust
/// Aggiunge uno al numero dato.
///
/// # Esempi
///
/// ```
/// let cinque = 5;
///
/// assert_eq!(6, somma_uno(5));
/// # fn somma_uno(x: i32) -> i32 {
/// #     x + 1
/// # }
/// ```
fn somma_uno(x: i32) -> i32 {
    x + 1
}
```

C'è un altro stile di commento di documentazione, `//!`, usato per commentare
gli elementi (per es. crate, moduli, o funzioni) che contengono tali commenti,
invece che per commentare gli elementi che li seguono. Sono usati tipicamente
all'interno delle radici di crate (lib.rs) o delle radici di moduli (mod.rs):

```
//! # The Rust Standard Library
//!
//! The Rust Standard Library provides the essential runtime
//! functionality for building portable Rust software.
```

Quando si scrivono commenti di documentazione, fornire degli esempi di utilizzo
è di enorme aiuto. Si noterà che qui abbiamo usato una nuova macro:
`assert_eq!`. Questa macro confronta due valori, e va in `panic!` se non sono
uguali tra di loro. È di grande aiuto nella documentazione. C'è un'altra macro,
`assert!`, che va in `panic!` se il valore passatole vale `false`.

Si può usare lo strumento [`rustdoc`](documentation.html) per generare
documentazione HTML da questi commenti di documentazione, e anche per eseguire
gli esempi di codice come collaudo!
