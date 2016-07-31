% Le stringhe

Le stringhe sono un concetto di cui è importante che ogni programmatore
si impadronisca. Il sistema di gestione delle stringhe di Rust è un po' diverso
da quello degli altri linguaggi, a causa del suo incentrarsi
sulla programmazione di sistema. Ogni volta che c'è una struttura dati
di dimensione variabile, le cose possono complicarsi, e le stringhe sono
una struttura dati ridimensionabile. Detto questo, le stringhe di Rust
funzionano diversamente anche da alcuni altri linguaggi di sistema, come il C.

Scaviamo nei dettagli. Una ‘stringa’ è una sequenza di valori scalari Unicode
codificati come flusso di byte UTF-8. Tutte le stringhe sono garantite essere
una ecodifica valida di sequenze UTF-8. In aggiunta, diversamente da alcuni
linguaggi di sistema, le stringhe non hanno un carattere terminatore e possono
contenere il carattere NUL, rappresentato dal byte 0.

Rust ha due principali tipi di stringhe: `&str` e `String`. Dapprima parliamo
di `&str`. Queste sono chiamate ‘slice di stringa’. Una slice di stringa ha
una dimensione fissa, e non può essere modificata. È un riferimento
a una sequenza di byte UTF-8.

```rust
let saluto = "Ciao là."; // saluto: &'static str
```

`"Ciao là."` è un letterale di stringa il cui tipo è `&'static str`.
Un letterale di stringa è uno slice di stringa che è allocato staticamente,
il che significa che è salvato dentro il nostro programma compilato, ed esiste
per l'intera durata dell'esecuzione. Il legamo `saluto` è un riferimento
a questa stringa staticamente allocata. Qualunque funzione che si aspetta
una slice di stringa accetterà anche un letterale di stringa.

I letterali di stringa possono estendersi su più righe. Ce ne sono due forme.
La prima includerà i caratteri a-capo e gli spazi che li seguono:

```rust
let s = "foo
    bar";

assert_eq!("foo\n    bar", s);
```

La seconda, con un `\`, rimuove gli a-capo e gli spazi che li seguono:

```rust
let s = "foo\
    bar";

assert_eq!("foobar", s);
```

Si noti che normalmente non si può accedere direttamente a una `str`,
ma solamente tramite un riferimento `&str`. Questo perché `str` è un tipo
non dimensionato che richiede informazioni aggiuntive in fase di esecuzione
per poter essere usata. Per avere maggiori informazioni, si veda il capitolo
sui [tipi non dimensionati][ut].

Però Rust ha di più oltre alle `&str`. Una `String` è una stringa allocata
sullo heap.
Questa string è estendibile, ed è anche garantita essere UTF-8. Le `String`
tipicamente sono create convertendo una slice di stringa, usando il metodo
`to_string`.

```rust
let mut s = "Ciao".to_string(); // mut s: String
println!("{}", s);

s.push_str(", mondo.");
println!("{}", s);
```

Le `String` vengono forzate ad essere un `&str` usando un `&`:

```rust
fn prendi_slice(slice: &str) {
    println!("Preso: {}", slice);
}

fn main() {
    let s = "Ciao".to_string();
    prendi_slice(&s);
}
```

Questa forzatura non avviene per le funzioni che accettano uno dei tratti
di `&str` invece di `&str` stessa. Per esempio, [`TcpStream::connect`][connect]
ha un argomento  di tipo `ToSocketAddrs`. Una `&str` va bene, ma una `String`
deve essere esplicitamente convertita usando `&*`.

```rust,no_run
use std::net::TcpStream;

TcpStream::connect("192.168.0.1:3000"); // argomento di tipo &str

let stringa_indirizzo = "192.168.0.1:3000".to_string();
TcpStream::connect(&*stringa_indirizzo); // converte stringa_indirizzo in &str
```

Vedere una `String` come una `&str` costa poco, ma convertire la `&str` in
una `String` comporta allocare della memoria. Non c'è ragione di farlo,
a meno che sia necessario!

## Indicizzazione

Siccome le stringhe sono UTF-8 valide, non supportano l'indicizzazione:

```rust,ignore
let s = "ciao";

println!("La prima lettera di s è {}", s[0]); // ERRORE!!!
```

Solitamente, l'accesso a un vettore con `[]` è molto veloce. Ma, siccome
ogni carattere una stringa codificata in UTF-8 può occupare più byte, si deve
percorrere la stringa per trovare l'ennesima lettera di una stringa.
Questa è un'operazione significativamente più costosa, e non si vuole essere
fuorvianti. Inoltre, il concetto di ‘lettera’ non è qualcosa di ben definito
in Unicode. Possiamo scegliere di guardare una stringa come una sequenza
di singoli byte, o come punti di codice ["codepoint"]:

```rust
let hachiko = "忠犬ハチ公";

for b in hachiko.as_bytes() {
    print!("{}, ", b);
}

println!("");

for c in hachiko.chars() {
    print!("{}, ", c);
}

println!("");
```

Questo stampa:

```text
229, 191, 160, 231, 138, 172, 227, 131, 143, 227, 131, 129, 229, 133, 172,
忠, 犬, ハ, チ, 公,
```

Come si vede, ci sono più byte che `char`.

Si può ottenere qualcosa di simile a un indice in questo modo:

```rust
# let hachiko = "忠犬ハチ公";
let dog = hachiko.chars().nth(1); // un po' come hachiko[1]
```

Questo evidenzia che dobbiamo percorrere la lista di `char` dall'inizio.

## Affettatura ["slicing"]

Si può ottenere una slice di una stringa con la sintassi dell'affettatura:

```rust
let dog = "hachiko";
let hachi = &dog[0..5];
```

Ma si noti che questi sono offset in _byte_, non offset in _caratturi_. Perciò
questo fallirà in fase di esecuzione:

```rust,should_panic
let dog = "忠犬ハチ公";
let hachi = &dog[0..2];
```

con questo errore:

```text
thread 'main' panicked at 'index 0 and/or 2 in `忠犬ハチ公` do not lie on
character boundary'
```

## Concatenazione

Avendo una `String`, si può concatenare una `&str` alla sua fine:

```rust
let ciao = "Ciao ".to_string();
let mondo = "mondo!";

let ciao_mondo = ciao + mondo;
```

Ma se avendo due `String`, serve un `&`:

```rust
let ciao = "Ciao ".to_string();
let mondo = "mondo!".to_string();

let ciao_mondo = ciao + &mondo;
```

Questo perché `&String` può essere automaticamente forzata in un `&str`.
Questa caratteristica si chiama ‘[forzatura `Deref`][dc]’.

[ut]: unsized-types.html
[dc]: deref-coercions.html
[connect]: ../std/net/struct.TcpStream.html#method.connect
