% Le forzature `Deref`

La libreria standard fornisce un tratto speciale, [`Deref`][deref].
Normalmente lo si usa per sovraccaricare `*`, l'operatore di dereferenziazione:

```rust
use std::ops::Deref;

struct EsempioDeref<T> {
    valore: T,
}

impl<T> Deref for EsempioDeref<T> {
    type Target = T;

    fn deref(&self) -> &T {
        &self.valore
    }
}

fn main() {
    let x = EsempioDeref { valore: 'a' };
    assert_eq!('a', *x);
}
```

[deref]: ../std/ops/trait.Deref.html

Questo è utile per scrivere dei tipi puntatore personalizzati. Però, c'è
una caratteristica del linguaggio correlata a `Deref`: le ‘forzature deref’.
Ecco la regola: Se si ha un tipo `U`, che implementa `Deref<Target=T>`,
i valori di tipo `&U` automaticamente potranno essere forzati al tipo `&T`.
Ecco un esempio:

```rust
fn foo(s: &str) {
    // prendi in prestito una stringa per un secondo
}

// String implementa Deref<Target=str>
let posseduta = "Hello".to_string();

// perciò, questo funziona:
foo(&posseduta);
```

Usare una e-commerciale davanti a un valore prende un riferimento ad esso.
Perciò `posseduta` è una `String`, `&posseduta` è un `&String`, e dato che
esiste `impl Deref<Target=str> for String`, si potrà chiamare `deref` su
`&String`, ottenendo un `&str`, che è accettato da `foo()`.

Ecco. Questa regola è un dei soli posti in cui Rust fa una conversione
automatica, ma ciò aggiunge molta flessibilità. Per esempio,
il tipo `Rc<T>` implementa `Deref<Target=T>`, e quindi questo funziona:

```rust
use std::rc::Rc;

fn foo(s: &str) {
    // prende in prestito una stringa per un secondo
}

// String implementa Deref<Target=str>
let posseduta = "Hello".to_string();
let contata = Rc::new(posseduta);

// perciò, questo funziona:
foo(&counted);
```

Quel che abbiamo fatto è avvolgere la nostra `String` in un `Rc<T>`. Ma adesso
possiamo passare in giro la `Rc<String>` ovunque sia accettata una `String`.
La firma di `foo` non è cambiata, ma funziona altrettanto con entrambi i tipi.
Questo esempio esegue due conversioni: da `Rc<String>` a `String`, e poi
da `String` a `&str`. Rust lo farà tante volte quanto possibile, fino a che
i tipi combaciano.

Un'altra implementazione molto tipica fornita dalla libreria standard è:

```rust
fn foo(s: &[i32]) {
    // prende in prestito una slice per un secondo
}

// Vec<T> implementa Deref<Target=[T]>
let posseduta = vec![1, 2, 3];

foo(&posseduta);
```

I vettori possono essere convertiti da `Deref` in una slice.

## Deref e le chiamate di metodo

`Deref` scatterà anche quando si chiama un metodo. Si consideri il seguente
esempio.

```rust
struct Foo;

impl Foo {
    fn foo(&self) { println!("Foo"); }
}

let f = &&Foo;

f.foo();
```

Anche se `f` è un `&&Foo` e `foo` prende invece un `&self`, questo funziona.
È perché le seguenti espressioni sono equivalenti:

```rust,ignore
f.foo();
(&f).foo();
(&&f).foo();
(&&&&&&&&f).foo();
```

Si possono chiamare su valore di tipo `&&&&&&&&&&&&&&&&Foo` anche metodi
definiti per `Foo`, perché il compilatore inserià tante operazioni `*`
quante ne servono per avere il tipo appropriato. E dato che sta inserendo
operatori `*`, usa `Deref`.
