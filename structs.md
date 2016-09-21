% Le struct

Le `struct` sono un modo di creare tipi di dati più complessi. Per esempio, se
stessimo facendo calcoli che coinvolgono coordinate nello spazio 2D,
ci servirebbero sia un valore `x` che un valore `y`:

```rust
let origine_x = 0;
let origine_y = 0;
```

Una `struct` ci permette di combinare questi due oggetti in un singolo tipo
di dato unificato, i cui campi sono etichettati `x` e `y`:

```rust
struct Punto {
    x: i32,
    y: i32,
}

fn main() {
    let origine = Punto { x: 0, y: 0 }; // origine: Punto

    println!("L'origine è in ({}, {})", origine.x, origine.y);
}
```

Qui ci sono molte cose, analizziamole a piccole dosi.
Una `struct` si dichiara con
la parola-chiave `struct`, e poi con un nome. Per convenzione, i nomi delle
`struct`s iniziano con la lettera maiuscola e sono "camel cased": 
`PuntoNelloSpazio`, non `Punto_Nello_Spazio`, né `punto_nello_spazio`.

Possiamo creare un'istanza della nostra `struct` usando `let`, come al solito,
ma per impostare ogni campo usiamo una sintassi con lo stile `chiave: valore`.
L'ordine non dev'essere il medesimo della dichiarazione originale.

Infine, siccome i campi hanno un nome, possiamo accedervi tramite la notazione
a punto: `origine.x`.

I valori nelle `struct` sono immutabili di default, come gli altri legami
in Rust. Si deve usare `mut` per renderli mutabili:

```rust
struct Punto {
    x: i32,
    y: i32,
}

fn main() {
    let mut punto = Punto { x: 0, y: 0 };

    punto.x = 5;

    println!("Il punto è in ({}, {})", punto.x, punto.y);
}
```

Questo stamperà `Il punto è in (5, 0)`.

Rust non supporta la mutabilità dei campi a livello del linguaggio, quindi
non si può scrivere qualcosa così:

```rust,ignore
struct Point {
    mut x: i32,
    y: i32,
}
```

La mutabilità è una proprietà del legame, non della struttura stessa. Chi
fosse abituato all mutabilità a livello di campo, lo può trovare strano
dapprima, ma semplifica parecchio le cose. Consente perfino di rendere
temporaneamente mutabili degli oggetti:

```rust,ignore
struct Punto {
    x: i32,
    y: i32,
}

fn main() {
    let mut punto = Punto { x: 0, y: 0 };

    punto.x = 5;

    let punto = punto; // adesso è immutabile

    punto.y = 6; // questo provoca un errore
}
```

Però una struttura può contenere dei puntatori `&mut`, che consentono
di applicare qualche tipo di mutazione:

```rust
struct Punto {
    x: i32,
    y: i32,
}

struct RifPunto<'a> {
    x: &'a mut i32,
    y: &'a mut i32,
}

fn main() {
    let mut punto = Punto { x: 0, y: 0 };

    {
        let r = RifPunto { x: &mut punto.x, y: &mut punto.y };

        *r.x = 5;
        *r.y = 6;
    }

    assert_eq!(5, punto.x);
    assert_eq!(6, punto.y);
}
```

# Sintassi di aggiornamento

Una `struct` può comprendere `..` per indicare che si vuole usare una copia
di qualche altra `struct` per alcuni dei valori. Per esempio:

```rust
struct Punto3d {
    x: i32,
    y: i32,
    z: i32,
}

let mut punto = Punto3d { x: 0, y: 0, z: 0 };
punto = Punto3d { y: 1, .. punto };
```

Questo dà a `punto` una nuova `y`, ma mantiene i vecchi valori `x` e `z`. Non
deve essere nemmeno la medesima `struct`; si può usare questa sintassi quando
se ne creano di nuove, e si copiano i valori che non vengono specificati:

```rust
# struct Punto3d {
#     x: i32,
#     y: i32,
#     z: i32,
# }
let origine = Punto3d { x: 0, y: 0, z: 0 };
let punto = Punto3d { z: 1, x: 2, .. origine };
```

# Strutture ennuple

Rust ha un altro tipo di dato che è come un ibrido fra una [ennupla][ennupla]
e una `struct`, e si chiama ‘struttura ennupla’. Le strutture ennuple hanno
un nome, ma i loro campi no. Sono dichiarate con la parola-chiave `struct`,
e poi con un nome seguito da una ennupla:

[ennupla]: primitive-types.html#tuples

```rust
struct Colore(i32, i32, i32);
struct Punto(i32, i32, i32);

let nero = Colore(0, 0, 0);
let origine = Punto(0, 0, 0);
```

Qui, `nero` ed `origine` non sono uguali, anche se contengono gli
stessi valori.

Si può accedere ai membri di una struttura ennupla tramite la notazione a punto
o il `let` destrutturante, proprio come le normali ennuple:

```rust
# struct Colore(i32, i32, i32);
# struct Punto(i32, i32, i32);
# let nero = Colore(0, 0, 0);
# let origine = Punto(0, 0, 0);
let nero_r = nero.0;
let Punto(_, origine_y, origine_z) = origine;
```

I pattern come `Punto(_, origine_y, origine_z)` sono usati anche nelle
[espressioni match][match].

Un caso in cui una struttura ennupla è molto utile è quando ha un solo
elemento. Questo viene chiamato il pattern ‘newtype’, perché consente di creare
un nuovo tipo che è distinto da quello del suo valore contenuto ed esprime
anche un suo significato semantico:

```rust
struct Pollici(i32);

let lunghezza = Pollici(10);

let Pollici(lunghezza_intera) = lunghezza;
println!("la lunghezza è {} pollici", lunghezza_intera);
```

Come sopra, si può estrarre il tipo intero interno tramite un `let`
destrutturante. In questo caso, il `let Pollici(lunghezza_intera)` assegna `10`
a `lunghezza_intera`. Avremmo potuto usare la notazione a punto per fare
la stessa cosa:

```rust
# struct Pollici(i32);
# let lunghezza = Pollici(10);
let lunghezza_intera = lunghezza.0;
```

È sempre possibile usare una `struct` invece di una struttura ennupla, e può
essere più chiara. Avremmo potuto scrivere `Colore` e `Punto` anche così:

```rust
struct Colore {
    rosso: i32,
    blu: i32,
    verde: i32,
}

struct Punto {
    x: i32,
    y: i32,
    z: i32,
}
```

I buoni nomi sono importanti, e mentre si può fare riferimento ai valori in una
struttura ennupla anche con la notazione a punto, una `struct` ci dà dei nomi
effettivi piuttosto che delle posizioni.

[match]: match.html

# Struct simili a unità

Si può anche definire una `struct` senza nessun membro:

```rust
struct Elettrone {} // si usano le graffe vuote...
struct Protone;     // ...o solo un punto-e-virgola

// che la struct sia stata dichiarata con le graffe oppure no,
// si deve fare lo stesso quando se ne istanzia una
let x = Elettrone {};
let y = Protone;
```

Una tale `struct` è chiamata ‘simile ad unità’ ("unit-like")
perché somiglia alla ennupla
vuota, `()`, che talvolta è chiamata ‘unità’. Come una struttura ennupla,
definisce un nuovo tipo.

Questo tipo è usato raramente da solo (sebbene talvolta può servire come tipo
marcatore), ma in combinazione con altre caratteristiche, può diventare utile.
Per esempio, una libreria può chiedere di creare una struttura che implementi
un certo [tratto][tratto] per gestire eventi. Se non si hanno dati da mettere
nella struttura, si può creare `struct` simile ad unità.

[tratto]: traits.html
