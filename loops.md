% Cicli

Attualmente Rust fornisce tre costrutti per eseguire attività iterative:
`loop`, `while` e `for`. Ogni costrutto serve a scopi diversi.

## loop

Il costrutto `loop` ("ciclo") è la forma più semplice di ciclo disponibile
in Rust. Usando la parola-chiave `loop`, Rust fornisce un modo
di ciclare indefinitamente finché si raggiunge qualche istruzione
di terminazione. Il ciclo infinito di Rust è fatto così:

```rust,ignore
loop {
    println!("Cicla per sempre!");
}
```

## while

Rust ha anche un ciclo `while` ("fintanto che"). È fatto così:

```rust
let mut x = 5; // mut x: i32
let mut fatto = false; // mut fatto: bool

while !fatto {
    x += x - 3;

    println!("{}", x);

    if x % 5 == 0 {
        fatto = true;
    }
}
```

I cicli `while` sono la scelta appropriata quando non si è sicuri
di quante volte si dovrà ciclare.

Se serve un ciclo infinito, si può essere tentati di scrivere:

```rust,ignore
while true {
```

Tuttavia, il costrutto `loop` è molto più adatto per gestire questo caso:

```rust,ignore
loop {
```

L'analisi del flusso di costrullo di Rust tratta questo costrutto
diversamente da un `while true`, dato che sappiamo che ciclerà per sempre.
In generale, più informazione possiamo dare al compilatore, meglio può fare
con la sicurezza e la generazione del codice, e perciò si dovrebbe sempre
preferire `loop` quando si intende ciclare indefinitamente.

## for

Il ciclo `for` viene usato per ciclare un particolare numero di volte.
Però i cicli `for` di Rust funzionano un po' diversamente dagli altri
linguaggi di sistema. Il ciclo `for` di Rust non somiglia al ciclo `for`
del linguaggio C:

```c
for (x = 0; x < 10; x++) {
    printf( "%d\n", x );
}
```

Invece, è fatto così:

```rust
for x in 0..10 {
    println!("{}", x); // x: i32
}
```

o, in termini più astratti,

```rust,ignore
for var in expression {
    code
}
```

L'espressione è un elemento che può essere convertito in un [iteratore] usando
[`IntoIterator`]. L'iteratore rende una serie di elementi. Ogni elemento è
un'iterazione del ciclo. Tale valore viene poi associato al nome `var`,
che è valido solo nel corpo del ciclo. Una volta che il corpo è finito,
il prossimo valore viene preso dall'iteratore, e si esegue un'altra iterazione.
Quando non ci sono più valori, il ciclo `for` è finito.

[iterator]: iterators.html
[`IntoIterator`]: ../std/iter/trait.IntoIterator.html

Nel nostro esempio, `0..10` è un'espressione che prende una posizione
di inizio e una di fine, e dà un iteratore su quei valori. Tuttavia, il limite
superiore è escluso, così che questo ciclo stamperà i numeri da `0` a `9`,
e non il `10`.

Rust non ha il ciclo `for` in "stile C" di proposito. Controllare manualmente
ogni elemento del ciclo è complicato e soggetto a errori, anche per
sviluppatori esperti nel linguaggio C.

### Enumerazione

Quando c'è bisogno di tener traccia di quante volte si ha già ciclato,
si può usare la funzione `.enumerate()`.

#### Sui range:

```rust
for (i, j) in (5..10).enumerate() {
    println!("i = {} e j = {}", i, j);
}
```

Emette:

```text
i = 0 e j = 5
i = 1 e j = 6
i = 2 e j = 7
i = 3 e j = 8
i = 4 e j = 9
```

In questo caso si devono aggiungere le parentesi intorno al range.

#### Sugli iteratori:

```rust
let linee = "ciao\nmondo".lines();

for (numerolinea, linea) in linee.enumerate() {
    println!("{}: {}", numerolinea, linea);
}
```

Emette:

```text
0: ciao
1: mondo
```

## Terminare precocemente l'iterazione

Diamo un'occhiata a quel ciclo `while` di prima:

```rust
let mut x = 5;
let mut fatto = false;

while !fatto {
    x += x - 3;

    println!("{}", x);

    if x % 5 == 0 {
        fatto = true;
    }
}
```

Abbiamo dovuto tenere apposta una variabile booleana, `fatto`, per sapere
quando dovremmo uscire dal ciclo. Rust ha due parole-chiave per aiutarci
a modificare le iterazioni: `break` ("interrompi") e `continue` ("continua").

In questo caso, possiamo scrivere il ciclo in un modo migliore usando `break`:

```rust
let mut x = 5;

loop {
    x += x - 3;

    println!("{}", x);

    if x % 5 == 0 { break; }
}
```

Adesso cicliamo per sempre usando `loop` e usiamo `break` per uscire
precocemente. Anche eseguire un'istruzione `return` potrebbe servire
a terminare il ciclo precocemente.

`continue` è simile, ma invece di terminare il ciclo, passa alla prossima
iterazione. Questo codice stamperà solamente i numeri dispari:

```rust
for x in 0..10 {
    if x % 2 == 0 { continue; }

    println!("{}", x);
}
```

## Etichette dei cicli

Si potrebbero anche incontrare situazioni in cui ci sono cicli annidati
si vuole specificare a quale ciclo si riferisce una particolare istruzione
`break` o `continue`. Come nella maggior parte degli altri linguaggi,
di default an'istruzione `break` o `continue` si applicheranno al ciclo
più interno. Dove si volesse applicare `break` o `continue` a uno dei cicli
esterni, si possono usare delle etichette. Il seguente codice stamperà
solamente quando sia `x` che `y` sono dispari:

```rust
'esterno: for x in 0..10 {
    'interno: for y in 0..10 {
        if x % 2 == 0 { continue 'esterno; } // continua al ciclo su x
        if y % 2 == 0 { continue 'interno; } // continua al ciclo su y
        println!("x: {}, y: {}", x, y);
    }
}
```
