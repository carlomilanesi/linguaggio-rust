% Tipi primitivi

Il linguaggio Rust ha vari tipi che sono considerati ‘primitivi’. Ciò
significa che fanno parte del linguaggio. Rust è strutturato in modo tale
che la libreria standard fornisce anche vari altri tipi utili, costruiti
basandosi su quelli primitivi, ma che non sono considerati primitivi.

# Booleani

Rust ha un tipo booleano primitivo, chiamato `bool`. Ha solo due valori,
`true` ("vero") e `false` ("falso"):

```rust
let x = true;

let y: bool = false;
```

I booleani sono usati tipicamente nei costrutti [`if`][if] e [`while`][while].

[if]: if.html
[while]: while.html

Nella [documentazione della libreria standard][bool] si trova ulteriore
documentazione sui `bool`.

[bool]: ../std/primitive.bool.html

# `char`

Il tipo `char` rappresenta un singolo valore scalare Unicode. Si possono
creare dei `char` racchudendolo tra apici singoli: (`'`)

```rust
let x = 'x';
let two_hearts = '💕';
```

Diversamente da alcuni altri linguaggi, cioò significa che il `char` di Rust
non è rappresentato a un singolo byte, ma da quattro byte.

Nella [documentazione della libreria standard][char] si trova ulteriore
documentazione sui `char`.

[char]: ../std/primitive.char.html

# Tipi numerici

Rust ha parecchi tipi numerici, appartenenti alle seguenti categories:
con segno e senza segno, fissi e variabili, a virgola mobile e interi.

Questi tipi consistono in due parti: la categoria, e la dimensione.
Per esempio, `u16` è un tipo senza segno con una dimensione di sedici bit.
Più bit consentono di rappresentare numeri più grandi.

Se un letterale numerico non specifica il tipo esatto a cui appartiene,
il suo tipo viene inferito nel seguente modo:

```rust
let x = 42; // x ha tipo i32

let y = 1.0; // y ha tipo f64
```

Ecco una lista dei diversi tipi numerici, con link alla loro documentazione
nella libreria standard:

* [i8](../std/primitive.i8.html)
* [i16](../std/primitive.i16.html)
* [i32](../std/primitive.i32.html)
* [i64](../std/primitive.i64.html)
* [u8](../std/primitive.u8.html)
* [u16](../std/primitive.u16.html)
* [u32](../std/primitive.u32.html)
* [u64](../std/primitive.u64.html)
* [isize](../std/primitive.isize.html)
* [usize](../std/primitive.usize.html)
* [f32](../std/primitive.f32.html)
* [f64](../std/primitive.f64.html)

Esaminiamoli in base alla loro categoria:

## Con segno e senza segno

Ci sono due categorie di tipi interi: con segno e senza segno. Per comprendere
la differenza, consideriamo un numero di 4 bit. Un numero di quattro bit,
con segno, consentirebbe di rappresentare i numeri da `-8` a `+7`. I numeri
con segno usano la "rappresentazione in complemento a due". Un numero
di quattro bit, senza segno, dato che non ha bisogno di rappresentare
valori negativi, può rappresentare valori da `0` a `+15`.

I tipi con segno usano una `u` per la loro categoria, e i tipi con segno
usano una `i`. La `u` sta per ‘unsigned’ ("senza segno"), mentre la `i`
sta per ‘integer’ ("intero"). Perciò `u8` è un numero senza segno, a otto bit,
 e `i8` è un numbero con segno, pure a otto bit.

## Tipi a dimensione fissa

I tipi dimensione fissa hanno uno specifico numero di bit nella loro
rappresentazione. Le dimensioni (in bit) valide sono `8`, `16`, `32`, e `64`.
Perciò, `u32` è un intero senza segno, a 32 bit,
mentre `i64` is a intero con segno, a 64 bit.

## Tipi a dimensione variabile

Rust fornisce anche dei tipi la cui effettiva dimensione dipende
dall'architettura di macchina target. L'ampiezza di questi tipi è sufficiente
a esprimere la dimensione di qualunque collezione, perciò questi tipi
appartengono alla categoria ‘size’ ("dimensione"). Anche loro hanno
la versione con segno e quella senza segno, e quindi sono due:
`isize` e `usize`.

## Tipi a virgola mobile

Rust ha anche due tipi a virgola mobile: `f32` e `f64`. Questi corrispondono
rispettivamente ai numeri a precisione singola e a precisione doppia
secondo lo stardard IEEE-754.

# Array

Come molti linguaggi di programmazione, Rust ha dei tipi compositi
per rappresentare sequenze di oggetti. Il più basilare è il tipo *array*
("schiera"), una lista a lunghezza fissa di elementi dello stesso tipo.
Di default, gli array sono immutabili.

```rust
let a = [1, 2, 3]; // a: [i32; 3]
let mut m = [1, 2, 3]; // m: [i32; 3]
```

Gli arrays hanno tipo `[T; N]`. Parleremo di questa notazione `T` nella
[sezione sulla genericità][generics]. La `N` è una costante nota in fase
di compilazione, che indica il numero di oggetti contenuto nell'array.

C'è un'abbreviazione per inizializzare ogni elemento di un array allo stesso
valore. Ecco come inizializzare a `0` ognuno dei 20 elementi dell'array `a`:

```rust
let a = [0; 20]; // a: [i32; 20]
```

Si può ottenere il numero di elementi di un array `a` con l'espressione
`a.len()`:

```rust
let a = [1, 2, 3];

println!("a ha {} elementi", a.len());
```

Si può accedere a un particolare elemento di un array con
la *notazione a indice*:

```rust
let nomi = ["Graydon", "Brian", "Niko"]; // nomi: [&str; 3]

println!("Il secondo nome è: {}", nomi[1]);
```

Gli indici partono da zero, come nella maggior parte dei linguaggi
di programmazione, e perciò il primo nome è `nomi[0]` e il secondo nome è
`nomi[1]`. L'esempio precedente stampa `Il secondo nome è: Brian`.
Provando a usare un indice non compreso nell'array, si ottiene un errore,
perché per ogni accesso a un array, in fase di esecuzione si verificati
che l'indice sia compreso nei limiti. Accessi erronei di questo tipo
causano molti bug in altri linguaggi di programmazione di sistema.

You can find more documentation for `array`s [in the standard library
documentation][array].

[array]: ../std/primitive.array.html

# Le slice ("fette")

Le ‘slice’ (pronunciato "slais") sono riferimenti a (o “viste" dentro)
altre strutture dati.
Servono a consentire un accesso sicuro ed efficiente a una porzione
di un array senza fare copie. Per esempio, si potrebbe voler far riferimento
solamente a una riga di un file letto in memoria. Per sua natura, una slice
non viene creata direttamente, ma partendo da una variabile esistente.
Le slice hanno punto d'inizio e lunghezza costanti, ma il loro contenuto
può essere mutabile o immutabile.

Internamente, le slice sono rappresentate come un puntatore all'inizio
dei dati e una lunghezza.

## Sintassi delle slice

Per creare una slice da vari oggetti si può usare la combinazione
del carattere `&` e della coppia di caratteri `[]`. Il carattere `&` indica
che le slice sono simili ai [riferimenti], che tratteremo in dettaglio
più avanti in questa sezione. La coppia di caratteri `[]` che racchiude
un range permette di definire la lunghezza della slice:

```rust
let a = [0, 1, 2, 3, 4];
let completo = &a[..]; // Una slice contenente tutti gli elementi di a
let mezzo = &a[1..4]; // Una slice contenente solo gli elementi 1, 2, e 3
```

Le slice sono di tipo `&[T]`. Parleremo di quella `T` quando tratteremo la
[genericità][genericità].

[genericità]: generics.html

You can find more documentation for slices [in the standard library
documentation][slice].

[slice]: ../std/primitive.slice.html

# `str`

Il tipo `str` di Rust è il tipo di stringa più primitivo. As an [unsized type][dst],
it’s not very useful by itself, but becomes useful when placed behind a
reference, like `&str`. We'll elaborate further when we cover
[Strings][strings] and [references].

[dst]: unsized-types.html
[strings]: strings.html
[references]: references-and-borrowing.html

You can find more documentation for `str` [in the standard library
documentation][str].

[str]: ../std/primitive.str.html

# Ennuple

Un'ennupla è una lista ordinata di lunghezza fissa. Come questa:

```rust
let x = (1, "ciao");
```

Le parentesi e le virgole formano questa ennupla di lunghezza due. Ecco
lo stesso codice, ma con il tipo annotato:

```rust
let x: (i32, &str) = (1, "ciao");
```

Come si vede, il tipo di un'ennupla somiglia all'ennupla, ma in ogni posizione
c'è il tipo invece del valore. I lettori attenti noteranno anche che
le ennuple sono eterogenee: in questa ennupla c'è un `i32` e un `&str`.
Nei linguaggi di programmazione di sistema, strings are a bit more complex than in other
languages. For now, read `&str` as a *string slice*, and we’ll learn more
soon.

You can assign one ennupla into another, if they have the same contained types
and [arity]. Le ennuple have the same arity when they have the same length.

[arity]: glossary.html#arity

```rust
let mut x = (1, 2); // x: (i32, i32)
let y = (2, 3); // y: (i32, i32)

x = y;
```

Si può accedere ai campi di un'ennupla usando un *`let` destrutturante*.
Ecco un esempio:

```rust
let (x, y, z) = (1, 2, 3);

println!("x is {}", x);
```

Remember [before][let] when I said the left-hand side of a `let` statement was more
powerful than assigning a binding? Here we are. We can put a pattern on
the left-hand side of the `let`, and if it matches up to the right-hand side,
we can assign multiple bindings at once. In this case, `let` “destructures”
or “breaks up” l'ennupla, and assigns the bits to three bindings.

[let]: variable-bindings.html

Questo pattern è molto potente, e lo ritroveremo ripetuto in seguito.

Per disambiguare un'ennupla con un solo elemento da un valore
tra parentesi, basta usare una virgola:

```rust
(0,); // ennupla con un solo elemento
(0); // zero tra parentesi
```

## Indicizzazione delle ennuple

I campi di un'ennupla possono esseree acceduti anche con la sintassi
di indicizzazione:


```rust
let ennupla = (1, 2, 3);

let x = ennupla.0;
let y = ennupla.1;
let z = ennupla.2;

println!("x is {}", x);
```

Come l'indicizzazione di array, parte da zero, ma diversamente
dall'indicizzazione di array, usa un carattere `.`, invece della coppia
di caratteri `[]`.

You can find more documentation per le ennuple [in the standard library
documentation][ennupla].

[ennupla]: ../std/primitive.tuple.html

# Funzioni

Anche le funzioni hanno un tipo! Ecco un esempio:

```rust
fn foo(x: i32) -> i32 { x }

let x: fn(i32) -> i32 = foo;
```

In questo caso, `x` è un ‘puntatore a funzione’ che punta a una funzione
che prende un `i32` e rende un `i32`.
