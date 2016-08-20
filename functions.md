% Funzioni

Ogni programma Rust ha almeno una funzione, la funzione `main`:

```rust
fn main() {
}
```

Questo è la dichiarazione di funzione più semplice possibile.
Come accennato prima, `fn` indica che ‘questa è una funzione’, ed è seguita
dal nome della funzione, da due parentesi vuote perché questa funzione
non prende argomenti, e poi da parentesi graffe che contengono il corpo della funzione.
Ecco una funzione chiamata `foo`:

```rust
fn foo() {
}
```

E per quanto riguarda gli argomenti? Ecco una funzione che stampa un numero:

```rust
fn stampa_numero(x: i32) {
    println!("x is: {}", x);
}
```

Ecco un programma completo che usa `stampa_numero`:

```rust
fn main() {
    stampa_numero(5);
}

fn stampa_numero(x: i32) {
    println!("x is: {}", x);
}
```

Come si vede, gli argomenti delle funzioni funzionano in modo molto simile
alle dichiarazioni `let`:
si aggiunge un tipo al nome dell'argomento, dopo i due punti `:`.

Ecco un programma completo che somma due numeri e stampa il risultato:

```rust
fn main() {
    stampa_somma(5, 6);
}

fn stampa_somma(x: i32, y: i32) {
    println!("la somma è: {}", x + y);
}
```

Si separano gli argomenti usando una virgola, sia quando si chiama la funzione,
che quando la si dichiara.

Diversamente dall'istruzione `let`, i tipi degli argomenti delle funzioni
_devono_ essere dichiarati. Pertanto questo non funziona:

```rust,ignore
fn stampa_somma(x, y) {
    println!("la somma è: {}", x + y);
}
```

Si ottiene l'errore:

```text
expected one of `!`, `:`, or `@`, found `)`
fn print_sum(x, y) {
```

Questa è una ponderata decisione progettuale. Per quanto sia possibile
l'inferenza di tipo sull'intero programma, i linguaggi che ce l'hanno,
come Haskell, spesso suggeriscono che sia meglio documentare esplicitamente
i propri tipi. Concordiamo che costringere le funzioni a dichiarare i tipi
mentre consentire l'inferenza all'interno dei corpi delle funzioni sia
un punto di equilibrio perfetto tra l'inferenza completa e nessuna inferenza.

Che dire del valore reso? Ecco una funzione che somma uno a un intero:

```rust
fn somma_uno(x: i32) -> i32 {
    x + 1
}
```

Le funzioni di Rust restituiscono esattamente un valore, e si dichiara
il tipo dopo una ‘freccia’, che è un trattino (`-`) seguito da un segno
di maggiore (`>`).
L'ultima riga di una funzione determina che cosa restituisce. Qui si noterà
la mancanza di un punto-e-virgola. Se l'avessimo aggiunto:

```rust,ignore
fn somma_uno(x: i32) -> i32 {
    x + 1;
}
```

Avremmo ottenuto un errore:

```text
error: not all control paths return a value
fn somma_uno(x: i32) -> i32 {
     x + 1;
}

help: consider removing this semicolon:
     x + 1;
          ^
```

Questo rivela due cose interessanti di Rust: è un linguaggio basato
sulle espressioni, e i punto-e-virgola sono diversi dai punto-e-virgola
in altri linguaggi basati su ‘graffe e punto-e-virgola’. Questi due aspetti
sono correlati.

## Espressioni contro istruzioni

Rust è primariamente un linguaggio basato sulle espressioni. Ci sono solamente
due tipi di istruzioni, e ogni altra cosa è un'espressione.

E qual è la differenza? Le espressioni restituiscono un valore, mentre
le istruzioni no. Ecco perché andiamo a finire con il messaggio d'errore
‘non tutti i percorsi di controllo restituiscono un valore’:
l'istruzione `x + 1;` non ritorna un valore.
Ci sono due tipi di istruzioni in Rust:
le ‘istruzioni di dichiarazione’ e le ‘istruzioni di espressione’. Tutto
il resto è un'espressione. Prima parliamo delle istruzioni di dichiarazione.

In alcuni linguaggi, i legami delle variabili possono essere scritti
come espressioni, non come istruzioni. Come in Ruby:

```ruby
x = y = 5
```

In Rust, però, l'uso di `let` per introdurre un legame _non_ è un'espressione.
La seguente riga produrrà un errore di compilazione:

```rust,ignore
let x = (let y = 5); // atteso un identificatore, trovata la parola-chiave `let`
```

Qui il compilatore ci sta dicendo che si stava aspettando di vedere l'inizio
di una espressione, mentre un `let` può iniziare solamente un'istruzione,
non un'espressione.

Si noti che assegnare a una variabile già legata (per es. `y = 5`) è ancora
un'espressione, per quanto il suo valore non sia particolarmente utile.
Diversamente da altri linguaggi, nei quali un assegnamento ha come valore
il valore assegnato (nell'esempio precedente, `5`), in Rust il valore
di un assegnamento è una ennupla vuota `()`, perché il valore assegnato può avere
[solamente un possessore](ownership.html), e ogni altro valore reso sarebbe
troppo sorprendente:

```rust
let mut y = 5;

let x = (y = 6);  // x ha valore `()`, non `6`
```

Il secondo genere di istruzioni in Rust è l'*istruzione espressione*. Il suo
scopo è trasformare qualunque espressione in un'istruzione. In pratica,
la grammatica di Rust si aspetta che delle istruzioni seguano altre istruzioni.
Ciò significa che si usano punti-e-virgola per separare diverse espressioni.
Ciò significa che Rust è molto simile alla maggior parte degli
altri linguaggi che richiedono di usare punti-e-virgola alla fine di ogni riga,
e si vedranno punti-e-virgola alla fine di quasi tutte le righe di codice Rust.

Cos'è questa eccezione che ci fa dire "quasi"? L'abbiamo già visto prima,
in questo codice:

```rust
fn somma_uno(x: i32) -> i32 {
    x + 1
}
```

La nostra funzione sostiene di restituire un `i32`, ma se ci fosse
un punto-e-virgola, restituirebbe un `()` invece. Rust si rende conto
che questo probabilmente non è ciò che vogliamo, e,
nel messaggio d'errore che abbiamo visto prima,
consiglia di togliere il punto-e-virgola.

## Uscite precoci

E che dire delle uscite precoci? Rust ha una parola-chiave farle, `return`:

```rust
fn foo(x: i32) -> i32 {
    return x;

    // non si eseguirà mai questo codice!
    x + 1
}
```

Usare un `return` come ultima riga di una funzione è corretto, ma è
considerato stile mediocre:

```rust
fn foo(x: i32) -> i32 {
    return x + 1;
}
```

La precedente definizione senza `return` può sembrare un po' strana a chi non
avesse mai lavorato con un linguaggio basato su espressioni, ma col tempo
diventa intuitivo.

## Funzioni divergenti

Rust ha alcune sintassi speciali per le ‘funzioni divergenti’, che sono
le funzioni che non rendono nessun valore:

```rust
fn diverge() -> ! {
    panic!("Questa funzione non rende nessun valore!");
}
```

`panic!` è una macro, come lo è `println!()` che abbiamo già visto.
Diversamente da `println!()`, `panic!()` manda in crash il thread corrente,
stampando il messaggio ricevuto come argomento. Dato che questa funzione
provocherà un crash, non renderà nessun valore, e quindi ha il tipo ‘`!`’,
che si legge ‘diverge’.

Se si aggiunge una funzione main che chiama `diverge()` e la si esegue,
si otterrà un output simile a questo:

```text
thread ‘main’ panicked at ‘Questa funzione non rende nessun valore!’, main.rs:2
```

Se si vogliono più informazioni, si può ottenere un backtrace impostando
la variabile d'ambiente `RUST_BACKTRACE`:

```text
$ rust_backtrace=1 ./diverge
thread 'main' panicked at 'Questa funzione non rende nessun valore!', main.rs:2
stack backtrace:
   1:     0x7f402773a829 - sys::backtrace::write::h0942de78b6c02817K8r
   2:     0x7f402773d7fc - panicking::on_panic::h3f23f9d0b5f4c91bu9w
   3:     0x7f402773960e - rt::unwind::begin_unwind_inner::h2844b8c5e81e79558Bw
   4:     0x7f4027738893 - rt::unwind::begin_unwind::h4375279447423903650
   5:     0x7f4027738809 - diverge::h2266b4c4b850236beaa
   6:     0x7f40277389e5 - main::h19bb1149c2f00ecfBaa
   7:     0x7f402773f514 - rt::unwind::try::try_fn::h13186883479104382231
   8:     0x7f402773d1d8 - __rust_try
   9:     0x7f402773f201 - rt::lang_start::ha172a3ce74bb453aK5w
  10:     0x7f4027738a19 - main
  11:     0x7f402694ab44 - __libc_start_main
  12:     0x7f40277386c8 - <unknown>
  13:                0x0 - <unknown>
```

Se serve sovrascrivere una variabile `RUST_BACKTRACE` già impostata, nel caso
in cui non si può semplicemente disimpostare la variabile, allora la si può
impostare a `0` per evitare di ottenere un backtrace. Qualunque altro valore
(anche nessun valore) attiva le informazioni di backtrace.

```text
$ export RUST_BACKTRACE=1
...
$ RUST_BACKTRACE=0 ./diverge
thread 'main' panicked at 'Questa funzione non rende nessun valore!', main.rs:2
note: Run with `RUST_BACKTRACE=1` for a backtrace.
```

`RUST_BACKTRACE` funziona anche con il comando `run` di Cargo:

```text
$ RUST_BACKTRACE=1 cargo run
     Running `target/debug/diverge`
thread 'main' panicked at 'Questa funzione non rende nessun valore!', main.rs:2
stack backtrace:
   1:     0x7f402773a829 - sys::backtrace::write::h0942de78b6c02817K8r
   2:     0x7f402773d7fc - panicking::on_panic::h3f23f9d0b5f4c91bu9w
   3:     0x7f402773960e - rt::unwind::begin_unwind_inner::h2844b8c5e81e79558Bw
   4:     0x7f4027738893 - rt::unwind::begin_unwind::h4375279447423903650
   5:     0x7f4027738809 - diverge::h2266b4c4b850236beaa
   6:     0x7f40277389e5 - main::h19bb1149c2f00ecfBaa
   7:     0x7f402773f514 - rt::unwind::try::try_fn::h13186883479104382231
   8:     0x7f402773d1d8 - __rust_try
   9:     0x7f402773f201 - rt::lang_start::ha172a3ce74bb453aK5w
  10:     0x7f4027738a19 - main
  11:     0x7f402694ab44 - __libc_start_main
  12:     0x7f40277386c8 - <unknown>
  13:                0x0 - <unknown>
```

Una funzione divergente può essere usata dove ci si aspetta un'espressione
di qualunque tipo:

```rust,should_panic
# fn diverge() -> ! {
#    panic!("Questa funzione non rende nessun valore!");
# }
let x: i32 = diverge();
let x: String = diverge();
```

## Puntatori di funzione

Possiamo anche creare legami di variabili che puntano a funzioni:

```rust
let f: fn(i32) -> i32;
```

`f` è un legame di variabile che punta a una funzione che prende un `i32` come
argomento e restituisce un `i32`. Per esempio:

```rust
fn piu_uno(i: i32) -> i32 {
    i + 1
}

// senza l'inferenza di tipo
let f: fn(i32) -> i32 = piu_uno;

// con l'inferenza di tipo
let f = piu_uno;
```

Poi possiamo usare `f` per chiamare la funzione:

```rust
# fn piu_uno(i: i32) -> i32 { i + 1 }
# let f = piu_uno;
let sei = f(5);
```
