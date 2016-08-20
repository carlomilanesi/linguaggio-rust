% Gioco-indovina

Impariamo un po' di Rust! Come nostro primo progetto, implementeremo
un classico problema di programmazione per principianti: l'indovina-numero.
Ecco come funziona: Il nostro programma genererà un intero casuale compreso
fra uno e cento. Poi ci chiederà di indovinarlo. Quando avremo inserito
un tentativo, ci dirà se siamo stati troppo bassi o troppo alti.
Quando indoviniamo, si congratulerà con noi. Mica male, no?

Lungo la strada, impareremo un pochino di Rust. Nel capitolo successivo,
‘Sintassi e semantica’, ci immergeremo in profondità in ogni parte.

# Impostazione

Impostiamo un nuovo progetto. Andiamo nella directory dei progetti.
Per il progetto `hello_world` abbiamo creato manualmente una struttura
di directory e il file `Cargo.toml`. Ma Cargo ha un comando che fa tutto
da solo. Proviamolo:

```bash
$ cd ~/projects
$ cargo new indovina_numero --bin
$ cd indovina_numero
```

Abbiamo passato il nome del nostro progetto al comando `cargo new`, e poi
il flag `--bin`, dato che stiamo creando un programma eseguibile,
invece di una libreria.

Verifichiamo il file `Cargo.toml` generato:

```toml
[package]

name = "indovina_numero"
version = "0.1.0"
authors = ["Your Name <you@example.com>"]
```

Cargo ricava queste informazioni dall'ambiente. Se non vanno bene, possono
sempre essere corrette.

Infine, Cargo ha generato un file ‘Hello, world!’. Verifichiamo `src/main.rs`:

```rust
fn main() {
    println!("Hello, world!");
}
```

Proviamo a compilare quello che Cargo ci ha fornito:

```{bash}
$ cargo build
   Compiling indovina_numero v0.1.0 (file:///home/you/projects/indovina_numero)
```

Ottimo! Riapriamo il file `src/main.rs`. Scriveremo tutto il nostro codice
in questo file.

Prima di procedere, vediamo un altro comando di Cargo: `run`. `cargo run`
è un po' come `cargo build`, ma inoltre lancia l'eseguibile prodotto.
Proviamolo:

```bash
$ cargo run
   Compiling indovina_numero v0.1.0 (file:///home/you/projects/indovina_numero)
     Running `target/debug/indovina_numero`
Hello, world!
```

Benone! Il comando `run` torna comodo quando si deve iterare rapidamente
su un progetto. Il nostro gioco è un tale progetto, e ci serve testare
rapidamente ogni iterazione prima di passare alla successiva.

# Elaborare un tentativo

Riprendiamo! La prima cosa che dobbiamo fare per il nostro gioco è
consentire al giocatore di inserire un tentativo. Scriviamo questo nel file
`src/main.rs`:

```rust,no_run
use std::io;

fn main() {
    println!("Indovina il numero!");

    println!("Prego, digita un tentativo.");

    let mut tentativo = String::new();

    io::stdin().read_line(&mut tentativo)
        .expect("Non si riesce a leggere la riga");

    println!("Hai digitato: {}", tentativo);
}
```

C'è parecchio qui! Esaminiamolo un pezzo alla volta.

```rust,ignore
use std::io;
```

Ci servirà prendere l'input dell'utente, e poi stampare il risultato in output.
Pertanto, abbiamo bisogno della libreria `io` presa dalla libreria standard.
Di default, Rust importa solamente un po' di cose in ogni programma,
[il ‘preludio’][preludio]. Se non è nel preludio, dovremo importarlo
direttamente, tramite `use`. C'è anche un secondo ‘preludio’, il
[preludio `io`][iopreludio], che serve per uno scopo simile: lo importi, ed
esso importa varie cose utili relative all'`io` (operazioni di input-output).

[preludio]: ../std/prelude/index.html
[iopreludio]: ../std/io/prelude/index.html

```rust,ignore
fn main() {
```

Come abbiamo visto prima, la funzione `main()` è il punto di ingresso
del programma. La sintassi `fn` dichiara una nuova funzione, le `()` indicano
che non ci sono argomenti, e `{` inizia il corpo della funzione. Siccome non
abbiamo specificato un tipo reso, questo si assume essere `()`,
cioè un'[ennupla][ennuple] vuota.

[ennuple]: primitive-types.html#tuples

```rust,ignore
    println!("Indovina il numero!");

    println!("Prego, digita un tentativo.");
```

Prima abbiamo imparato che `println!()` è una [macro][macro] che stampa
una [stringa][stringhe] sullo schermo.

[macro]: macros.html
[stringhe]: strings.html

```rust,ignore
    let mut tentativo = String::new();
```

Adesso si fa interessante! Succedono molte cose in questa breve riga.
La prima cosa da notare è che questa è un'istruzione [istruzione let][let],
che viene usata per creare ‘legami a variabili’ [variable bindings].
Questi ultimi assumono la forma:

```rust,ignore
let foo = bar;
```

[let]: variable-bindings.html

Questa istruzione crea un nuovo legame chiamato `foo`, e lo lega al valore
`bar`. In molti linguaggi, questo si chiama ‘variabile’, ma i legami
di variabile di Rust hanno alcuni assi nella manica.

Per esempio, di default sono immutabili [immutabile][immutabile]. Ecco perché
il nostro esempio usa `mut`: rende mutabile il legame, invece che immutabile.
`let` non prende un nome sul lato sinistro dell'assegnamento, in realtà
accetta un ‘[pattern][pattern]’. Useremo i pattern più avanti. Per adesso è
abbastanza facile da usare:

```rust
let foo = 5; // immutabile.
let mut bar = 5; // mutabile
```

[immutabile]: mutability.html
[pattern]: patterns.html

Oh, e `//` inizia un commento, che finisce col finire della riga. Rust ignora
tutto il contenuto dei [commenti][commenti].

[comments]: comments.html

Perciò adesso sappiamo che `let mut tentativo` introduce un legame mutabile
di nome `tentativo`, ma dobbiamo guardare dall'altra parte dell'`=` per sapere
a che cosa è legato: `String::new()`.

`String` è un tipo stringa, fornito dalla libreria standard. Una
[`String`][string] è un pezzo di testo estendibile, codificato in UTF-8.

[string]: ../std/string/struct.String.html

La sintassi `::new()` usa i `::` perché questa è una ‘funzione associata’
di un particolare tipo. Il che significa che è associata allo stesso tipo
`String`, invece che ad una particolare istanza di `String`. Altri linguaggi
lo chiamerebbero ‘metodo statico’ o ‘metodo di classe’.

Questa funzione si chiama `new()`, perché crea un a nuova `String`, vuota.
Molti tipi hanno una funzione `new()`, dato che è un nome naturale
per creare un nuovo valore ti qualche tipo.

Andiamo avanti:

```rust,ignore
    io::stdin().read_line(&mut tentativo)
        .expect("Non si riesce a leggere la riga");
```

C'è molto di più! Prendiamo un pezzetto alla volta. La prima riga ha due parti.
Ecco la prima:

```rust,ignore
io::stdin()
```

Alla prima riga del programma avevamo scritto `use std::io`. Adesso stiamo
chiamando una funzione associata a tale tipo. Comunque, se non avessimo scritto
`use std::io`, ce la saremmo cavata scrivendo in questa riga
`std::io::stdin()`.

Questa funzione particolare restituisce un handle al flusso standard di input
per la console. Più specificamente, a [std::io::Stdin][iostdin].

[iostdin]: ../std/io/struct.Stdin.html

La prossima parte userà questo handle per ottenere l'input dall'utente:

```rust,ignore
.read_line(&mut tentativo)
```

Qui, chiamiamo il metodo [`read_line()`][read_line] sul nostro handle.
I [metodi][metodo] sono come funzioni associate, ma sono disponibili solamente
per un'istanza particolare di un tipo, invece che sul tipo stesso. Stiamo anche
passando un argomento a `read_line()`: `&mut tentativo`.

[read_line]: ../std/io/struct.Stdin.html#method.read_line
[metodo]: method-syntax.html

Come abbiamo legato il nome `tentativo` prima? Abbiamo detto che era mutabile.
Però, `read_line` non prende una `String` come argomento: prende
una `&mut String`. Rust ha una caratteristica chiamata ‘[riferimenti]
[riferimenti]’, che consente di avere più riferimenti ad un singolo dato,
la quale riduce la necessità di copiare. I riferimenti sono
una caratteristica complessa, dato che uno dei principali vantaggi di Rust
è quanto sia sicuro e facile usare i riferimenti. Però, per adesso, non
ci serve sapere molti dettagli per finire il nostro programma. Per adesso,
l'unica cosa che dobbiamo sapere è che anche i riferimenti, come i legami
`let`, sono immutabili di default. Pertanto, dobbiamo scrivere
`&mut tentativo`, invece che semplicemente `&tentativo`.

Perché `read_line()` prende un riferimento mutabile a una stringa? Il suo
compito è prendere i caratteri digitati dall'utente tramite lo standard input,
e collocarli in una stringa. Quindi prende quella come argomento, e per poterci
aggiungere l'input, ha bisogno che sia mutabile.

[riferimenti]: references-and-borrowing.html

Ma non abbiamo proprio finito con questa riga di codice. Anche se è una singola
riga di testo, è solamente la prima parte della singola riga logica di codice:

```rust,ignore
        .expect("Non si riesce a leggere la riga");
```

Quando si chiama un metodo con la sintassi `.foo()`, si può andare a capo
o si possono inserire spazi. Ciò aiuta a spezzare righe lunghe. _Avremmo anche
potuto_ scrivere:

```rust,ignore
    io::stdin().read_line(&mut tentativo).expect("non si riesce a leggere la riga");
```

Ma sarebbe diventato più difficile da leggere. E allora l'abbiamo spezzato;
due righe per due chiamate di metodo. Abbiamo già parlato di `read_line()`,
ma che dire di `expect()`? Beh, abbiamo già accennato che `read_line()` mette
ciò che viene digitato dall'utente nella `&mut String` che le passiamo.
Ma restituisce anche un valore: in questo caso, un [`io::Result`][ioresult].
Rust ha vari tipi chiamati `Result` nella sua libreria standard: un [`Result`]
[result] generico, e poi delle versioni specifiche per delle sotto-librerie,
come `io::Result`.

[ioresult]: ../std/io/type.Result.html
[result]: ../std/result/enum.Result.html

Lo scopo di questi tipi `Result` è codificare le informazioni di gestione
degli errori. Per i valori del tipo `Result`, come per quelli di ogni altro
tipo, sono definiti dei metodi. In questo caso, `io::Result` ha un [metodo
`expect()`][expect] che prende un valore su cui è chiamato, e se tale valore
non rappresenta un successo, va in [`panic!`][panic] emettendo il messaggio
che gli è stato passato. Un `panic!` come questo manderà in crash il programma,
mostrando il messaggio.

[expect]: ../std/result/enum.Result.html#method.expect
[panic]: error-handling.html

Se omettiamo la chiamata a questo metodo, il nostro programma compilerà,
ma otterremo un avvertimento:

```bash
$ cargo build
   Compiling indovina_numero v0.1.0 (file:///home/you/projects/indovina_numero)
src/main.rs:10:5: 10:43 warning: unused result which must be used,
#[warn(unused_must_use)] on by default
src/main.rs:10     io::stdin().read_line(&mut tentativo);
                   ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
```

Rust ci avvisa che non abbiamo utilizzato il valore `Result`. Questo
avvertimento viene generato da una speciale annotazione utilizzata da `io::Result`.
Rust sta cercando di dirci che non abbiamo gestito un possibile errore. Il
modo giusto per sopprimere l'errore è scrivere il codice per la gestione
dell'errore stesso. Fortunatamente, se vogliamo abortire il programma in caso
ci sia un problema, possiamo usare `expect()`. Se possiamo recuperare
dall'errore in qualche modo, faremo qualcosa di diverso, ma riserviamo
l'argomento per un progetto futuro.

C'è solo una riga di codice rimasta per questo primo esempio:

```rust,ignore
    println!("Hai digitato: {}", tentativo);
}
```

Questa riga stampa la stringa dove abbiamo salvato il nostro input. Le
parentesi graffe `{}` sono dei segna-posto per passare `tentativo` come
argomento. Se avessimo scritto diversi `{}`, avremmo passato diversi
argomenti:

```rust
let x = 5;
let y = 10;

println!("x e y: {} e {}", x, y);
```

Facile.

Possiamo lanciare quello che abbiamo con `cargo run`:

```bash
$ cargo run
   Compiling indovina_numero v0.1.0 (file:///home/you/projects/indovina_numero)
     Running `target/debug/indovina_numero`
Indovina il numero!
Prego, digita un tentativo.
6
Hai digitato: 6
```

Tutto bene! La prima parte del progetto è fatta: possiamo prendere input da
tastiera, e stamparli a schermo.

# Generare un numero segreto

Poi, dobbiamo generare un numero segreto. La libreria standard di Rust
non comprende ancora un generatore di numeri casuali. Però, la squadra
di Rust fornisce un [crate `rand`][randcrate]. Un ‘crate’ ('cassone') è
un pacchetto di codice Rust. Stiamo costruendo un ‘crate binario’, mentre
`rand` è un ‘crate libreria’, cioè contiene del codice pronto per essere
usato da altri programmi.

[randcrate]: https://crates.io/crates/rand

Usare crate esterni è la cosa per cui Cargo fa faville. Prima di poter scrivere
del codice che usa `rand`, dobbiamo modificare il nostro `Cargo.toml`.
Apriamolo, e aggiungiamo in fondo queste due righe:

```toml
[dependencies]

rand="0.3.0"
```

La sezione `[dependencies]` di `Cargo.toml` è simile alla sezione `[package]`:
ogni cosa che la segue ne fa parte, fino all'inizio della sezione successiva.
Cargo usa la sezione dependencies per sapere quali dipendenze ci sono da crate
esterni, e quali versioni di essi sono richieste. In questo caso, abbiamo
specificato la versione `0.3.0`, che Cargo capisce essere qualunque rilascio
che è compatibile con questa specifica versione. Cargo capisce
il [Versionamento semantico][semver], che è uno standard per scrivere
numeri di versione. Un semplice numero come quello sopra è in realtà
un'abbreviazione di `^0.3.0`, che significa "tutte le versioni compatibili
con la versione 0.3.0".
Se avessimo voluto usare solamente proprio la versione `0.3.0`, avremmo potuto
dire `rand="=0.3.0"` (si notino i due segni di uguaglianza).
E se volessimo usare sempre la versione più recente, potremmo usare `*`.
Potremmo usare anche una gamma di versioni.
La [documentazione di Cargo][cargodoc] contiene ulteriori dettagli.

[semver]: http://semver.org
[cargodoc]: http://doc.crates.io/specifying-dependencies.html

Adesso, senza cambiare nient'altro del codice, costruiamo il nostro progetto:

```bash
$ cargo build
    Updating registry `https://github.com/rust-lang/crates.io-index`
 Downloading rand v0.3.8
 Downloading libc v0.1.6
   Compiling libc v0.1.6
   Compiling rand v0.3.8
   Compiling indovina_numero v0.1.0 (file:///home/you/projects/indovina_numero)
```

(Naturalmente, si potranno vedere diversi numeri di versione.)

Un sacco di nuovo output! Adesso che abbiamo una dipendenza esterna, Cargo
va a prendere le versioni più recenti di ogni cosa dal registry, che è
una copia di dati presi da [Crates.io][cratesio]. Crates.io è il posto dove
la gente nell'ecosistema di Rust invia i suoi progetti open source in Rust
per farli usare ad altri.

[cratesio]: https://crates.io

Dopo aver aggiornato il registry, Cargo verifica la nostra sezione
`[dependencies]` e scarica tutti i pacchetti che non abbiamo ancora. In questo
caso, mentre abbiamo detto soltanto che volevamo dipendere da `rand`, ci siamo
presi anche una copia di `libc`. Questo perché `rand` dipende da `libc`
per funzionare. Dopo averli scaricati, li compila, e poi compila
il nostro progetto.

Se eseguiamo ancora `cargo build`, otteniamo un output diverso:

```bash
$ cargo build
```

Va bene, nessun output! Cargo sa che il nostro progetto è stato costruito, e
che tutte le sue dipendenze sono state scaricate e costruite, e quindi non c'è
ragione di fare tutta quella roba. Non avendo niente da fare, semplicemente
termina. Se riapriamo `src/main.rs`, facciamo una modifica banale, e lo
salviamo di nuovo, vedremo solamente una riga:

```bash
$ cargo build
   Compiling indovina_numero v0.1.0 (file:///home/you/projects/indovina_numero)
```

Perciò, abbiamo detto a Cargo che volevamo la versione `0.3.x` di `rand`, e
quindi ci ha preso la versione più recente che c'era all'istante in cui questo
documento è stato scritto, la `v0.3.8`. Ma cosa accadrà la prossima settimana,
quando uscirà la versione `v0.3.9`, contenente una correzione importante?
Ricevere le correzioni è importante, ma se la versione `0.3.9` contiene
una regressione incompatibile con il nostro codice?

La risposta a questo problema è il file `Cargo.lock` che adesso si troverà
nella directory del nostro progetto. Quando costruiamo un progetto per
la prima volta, Cargo calcola tutti i numeri di versione che soddisfano
i nostri criteri, e poi li scrive nel file `Cargo.lock`. Quando si costruisce
nuovamente il progetto, Cargo vedrà che il file `Cargo.lock` esiste, e quindi
usa quella specifica versione invece di rifare tutto il lavoro di calcolare
i numeri di versione. Questo consente di avere un build ripetibile automaticamente.
In altri termini, rimarremo alla versione `0.3.8` finché la
aggiorniamo esplicitamente, e così farà chiunque condivida il nostro codice,
grazie al file `.lock`.

E che fare quando _vogliamo_ usare la versione `v0.3.9`? Cargo ha un altro
comando, `update`, che significa ‘ignora il file `.lock`, calcola tutte
le versioni più recenti che soddisfano quello che abbiamo specificato.
Se funziona, scrivi quelle versioni nel file `.lock`’. Ma, di default, Cargo
cercherà solamente versioni superiori alla `0.3.0` e inferiori alla `0.4.0`. Se volessimo
aggiornare ad una versione `0.4.x`, dovremmo aggiornare direttamente il file
`Cargo.toml`. Quando lo facciamo, la prossima volta che eseguiamo il comando
`cargo build`, Cargo aggiornerà l'indice e rivaluterà i requisiti del nostro
`rand`.

C'è molto più da dire su [Cargo][doccargo] e sul [suo ecosistema][doccratesio],
ma per adesso, è tutto  quello che ci serve. Cargo rende davvero facile il
riutilizzo delle
librerie, e quindi i Rustaciani tendono a scrivere progetti piccoli, assemblati
usando vari sotto-progetti.

[doccargo]: http://doc.crates.io
[doccratesio]: http://doc.crates.io/crates-io.html

Procediamo a _usare_ `rand`. Ecco il nostro prossimo passo:

```rust,ignore
extern crate rand;

use std::io;
use rand::Rng;

fn main() {
    println!("Indovina il numero!");

    let numero_segreto = rand::thread_rng().gen_range(1, 101);

    println!("Il numero segreto è: {}", numero_segreto);

    println!("Prego, digita un tentativo.");

    let mut tentativo = String::new();

    io::stdin().read_line(&mut tentativo)
        .expect("Non si riesce a leggere la riga");

    println!("Hai digitato: {}", tentativo);
}
```

La prima cosa che abbiamo fatto è cambiare la prima riga. Adesso dice
`extern crate rand`. Dato che abbiamo dichiarato `rand` nelle nostre
`[dependencies]`, possiamo usare `extern crate` per far sapere a Rust che ne
faremo uso. Questa istruzione fa anche l'equivalente di un `use rand;`,
e quindi possiamo fare uso di ogni cosa presente nel crate `rand` pur di
qualificarla con il prefisso `rand::`.

Poi, abbiamo aggiunto un'altra riga `use`: `use rand::Rng`. Fra un attimo
useremo un metodo, e tale metodo richiede che `Rng` sia nell'ambito del
programma. L'idea
di base è questa: i metodi sono definiti su cose chiamate ‘tratti’ ('traits'),
e affinché
un metodo funzioni, ha bisogno che il suo tratto sia attivo nell'ambito
corrente. Per maggiori dettagli, si legga la sezione sui [tratti][tratti].

[tratti]: traits.html

Abbiamo aggiunto due altre righe, nel mezzo:

```rust,ignore
    let numero_segreto = rand::thread_rng().gen_range(1, 101);

    println!("Il numero segreto è: {}", numero_segreto);
```

Usiamo la funzione `rand::thread_rng()` per ottenere una copia del generatore
di numeri casuali, che è locale al particolare [thread][concurrency] di
esecuzione in cui siamo.
Siccome sopra abbiamo scritto `use rand::Rng`, adesso il metodo
`gen_range()` è disponibile. Questo metodo prende due argomenti, e genera
un numero compreso tra di essi. Il limite inferiore è incluso, mentre il limite
superiore è escluso, e quindi ci servono `1` e `101` per ottenere un numero
che può andare da uno a cento.

[concurrency]: concurrency.html

La seconda riga stampa il numero segreto. Questa è utile fintanto che stiamo
sviluppando il nostro programma, così che possiamo collaudarlo facilmente.
Ma la toglieremo per la versione finale. Non sarebbe un gran gioco se stampasse
la risposta esatta quando lo si lancia!

Proviamo a eseguire alcune volte il nostro nuovo programma:

```bash
$ cargo run
   Compiling indovina_numero v0.1.0 (file:///home/you/projects/indovina_numero)
     Running `target/debug/indovina_numero`
Indovina il numero!
Il numero segreto è: 7
Prego, digita un tentativo.
4
Hai digitato: 4
$ cargo run
     Running `target/debug/indovina_numero`
Indovina il numero!
Il numero segreto è: 83
Prego, digita un tentativo.
5
Hai digitato: 5
```

Ottimo! Prossimo passo: confrontare il nostro tentativo con il numero segreto.

# Confrontare i tentativi

Adesso che abbiamo l'input dell'utente, confrontiamo il nostro tentativo
con il numero segreto. Ecco il nostro nuovo passo, anche se non compila ancora:

```rust,ignore
extern crate rand;

use std::io;
use std::cmp::Ordering;
use rand::Rng;

fn main() {
    println!("Indovina il numero!");

    let numero_segreto = rand::thread_rng().gen_range(1, 101);

    println!("Il numero segreto è: {}", numero_segreto);

    println!("Prego, digita un tentativo.");

    let mut tentativo = String::new();

    io::stdin().read_line(&mut tentativo)
        .expect("Non si riesce a leggere la riga");

    println!("Hai digitato: {}", tentativo);

    match tentativo.cmp(&numero_segreto) {
        Ordering::Less    => println!("Troppo piccolo!"),
        Ordering::Greater => println!("Troppo grande!"),
        Ordering::Equal   => println!("Hai vinto!"),
    }
}
```

Ci sono alcune aggiunte. La prima è un altro `use`. Portiamo nell'ambito
un tipo chiamato `std::cmp::Ordering`. Poi, ci sono cinque nuove righe
in fondo che usano tale tipo:

```rust,ignore
match tentativo.cmp(&numero_segreto) {
    Ordering::Less    => println!("Troppo piccolo!"),
    Ordering::Greater => println!("Troppo grande!"),
    Ordering::Equal   => println!("Hai vinto!"),
}
```

Il metodo `cmp()` può essere chiamato su qualunque oggetto che può essere
confrontato, e prende un riferimento all'oggetto con cui lo si vuole
confrontare. Il confronto restituisce il tipo `Ordering`
che abbiamo importato prima. Usiamo
un'istruzione [`match`][match] per determinare esattamente quale `Ordering`
abbiamo ottenuto dal confronto. `Ordering` è una [`enum`][enum], abbreviazione
di ‘enumerazione’. Le enumerazioni si presentano così:

```rust
enum Foo {
    Bar,
    Baz,
}
```

[match]: match.html
[enum]: enums.html

Con questa definizione, ogni oggetto di tipo `Foo` può essere o una `Foo::Bar`
o un `Foo::Baz`. Usiamo il `::` per indicare lo spazio dei nomi
di una paricolare variante della `enum`.

L'`enum` [`Ordering`][ordering] ha tre possibili varianti: `Less`, `Equal`,
e `Greater`. L'istruzione `match` prende un valore di un tipo, e permette
di creare un ‘braccio’ per ogni valore possibile. Dato che abbiamo tre varianti
di `Ordering`, dobbiamo avere tre bracci:

```rust,ignore
match tentativo.cmp(&numero_segreto) {
    Ordering::Less    => println!("Troppo piccolo!"),
    Ordering::Greater => println!("Troppo grande!"),
    Ordering::Equal   => println!("Hai vinto!"),
}
```

[ordering]: ../std/cmp/enum.Ordering.html

Se vale `Less`, stampiamo `Troppo piccolo!`, se vale `Greater`,
`Troppo grande!`, e se vale `Equal`, `Hai vinto!`. `match` è davvero utile,
ed è usato spesso in Rust.

Ho già detto che questo codice non compilava, però. Proviamolo:

```bash
$ cargo build
   Compiling indovina_numero v0.1.0 (file:///home/you/projects/indovina_numero)
src/main.rs:28:25: 28:40 error: mismatched types:
 expected `&collections::string::String`,
    found `&_`
(expected struct `collections::string::String`,
    found integral variable) [E0308]
src/main.rs:28     match tentativo.cmp(&numero_segreto) {
                                       ^~~~~~~~~~~~~~~
error: aborting due to previous error
Could not compile `indovina_numero`.
```

Urca! Questo è un grosso errore. Il sua sintesi è che abbiamo dei ‘tipi male
accoppiati’ ["‘mismatched types’"]. Rust ha un sistema dei tipi forte
e statico. Però, ha anche l'inferenza dei tipi. Quando abbiamo scritto
`let tentativo = String::new()`, Rust è stato capace di inferire che
`tentativo` doveva essere una `String`, e quindi non ci ha costretto
a scriverne il tipo. E con il nostro `numero_secreto`, ci sono vari tipi
che possono avere un valore fra uno e cento: `i32`, cioè un numero intero
con segno a trentadue bit, o `u32`, cioè un numero intero senza segno
a trentadue bit, o `i64`, un numero intero con segno a sessantaquattro bit
oppure altri. Finora questo fatto non ha importato, e quindi Rust ha preso come default
un `i32`. Però, qui, Rust non sa come confrontare `tentativo` con
`secret_number`. Devono essere dello stesso tipo. Pertanto, dobbiamo convertire
la `String` che abbiamo letto come input in un vero tipo numerico, per poterlo
confrontare. Lo possiamo fare aggiungendo due righe. Ecco il nostro nuovo
programma:

```rust,ignore
extern crate rand;

use std::io;
use std::cmp::Ordering;
use rand::Rng;

fn main() {
    println!("Indovina il numero!");

    let numero_segreto = rand::thread_rng().gen_range(1, 101);

    println!("Il numero segreto è: {}", numero_segreto);

    println!("Prego, digita un tentativo.");

    let mut tentativo = String::new();

    io::stdin().read_line(&mut tentativo)
        .expect("Non si riesce a leggere la riga");

    let tentativo: u32 = tentativo.trim().parse()
        .expect("Prego, digita un numero!");

    println!("Hai digitato: {}", tentativo);

    match tentativo.cmp(&numero_segreto) {
        Ordering::Less    => println!("Troppo piccolo!"),
        Ordering::Greater => println!("Troppo grande!"),
        Ordering::Equal   => println!("Hai vinto!"),
    }
}
```

Le due nuove righe:

```rust,ignore
    let tentativo: u32 = tentativo.trim().parse()
        .expect("Prego, digita un numero!");
```

Aspetta un attimo, pensavo che avevamo già un `tentativo`? Sì, ma Rust ci
permette di ’oscurare’ ["‘shadow’"] il precedente `tentativo` con uno nuovo.
Questo si usa spesso proprio in questa situazione, in cui `tentativo` inizia
come `String`, ma lo vogliamo convertire in un `u32`. L'oscuramento ci permette
di riusare il nome `tentativo`, invece di costringerci a inventare due nomi
diversi come `tentativo_str` e `tentativo`, o qualcos'altro.

Leghiamo `tentativo` ad una espressione che somiglia a qualcosa che abbiamo
scritto prima:

```rust,ignore
tentativo.trim().parse()
```

Qui, `tentativo` si riferisce al vecchio `tentativo`, quello che era
una `String` contenente il nostro input. Il metodo `trim()` applicato ad
una `String` eliminerà tutti gli spazi all'inizio e alla fine della stringa.
Questo è importante, dato che abbiamo dovuto premere il tasto ‘Invio’
per soddisfare la funzione `read_line()`. Ciò significa che se digitiamo `5`
e battiamo Invio, `tentativo` conterrà `5\n`. La stringa `\n` rappresenta un
carattere ‘a capo’, inserito dal tasto Invio. `trim()` se ne sbarazza,
lasciando nella nostra stringa solamente il `5`. Il [metodo `parse()` applicato
a una stringa][parse] analizza la stringa estraendone un numero di qualche
tipo. Dato che tale metodo può riconoscere vari tipi di numeri, dobbiamo
suggerire a Rust il tipo esatto del numero che vogliamo. Pertanto, scriviamo
`let tentativo: u32`. I due-punti (`:`) dopo `tentativo` dicono a Rust
che stiamo annotando il tipo del legame. `u32` è il tipo intero senza segno
a trentadue bit. Rust ha [vari tipi numerici predefiniti][number], ma abbiamo
scelto `u32`. È una buona scelta di default per un numero positivo piccolo.

[parse]: ../std/primitive.str.html#method.parse
[number]: primitive-types.html#numeric-types

Proprio come `read_line()`, anche la nostra chiamata a `parse()` potrebbe
provocare un errore. Che fare se la nostra stringa contenesse `A👍%`? Non ci
sarebbe modo di convertirla in un numero. Pertanto, faremo la stessa cosa che
abbiamo fatto con `read_line()`: usiamo il metodo `expect()` per andare
in crash se c'è un errore.

Proviamo il nostro programma!

```bash
$ cargo run
   Compiling indovina_numero v0.1.0 (file:///home/you/projects/indovina_numero)
     Running `target/indovina_numero`
Indovina il numero!
Il numero segreto è: 58
Prego, digita un tentativo.
  76
Hai digitato: 76
Troppo grande!
```

Carino! Si può vedere che abbiamo perfino aggiunto degli spazi prima
di digitare il tentativo, ma il programma è comunque riuscito a valutarlo come 76.
Eseguiamo il programma alcune volte, e verifichiamo che funzioni quando
si indovina il numero, e anche quando il tentativo è un numero troppo piccolo.

Adesso la maggior parte del gioco funziona, ma possiamo fare
un solo tentativo. Modifichiamolo aggiungendo i cicli!

# Ciclare

La parola-chiave `loop` ci dà un ciclo infinito. Aggiungiamola:

```rust,ignore
extern crate rand;

use std::io;
use std::cmp::Ordering;
use rand::Rng;

fn main() {
    println!("Indovina il numero!");

    let numero_segreto = rand::thread_rng().gen_range(1, 101);

    println!("Il numero segreto è: {}", numero_segreto);

    loop {
        println!("Prego, digita un tentativo.");

        let mut tentativo = String::new();

        io::stdin().read_line(&mut tentativo)
            .expect("Non si riesce a leggere la riga");

        let tentativo: u32 = tentativo.trim().parse()
            .expect("Prego, digita un numero!");

        println!("Hai digitato: {}", tentativo);

        match tentativo.cmp(&numero_segreto) {
            Ordering::Less    => println!("Troppo piccolo!"),
            Ordering::Greater => println!("Troppo grande!"),
            Ordering::Equal   => println!("Hai vinto!"),
        }
    }
}
```

E proviamola. Ma, un momento, non abbiamo appena aggiunto un ciclo infinito?
Già. Prima, parlando di `parse()`, abbiamo detto che se gli diamo una risposta
che non è un numero, andrà in `panic!` e terminerà. Osserviamo:

```bash
$ cargo run
   Compiling indovina_numero v0.1.0 (file:///home/you/projects/indovina_numero)
     Running `target/indovina_numero`
Indovina il numero!
Il numero segreto è: 59
Prego, digita un tentativo.
45
Hai digitato: 45
Troppo piccolo!
Prego, digita un tentativo.
60
Hai digitato: 60
Troppo grande!
Prego, digita un tentativo.
59
Hai digitato: 59
Hai vinto!
Prego, digita un tentativo.
basta
thread 'main' panicked at 'Prego, digita un numero!'
```

Ah! Digitando `basta` in effetti si termina il programma. Come fa qualunque
input non numerico. Beh, questo non è il massimo, a dir poco. Come prima cosa,
terminiamo effettivamente quando si vince il gioco:

```rust,ignore
extern crate rand;

use std::io;
use std::cmp::Ordering;
use rand::Rng;

fn main() {
    println!("Indovina il numero!");

    let numero_segreto = rand::thread_rng().gen_range(1, 101);

    println!("Il numero segreto è: {}", numero_segreto);

    loop {
        println!("Prego, digita un tentativo.");

        let mut tentativo = String::new();

        io::stdin().read_line(&mut tentativo)
            .expect("Non si riesce a leggere la riga");

        let tentativo: u32 = tentativo.trim().parse()
            .expect("Prego, digita un numero!");

        println!("Hai digitato: {}", tentativo);

        match tentativo.cmp(&numero_segreto) {
            Ordering::Less    => println!("Troppo piccolo!"),
            Ordering::Greater => println!("Troppo grande!"),
            Ordering::Equal   => {
                println!("Hai vinto!");
                break;
            }
        }
    }
}
```

Aggiungendo la riga `break` dopo la stampa di `Hai vinto!`, usciremo dal ciclo
quando si vince. Uscire dal ciclo comporta anche uscire dal programma, dato che
non c'è altro in `main()`. Abbiamo solamente un altro ritocco da fare: quando
si digita un input non numerico, non vogliamo terminare, vogliamo ignorarlo.
Lo possiamo fare così:

```rust,ignore
extern crate rand;

use std::io;
use std::cmp::Ordering;
use rand::Rng;

fn main() {
    println!("Indovina il numero!");

    let numero_segreto = rand::thread_rng().gen_range(1, 101);

    println!("Il numero segreto è: {}", numero_segreto);

    loop {
        println!("Prego, digita un tentativo.");

        let mut tentativo = String::new();

        io::stdin().read_line(&mut tentativo)
            .expect("Non si riesce a leggere la riga");

        let tentativo: u32 = match tentativo.trim().parse() {
            Ok(num) => num,
            Err(_) => continue,
        };

        println!("Hai digitato: {}", tentativo);

        match tentativo.cmp(&numero_segreto) {
            Ordering::Less    => println!("Troppo piccolo!"),
            Ordering::Greater => println!("Troppo grande!"),
            Ordering::Equal   => {
                println!("Hai vinto!");
                break;
            }
        }
    }
}
```

Le righe modificate sono le seguenti:

```rust,ignore
let tentativo: u32 = match tentativo.trim().parse() {
    Ok(num) => num,
    Err(_) => continue,
};
```

Il modo tipico per passare da un ‘crash dovuto a errore’ a una ‘gestione
effettiva dell'errore’, è passare dall'uso del metodo `expect()` all'uso
dell'istruzione `match`. La chiamata a `parse()` restituisce un `Result`;
questo è una `enum`, come `Ordering`, ma in questo caso,
ogni variante ha alcuni dati associati ad essa:
`Ok` indica un successo, e `Err` indica un fallimento, ma
entrambi contengono ulteriori informazioni: per il primo, l'intero estratto
con successo, e per l'altro il genere di errore. In questo caso, `match`
proverà a far combaciare il suo argomento con `Ok(num)`, che assegnerà al nome
il valore contenuto in `Ok` (cioè l'intero estratto), e restituirà
quest'ultimo nel lato destro. Nel caso `Err`, non ci interessa il genere
di errore, e così usiamo il carattere piglia-tutto `_`, invece di un nome.
Questo jolly combacia con qualunque codice d'errore, e poi `continue` ci
sposterà alla prossima iterazione del ciclo; in effetti, questo codice
ci consente di ignorare tutti gli errori e continuare nel programma.

Adesso dovremmo essere a posto! Proviamo:

```bash
$ cargo run
   Compiling indovina_numero v0.1.0 (file:///home/you/projects/indovina_numero)
     Running `target/indovina_numero`
Indovina il numero!
Il numero segreto è: 61
Prego, digita un tentativo.
10
Hai digitato: 10
Troppo piccolo!
Prego, digita un tentativo.
99
Hai digitato: 99
Troppo grande!
Prego, digita un tentativo.
foo
Prego, digita un tentativo.
61
Hai digitato: 61
Hai vinto!
```

Magnifico! Con un ultimo ritocchino, finiremo il gioco. Riesci ad indovinare
quale sarà? Giusto, non vogliamo stampare subito il numero segreto. Andava
bene per il collaudo, ma rovinava il gioco. Ecco il nostro sorgente finale:

```rust,ignore
extern crate rand;

use std::io;
use std::cmp::Ordering;
use rand::Rng;

fn main() {
    println!("Indovina il numero!");

    let numero_segreto = rand::thread_rng().gen_range(1, 101);

    loop {
        println!("Prego, digita un tentativo.");

        let mut tentativo = String::new();

        io::stdin().read_line(&mut tentativo)
            .expect("Non si riesce a leggere la riga");

        let tentativo: u32 = match tentativo.trim().parse() {
            Ok(num) => num,
            Err(_) => continue,
        };

        println!("Hai digitato: {}", tentativo);

        match tentativo.cmp(&numero_segreto) {
            Ordering::Less    => println!("Troppo piccolo!"),
            Ordering::Greater => println!("Troppo grande!"),
            Ordering::Equal   => {
                println!("Hai vinto!");
                break;
            }
        }
    }
}
```

# Finito!

Questo progetto ha mostrato parecchie cose: `let`, `match`, i metodi,
le funzioni associate, l'uso di crate esterni, e altro.

A questo punto, abbiamo costruito un gioco Gioco-Indovina funzionante!
Congratulazioni!
