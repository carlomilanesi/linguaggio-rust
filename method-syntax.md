% Sintassi dei metodi

Le funzioni sono ottime, ma se si vuole chiamarne un po' su alcuni dati, può
diventare scomodo. Si consideri questo codice:

```rust,ignore
baz(bar(foo(x)));
```

Normalmente leggiamo questo codice da sinistra a destra, e quindi diciamo
‘baz di bar di foo di x’. Ma questo non è l'ordine con cui le funzioni
verrebbero chiamate; l'ordine di chiamata è invece il contrario:
‘applica a x prima foo, poi bar, e poi baz’.
Non sarebbe carino se potessimo scrivere il seguente codice?

```rust,ignore
x.foo().bar().baz();
```

Fortunatamente, come si potrebbe immaginare, si può! Rust fornisce
la capacità di usare questa ‘sintassi di chiamata di metodo’ tramite
la parola-chiave `impl`.

# Chiamate di metodo

Ecco come funziona:

```rust
struct Cerchio {
    x: f64,
    y: f64,
    raggio: f64,
}

impl Cerchio {
    fn area(&self) -> f64 {
        std::f64::consts::PI * (self.raggio * self.raggio)
    }
}

fn main() {
    let c = Cerchio { x: 0.0, y: 0.0, raggio: 2.0 };
    println!("{}", c.area());
}
```

Questo stamperà `12.566371`.

Abbiamo definito una `struct` che rappresenta un cerchio. Poi abbiamo scritto
un blocco `impl`, e al suo interno abbiamo definito un metodo, `area`.

I metodi prendono un primo argomento speciale, di cui ci sono tre varianti:
`self`, `&self`, e `&mut self`. Si può pensare a questo primo argomento come
se fosse il `foo` in `foo.bar()`. Le tre varianti corrispondono ai tre tipi
di cose che `foo` potrebbe essere: `self` se è un valore sullo stack,
`&self` se è un riferimento, e `&mut self` se è un riferimento mutabile.
Siccome abbiamo preso l'argomento `&self` da `area`, possiamo usarlo
come qualunque altro argomento. Siccome sappiamo che tale argomento è di tipo
`Cerchio`, possiamo accedere al suo membro `raggio` come faremmo con qualunque
altra `struct`.

Di regola dovremmo usare `&self`, dato che dovremmo preferire prendere
a prestito rispetto a prendere il possesso, e pure dovremmo preferire
prendere un riferimenti immutabili rispetto a qulli mutabili. Ecco un esempio
di tutte e tre le varianti:

```rust
struct Cerchio {
    x: f64,
    y: f64,
    raggio: f64,
}

impl Cerchio {
    fn riferimento(&self) {
       println!("presa di sé per riferimento!");
    }

    fn riferimento_mutabile(&mut self) {
       println!("presa di sé per riferimento mutabile!");
    }

    fn prendi_possesso(self) {
       println!("presa di possesso di sé!");
    }
}
```

Si possono usare tanti blocchi `impl` quanti se ne vuole. L'esempio precedente
poteva anche essere scritto così:

```rust
struct Cerchio {
    x: f64,
    y: f64,
    raggio: f64,
}

impl Cerchio {
    fn riferimento(&self) {
       println!("presa di sé per riferimento!");
    }
}

impl Cerchio {
    fn riferimento_mutabile(&mut self) {
       println!("presa di sé per riferimento mutabile!");
    }
}

impl Cerchio {
    fn prendi_possesso(self) {
       println!("presa di possesso di sé!");
    }
}
```

# Concatenamento di chiamate di metodi

Perciò, adesso sappiamo come chiamare un metodo, come `foo.bar()`. E che dire
del nostro esempio originale, `x.foo().bar().baz()`? Questo è chiamato
‘concatenamento di metodi’. Vediamo un esempio:

```rust
struct Cerchio {
    x: f64,
    y: f64,
    raggio: f64,
}

impl Cerchio {
    fn area(&self) -> f64 {
        std::f64::consts::PI * (self.raggio * self.raggio)
    }

    fn cresci(&self, incremento: f64) -> Cerchio {
        Cerchio { x: self.x, y: self.y, raggio: self.raggio + incremento }
    }
}

fn main() {
    let c = Cerchio { x: 0.0, y: 0.0, raggio: 2.0 };
    println!("{}", c.area());

    let d = c.cresci(2.0).area();
    println!("{}", d);
}
```

Verifica il tipo del valore restituito:

```rust
# struct Cerchio;
# impl Cerchio {
fn grow(&self, increment: f64) -> Cerchio {
# Cerchio } }
```

Diciamo che stiamo restituendo un `Cerchio`. Con questo metodo, possiamo
far crescere un nuovo `Cerchio` a qualunque dimensione.

# Funzioni associate

Si possono anche definire funzioni associate che non prendono un argomento
`self`. Ecco un pattern molto comune nel codice Rust:

```rust
struct Cerchio {
    x: f64,
    y: f64,
    raggio: f64,
}

impl Cerchio {
    fn new(x: f64, y: f64, raggio: f64) -> Cerchio {
        Cerchio {
            x: x,
            y: y,
            raggio: raggio,
        }
    }
}

fn main() {
    let c = Cerchio::new(0.0, 0.0, 2.0);
}
```

Questa ‘funzione associata’ ci costruisce un nuovo `Cerchio`. Si noti che
le funzioni associate vengono chiamate usando la sintassi `Struct::function()`,
invece che con la sintassi `ref.method()`. In alcuni altri linguaggi,
le funzioni associate sono chiamate ‘funzioni membro statiche’
o ‘metodi statici’ o ‘metodi di classe’.

# Il pattern del costruttore

Diciamo che vogliamo che i nostri utenti possano creare delle istanze
di `Cerchio`, ma permetteremo loro di impostare solamente le proprietà
a cui sono interessati. Se non specificati, gli attributi `x` e `y` varranno
`0.0`, e l'attributo `raggio` varrà `1.0`. Rust non ha il sovraccaricamento
dei metodi, né gli argomenti con nome, né un numero variabile di argomenti.
Invece si impiega il pattern del costruttore. Si presenta così:

```rust
struct Cerchio {
    x: f64,
    y: f64,
    raggio: f64,
}

impl Cerchio {
    fn area(&self) -> f64 {
        std::f64::consts::PI * (self.raggio * self.raggio)
    }
}

struct CostruttoreDiCerchi {
    x: f64,
    y: f64,
    raggio: f64,
}

impl CostruttoreDiCerchi {
    fn new() -> CostruttoreDiCerchi {
        CostruttoreDiCerchi { x: 0.0, y: 0.0, raggio: 1.0, }
    }

    fn x(&mut self, coordinata: f64) -> &mut CostruttoreDiCerchi {
        self.x = coordinata;
        self
    }

    fn y(&mut self, coordinata: f64) -> &mut CostruttoreDiCerchi {
        self.y = coordinata;
        self
    }

    fn raggio(&mut self, raggio: f64) -> &mut CostruttoreDiCerchi {
        self.raggio = raggio;
        self
    }

    fn finalizza(&self) -> Cerchio {
        Cerchio { x: self.x, y: self.y, raggio: self.raggio }
    }
}

fn main() {
    let c = CostruttoreDiCerchi::new()
        .x(1.0)
        .y(2.0)
        .raggio(2.0)
        .finalizza();

    println!("area: {}", c.area());
    println!("x: {}", c.x);
    println!("y: {}", c.y);
}
```

Ciò che abbiamo fatto qui è creare un'altra `struct`, `CostruttoreDiCerchi`.
Su di essa abbiamo definito i nostri metodi di costruttore. Abbiamo anche
definito il nostro metodo `area()` su `Cerchio`. Inoltre abbiamo creato
un altro metodo su `CostruttoreDiCerchi`: `finalizza()`. Questo metodo crea
il nostro `Cerchio` finale dal costruttore. Adesso, abbiamo usato il sistema
dei tipi per imporre le nostre intenzioni: possiamo usare i metodi su
`CostruttoreDiCerchi` per vincolare la costruzione di istanze di `Cerchio`
in qualunque modo desideriamo.
