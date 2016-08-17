% I pattern

I pattern sono molto comuni in Rust. Li usiamo nei [legami di variabile]
[bindings], nelle [espressioni match][match], e anche in altri posti. Facciamo
una carrellata di tutte le cose che i pattern possono fare!

[bindings]: variable-bindings.html
[match]: match.html

Un rapido ripasso: si può far combaciare direttamente con letterali, e
il carattere `_` agisce come caso ‘qualunque’:

```rust
let x = 1;

match x {
    1 => println!("uno"),
    2 => println!("due"),
    3 => println!("tre"),
    _ => println!("qualunque cosa"),
}
```

Questo stampa `uno`.

C'è un trabocchetto con i pattern: come ogni cosa che introduce
un nuovo legame, anche i pattern possono introdurre l'oscuramento. Per esempio:

```rust
let x = 1;
let c = 'c';

match c {
    x => println!("x: {} c: {}", x, c),
}

println!("x: {}", x)
```

Questo stampa:

```text
x: c c: c
x: 1
```

In altre parole, `x =>` combacia con il valore di `c` e introduce un nuovo
legame avente nome `x`. Questo nuovo legame è ha come ambito il braccio
di match e prende il valore di `c`. Si noti che il valore di `x` all'esterno
dell'ambito di match è ininfluente sul valore `x` al suo interno. Siccome
avevamo già un legame chiamato `x`, questo nuovo `x` lo oscura.

# Pattern multipli

Possiamo far combaciare più pattern usando `|`:

```rust
let x = 1;

match x {
    1 | 2 => println!("uno o due"),
    3 => println!("tre"),
    _ => println!("qualunque cosa"),
}
```

Questo stampa `uno o due`.

# Destrutturazione

Se si ha un tipo di dati composito, come una [`struct`][struct], lo si può
destrutturare dentro un pattern:

```rust
struct Punto {
    x: i32,
    y: i32,
}

let origine = Punto { x: 0, y: 0 };

match origine {
    Punto { x, y } => println!("({},{})", x, y),
}
```

[struct]: structs.html

Possiamo usare `:` per dare un altro nome a un valore.

```rust
struct Punto {
    x: i32,
    y: i32,
}

let origine = Punto { x: 0, y: 0 };

match origine {
    Punto { x: x1, y: y1 } => println!("({},{})", x1, y1),
}
```

Se ci interessano solamente alcuni valori, non dobbiamo dare dei nomi a tutti:

```rust
struct Punto {
    x: i32,
    y: i32,
}

let origine = Punto { x: 0, y: 0 };

match origine {
    Punto { x, .. } => println!("x is {}", x),
}
```

Questo stampa `x è 0`.

si può fare questo genere di match su qualunque membro, non solamente il primo:

```rust
struct Punto {
    x: i32,
    y: i32,
}

let origine = Punto { x: 0, y: 0 };

match origine {
    Punto { y, .. } => println!("y is {}", y),
}
```

Questo stampa `y è 0`.

Questo comportamento ‘destrutturante’ funziona su qualunque tipo di dati
composito, come le [ennuple][ennuple] o le [enum][enum].

[ennuple]: primitive-types.html#tuples
[enum]: enums.html

# Ignorare i legami

Si può usare `_` in un pattern per non tener conto del tipo e del valore.
Per esempio, ecco un `match` con un `Result<T, E>`:

```rust
# let qualche_valore: Result<i32, &'static str> = Err("C'era un errore");
match qualche_valore {
    Ok(valore) => println!("preso un valore: {}", valore),
    Err(_) => println!("è avvenuto un errore"),
}
```

Nel primo braccio, leghiamo il valore dentro la variante `Ok` a `valore`. Ma
nel braccio `Err`, usiamo `_` per non tener conto dello specifico errore, e
stampiamo un messaggio d'errore generico.

`_` è valido in qualunque pattern che crea un legame. Ciò può essere utile
per ignorare parti di una struttura più grande:

```rust
fn coordinate() -> (i32, i32, i32) {
    // genera e restituisci una terna
# (1, 2, 3)
}

let (x, _, z) = coordinate();
```

Qui, leghiamo il primo e l'ultimo elemento dell'ennupla a `x` e a `z`, ma
ignoriamo l'elemento di mezzo.

Vale la pena notare che usando `_` il valore combaciante non viene affatto
legato, il che comporta che tale valore non viene spostato:

```rust
let ennupla: (u32, String) = (5, String::from("cinque"));

// Qui, ennupla viene spostata, perché l'oggetto String è stato spostato:
let (x, _s) = ennupla;

// La prossima riga darebbe "error: use of partially moved value: `ennupla`"
// println!("L'ennupla è: {:?}", ennupla);

// Però,

let ennupla = (5, String::from("five"));

// Qui, ennupla _non_ vien spostata, dato che l'oggetto String non è
// mai stato spostato, e l'oggetto u32 è Copy:
let (x, _) = ennupla;

// Ciò comporta che questo funziona:
println!("L'ennupla è: {:?}", ennupla);
```

Ciò comporta anche che ogni variabile temporanea verrà distrutta alla fine
dell'istruzione:

```rust
// Qui, la String creata verrà distrutta immediatamente, dato che non è legata:
let _ = String::from("  hello  ").trim();
```

Si può usare anche `..` in un pattern per ignorare più valori:

```rust
enum EnnuplaOpzionale {
    Valore(i32, i32, i32),
    Mancante,
}

let x = EnnuplaOpzionale::Valore(5, -2, 3);

match x {
    EnnuplaOpzionale::Valore(..) => println!("Ho un'ennupla!"),
    EnnuplaOpzionale::Mancante => println!("Non ho tale fortuna."),
}
```

Questo stampa `Ho un'ennupla!`.

# `ref` e `ref mut`

Se si vuole ottenere un [riferimento][ref], si usi la parola-chiave `ref`:

```rust
let x = 5;

match x {
    ref r => println!("Ho un riferimento a {}", r),
}
```

Questo stampa `Ho un riferimento a 5`.

[ref]: references-and-borrowing.html

Qui, la `r` dentro il `match` ha il tipo `&i32`. In altre parole,
la parola-chiave `ref` _crea_ un riferimento, da usare nel pattern. Se serve
un riferimento mutabile, `ref mut` funzionerà allo stesso modo:

```rust
let mut x = 5;

match x {
    ref mut mr => println!("Ho un riferimento mutabile a {}", mr),
}
```

# Gamme

Si può far combaciare una gamma di valori usando `...`:

```rust
let x = 1;

match x {
    1 ... 5 => println!("da uno a cinque"),
    _ => println!("qualunque cosa"),
}
```

Questo stampa `da uno a cinque`.

Le gamme sono usate per lo più con gli interi e i `char`:

```rust
let x = '💅';

match x {
    'a' ... 'j' => println!("lettera precoce"),
    'k' ... 'z' => println!("lettera tardiva"),
    _ => println!("qualcos'altro"),
}
```

Questo stampa `qualcos'altro`.

# Legami

Si possono legare valori a nomi usando `@`:

```rust
let x = 1;

match x {
    e @ 1 ... 5 => println!("ho un elemento della gamma: {}", e),
    _ => println!("qualunque cosa"),
}
```

Questo stampa `ho un elemento della gamma: 1`. Questo operatore serve anche
quando si vuole estrarre una parte di una struttura dati complicata:

```rust
#[derive(Debug)]
struct Persona {
    nome: Option<String>,
}

let nome = "Steve".to_string();
let x: Option<Persona> = Some(Persona { nome: Some(nome) });
match x {
    Some(Persona { nome: ref a @ Some(_), .. }) => println!("{:?}", a),
    _ => {}
}
```

Questo stampa `Some("Steve")`: abbiamo legato il `nome` interno ad `a`.

Se si usa `@` con `|`, bisogna assicurarsi che il nome sia legato in ogni
parte del pattern:

```rust
let x = 5;

match x {
    e @ 1 ... 5 | e @ 8 ... 10 => println!("ho un elemento della gamma: {}", e),
    _ => println!("qualunque cosa"),
}
```

# Guardie

Si possono introdurre le ‘guardie di match’ usando `if`:

```rust
enum OptionalInt {
    Valore(i32),
    Mancante,
}

let x = OptionalInt::Valore(5);

match x {
    OptionalInt::Valore(i) if i > 5 => println!("Ho un int maggiore di cinque!"),
    OptionalInt::Valore(..) => println!("Ho un int!"),
    OptionalInt::Mancante => println!("Non ho tale fortuna."),
}
```

Questo stampa `Ho un int!`.

Se si sta usando `if` con più pattern, la `if` si applica a entrambi i lati:

```rust
let x = 4;
let y = false;

match x {
    4 | 5 if y => println!("sì"),
    _ => println!("no"),
}
```

Questo stampa `no`, perché la `if` si applica a tutta l'espressione `4 | 5`, e
non solamente al `5`. In altre parole, la precedenza di `if` si comporta così:

```text
(4 | 5) if y => ...
```

non così:

```text
4 | (5 if y) => ...
```

# Mescolare e abbinare

Urca! Ci sono molti modi diversi di far combaciare le cose, e tutti quanti
possono essere mescolati e abbinati, a seconda di ciò che si sta facendo:

```rust,ignore
match x {
    Foo { x: Some(ref nome), y: None } => ...
}
```

I pattern sono molto potenti. Facciamone buon uso.
