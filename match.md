% Match

Spesso, un semplice [`if`][if]/`else` non basta, perché ci sono più di due
opzioni possibili. Inoltre, le condizioni possono diventare parecchio
complesse. Rust ha una parola-chiave, `match` ("combacia"), che consente
di sostituire dei complicati raggruppamenti di `if`/`else`
con qualcosa di più potente. Ecco qua:

```rust
let x = 5;

match x {
    1 => println!("uno"),
    2 => println!("due"),
    3 => println!("tre"),
    4 => println!("quattro"),
    5 => println!("cinque"),
    _ => println!("qualcos'altro"),
}
```

[if]: if.html

`match` prende un'espressione ed esegue una diramazione in base al suo valore.
Ogni ‘braccio’ della diramazione ha la forma `valore => espressione`.
Quando il valore combacia, l'espressione di quel braccio viene valutata.
Viene chiamata `match` a causa del concetto di ‘pattern matching’, di cui
`match` è un'implementazione. C'è un [separate section on patterns][patterns]
che tratta di tutti i pattern che sono ammessi qui.

[patterns]: patterns.html

Uno dei molti vantaggi di `match` è che impone la ‘verifica di esaustività’.
Per esempio, se si toglie l'ultimo braccio, quello con il carattere `_`,
il compilatore darà l'errore:

```text
error: non-exhaustive patterns: `_` not covered
```

Rust ci dice che abbiamo dimenticato qualche valore. Il compilatore inferisce
dall'`x` che può avere qualunque valore a 32 bit, da -2.147.483.648
a 2.147.483.647. Il carattere `_` agisce da 'prendi-tutto', e prenderà
tutti i possibili valori che *non sono* specificati in un braccio
dello stesso `match`. Come si vede nell'esempio precedente, al `match` 
vengono forniti bracci per gli interi da 1 a 5, se `x` vale 6 o qualunque
altro valore, viene preso dal caso `_`.

Il costrutto `match` è anche un'espressione, il che significa che lo si può
usare al lato destro di un'istruzione `let` o direttamente dove è ammessa
una espressione:

```rust
let x = 5;

let numero = match x {
    1 => "uno",
    2 => "due",
    3 => "tre",
    4 => "quattro",
    5 => "cinque",
    _ => "qualcos'altro",
};
```

Talvolta è un modo carino di convertire qualcosa da un tipo a un altro; in
questo esempio gli interi vengono convertiti in `String`.

# Combaciare le enumerazioni

Un altro impiego importante della parola-chiave `match` sta nell'elaborare
le possibili varianti di un'enumerazione:

```rust
enum Messaggio {
    Abbandona,
    CambiaColore(i32, i32, i32),
    Sposta { x: i32, y: i32 },
    Scrivi(String),
}

fn abbandona() { /* ... */ }
fn cambia_colore(r: i32, g: i32, b: i32) { /* ... */ }
fn sposta_cursore(x: i32, y: i32) { /* ... */ }

fn elabora_messaggio(msg: Messaggio) {
    match msg {
        Messaggio::Abbandona => abbandona(),
        Messaggio::CambiaColore(r, g, b) => cambia_colorw(r, g, b),
        Messaggio::Sposta { x: x, y: y } => sposta_cursore(x, y),
        Messaggio::Scrivi(s) => println!("{}", s),
    };
}
```

Ancora, il compilatore Rust verifica l'esaustività, e richiede di avere
un braccio combaciante per ogni variante dell'enum. Se ne manca qualcuno, darà
un errore di compilazione, a meno che si usi il braccio `_`.

Diversamente dai precedenti utilizzi di `match`, questo caso non è
sostituibile da un semplice uso del costrutto `if`. Si può però usare
il costrutto  [`if let`][if-let], che può essere visto come una forma
abbreviata di `match`.

[if-let]: if-let.html
