% Alias di tipo

La parola-chiave `type` permette di dichiarare un alias di un altro tipo:

```rust
type Nome = String;
```

Si può poi usare questo tipo come se fosse un vero tipo:

```rust
type Nome = String;

let x: Nome = "Hello".to_string();
```

Si noti, però, che questo è un _alias_, non un tipo interamente nuovo. In altre
parole, siccome Rust è fortemente tipizzato, ci si aspetterebbe che
un confronto fra due tipi diversi fallisse:

```rust,ignore
let x: i32 = 5;
let y: i64 = 5;

if x == y {
   // ...
}
```

questo dà

```text
error: mismatched types:
 expected `i32`,
    found `i64`
(expected i32,
    found i64) [E0308]
     if x == y {
             ^
```

Ma, se avessimo un alias:

```rust
type Num = i32;

let x: i32 = 5;
let y: Num = 5;

if x == y {
   // ...
}
```

Questo compila senza errori. Il valore di tipo `Num` è identico in ogni aspetto
al valore di tipo `i32`. Per ottenere un tipo veramente nuovo, si può usare
una [struttura ennupla].

[struttura ennupla]: structs.html#tuple-structs

Gli alias di tipo possono essere usati anche con i generici:

```rust
use std::result;

enum ErroreConcreto {
    Foo,
    Bar,
}

type Result<T> = result::Result<T, ErroreConcreto>;
```

Questo codice crea una versione specializzata del tipo `Result`, che ha sempre
un `ErroreConcreto` nella parte `E` di `Result<T, E>`. Questo viene usato
tipicamente nella libreria standard per creare errori personalizzati per ogni
sottosezione. Per esempio, [io::Result][ioresult].

[ioresult]: ../std/io/type.Result.html
