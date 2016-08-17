% Gli oggetti-tratto

Quando del codice coinvolge del polimorfismo, serve un meccanismo
per determinare quale specifica versione viene effettivamente eseguita.
Questo si chiama ‘dispatch’ (in italiano, "disbrigo"). Ci sono due forme
principali di dispatch: il dispatch statico e il dispatch dinamico. Rust
favorisce il dispatch statico, ma supporta anche il dispatch dinamico
tramite un meccanismo chiamato ‘oggetti-tratto’ ["trait object"].

## Premesse

Per il resto di questa sezione, ci serviranno un tratto e alcune
implementazioni. Facciamone una semplice, `Foo`. Ha un metodo che ci si
aspetta che restituisca una `String`.

```rust
trait Foo {
    fn metodo(&self) -> String;
}
```

Implementeremo anche questo tratto per `u8` e `String`:

```rust
# trait Foo { fn metodo(&self) -> String; }
impl Foo for u8 {
    fn metodo(&self) -> String { format!("u8: {}", *self) }
}

impl Foo for String {
    fn metodo(&self) -> String { format!("string: {}", *self) }
}
```

## Dispatch statico

Possiamo usare questo tratto per eseguire un dispatch static con i legami
dei tratti:

```rust
# trait Foo { fn metodo(&self) -> String; }
# impl Foo for u8 { fn metodo(&self) -> String { format!("u8: {}", *self) } }
# impl Foo for String { fn metodo(&self) -> String { format!("string: {}", *self) } }
fn fai_qualcosa<T: Foo>(x: T) {
    x.metodo();
}

fn main() {
    let x = 5u8;
    let y = "Hello".to_string();

    fai_qualcosa(x);
    fai_qualcosa(y);
}
```

Qui Rust usa la ‘monomorfizzazione’ per eseguire il dispatch statico. Questo
significa che Rust creerà una versione speciale di `fai_qualcosa()` sia
per `u8` che per `String`, e poi sostituirà i punti di chiamata con chiamate
a queste funzioni specializzate. In altre parole, Rust genera qualcosa così:

```rust
# trait Foo { fn metodo(&self) -> String; }
# impl Foo for u8 { fn metodo(&self) -> String { format!("u8: {}", *self) } }
# impl Foo for String { fn metodo(&self) -> String { format!("string: {}", *self) } }
fn fai_qualcosa_u8(x: u8) {
    x.metodo();
}

fn fai_qualcosa_string(x: String) {
    x.metodo();
}

fn main() {
    let x = 5u8;
    let y = "Hello".to_string();

    fai_qualcosa_u8(x);
    fai_qualcosa_string(y);
}
```

Questo ha un aspetto molto positivo: il dispatch statico consente che
le chiamate di funzione siano espanse in linea, perché il chiamato è noto
in fase di compilazione, e l'espansione in linea è la chiave per una buona
ottimizzazione. Il dispatch statico è veloce, ma si paga: il gonfiore
del codice ["‘code bloat’"], dovuto alle numerose copie della stessa funzione
esistente nel binario, uno per ogni tipo.

Inoltre, i compilatori non sono perfetti e possono “ottimizzare” il codice
fino a farlo diventare più lento. Per esempio, le funzioni espanse in linea
troppo avidamente riempiranno la cache delle istruzioni (le cache governano
le prestazioni). Questo fa parte della ragione per cui `#[inline]`
e `#[inline(always)]` dovrebbero essere usati attentamente, e una ragione
per cui usare un dispatch in alcuni casi è più efficiente.

Però, il caso più tipico è quello che è più efficiente usare il dispatch
statico, e si può sempre avere una piccola funzione chiamata con dispatch
statico che esegue il dispatch dinamico, mentre non è possibile il contrario,
ossia trasformate il dispatch dinamico in statico, il che significa che
le chiamate statiche sono più flessibili. Per questo ragione, la libreria
standard cerca di usare il dispatch statico il più possibile.

## Dispatch dinamico

Rust fornisce il dispatch dinamico tramite una caratteristica chiamata
‘oggetti-tratto’. Gli oggetti-tratto, come `&Foo` o `Box<Foo>`, sono valori
normali che immagazzinano un valore di *qualunque* tipo che implementa il dato
tratto, dove il preciso tipo può essere noto solo in fase di esecuzione.

Un oggetto-tratto può essere ottenuto da un puntatore a un tipo concreto che
implementa il tratto, *convertendolo* (per es. `&x as &Foo`) o *forzandolo*
(per es. usando `&x` come argomento a una funzione che prende un `&Foo`).

Queste forzature e conversioni di oggetti-tratto funzionano anche per
i puntatori come `&mut T` convertito in `&mut Foo`, e per `Box<T>`
convertito in `Box<Foo>`, e per il momento per nient'altro. Le forzature e
le conversioni sono identiche.

Questa operazione può essere vista come un ‘cancellare’ la conoscenza che
il compilatore ha sullo specifico tipo del puntatore, e quindi talvolta si fa
riferimento agli oggetti-tratto come a ’cancellazioni di tipo’.

Tornando all'esempio di prima, possiamo usare il medesimo tratto per eseguire
un dispatch dinamico con gli oggetti-tratto, sia tramite la conversione:

```rust
# trait Foo { fn metodo(&self) -> String; }
# impl Foo for u8 { fn metodo(&self) -> String { format!("u8: {}", *self) } }
# impl Foo for String { fn metodo(&self) -> String { format!("string: {}", *self) } }

fn fai_qualcosa(x: &Foo) {
    x.metodo();
}

fn main() {
    let x = 5u8;
    fai_qualcosa(&x as &Foo);
}
```

che tramite la forzatura:

```rust
# trait Foo { fn metodo(&self) -> String; }
# impl Foo for u8 { fn metodo(&self) -> String { format!("u8: {}", *self) } }
# impl Foo for String { fn metodo(&self) -> String { format!("string: {}", *self) } }

fn fai_qualcosa(x: &Foo) {
    x.metodo();
}

fn main() {
    let x = "Hello".to_string();
    fai_qualcosa(&x);
}
```

Una funzione che prende un oggetto-tratto non è specializzata per ognuno
dei tipi che implementano `Foo`: ne viene generata solamente una copia, con
la conseguenza che spesso (ma non sempre) si riduce il gonfiore di codice.
Però, ciò comporta il costo di richiedere le più lente chiamate di funzioni
virtuali, e di inibire ogni possibilità di espandere in linea e di applicare
le relative ottimizzazioni.

### Perché i puntatori?

Rust non mette le cose dietro un puntatore di default, diversamente da molti
linguaggi gestiti ["managed"], e quindi i tipi possono avere dimensioni
diverse. Conoscere la dimensione del valore in fase di compilazione è
importante per cose come passarlo come argomento a una funzione, spostarlo
in giro per lo stack, e allocare (e deallocare) spazio sullo heap
per immagazzinarlo.

Per `Foo`, avrebbo bisogno di avere un valore che potesse rappresentare almeno
una `String` (24 byte) o un `u8` (1 byte), e così puro qualunque altro tipo
per cui i crate dipendenti possono implementare `Foo` (assolutamente qualunque
numero di bytes). Non c'è modo di garantire che quest'ultimo punto possa
funzionare se i valori vengono immagazzinati senza un puntatore, perché
quegli altri tipi possono essere arbitrariamente grandi.

Mettere il valore dietro un puntatore significa che la dimensione di tale
valore non è rilevante quando si maneggia un oggetto-tratto, e importa solo
la dimensione del puntatore stesso.

### Rappresentazione

I metodi del tratto possono essere chiamati su un oggetto-tratto tramite
uno speciale record di puntatori di funzione tradizionalmente chiamato ‘vtable’
(creato e gestito dal compilatore).

Gli oggetti-tratto sono sia semplici che complicati: la loro rappresentazione e
disposizione interna è davvero immediata, ma ci sono da scoprire
alcuni messaggi d'errore contorti e alcuni comportamenti sorprendenti.

Iniziamo dal facile, con la rappresentazione in fase di esecuzione di un
oggetto-tratto. Il modulo `std::raw` contiene le struct con le disposizioni
che sono le medesime dei complicati tipi incorporati, [compresi
gli oggetti-tratto][stdraw]:

```rust
# mod foo {
pub struct TraitObject {
    pub data: *mut (),
    pub vtable: *mut (),
}
# }
```

[stdraw]: ../std/raw/struct.TraitObject.html

Cioè, un oggetto-tratto come `&Foo` consiste in un puntatore ‘data’
e un puntatore ‘vtable’.

Il puntatore data indirizza i dati (di qualche tipo sconosciuto `T`)
che l'oggetto-tratto sta immagazzinando, e il puntatore vtable punta
alla vtable (‘tabella dei metodi virtuali’) corrispondente all'implementazione
di `Foo` per `T`.

Una vtable è essenzialmente una struct di puntatori a funzione, che puntano
al pezzo concreto di codice macchina per ogni metodo nell'implementazione.
Una chiamata di metodo come `oggetto_tratto.metodo()` recupererà il puntatore
corretto dalla vtable e poi farà una chiamata dinamica ad esso. Per esempio:

```rust,ignore
struct FooVtable {
    destructor: fn(*mut ()),
    size: usize,
    align: usize,
    method: fn(*const ()) -> String,
}

// u8:

fn call_method_on_u8(x: *const ()) -> String {
    // il compilatore garantisce che questa funzione è chiamata solamente
    // con `x` che punta a un u8
    let byte: &u8 = unsafe { &*(x as *const u8) };

    byte.method()
}

static Foo_for_u8_vtable: FooVtable = FooVtable {
    destructor: /* magia del compilatore */,
    size: 1,
    align: 1,

    // cast a un puntatore a funzione
    method: call_method_on_u8 as fn(*const ()) -> String,
};


// String:

fn call_method_on_String(x: *const ()) -> String {
    // il compilatore garantisce che questa funzione è chiamata solamente
    // con `x` che punta a una String
    let string: &String = unsafe { &*(x as *const String) };

    string.method()
}

static Foo_for_String_vtable: FooVtable = FooVtable {
    destructor: /* magia del compilatore */,
    // questi sono i valori per un target a 64-bit;
    // bisogna dimezzarli per un target a 32-bit
    size: 24,
    align: 8,

    method: call_method_on_String as fn(*const ()) -> String,
};
```

Il campo `destructor` in ogni vtable punta a una funzione che rilascerà
qualunque risorsa del tipo vtable: per `u8` è banale, ma per `String` libererà
della memoria. Questo è necessario per gli oggetti-tratto che possiedono, come
`Box<Foo>`, che ha bisogno di rilasciare sia l'allocazione di `Box` che
il tipo interno, quando escono dall'ambito. I campi `size` e `align`
immagazzinano la dimensione del tipo cancellato, nonché i suoi requisiti
di allineamento; questi campi sono essenzialmente inutilizzati al momento,
dato che questa informazione è incorporata nel distruttore, ma verrà usata in
futuro, dato che gli oggetti-tratto sono resi progressivamente più flessibili.

Supponiamo di avere alcuni valori che implementano `Foo`. La forma esplicita
di costruzione e utilizzo degli oggetti-tratti di `Foo` potrebbe sembrare
un po' così (ignorando gli errori di tipo: comunque sono tutti puntatori):

```rust,ignore
let a: String = "foo".to_string();
let x: u8 = 1;

// let b: &Foo = &a;
let b = TraitObject {
    // immagazzina i dati
    data: &a,
    // immagazzina i metodi
    vtable: &Foo_for_String_vtable
};

// let y: &Foo = x;
let y = TraitObject {
    // immagazzina i dati
    data: &x,
    // immagazzina i metodi
    vtable: &Foo_for_u8_vtable
};

// b.method();
(b.vtable.method)(b.data);

// y.method();
(y.vtable.method)(y.data);
```

## Sicurezza come oggetto

Non tutti i tratti possono essere usati per costruire un oggetto-tratto.
Per esempio, i vettori implementano `Clone`, ma se proviamo a costruirne
un oggetto-tratto:

```rust,ignore
let v = vec![1, 2, 3];
let o = &v as &Clone;
```

Otteniamo un errore:

```text
error: cannot convert to a trait object because trait `core::clone::Clone` is not object-safe [E0038]
let o = &v as &Clone;
        ^~
note: the trait cannot require that `Self : Sized`
let o = &v as &Clone;
        ^~
```

L'errore dice che `Clone` non è ‘object-safe’, cioè ’sicuro come oggetto’.
Solamente i tratti che sono sicuri come oggetti possono essere trasformati
in oggetti-tratto. Un tratto è sicuro come oggetto se sono vere entrambe
le seguenti proprietà:

* il tratto non richiede che valga `Self: Sized`
* tutti i suoi metodi sono sicuri come oggetti

Ma che cosa rende un metodo sicuro come oggetto? Ogni metodo
deve richiedere che valga `Self: Sized` oppure che valga tutto il seguente:

* deve non avere parametri di tipo
* deve non usare `Self`

Urca! Come si può vedere, quasi tutte queste regole parlano di `Self`.
Una buona intuizione è “eccetto in circostanze speciali, se il metodo
del tratto usa `Self`, non è object-safe.”
