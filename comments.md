% Commenti

Adesso che abbiamo un po' di funzioni, è una buona idea imparare riguardo
ai commenti.
I commenti sono annotazioni che si lasciano per gli altri programmatori,
per aiutarli a spiegare il proprio codice. Il compilatore prevalentemente
li ignora.

Rust ha due tipi di commenti a cui si dovrebbe essere interessati:
i *commenti di riga* e i *commenti di documentazione* ["doc comment"].

```rust
// I commenti di riga sono tutti i caratteri dopo ‘//’ e la fine della riga.

let x = 5; // anche questo è un commento di riga.

// Se si ha una lunga spiegazione da scrivere, si possono mettere più
// commenti di riga, uno dopo l'altro. Mettere uno spazio tra // e il testo
// rende più leggibile il commento.
```

L'altro typo di commento è il commento di documentazione. I commenti
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
gli elementi che contengono tali commenti, (per es. crate, moduli, o funzioni)
invece che per commentare gli elementi che li seguono. Sono usati tipicamente
all'interno delle radici di crate (lib.rs) o delle radici di moduli (mod.rs):

```
//! # La libreria Standard di Rust
//!
//! La libreria Standard di Rust fornisce la funzionalità
//! essenziale di runtime per costruire software portabile
//! in Rust.
```

Quando si scrivono commenti di documentazione, fornire degli esempi di utilizzo
è di enorme aiuto. Si noterà che qui abbiamo usato una nuova macro:
`assert_eq!`. Questa macro confronta due valori, e va in `panic!` se non sono
uguali tra di loro. È di grande aiuto nella documentazione. C'è un'altra macro,
`assert!`, che va in `panic!` se il valore passatole è `false`.

Si può usare lo strumento [`rustdoc`](documentation.html) per generare
documentazione HTML da questi commenti di documentazione, e anche per eseguire
gli esempi di codice commentato come collaudo!
