% Gli Slice pattern

Se si vuole combaciare con una slice o un array, si può usare `&`
con la caratteristica degli `slice_pattern`:

```rust
#![feature(slice_patterns)]

fn main() {
    let v = vec!["combacia_questo", "1"];

    match &v[..] {
        &["combacia_questo", secondo] => println!("Il secondo elemento è {}",
            secondo),
        _ => {},
    }
}
```

Il gate degli `advanced_slice_patterns` consente di usare `..` per indicare
qualunque numero di elementi dentro un pattern che combacia con una slice.
Questo jolly può essere usato solamente una volta per ogni dato array.
se c'è un identificatore prima dei `..`, il risultato della slice verrà legato
a quel nome. Per esempio:

```rust
#![feature(advanced_slice_patterns, slice_patterns)]

fn e_simmetrico(elenco: &[u32]) -> bool {
    match elenco {
        &[] | &[_] => true,
        &[x, ref dentro.., y] if x == y => e_simmetrico(dentro),
        _ => false
    }
}

fn main() {
    let sim = &[0, 1, 4, 2, 4, 1, 0];
    assert!(e_simmetrico(sim));

    let non_sim = &[0, 1, 7, 2, 4, 1, 0];
    assert!(!e_simmetrico(non_sim));
}
```
