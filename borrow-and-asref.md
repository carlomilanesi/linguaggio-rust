% Borrow e AsRef

I tratti [`Borrow`][borrow] e [`AsRef`][asref] sono molto simili, ma diversi.
Ecco un rapido ripasso di quello che questi due tratti significano.

[borrow]: ../std/borrow/trait.Borrow.html
[asref]: ../std/convert/trait.AsRef.html

# Borrow

Il tratto `Borrow` si usa quando si sta scrivendo una struttura dati,
e, per qualche scopo, si vuole usare come sinonimo un tipo o posseduto
o preso in prestito.

Per esempio, [`HashMap`][hashmap] ha un metodo [`get`][get] che usa `Borrow`:

```rust,ignore
fn get<Q: ?Sized>(&self, k: &Q) -> Option<&V>
    where K: Borrow<Q>,
          Q: Hash + Eq
```

[hashmap]: ../std/collections/struct.HashMap.html
[get]: ../std/collections/struct.HashMap.html#method.get

Questa firma è parecchio complicata. Quello che ci interessa qui è
il parametro `K`. Si riferisce a un parametro di `HashMap` stesso:

```rust,ignore
struct HashMap<K, V, S = RandomState> {
```

Il parametro `K` è il tipo della _chiave_ usata da `HashMap`. Quindi,
riguardando la firma di `get()`, vediamo che possiamo usare`get()` quando
la chiave implementa `Borrow<Q>`. In quel modo, possiamo costruire
un `HashMap` che usa chiavi `String`, ma usare delle `&str` quando cerchiamo:

```rust
use std::collections::HashMap;

let mut map = HashMap::new();
map.insert("Foo".to_string(), 42);

assert_eq!(map.get("Foo"), Some(&42));
```

Questo perché la libreria standard ha `impl Borrow<str> for String`.

Per la maggior parte dei tipi, quando si vuole prenderne un tipo posseduto
o preso in prestito, un `&T` può bastare. Ma un'area dove `Borrow` è efficace
è quando c'è più di un genere di valore preso in prestito. Ciò è vero
specialmente per i riferimenti e le slice: si può avere sia un `&T`
che un `&mut T`. Se vogliamo accettare entrambi questi tipi, `Borrow` è
quel che ci vuole:

```rust
use std::borrow::Borrow;
use std::fmt::Display;

fn foo<T: Borrow<i32> + Display>(a: T) {
    println!("a è preso in prestito: {}", a);
}

let mut i = 5;

foo(&i);
foo(&mut i);
```

Questo stamperà due volte `a è preso in prestito: 5`.

# AsRef

Il tratto `AsRef` è un tratto di conversione. Serve a convertire
qualche valore in un riferimento nel codice generico. Così:

```rust
let s = "Hello".to_string();

fn foo<T: AsRef<str>>(s: T) {
    let slice = s.as_ref();
}
```

# Quale usare?

Come si vede, sono in qualche modo uguali: entrambi trattano versioni
possedute o prese in prestito di qualche tipo. Però, sono un po' diversi.

Si scelga `Borrow` quando si vuole astrarre su diversi generi di prestiti,
oppure quando si sta costruendo una struttura dati che tratta i valori
posseduti e i valori presi in prestito in modi equivalenti, come lo hashing
e il confronto.

Si scelga `AsRef` quando si vuole convertire qualcosa direttamente
a un riferimento, e si sta scrivendo del codice generico.
