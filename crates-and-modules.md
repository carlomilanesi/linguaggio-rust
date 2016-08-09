% I crate e i moduli

Quando un progetto inizia a diventare grande, è considerata buona pratica
di ingegneria del software scomporlo in un gruppo di pezzi più piccoli, e poi
incastrarli insieme. È importante anche avere un'interfaccia bene definita,
così che parte delle proprie funzionalità siano private, e parte pubbliche.
Per facilitare questo genere di cose, Rust ha un sistema di moduli.

# Terminologia di base: i crate e i moduli

Rust ha due termini distinti relativi al sistema dei moduli: ‘crate’ e ‘modulo’
["‘module’"]. Un crate è sinonimo di ‘libreria’ o di ‘pacchetto’ in altri
linguaggi. Da questo deriva il nome di “Cargo”, lo strumento di gestione
dei pacchetti di Rust: si inviano i propri "cassoni" ["crate"] ad altri
usando Cargo. Un crate può produrre un programma o una libreria, a seconda
del progetto.

Ogni crate ha un *modulo radice* implicito che contiene il codice
per quel crate. Poi si può definire un albero di sotto-moduli sotto quel
modulo radice. I moduli permettono di partizionare il proprio codice
internamente allo stesso crate.

Come esempio, creiamo un create *frasi*, che ci darà varie frasi in diverse
lingue. Per mantenerlo semplice, ci limiteremo ai due categorie di frasi
‘saluti’ e ‘commiati’, e useremo solamente le due lingue inglese e giapponese
(日本語) per scrivere quelle frasi. Useremo questa organizzazione di moduli:

```text
                                  +--------+
                              +---| saluti |
                +---------+   |   +--------+
            +---| inglese |---+
            |   +---------+   |   +----------+
            |                 +---| commiati |
+-------+   |                     +----------+
| frasi |---+
+-------+   |                       +--------+
            |                   +---| saluti |
            |   +------------+  |   +--------+
            +---| giapponese |--+
                +------------+  |   +----------+
                                +---| commiati |
                                    +----------+
```

In questo esempio, `frasi` è il nome del nostro crate. Tutto il resto sono
moduli. Si può vedere che formano un albero, che si dirama dal crate *radice*,
che è la radice dell'albero: `frasi`.

Adesso che abbiamo un piano, definiamo questi moduli nel codice. Per iniziare,
generiamo un nuovo crate usando Cargo:

```bash
$ cargo new frasi
$ cd frasi
```

Come già detto, questo genera un semplice progetto:

```bash
$ tree .
.
├── Cargo.toml
└── src
    └── lib.rs

1 directory, 2 files
```

`src/lib.rs` è la radice del nostro crate, corrispondente al `frasi` nello
schema disegnato prima.

# Definire i moduli

Per definire ogni modulo, usiamo la parola-chiave `mod`. Facciamo che il nostro
`src/lib.rs` si presenti così:

```rust
mod inglese {
    mod saluti {
    }

    mod commiati {
    }
}

mod giapponese {
    mod saluti {
    }

    mod commiati {
    }
}
```

Dopo la parola-chiave `mod`, si mette il nome del modulo. I nomi dei moduli
seguono le convenzioni degli altri identificatori di Rust: `minuscolo
a serpente`. Il contenuto di ogni modulo sta entro graffe (`{}`).

Entro un dato `mod`, si possono dichiarare dei sotto-`mod`. Si può far
riferimento a sotto-moduli usando la notazione (`::`): i nostri quattro moduli
annidati sono `inglese::saluti`, `inglese::commiati`, `giapponese::saluti`, e
`giapponese::commiati`. Siccome questi sotto-moduli sono definiti all'interno
dei loro moduli genitore, i loro nomi non sono ambigui: `inglese::saluti` e
`giapponese::saluti` sono distinti, anche se i loro nomi sono entrambi
`saluti`.

Siccome questo crate non ha una funzione `main()`, e si chiama `lib.rs`,
Cargo costruirà questo crate come libreria:

```bash
$ cargo build
   Compiling frasi v0.0.1 (file:///home/you/projects/frasi)
$ ls target/debug
build  deps  examples  libfrasi-a7448e02a0468eaa.rlib  native
```

`libfrasi-<hash>.rlib` è il crate compilato. Prima di vedere come usare
questo crate da un altro crate, scomponiamolo in più file.

# Crate aventi più file

Se ogni crate stesse in un solo file, questo file diventerebbe molto grande.
Spesso è più facile scomporre i crate in più file, e Rust supporta
questa operazione in due modi.

Invece di dichiarare un modulo così:

```rust,ignore
mod inglese {
    // il contenuto del modulo va qui
}
```

Si può invece dichiarare il modulo così:

```rust,ignore
mod inglese;
```

Se facciamo così, Rust si aspetterà di trovare o un file `inglese.rs`, o
un file `inglese/mod.rs` con il contenuto del modulo.

Si noti che in questi file, non c'è bisogno di ri-dichiarare il modulo:
è già stato fatto con la dichiarazione `mod` iniziale.

Usando queste due tecniche, possiamo scomporre il nostro crate
in due directory e sette file:

```bash
$ tree .
.
├── Cargo.lock
├── Cargo.toml
├── src
│   ├── inglese
│   │   ├── commiati.rs
│   │   ├── saluti.rs
│   │   └── mod.rs
│   ├── giapponese
│   │   ├── commiati.rs
│   │   ├── saluti.rs
│   │   └── mod.rs
│   └── lib.rs
└── target
    └── debug
        ├── build
        ├── deps
        ├── examples
        ├── libfrasi-a7448e02a0468eaa.rlib
        └── native
```

`src/lib.rs` è la radice del nostro crate, e si presenta così:

```rust,ignore
mod inglese;
mod giapponese;
```

Queste due dichiarazioni dicono a Rust di cercare o `src/inglese.rs` e
`src/giapponese.rs`, oppure `src/inglese/mod.rs` e `src/giapponese/mod.rs`,
a seconda della nostra preferenza. In questo caso, siccome i nostri moduli
hanno dei sotto-moduli, abbiamo scelto la seconda possibilità.
Sia `src/inglese/mod.rs` che `src/giapponese/mod.rs` si presentano così:

```rust,ignore
mod saluti;
mod commiati;
```

Di nuovo, queste dichiarazioni dicono a Rust di cercare o
`src/inglese/saluti.rs`, `src/inglese/commiati.rs`,
`src/giapponese/saluti.rs` e `src/giapponese/commiati.rs` oppure
`src/inglese/saluti/mod.rs`, `src/inglese/commiati/mod.rs`,
`src/giapponese/saluti/mod.rs` e `src/giapponese/commiati/mod.rs`.
Siccome questi sotto-moduli non hanno i loro sotto-moduli, abbiamo scelto
la prima possibilità. Urca!

I file `src/inglese/saluti.rs`, `src/inglese/commiati.rs`,
`src/giapponese/saluti.rs` e `src/giapponese/commiati.rs` al momento sono
vuoti. Aggiungiamoci alcune funzioni.

Nel file `src/inglese/saluti.rs` si metta questa:

```rust
fn ciao() -> String {
    "Hello!".to_string()
}
```

Nel file `src/inglese/commiati.rs` si metta questa:

```rust
fn arrivederci() -> String {
    "Goodbye.".to_string()
}
```

Nel file `src/giapponese/saluti.rs` si metta questa:

```rust
fn ciao() -> String {
    "こんにちは".to_string()
}
```

Naturalmente, lo si può copiare e incollare da questa pagina web, oppure
digitare qualcos'altro. Non è importante mettere proprio ‘konnichiwa’
per imparare il sistema dei moduli.

E nel file `src/giapponese/commiati.rs` si metta questa:

```rust
fn arrivederci() -> String {
    "さようなら".to_string()
}
```

(Questo commiato è ‘Sayōnara’, per chi interessasse.)

Adesso che abbiamo un po' di funzionalità nel nostro crate, proviamo a usarlo
da un altro crate.

# Importare crate esterni

Abbiamo un crate di libreria. Creaiamo un crate di programma che importa e usa
la nostra libreria.

Creiamo il file `src/main.rs` e mettiamoci dentro questo (anche se
non compilerà ancora):

```rust,ignore
extern crate frasi;

fn main() {
    println!("Ciao in inglese: {}", frasi::inglese::saluti::ciao());
    println!("Arrivederci in inglese: {}", frasi::inglese::commiati::arrivederci());

    println!("Ciao in giapponese: {}", frasi::giapponese::saluti::ciao());
    println!("Arrivederci in giapponese: {}", frasi::giapponese::commiati::arrivederci());
}
```

La dichiarazione `extern crate` dice a Rust che deve linkare il programma
con il crate `frasi`. Poi possiamo usare il modulo `frasi`’ in tale crate.
Come detto prima, si possono usare i doppi due-punti per riferirsi
ai sotto-moduli e alle funzioni al loro interno.

(Nota: quando si importa un crate che ha trattini nel suo nome "come-questo",
che non è un identificatore Rust valido, tale nome verrà convertito sostituendo
i trattini con underscore ("_"), e quindi si dovrà scrivere
`extern crate come_questo;`.)

Inoltre, Cargo assume che il file `src/main.rs` sia il crate radice di
un crate di programma, invece che un crate di libreria. Il nostro pacchetto
adesso contiene due crate: `src/lib.rs` e `src/main.rs`. Questo pattern è
parecchio tipico per i crate di programma: la maggior parte della funzionalità
sta in un crate di libreria, e il crate di programma usa quella libreria.
In questo modo, anche altri programmi possono usare il crate di libreria,
ed è anche una bella separazione di compiti.

Però quello che abbiamo scritto finora non funziona ancora. Otteniamo quattro
errori analoghi al seguente:

```bash
$ cargo build
   Compiling frasi v0.0.1 (file:///home/you/projects/frasi)
src/main.rs:4:37: 4:65 error: function `ciao` is private
src/main.rs:4     println!("Ciao in inglese: {}", frasi::inglese::saluti::ciao());
                                                  ^~~~~~~~~~~~~~~~~~~~~~~~~~~~
note: in expansion of format_args!
<std macros>:2:25: 2:58 note: expansion site
<std macros>:1:1: 2:62 note: in expansion of print!
<std macros>:3:1: 3:54 note: expansion site
<std macros>:1:1: 3:58 note: in expansion of println!
frasi/src/main.rs:4:5: 4:69 note: expansion site
```

Di default, tutto è privato in Rust. Parliamone un po' più in profondità.

# Exportare un'interfaccia pubblica

Rust consente di controllare con precisione quali aspetti della propria
interfaccia sono pubblici, e quindi il default è essere privato. Per rendere
pubbliche le cose, si usa la parola-chiave `pub`. Prima concentriamoci
sul modulo `inglese`, e perciò riduciamo il file `src/main.rs` a questo:

```rust,ignore
extern crate frasi;

fn main() {
    println!("Ciao in inglese: {}", frasi::inglese::saluti::ciao());
    println!("Arrivederci in inglese: {}", frasi::inglese::commiati::arrivederci());
}
```

Nel file `src/lib.rs`, aggiungiamo `pub` alla dichiarazione del modulo
`inglese`:

```rust,ignore
pub mod inglese;
mod giapponese;
```

E nel file `src/inglese/mod.rs`, rendiamo entrambi `pub`:

```rust,ignore
pub mod saluti;
pub mod commiati;
```

Nel file `src/inglese/saluti.rs`, aggiungiamo `pub` alla dichiarazione `fn`:

```rust,ignore
pub fn ciao() -> String {
    "Hello!".to_string()
}
```

E anche nel file `src/inglese/commiati.rs`:

```rust,ignore
pub fn arrivederci() -> String {
    "Goodbye.".to_string()
}
```

Adesso, il nostro crate compila, sebbene con degli avvertimenti su fatto che
non stiamo usando le funzioni del modulo `giapponese`:

```bash
$ cargo run
   Compiling frasi v0.0.1 (file:///home/you/projects/frasi)
src/giapponese/saluti.rs:1:1: 3:2 warning: function is never used: `ciao`, #[warn(dead_code)] on by default
src/giapponese/saluti.rs:1 fn ciao() -> String {
src/giapponese/saluti.rs:2     "こんにちは".to_string()
src/giapponese/saluti.rs:3 }
src/giapponese/commiati.rs:1:1: 3:2 warning: function is never used: `arrivederci`, #[warn(dead_code)] on by default
src/giapponese/commiati.rs:1 fn arrivederci() -> String {
src/giapponese/commiati.rs:2     "さようなら".to_string()
src/giapponese/commiati.rs:3 }
     Running `target/debug/frasi`
Ciao in inglese: Hello!
Arrivederci in inglese: Goodbye.
```

`pub` si applica anche alle `struct` e ai loro campi. Rispettando la tendenza
di Rust verso la sicurezza, semplicemente rendendo pubblica una `struct` non
rende automaticamente pubblici i suoi membri: i campi devono essere marcati
individualmente con `pub`.

Adesso che le nostre funzioni sono pubbliche, possiamo usarle. Ottimo! Però,
digitare `frasi::inglese::saluti::ciao()` è molto lungo e ripetitivo. Rust ha
un'altra parola-chiave per importare i nomi nell'ambito corrente, così che
ci si possa riferire ad essi con nomi più brevi. Parliamo di `use`.

# Importare modulo usando `use`

Rust ha la parola-chiave `use`, che ci consente di importare dei nomi
nel nostro ambito locale. Modifichiamo il file `src/main.rs` così:

```rust,ignore
extern crate frasi;

use frasi::inglese::saluti;
use frasi::inglese::commiati;

fn main() {
    println!("Ciao in inglese: {}", saluti::ciao());
    println!("Arrivederci in inglese: {}", commiati::arrivederci());
}
```

Le due righe `use` importano ogni modulo nell'ambito locale, e quindi possiamo
far riferimento alle funzioni con un nome molto più breve. Per convenzione,
quando si importano funzioni, la pratica migliore è considerata importare
il modulo, invece che direttamente la funzione. In altre parole, si _può_ fare
così:

```rust,ignore
extern crate frasi;

use frasi::inglese::saluti::ciao;
use frasi::inglese::commiati::arrivederci;

fn main() {
    println!("Ciao in inglese: {}", ciao());
    println!("Arrivederci in inglese: {}", arrivederci());
}
```

Ma non è tipico. Così è significativamente più probabile che si verifichi
un conflitto di nomi. Nel nostro breve programma, non è una grossa questione,
ma man mano che cresce, diventa un problema. Se abbiamo nomi in conflitto,
Rust darà un errore di compilazione. Per esempio, se avessimo reso pubbliche
le funzioni del modulo `giapponese`, e provassimo a fare così:

```rust,ignore
extern crate frasi;

use frasi::inglese::saluti::ciao;
use frasi::giapponese::saluti::ciao;

fn main() {
    println!("Ciao in inglese: {}", ciao());
    println!("Ciao in giapponese: {}", ciao());
}
```

Rust ci darà un errore di compilazione:

```text
   Compiling frasi v0.0.1 (file:///home/you/projects/frasi)
src/main.rs:4:5: 4:36 error: a value named `ciao` has already been imported in this module [E0252]
src/main.rs:4 use frasi::giapponese::saluti::ciao;
                  ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
error: aborting due to previous error
Could not compile `frasi`.
```

Se stiamo importando più nomi dallo stesso modulo, non dobbiamo digitarlo
due volte. Invece di questo:

```rust,ignore
use frasi::inglese::saluti;
use frasi::inglese::commiati;
```

Possiamo usare questa abbreviazione:

```rust,ignore
use frasi::inglese::{saluti, commiati};
```

## Ri-esportare usando `pub use`

Non si usa `use` solamente per abbreviare gli identificatori. Lo si può anche
usare dentro il proprio crate per ri-esportare una funzione all'interno
di un altro modulo. Ciò consente di presentare un'interfaccia esterna che
può non mapparsi direttamente all'organizzazione interna del proprio codice.

Guardiamo un esempio. Modifichiamo il file `src/main.rs` così:

```rust,ignore
extern crate frasi;

use frasi::inglese::{saluti,commiati};
use frasi::giapponese;

fn main() {
    println!("Ciao in inglese: {}", saluti::ciao());
    println!("Arrivederci in inglese: {}", commiati::arrivederci());

    println!("Ciao in giapponese: {}", giapponese::ciao());
    println!("Arrivederci in giapponese: {}", giapponese::arrivederci());
}
```

Poi, modifichiamo il file `src/lib.rs` per rendere pubblico il modulo
`giapponese`:

```rust,ignore
pub mod inglese;
pub mod giapponese;
```

Poi, rendiamo pubbliche le due funzioni, prima nel file
`src/giapponese/saluti.rs`:

```rust,ignore
pub fn ciao() -> String {
    "こんにちは".to_string()
}
```

E poi nel file `src/giapponese/commiati.rs`:

```rust,ignore
pub fn arrivederci() -> String {
    "さようなら".to_string()
}
```

Infine, modifichiamo il file `src/giapponese/mod.rs` così:

```rust,ignore
pub use self::saluti::ciao;
pub use self::commiati::arrivederci;

mod saluti;
mod commiati;
```

La dichiarazione `pub use` porta la funzione nell'ambito di questa parte
della gerarchia di moduli. Siccome abbiamo inserito l'istruzione `pub use`
nel modulo `giapponese`, adesso abbiamo a disposizione la funzione
`frasi::giapponese::ciao()` e la funzione `frasi::giapponese::arrivederci()`,
anche se il loro codice risiede rispettivamente in
`frasi::giapponese::saluti::ciao()` and in
`frasi::giapponese::commiati::arrivederci()`. La nostra organizzazione interna
non definisce la nostra interfaccia esterna.

Qui abbiamo un `pub use` per ogni funzione che vogliamo portare nell'ambito di
`giapponese`. Alternativamente potevamo usare la sintassi jolly per includere
nell'ambito corrente tutto il contenuto di `saluti`: `pub use self::saluti::*`.

Che dire di `self`? Beh, di default, le dichiarazioni `use` sono percorsi
assoluti, che iniziano dal crate radice. `self` invece fa sì che il percorso
sia relativo alla posizione corrente nella gerarchia. C'è un'altra forma
speciale di `use`: si può scrivere `use super::` per salire di un livello
nell'albero dalla posizione attuale. Ad alcuni piace pensare a `self` come
alla directory `.`, e a `super` come alla directory `..`.

Fuori dalle istruzioni `use`, i percorsi sono normalmente relativi:
`foo::bar()` si referisce a una funzione interna a `foo`, relativo a dove
siamo. Se viene preceduto da `::`, come in `::foo::bar()`, allora si riferisce
a un diverso `foo`, inquanto è un percorso assoluto dal crate radice.

Questo verrà compilato ed eseguito così:

```bash
$ cargo run
   Compiling frasi v0.0.1 (file:///home/you/projects/frasi)
     Running `target/debug/frasi`
Ciao in inglese: Hello!
Arrivederci in inglese: Goodbye.
Ciao in giapponese: こんにちは
Arrivederci in giapponese: さようなら
```

## Importazioni complesse

Rust offre varie opzioni avanzate che possono aggiungere compattezza e comodità
alle istruzioni `extern crate` e `use`. Ecco un esempio:

```rust,ignore
extern crate frasi as detti;

use detti::giapponese::saluti as ja_saluti;
use detti::giapponese::commiati::*;
use detti::inglese::{self, saluti as en_saluti, commiati as en_commiati};

fn main() {
    println!("Ciao in inglese; {}", en_saluti::ciao());
    println!("E in giapponese: {}", ja_saluti::ciao());
    println!("Arrivederci in inglese: {}", inglese::commiati::arrivederci());
    println!("Ancora: {}", en_commiati::arrivederci());
    println!("E in giapponese: {}", arrivederci());
}
```

Cosa succede qui?

Primo, sia `extern crate` che `use` consentono di rinominare la cosa che viene
importata. Quindi il crate si chiama ancora "frasi", ma qui ci riferiremo
ad esso come a "detti". Analogamente, la prima istruzione `use` tira dentro
il modulo `giapponese::saluti` dal crate, ma lo rende disponibile come
`ja_saluti` invece che semplicemente `saluti`. Ciò può aiutare a evitare
ambiguità quando si importano elementi con nomi simili da altri posti.

La seconda istruzione `use` usa un asterisco jolly per portare dentro tutti
i simboli pubblici dal modulo `detti::giapponese::commiati`. Come si vede,
poi possiamo far riferimento alla funzione giapponese `arrivederci` senza
qualificatori di modulo. Questo genere di jolly dovrebbe essere usato
con parsimonia. Vale la pena notare che importa solamente i simboli pubblici,
anche se il codice che usa il jolly è nel medesimo modulo.

La terza istruzione `use` porta più spiegazioni. Usa l'"espansione a graffa"
per comprimere tre istruzioni `use` in una sola (questo genere di sintassi
può essere familiare a chi ha già scritto script di shell Linux). La forma
non compressa di questa istruzione sarebbe:

```rust,ignore
use detti::inglese;
use detti::inglese::saluti as en_saluti;
use detti::inglese::commiati as en_commiati;
```

Come si vede, le graffe comprimono le istruzioni `use` di varie voci
sotto il medesimo percorso, e in questo contesto `self` si riferisce
a quel percorso.
Nota: Le graffe non possono essere annidate né mescolate con asterischi jolly.
