% Vettori

Un ‘vettore’ è un array dinamico ossia ‘estendibile’, implementato dal tipo
[`Vec<T>`][vec] nella libreria standard. Il `T` significa che si possono avere
vettori di ogni tipo (si veda il capitolo su [generici][generic] per avere
maggiori informazioni).
I vettori allocano sempre i loro dati nello heap.
Possono essere creati con la macro `vec!`:

```rust
let v = vec![1, 2, 3, 4, 5]; // v: Vec<i32>
```

(Si noti che diversamente dalla macro `println!` che abbiamo usato in passato,
con la macro `vec!` usiamo le parentesi quadre `[]`. Rust permette di usare
entrambi i tipi di parentesi in entrambe le situazioni, questo uso è solo
una convenzione.)

C'è una forma alternativa di `vec!` per ripetere un valore iniziale:

```rust
let v = vec![0; 10]; // dieci zeri
```

I vettori immagazzinano il loro contenuto sullo heap come array contigui
di `T`. Ciò significa che devono essere capaci di sapere la dimensione di `T`
in fase di compilazione (cioè, quanti byte servono per memorizzare un `T`?).
La dimensione di alcuni oggetti non si può sapere in fase di compilazione.
Per tali oggetti si dovrà immagazzinare un puntatore a quell'oggetto:
fortunatamente, il tipo [`Box`][box] funziona perfettamente a questo scopo.

## Accedere agli elementi

Per ottenere il valore a un particolare indice nel vettore, si usano le `[]`:

```rust
let v = vec![1, 2, 3, 4, 5];

println!("Il terzo elemento di v è {}", v[2]);
```

Gli indici contano da `0`, e perciò il terzo elemento è `v[2]`.

È anche importante notare che si deve indicizzare con il tipo `usize`:

```rust,ignore
let v = vec![1, 2, 3, 4, 5];

let i: usize = 0;
let j: i32 = 0;

// funziona
v[i];

// non funziona
v[j];
```

Indicizzare con un tipo diverso da `usize` dà un errore come questo:

```text
error: the trait bound `collections::vec::Vec<_> : core::ops::Index<i32>`
is not satisfied [E0277]
v[j];
^~~~
note: the type `collections::vec::Vec<_>` cannot be indexed by `i32`
error: aborting due to previous error
```

C'è molta punteggiatura in quel messaggio, ma il suo nucleo significa:
non si può indicizzare con un `i32`.

## Accesso fuori dai limiti

Se si prova ad accedere un indice che non esiste:

```rust,ignore
let v = vec![1, 2, 3];
println!("L'elemento 7 è {}", v[7]);
```

allora il thread attuale andrà in [panico] con un messaggio come questo:

```text
thread 'main' panicked at 'index out of bounds: the len is 3 but the index is 7'
```

Se si vuole gestire gli errori di accesso fuori dai limiti senza andare
in panico, si possono usare metodi come [`get`][get] o [`get_mut`][get_mut],
che restituiscono `None` quando gli viene dato un indice invalido:

```rust
let v = vec![1, 2, 3];
match v.get(7) {
    Some(x) => println!("Item 7 is {}", x),
    None => println!("Spiacente, questo vettore è troppo corto.")
}
```

## Iterare

Una volta che si ha un vettore, si può iterare sui suoi elementi usando `for`.
Ce ne sono tre versioni:

```rust
let mut v = vec![1, 2, 3, 4, 5];

for i in &v {
    println!("Un riferimento a {}", i);
}

for i in &mut v {
    println!("Un riferimento mutabile a {}", i);
}

for i in v {
    println!("Prendi il possesso del vettore e del suo elemento {}", i);
}
```

Nota: Non si può usare ancora il vettore dopo averlo iterato prendendone
il possesso. Invece, si può iterare il vettore più volte se quando lo si itera
se ne prende un riferimento.
Per esempio, il seguente codice non compila.

```rust,ignore
let v = vec![1, 2, 3, 4, 5];

for i in v {
    println!("Prendi possesso del vettore e del suo elemento {}", i);
}

for i in v {
    println!("Prendi possesso del vettore e del suo elemento {}", i);
}
```

Mentre il seguente funziona perfettamente:

```rust
let v = vec![1, 2, 3, 4, 5];

for i in &v {
    println!("Questo è un riferimento a {}", i);
}

for i in &v {
    println!("Questo è un riferimento a {}", i);
}
```

I vettori hanno molti altri metodi utili, di cui si può leggere nella
[documentazione della loro API][vec].

[vec]: ../std/vec/index.html
[box]: ../std/boxed/index.html
[generici]: generics.html
[panic]: concurrency.html#panics
[get]: ../std/vec/struct.Vec.html#method.get
[get_mut]: ../std/vec/struct.Vec.html#method.get_mut
