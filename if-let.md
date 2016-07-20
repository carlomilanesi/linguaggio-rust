% if let

Il costrutto `if let` consente di combinare i costrutti `if` e `let`,
per migliorare efficienza di certi tipi di pattern match.

Per esempio, diciamo che abbiamo un certo `Option<T>`. Vogliamo chiamare una
funzione su di esso, ma solo se vale `Some<T>`, e non fare niente se vale
`None`. Si potrebbe fare così:

```rust
# let opzione = Some(5);
# fn foo(x: i32) { }
match opzione {
    Some(x) => { foo(x) },
    None => {},
}
```

Qui non è necessario usare un `match`, perché potremmo anche usare un `if`:

```rust
# let opzione = Some(5);
# fn foo(x: i32) { }
if opzione.is_some() {
    foo(opzione.unwrap());
}
```

Però nessuna delle due soluzioni è particolarmente soddisfacente.
Invece si può usare un `if let` per fare la stessa cosa in un modo più carino:

```rust
# let opzione = Some(5);
# fn foo(x: i32) { }
if let Some(x) = opzione {
    foo(x);
}
```

Se un [pattern][pattern] combacia con successo, lega tutte le parti
appropriate del valore agli identificatori del pattern, e poi valuta
l'espressione. Se il pattern non combacia, non succede niente.

Se si vuole fare qualcos'altro quando il patter non combacia, si può usare
un `else`:

```rust
# let opzione = Some(5);
# fn foo(x: i32) { }
# fn bar() { }
if let Some(x) = opzione {
    foo(x);
} else {
    bar();
}
```

## `while let`

Similmente, il costrutto `while let` può essere usato quando si vuole ciclare
condizionalmente fin tanto che un valore combacia con un certo pattern.
Trasforma il seguente codice:

```rust
let mut v = vec![1, 3, 5, 7, 11];
loop {
    match v.pop() {
        Some(x) =>  println!("{}", x),
        None => break,
    }
}
```

nel seguente codice:

```rust
let mut v = vec![1, 3, 5, 7, 11];
while let Some(x) = v.pop() {
    println!("{}", x);
}
```

[pattern]: patterns.html
