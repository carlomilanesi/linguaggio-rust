% Funzioni

Ogni programma Rust ha almeno una funzione, la funzione `main`:

```rust
fn main() {
}
```

Questo è la dichiarazione di funzione più semplice possibile.
Come accennato prima, c'è `fn` che dice ‘questa è una funzione’, ed è seguita
dal nome della funzione, da due parentesi vuote perché questa funzione
non prende argomenti, e poi da graffe che contengono il corpo della funzione.
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
alle dichiarazoni `let`:
si aggiunge un tipo al nome dell'argomento, dopo un carattere `:`.

Ecco un programma completo che addiziona due numeri e stampa il risultato:

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

Questa è una precisa decisione progettuale. Per quanto sia possibile
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

Le funzioni di Rust rendono esattamente un valore, e si dichiara il tipo dopo
una ‘freccia’, che è un trattino (`-`) seguito da un segno di maggiore (`>`).
L'ultima riga di una funzione determina che cosa rende. Qui si noterà
la mancanza di un punto-e-virgola. Se l'avessimo aggiunto:

```rust,ignore
fn somma_uno(x: i32) -> i32 {
    x + 1;
}
```

Avremmo ottenuto un errore:

```text
error: not all control paths return a value
fn add_one(x: i32) -> i32 {
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

E qual è la differenza? Le espressioni rendono un valore, mentre
le istruzioni no. Ecco perché andiamo a finire con il messaggio d'errore
‘non tutti i percorsi di controllo rendono un valore’: l'istruzione `x + 1;`
non ritorna un valore. Ci sono due tipi di istruzioni in Rust:
le ‘istruzioni di dichiarazione’ e le ‘istruzioni di espressione’. Tutto
il resto è un'espressione. Prima parliamo delle istruzioni di dichiarazione.

In alcuni linguaggi, i legami delle variabili possono essere scritti
come espressioni, non come istruzioni. Come in Ruby:

```ruby
x = y = 5
```

In Rust, however, using `let` to introduce a binding is _not_ an expression. The
following will produce a compile-time error:

```rust,ignore
let x = (let y = 5); // expected identifier, found keyword `let`
```

The compiler is telling us here that it was expecting to see the beginning of
an expression, and a `let` can only begin a statement, not an expression.

Note that assigning to an already-bound variable (e.g. `y = 5`) is still an
expression, although its value is not particularly useful. Unlike other
languages where an assignment evaluates to the assigned value (e.g. `5` in the
previous example), in Rust the value of an assignment è un'ennupla vuota `()`
because the assigned value can have [only one owner](ownership.html), and any
other returned value would be too surprising:

```rust
let mut y = 5;

let x = (y = 6);  // x has the value `()`, not `6`
```

The second kind of statement in Rust is the *expression statement*. Its
purpose is to turn any expression into a statement. In practical terms, Rust's
grammar expects statements to follow other statements. This means that you use
semicolons to separate expressions from each other. This means that Rust
looks a lot like most other languages that require you to use semicolons
at the end of every line, and you will see semicolons at the end of almost
every line of Rust code you see.

What is this exception that makes us say "almost"? You saw it already, in this
code:

```rust
fn add_one(x: i32) -> i32 {
    x + 1
}
```

Our function claims to return an `i32`, but with a semicolon, it would return
`()` instead. Rust realizes this probably isn’t what we want, and suggests
removing the semicolon in the error we saw before.

## Early returns

But what about early returns? Rust does have a keyword for that, `return`:

```rust
fn foo(x: i32) -> i32 {
    return x;

    // we never run this code!
    x + 1
}
```

Using a `return` as the last line of a function works, but is considered poor
style:

```rust
fn foo(x: i32) -> i32 {
    return x + 1;
}
```

The previous definition without `return` may look a bit strange if you haven’t
worked in an expression-based language before, but it becomes intuitive over
time.

## Diverging functions

Rust has some special syntax for ‘diverging functions’, which are functions that
do not return:

```rust
fn diverges() -> ! {
    panic!("This function never returns!");
}
```

`panic!` is a macro, similar to `println!()` that we’ve already seen. Unlike
`println!()`, `panic!()` causes the current thread of execution to crash with
the given message. Because this function will cause a crash, it will never
return, and so it has the type ‘`!`’, which is read ‘diverges’.

If you add a main function that calls `diverges()` and run it, you’ll get
some output that looks like this:

```text
thread ‘main’ panicked at ‘This function never returns!’, hello.rs:2
```

If you want more information, you can get a backtrace by setting the
`RUST_BACKTRACE` environment variable:

```text
$ RUST_BACKTRACE=1 ./diverges
thread 'main' panicked at 'This function never returns!', hello.rs:2
stack backtrace:
   1:     0x7f402773a829 - sys::backtrace::write::h0942de78b6c02817K8r
   2:     0x7f402773d7fc - panicking::on_panic::h3f23f9d0b5f4c91bu9w
   3:     0x7f402773960e - rt::unwind::begin_unwind_inner::h2844b8c5e81e79558Bw
   4:     0x7f4027738893 - rt::unwind::begin_unwind::h4375279447423903650
   5:     0x7f4027738809 - diverges::h2266b4c4b850236beaa
   6:     0x7f40277389e5 - main::h19bb1149c2f00ecfBaa
   7:     0x7f402773f514 - rt::unwind::try::try_fn::h13186883479104382231
   8:     0x7f402773d1d8 - __rust_try
   9:     0x7f402773f201 - rt::lang_start::ha172a3ce74bb453aK5w
  10:     0x7f4027738a19 - main
  11:     0x7f402694ab44 - __libc_start_main
  12:     0x7f40277386c8 - <unknown>
  13:                0x0 - <unknown>
```

If you need to override an already set `RUST_BACKTRACE`, 
in cases when you cannot just unset the variable, 
then set it to `0` to avoid getting a backtrace. 
Any other value (even no value at all) turns on backtrace.

```text
$ export RUST_BACKTRACE=1
...
$ RUST_BACKTRACE=0 ./diverges 
thread 'main' panicked at 'This function never returns!', hello.rs:2
note: Run with `RUST_BACKTRACE=1` for a backtrace.
```

`RUST_BACKTRACE` also works with Cargo’s `run` command:

```text
$ RUST_BACKTRACE=1 cargo run
     Running `target/debug/diverges`
thread 'main' panicked at 'This function never returns!', hello.rs:2
stack backtrace:
   1:     0x7f402773a829 - sys::backtrace::write::h0942de78b6c02817K8r
   2:     0x7f402773d7fc - panicking::on_panic::h3f23f9d0b5f4c91bu9w
   3:     0x7f402773960e - rt::unwind::begin_unwind_inner::h2844b8c5e81e79558Bw
   4:     0x7f4027738893 - rt::unwind::begin_unwind::h4375279447423903650
   5:     0x7f4027738809 - diverges::h2266b4c4b850236beaa
   6:     0x7f40277389e5 - main::h19bb1149c2f00ecfBaa
   7:     0x7f402773f514 - rt::unwind::try::try_fn::h13186883479104382231
   8:     0x7f402773d1d8 - __rust_try
   9:     0x7f402773f201 - rt::lang_start::ha172a3ce74bb453aK5w
  10:     0x7f4027738a19 - main
  11:     0x7f402694ab44 - __libc_start_main
  12:     0x7f40277386c8 - <unknown>
  13:                0x0 - <unknown>
```

A diverging function can be used as any type:

```rust,should_panic
# fn diverges() -> ! {
#    panic!("This function never returns!");
# }
let x: i32 = diverges();
let x: String = diverges();
```

## Function pointers

We can also create variable bindings which point to functions:

```rust
let f: fn(i32) -> i32;
```

`f` is a variable binding which points to a function that takes an `i32` as
an argument and returns an `i32`. For example:

```rust
fn plus_one(i: i32) -> i32 {
    i + 1
}

// without type inference
let f: fn(i32) -> i32 = plus_one;

// with type inference
let f = plus_one;
```

We can then use `f` to call the function:

```rust
# fn plus_one(i: i32) -> i32 { i + 1 }
# let f = plus_one;
let six = f(5);
```
