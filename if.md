% if

La versione di Rust dell'istruzione `if` non è particolarmente complessa,
ma somiglia più all'`if` che si trova nei linguaggi tipizzati dinamicamente
che a quello dei linguaggi di sistema tradizionali.
Perciò parliamone, per assicurarsi che ne siano afferrate le sfumature.

`if` è una forma specifica di un concetto più generale, la ‘diramazione’,
il cui nome deriva dai rami degli alberi: è un punto di decisione, dove
a seconda di un valore, si possono prendere più strade.

Nel caso dell'`if`, c'è una sola scelta che conduce a due strade:

```rust
let x = 5;

if x == 5 {
    println!("x vale cinque!");
}
```

Se cambiassimo il valore di `x` con qualcos'altro, questa riga non verrebbe
stampata. Più specificamente, se l'espressione dopo l'`if` vale `true`, allora
il blocco viene eseguito; se vale `false`, no.

Se si vuole che accada qualcosa nel caso `false`, si usi un `else`:

```rust
let x = 5;

if x == 5 {
    println!("x vale cinque!");
} else {
    println!("x non vale cinque :(");
}
```

Se c'è più di un caso, si usi un `else if`:

```rust
let x = 5;

if x == 5 {
    println!("x vale cinque!");
} else if x == 6 {
    println!("x vale sei!");
} else {
    println!("x non vale né cinque né sei :(");
}
```

Tutto ciò è abbastanza normale. Però, si può fare anche questo:

```rust
let x = 5;

let y = if x == 5 {
    10
} else {
    15
}; // y: i32
```

che possiamo (e forse dovremmo) scrivere così:

```rust
let x = 5;

let y = if x == 5 { 10 } else { 15 }; // y: i32
```

Ciò funziona perché l'`if` è un'espressione. Il valore di tale espressione è
il valore dell'ultima espressione della diramazione scelta. Un `if` senza un
`else` ha sempre il valore `()`.
