% La gestione degli errori

Come la maggior parte dei linguaggi di programmazione, Rust incoraggia
il programmatore a gestire gli errori in un modo particolare. In generale,
la gestione degli errori è divisa in due ampie categorie: le eccezioni e
i valori restituiti. Rust opta per i valori restituiti.

In questa sezione, intendiamo fornire una descrizione approfondita di come
trattare gli errori in Rust. In aggiunta, tenteremo di introdurre la gestione
degli errori un pezzo per volta, così che se ne ricaverà una buona conoscenza
operativa di come tutto si adatta insieme.

Quando è fatta in modo ingenuo, la gestione degli errori in Rust può essere
verbosa e fastidiosa. Questa sezione esplorerà quegli ostacoli e mostrerà come
usarela libreria standard per rendere la gestione degli errori concisa
ed ergonomica.

# Sommario

Questa sezione è molto lunga, principalmente perché iniziamo proprio
dall'inizio con i tipi somma e i combinatori, e proviamo a motivare il modo
con cui Rust gestisce gli errori incrementalmente. Pertanto, i programmatori
con esperienza in altri sistemi di tipi espressivi potranno voler saltare più
avanti.

* [Le basi](#the-basics)
    * [Lo svolgimento spiegato](#unwrapping-explained)
    * [Il tipo `Option`](#the-option-type)
        * [Comporre i valori `Option<T>`](#composing-optiont-values)
    * [Il tipo `Result`](#the-result-type)
        * [Parsing integers](#parsing-integers)
        * [L'idioma dell'alias del tipo `Result`](#the-result-type-alias-idiom)
    * [Un breve interludio: lo svolgimento non è male](#a-brief-interlude-unwrapping-isnt-evil)
* [Lavorare con più tipi di errori](#working-with-multiple-error-types)
    * [Comporre `Option` e `Result`](#composing-option-and-result)
    * [I limiti dei combinatori](#the-limits-of-combinators)
    * [Uscite precoci](#early-returns)
    * [La macro `try!`](#the-try-macro)
    * [Definire il proprio tipo di errori](#defining-your-own-error-type)
* [I tratti della libreria standard usati per la gestione degli errori](#standard-library-traits-used-for-error-handling)
    * [Il tratto `Error`](#the-error-trait)
    * [Il tratto `From`](#the-from-trait)
    * [La vera macro `try!`](#the-real-try-macro)
    * [Comporre tipi di errore personalizzati](#composing-custom-error-types)
    * [Consigli per autori di librerie](#advice-for-library-writers)
* [Studio di un caso: Un programma per leggere dati sulla popolazione](#case-study-a-program-to-read-population-data)
    * [Impostazione iniziale](#initial-setup)
    * [Analisi degli argomenti](#argument-parsing)
    * [Scrivere la logica](#writing-the-logic)
    * [Usare `Box<Error>` per la gestione degli errori](#error-handling-with-boxerror)
    * [Leggere da stdin](#reading-from-stdin)
    * [Usare un tipo personalizzato per la gestione degli errori](#error-handling-with-a-custom-type)
    * [Aggiungere funzionalità](#adding-functionality)
* [La storia breve](#the-short-story)

# Le basi

Si può pensare alla gestione degli errori come all'uso dell'*analisi dei casi*
per determinare se un'elaborazione ha avuto successo oppure no. Come vedremo,
la chiave alla gestione ergonomica degli errori è ridurre la quantità
di analisi esplicita dei casi che il programmatore deve fare, pur mantenendo
componibile il codice.

Mantenere il codice componibile è importante, perché senza quel requisito,
potremmo andare in [`panic`](../std/macro.panic!.html) ogni volta ci imbattiamo
in qualcosa di inaspettato. (Il `panic` fa sì che il thread corrente si svolga,
e nella maggior parte dei casi, che l'intero programma abortisca.)
Ecco un esempio:

```rust,should_panic
// Indovina un numero fra 1 e 10.
// Se combacia col numero che avevamo in mente, restituisci true.
// Altrimenti, restituisci false.
fn indovina(n: i32) -> bool {
    if n < 1 || n > 10 {
        panic!("Numero non valido: {}", n);
    }
    n == 5
}

fn main() {
    guess(11);
}
```

se si prova a eseguire questo codice, il programma andrà in crash
con un message così:

```text
thread 'main' panicked at 'Numero non valido: 11', src/bin/panic-simple.rs:5
```

Ecco un altro esempio che è leggermente meno artificioso. Un programma
che accetta un intero come argomento, lo raddoppia e lo stampa.

<span id="code-unwrap-double"></span>

```rust,should_panic
use std::env;

fn main() {
    let mut argv = env::args();
    let arg: String = argv.nth(1).unwrap(); // errore 1
    let n: i32 = arg.parse().unwrap(); // errore 2
    println!("{}", 2 * n);
}
```

Se si dà a questo programma zero argomenti (errore 1) o se il primo argomento
non è un intero (errore 2), il programma andrà in panico proprio come
nel primo esempio.

Si può pensare a questo stile di gestione degli errori come simile a un toro
che corre in un negozio di porcellana. Il toro arriverà dove vuole andare,
ma calpesterà tutto nel farlo.

## Lo svolgimento spiegato

Nell'esempio precedente, abbiamo detto che il programma andrebbe semplicemente
in panico se raggiungesse una delle due condizioni d'errore, però,
il programma non comprende un'esplicita chiamata a `panic` come il primo
esempio. Questo perché il panico è incorporato nelle chiamate a `unwrap`.

Eseguire “unwrap” (["svolgere"]) qualcosa in Rust è dire, “Dammi il risultato
dell'elaborazione, e se c'era un errore, va in panico e ferma il programma.”
Sarebbe meglio se mostrassimo il codice per svolgere, perché è così semplice,
ma per farlo, prima dovremo esplorare i tipi `Option` e `Result`. Entrambi
questi tipi hanno un metodo chiamato `unwrap` definito su di essi.

### Il tipo `Option`

Il tipo `Option` è [definito nella libreria standard][5] come:

```rust
enum Option<T> {
    None,
    Some(T),
}
```

Il tipo `Option` è un modo di usare il sistema dei tipi di Rust per esprimere
la *possibilità di assenza*. Codificare la possibilità di assenza nel sistema
dei tipo è un concetto importante perché farà sì che il compilatore costringa
il programmatore a gestire quell'assenza. Diamo un'occhiata a un esempio
che prova a trovare un carattere in una stringa:

<span id="code-option-ex-string-find"></span>

```rust
// Cerca in `pagliaio` il carattere Unicode `ago`. Se ne viene trovato uno,
// viene restituito lo scostamento in byte di tale carattere.
// Altrimenti, viene restituito `None`.
fn trova(pagliaio: &str, ago: char) -> Option<usize> {
    for (scostamento, c) in pagliaio.char_indices() {
        if c == ago {
            return Some(scostamento);
        }
    }
    None
}
```

Si noti che quando questa funzione trova un carattere corrispondente,
non restituisce solamente lo `scostamento`.
Invece, restituisce `Some(scostamento)`.
`Some` è una variante o un *costruttore di valore* per il tipo `Option`.
Si può pensare ad esso come a una funzione di tipo
`fn<T>(valore: T) -> Option<T>`. Analogamente, anche `None` è un costruttore
di valore, a parte il fatto che non ha argomenti. Si può pensare a `None`
come a una funzione di tipo `fn<T>() -> Option<T>`.

Questo potrebbe sembrare come tanta briga per niente, ma questa è solo metà
della storia. L'altra metà è *usare* la funzione `trova` che abbiamo scritto.
proviamo a usarla per trovare l'estensione in un nome di file.

```rust
# fn find(_: &str, _: char) -> Option<usize> { None }
fn main() {
    let nome_di_file = "foobar.rs";
    match find(nome_di_file, '.') {
        None => println!("Nessuna estensione di file trovata."),
        Some(i) => println!("Estensione di file: {}", &nome_di_file[i+1..]),
    }
}
```

Questo codice usa il [pattern matching][1] per fare *l'analisi dei casi*
sulla `Option<usize>` restituita dalla funzione `trova`.
Di fatto, l'analisi dei casi
è l'unico modo per arrivare al valore immagazzinato in un `Option<T>`.
Questo significa che il programmatore deve gestire il caso in cui `Option<T>`
vale `None` invece di `Some(t)`.

Ma, un momento, che dire di `unwrap`, che abbiamo usato [prima]
(#code-unwrap-double)? Là non c'è stata un'analisi dei casi! Invece, l'analisi
dei casi è stata messa dentro il metodo `unwrap`. Ognuno se lo potrebbe
definire, se volesse:

<span id="code-option-def-unwrap"></span>

```rust
enum Option<T> {
    None,
    Some(T),
}

impl<T> Option<T> {
    fn unwrap(self) -> T {
        match self {
            Option::Some(val) => val,
            Option::None =>
              panic!("chiamata `Option::unwrap()` su un valore `None`"),
        }
    }
}
```

Il metodo `unwrap` *astrae l'analisi dei casi*. Questa è precisamente la cosa
che rende `unwrap` ergonomica da usare. Sfortunatamente, quel `panic!`
significa che `unwrap` non è componibile: è il toro nel negozio di porcellana.

### Comporre i valori `Option<T>`

In un [esempio precedente](#code-option-ex-string-find), abbiamo visto
come usare `find` per scoprire l'estensione in un nome di file. Naturalmente,
non tutti i nomi di file contengono un `.`, e quindi è possibile che il nome
di file non abbia alcuna estensione. Questa *possibilità di assenza* è
codificata nei tipi usando `Option<T>`. In altre parole, il compilatore ci
costringerà ad affrontare la possibilità che un'estensione non esista.
Nel nostro caso, stampiamo solamente un messaggio che lo dice.

Ottenere l'estensione di un nome di file è un'operazione piuttosto tipica, e
quindi ha senso metterla in una funzione:

```rust
# fn trova(_: &str, _: char) -> Option<usize> { None }
// Restituisce l'estensione del dato nome di file, dove l'estensione è definita
// come tutti i caratteri che seguono il primo `.`.
// Se `nome_di_file` non contiene nessun `.`, allora viene restituito `None`.
fn estensione_esplicita(nome_di_file: &str) -> Option<&str> {
    match trova(nome_di_file, '.') {
        None => None,
        Some(i) => Some(&nome_di_file[i+1..]),
    }
}
```

(Consiglio professionale: non usate questo codice. Usate il metodo
[`extension`](../std/path/struct.Path.html#method.extension)
della libreria standard invece.)

Il codice rimane semplice, ma la cosa importante da notare è che il tipo di
`find` ci costringe a considerare la possibilità di assenza. questa è una buona
cosa, perché significa che il compilatore non ci lascierà accidentalmente
dimenticare del caso in cui un nome di file non ha un'estensione.
D'altra parte, fare ogni volta un'analisi dei casi esplicita, come abbiamo
fatto in `estensione_esplicita`, può diventare un po' seccante.

Di fatto, l'analisi dei casi in `estensione_esplicita` segue un pattern molto
tipico: *mappare* una funzione al valore interno di un `Option<T>`, a meno che
l'opzione sia `None`, nel qual caso, restituire `None`.

Rust ha il polimorfismo parametrico, perciò è molto facile definire
un combinatore che astrae questo pattern:

<span id="code-option-map"></span>

```rust
fn map<F, T, A>(opzione: Option<T>, f: F) -> Option<A>
        where F: FnOnce(T) -> A {
    match opzione {
        None => None,
        Some(valore) => Some(f(valore)),
    }
}
```

E in effetti, `map` è [definito come metodo][2] di `Option<T>` nella libreria
standard. Essendo un metodo, ha una firma leggermente diversa: i metodi
prendono `self`, `&self`, o `&mut self` come loro primo argomento.

Armati del nostro nuovo combinatore, possiamo riscrivere il nostro metodo
`estensione_esplicita` per sbarazzarci dell'analisi dei casi:

```rust
# fn trova(_: &str, _: char) -> Option<usize> { None }
// Restituisce l'estensione del dato nome di file, dove l'estensione è definita
// come tutti i caratteri che seguono il primo `.`.
// se `nome_di_file` non contiene nessun `.`, allora viene restituito `None`.
fn estensione(nome_di_file: &str) -> Option<&str> {
    trova(nome_di_file, '.').map(|i| &nome_di_file[i+1..])
}
```

Un altro pattern che si trova tipicamente è assegnare un valore di default al
caso in cui un valore di `Option` è `None`. Per esempio, può darsi che
in nostro programma assuma che l'estensione di un file sia `rs` anche se non
c'è nessuna estensione. Come si potrebbe immaginare, l'analisi dei casi
per questo non è specifica delle estensioni dei nomi di file - può funzionare
con qualunque `Option<T>`:

```rust
fn unwrap_or<T>(option: Option<T>, default: T) -> T {
    match option {
        None => default,
        Some(value) => value,
    }
}
```

Come con la funzione `map` di prima, l'implementazione della libreria standard
è un metodo invece di una funzione libera.

Qui il trucco è che il valore di default deve avere lo stesso tipo del valore
che potrebbe essere dentro l'`Option<T>`. Usarlo è facilissimo nel nostro caso:

```rust
# fn trova(pagliaio: &str, ago: char) -> Option<usize> {
#     for (scostamento, c) in pagliaio.char_indices() {
#         if c == ago {
#             return Some(scostamento);
#         }
#     }
#     None
# }
#
# fn estensione(nome_di_file: &str) -> Option<&str> {
#     trova(nome_di_file, '.').map(|i| &nome_di_file[i+1..])
# }
fn main() {
    assert_eq!(estensione("foobar.csv").unwrap_or("rs"), "csv");
    assert_eq!(estensione("foobar").unwrap_or("rs"), "rs");
}
```

(Si noti che `unwrap_or` è [definito come metodo][3] in `Option<T>` nella
libreria standard, e quindi qui usiamo quello invece della funzione libera che
abbiamo definito prima. Non ci si dimentichi andare a vedere il più generale
metodo [`unwrap_or_else`][4].)

C'è ancora un combinatore a cui pensiamo valga la pena prestare una speciale
attenzione: `and_then`. Rende facile comporre elaborazioni distinte
che ammettono la *possibilità di assenza*. Per esempio, molto del codice
in questa sezione riguarda il trovare un'estensione dato un nome di file.
Per poterlo fare, prima serve il nome di file che tipicamente è estratto da
un *percorso* di file. Mentre la maggior parte dei percorsi di file contengono
un nome di file, non è così per *tutti*. Per esempio, `.`, `..` o `/`.

Perciò, abbiamo il compito di trovare un'estensione dato un *percorso* di file.
Iniziamo con l'analisi esplicita dei casi:

```rust
# fn estensione(nome_di_file: &str) -> Option<&str> { None }
fn estensione_di_percorso_di_file_esplicita(percorso_di_file: &str) -> Option<&str> {
    match nome_di_file(percorso_di_file) {
        None => None,
        Some(name) => match estensione(name) {
            None => None,
            Some(ext) => Some(ext),
        }
    }
}

fn nome_di_file(percorso_di_file: &str) -> Option<&str> {
  // implementazione elisa
  unimplemented!()
}
```

Si potrebbe pensare che potremmo usare il combinatore `map` per ridurre
l'analisi dei casi, ma il suo tipo non è proprio adatto...

```rust,ignore
fn estensione_di_percorso_di_file(percorso_di_file: &str) -> Option<&str> {
    nome_di_file(percorso_di_file).map(|x| estensione(x)) // Errore di compilazione
}
```

Qui la funzione `map` avvolge il valore restituito dalla funzione `estensione`
dentro una `Option<_>` e siccome la stessa funzione `estensione` restituisce
una `Option<&str>`, l'espressione
`nome_di_file(percorso_di_file).map(|x| estensione(x))`
restituisce effettivamente un `Option<Option<&str>>`.

Ma siccome `estensione_di_percorso_di_file` restituisce appena `Option<&str>`
(e non `Option<Option<&str>>`) otteniamo un errore di compilazione.

Il risultato della funzione presa da `map` come input è *sempre* [riavvolto
da `Some`](#code-option-map). Invece, ci serve qualcosa come `map`, ma che
consenta al chiamante di restituire direttamente un'`Option<_>`
senza avvolgerlo in un altro `Option<_>`.

La sua implementazione generica è perfino più semplice di `map`:

```rust
fn and_then<F, T, A>(opzione: Option<T>, f: F) -> Option<A>
        where F: FnOnce(T) -> Option<A> {
    match opzione {
        None => None,
        Some(valore) => f(valore),
    }
}
```

Adesso possiamo riscrivere la nostra funzione `estensione_di_percorso_di_file`
senza l'esplicita analisi dei casi:

```rust
# fn estensione(nome_di_file: &str) -> Option<&str> { None }
# fn nome_di_file(percorso_di_file: &str) -> Option<&str> { None }
fn estensione_di_percorso_di_file(percorso_di_file: &str) -> Option<&str> {
    nome_di_file(percorso_di_file).and_then(estensione)
}
```

Nota a fianco: `and_then`, siccome funziona essenzialmente come `map`,
ma restituisce un `Option<_>` invece di un `Option<Option<_>>`,
è noto come `flatmap` in alcuni altri linguaggi.

Il tipo `Option` ha molti altri combinatori [definiti nella libreria standard]
[5]. È una buona idea scorrere questo elenco e familiarizzarsi con ciò che
è disponibile—spesso consentono di ridurre l'analisi dei casi. Familiarizzarsi
con questi combinatori tornerà utile perché molti di loro sono definiti anche
(con una simile semantica) per `Result`, di cui poi parleremo.

I combinatori rendono ergonomico l'uso di tipi come `Option`, perché riducono
l'analisi esplicita dei casi. Sono anche componibili, perché permettono
al chiamante di gestire la possibilità di assenza a modo suo. I metodi come
`unwrap` tolgono delle scelte, perché andranno in panico se `Option<T>`
vale `None`.

## Il tipo `Result`

Anche il tipo `Result` [definito nella libreria standard][6]:

<span id="code-result-def"></span>

```rust
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

Il tipo `Result` è una versione più ricca di `Option`. Invece di esprimere
la possibilità di *assenza* come fa `Option`, `Result` esprime la possibilità
di *errore*. Solitamente, l'*errore* serve a spiegare perché l'esecuzione
di qualche elaborazione è fallita. Questa è una forma strettamente più generale
di `Option`. Si consideri il seguente alias di tipo, che è semanticamente
del tutto equivalente al vero `Option<T>`:

```rust
type Option<T> = Result<T, ()>;
```

Questo fissa il secondo parametro di tipo di `Result` a essere sempre `()`
(pronunciato “unità” o “ennupla vuota"). Il tipo `()` è abitato da esattamente
un solo valore: `()`. (Già, i termini a livello di tipo e di valore hanno
la medesima notazione!)

Il tipo `Result` è un modo di rappresentare uno di due possibili esiti
di un'elaborazione. Per convenzione, un esito è pensato come atteso o “`Ok`”
mentre l'altro esito è pensato come inatteso o “`Err`”.

Proprio com `Option`, il tipo `Result` ha anche un [metodo `unwrap` definito]
[7] nella libreria standard. Definiamolo:

```rust
# enum Result<T, E> { Ok(T), Err(E) }
impl<T, E: ::std::fmt::Debug> Result<T, E> {
    fn unwrap(self) -> T {
        match self {
            Result::Ok(val) => val,
            Result::Err(err) =>
              panic!("chiamata `Result::unwrap()` su un valore `Err`: {:?}", err),
        }
    }
}
```

Questa è effettivamente la medesima cosa della nostra [definizione
di `Option::unwrap`](#code-option-def-unwrap), eccetto che comprende il valore
di errore nel messaggio di `panic!`. Ciò facilita il debugging, ma ci obbliga
anche ad aggiungere il vincolo [`Debug`][8] sul parametro di tipo `E` (che
rappresenta il nostro tipo di errore). Siccome la vasta maggioranza dei tipi
dovrebbe soddisfare il vincolo `Debug`, questo in pratica tende a funzionare.
(`Debug` su un tipo significa semplicemente che c'è un modo ragionevole
di stampare una descrizione umanamente leggibile dei valori di quel tipo.)

OK, passiamo a un esempio.

### Analizzare gli interi

La libreria standard di Rust rende estremamente semplice la conversione
di stringhe in interi. Ma è talmente facile, che viene la forte tentazione
di scrivere qualcosa come:

```rust
fn raddoppia_numero(numero_stringa: &str) -> i32 {
    2 * numero_stringa.parse::<i32>().unwrap()
}

fn main() {
    let n: i32 = raddoppia_numero("10");
    assert_eq!(n, 20);
}
```

A questo punto, si dovrebbe essere scettici sul chiamare `unwrap`. Per esempio,
se la stringa non è analizzabile come numero, si otterrà un panico:

```text
thread 'main' panicked at 'called `Result::unwrap()` on an `Err` value: ParseIntError { kind: InvalidDigit }', /home/rustbuild/src/rust-buildbot/slave/beta-dist-rustc-linux/build/src/libcore/result.rs:729
```

Questo è abbastanza inguardabile, e se è accaduto dentro una libreria che
si sta usando, si potrebbe essere comprensibilmente seccati. Invece, dovremmo
provare a gestire l'errore nella nostra funzione e consentire al chiamante
di decidere cosa fare. Ciò comporta la modifica del tipo del valore restituito
da `raddoppia_numero`. Ma a quale tipo? Beh, bisogna guardare la firma
del [metodo `parse`][9] nella libreria standard:

```rust,ignore
impl str {
    fn parse<F: FromStr>(&self) -> Result<F, F::Err>;
}
```

Ehm. Allora almeno sappiamo che dobbiamo usare un `Result`. Certamente, avrebbe
potuto restituire un `Option`. Dopo tutto, una stringa o è valida come numero
o non lo è, no? Sarebbe certamente un modo ragionevole di procedere, ma
l'implementazione distingue internamente *perché* la stringa non è un intero
valido. (Potrebbe essere una stringa vuota, avere una cifra non valida, essere
un numero troppo grande o troppo piccolo.) Perciò, usare un `Result` ha senso
perché vogliamo fornire più informazione della semplice “assenza.” Vogliamo
dire *perché* la conversione è fallita. Si dovrebbe provare a emulare questa
linea di ragionamento quando si affronta una scelta fra `Option` e `Result`.
Se si può fornire qualche dettaglio sull'errore, allora probabilmente si
dovrebbe farlo. (Ritorneremo su questa questione più avanti.)

OK, ma come scriviamo il nostro tipo del valore restituito? Il metodo `parse`
definito sopra è generico su tutti i diversi tipi numerici definiti nella
libreria standard. Potremmo (e probabilmente dovremmo) rendere generica anche
la nostra funzione, ma per il momento cerchiamo di favorire l'esplicitazione.
Ci interessa solamente `i32`, perciò dobbiamo [trovare la sua implementazione
di `FromStr`](../std/primitive.i32.html) (si cerchi “FromStr” nella pagina
di `i32`) e si guardi il suo [tipo associato][10] `Err`. In questo caso,
è [`std::num::ParseIntError`](../std/num/struct.ParseIntError.html).
Infine, possiamo riscrivere la nostra funzione:

```rust
use std::num::ParseIntError;

fn raddoppia_numero(number_str: &str) -> Result<i32, ParseIntError> {
    match number_str.parse::<i32>() {
        Ok(n) => Ok(2 * n),
        Err(err) => Err(err),
    }
}

fn main() {
    match raddoppia_numero("10") {
        Ok(n) => assert_eq!(n, 20),
        Err(err) => println!("Error: {:?}", err),
    }
}
```

Questo è un po' meglio, ma adesso abbiamo scritto molto più codice! L'analisi
dei casi ci ha morso ancora una volta.

I combinators vengono in aiuto! Proprio come `Option`, `Result` ha molti
combinatori definiti come metodi. Ci sono molti combinatori in comune tra
`Result` e `Option`. In particolare, `map` fa parte di questa intersezione:

```rust
use std::num::ParseIntError;

fn raddoppia_numero(number_str: &str) -> Result<i32, ParseIntError> {
    number_str.parse::<i32>().map(|n| 2 * n)
}

fn main() {
    match raddoppia_numero("10") {
        Ok(n) => assert_eq!(n, 20),
        Err(err) => println!("Error: {:?}", err),
    }
}
```

I soliti sospetti sono tutti liì per `Result`, inclusi
[`unwrap_or`](../std/result/enum.Result.html#method.unwrap_or) e
[`and_then`](../std/result/enum.Result.html#method.and_then).
Inoltre, siccome `Result` ha un secondo parametri di tipo, ci sono dei
combinatori che influenzano solamente il tipo di errore, come
[`map_err`](../std/result/enum.Result.html#method.map_err) (invece di
`map`) e [`or_else`](../std/result/enum.Result.html#method.or_else)
(invece di `and_then`).

### L'idioma dell'alias del tipo `Result`

Nella libreria standard, si possono vedere frequentemente dei tipi come
`Result<i32>`. Ma, un momento, [abbiamo definito `Result`](#code-result-def)
in modo che abbia due parametri di tipo. Come possiamo cavarcela specificandone
solamente uno? La chiave è definire un alias di tipo `Result` che *fissa* uno
dei parametri di tipo a un particolare tipo. Solitamente il tipo fissato è
il tipo di errore. Per esempio, il nostro esempio precedente che convertiva
gli interi potrebbe essere riscritto così:

```rust
use std::num::ParseIntError;
use std::result;

type Result<T> = result::Result<T, ParseIntError>;

fn raddoppia_numero(number_str: &str) -> Result<i32> {
    unimplemented!();
}
```

Perché faremmo così? Beh, se abbiamo molte funzioni che potrebbero restituire
`ParseIntError`, allora è molto più comodo definire un alias che usa sempre
`ParseIntError`, così che non dobbiamo riscriverlo tutte le volte.

Il luogo più importante nella libreria standard in cui questo idioma
è utilizzato è con [`io::Result`](../std/io/type.Result.html). Tipicamente,
si scrive `io::Result<T>`, che rende chiaro che si sta usando l'alias di tipo
del modulo `io` invece della definizione base tratta da `std::result`. (Questo
idioma viene usato anche per [`fmt::Result`](../std/fmt/type.Result.html).)

## Un breve interludio: lo svolgimento non è male

Chi avesse seguito fin qui, potrebbe aver notato che è stata presa una linea
piuttosto dura contro il chiamare i metodi come `unwrap` che potrebbero andare
in `panic` e far abortire il programma. *In generale*, questo è un buon
consiglio.

Però, `unwrap` può ancora essere usato con giudizio. Ciò che giustifica
esattamente l'utilizzo di `unwrap` è un'area indefinita e varie persone
ragionevoli possono pensarla diveramente. Ecco alune *opinioni* in materia.

* **Nel codice di esempio o abborracciato.** Talvolta si scrivono esempi
  o un programma di getto, e la gestione degli errori non è così importante.
  In tali scenari, può essere difficile battere la comodità di `unwrap`, quindi
  è molto attraente.
* **Quando andare in panico indica un grave difetto del programma.** Quando
  gli invarianti del proprio codice dovrebbero prevenire che una certa
  situazione avvenga (come, diciamo, rimuove un elemento da uno stack vuoto),
  allora andare in panico può essere accettato. Questo è perché espone
  un grave difetto del proprio programma. Questo panico può essere esplicito,
  come quando una `assert!` fallisce, o potrebbe essere perché l'indice in
  un array era uscito dai limiti.

Questa probabilmente non è una lista esauriente. Inoltre, quando si usa
un `Option`, è spesso meglio usare il suo metodo [`expect`]
(../std/option/enum.Option.html#method.expect). `expect` fa esattamente
la medesima cosa di `unwrap`, eccetto che stampa il messaggio ricevuto come
argomento. Ciò rende il panico risultante un po' più carino da trattare, dato
che mostrerà il nostro messaggio invece di “called unwrap on a `None` value.”

Questi consigli si riducono a questo: usare giudizio. C'è una ragione per cui
le parole “non fare mai X” o “Y è considerato dannoso” non appaiono in questo
libro. Ci sono pro e contro in tutte le cose, e sta allo sviluppatore
determinare che cosa è accettabile per i propri casi d'uso. L'obiettivo
di questo libro è solamente aiutare a valutare i pro e i contro il più
accuratamente possibile.

Adesso che abbiamo trattato le basi della gestione degli errori in Rust, e
spiegato lo svolgimento, iniziamo a esplorare di più la libreria standard.

# Lavorare con più tipi di errori

Finora, abbiamo visto la gestione degli errori dove tutto era o un `Option<T>`
o un `Result<T, SomeError>`. Ma che succede quando ci sono sia un `Option` che
un `Result`? O che fare se ci sono sia un `Result<T, Error1>` che un
`Result<T, Error2>`? Gestire la *composizione di tipi distinti di errori* è
la prossima sfida che affrontiamo, e sarà il tema principale per tutto
il resto di questa sezione.

## Comporre `Option` e `Result`

Finora, abbiamo parlato dei combinatori definiti per `Option` e dei combinatori
definiti per `Result`. Possiamo usare questi combinatori per comporre
i risultati of di diverse elaborazioni senza fare un'analisi esplicita
dei casi.

Naturalmente, nel vero codice, le cose non sono sempre così pulite. Talvolta
c'è un miscuglio di tipi `Option` e `Result`. Dobbiamo ricorrere all'analisi
esplicita dei casi, o possiamo continuare a usare i combinatori?

Per adesso, rivediamo uno dei primi esempi in questa sezione:

```rust,should_panic
use std::env;

fn main() {
    let mut argv = env::args();
    let arg: String = argv.nth(1).unwrap(); // error 1
    let n: i32 = arg.parse().unwrap(); // error 2
    println!("{}", 2 * n);
}
```

Data la nostra nuova conoscenza di `Option`, `Result` e dei loro vari
combinatori, dovremmo provare e riscriverlo in modo che gli errori siano
gestiti appropriatamente e il programma non vada in panico se c'è un errore.

Qui l'aspetto delicato è che `argv.nth(1)` produce un `Option`, mentre
`arg.parse()` produce un `Result`, che non sono direttamente componibili.
Quando si affrontano sia un `Option` che un `Result`, la soluzione
*solitamente* è convertire l'`Option` in un `Result`. Nel nostro caso,
l'assenza di un argomento di riga di comando (da `env::args()`) significa
che l'utente non ha invocato correttamente il programma. Potremmo usare
una `String` per descrivere questo errore. Proviamo:

<span id="code-error-double-string"></span>

```rust
use std::env;

fn raddoppia_arg(mut argv: env::Args) -> Result<i32, String> {
    argv.nth(1)
        .ok_or("Per favore, inserisci almeno un argomento".to_owned())
        .and_then(|arg| arg.parse::<i32>().map_err(|err| err.to_string()))
        .map(|n| 2 * n)
}

fn main() {
    match double_arg(env::args()) {
        Ok(n) => println!("{}", n),
        Err(err) => println!("Error: {}", err),
    }
}
```

Ci sono un paio di cose nuove in questo esempio. La prima è l'uso
del combinatore [`Option::ok_or`](../std/option/enum.Option.html#method.ok_or).
Questo è un modo di convertire un `Option` in un `Result`. La conversione
obbliga a specificare quale errore usare se `Option` è `None`. Come gli altri
combinatori che abbiamo visto, la sua definizione è molto semplice:

```rust
fn ok_or<T, E>(option: Option<T>, err: E) -> Result<T, E> {
    match option {
        Some(val) => Ok(val),
        None => Err(err),
    }
}
```

L'altro nuovo combinatore usato qui è [`Result::map_err`]
(../std/result/enum.Result.html#method.map_err).
Questo è come `Result::map`, eccetto che mappa una funzione sulla porzione
*errore* di un valore `Result`. Se il `Result` è un valore `Ok(...)`, allora
viene restituito non modificato.

Qui usiamo `map_err` perché è necessario per i tipi di errori rimanere
i medesimi (a causa del nostro uso di `and_then`). Siccome abbiamo scelto
di convertire il `Option<String>` (da `argv.nth(1)`)
a un `Result<String, String>`, dobbiamo convertire anche il `ParseIntError`
da `arg.parse()` a una `String`.

## I limiti dei combinatori

Fare dell'I/O e analizzare l'input sono attività molto tipiche. Pertanto,
continueremo a usare varie routine di I/O e di analisi per esemplificare
la gestione degli errori.

Iniziamo dal semplice. Abbiamo il compito di aprire un file testuale, leggerlo
tutto e convertire il suo contenuto in un numero, che moltiplichiamo per `2`,
e poi stampiamo il risultato.

Sebbene abbia provato a convincere di non usare `unwrap`, può essere utile
scrivere dapprima il codice usando `unwrap`. Consente di concentrarsi
sul problema invece che sulla gestione degli errori, ed espone i punti in cui
deve avvenire l'appropriata gestione degli errori. Iniziamo così, in modo
da poter avere del codice su cui ragionare, e poi lo rifattorizziamo per usare
una migliore gestione degli errori.

```rust,should_panic
use std::fs::File;
use std::io::Read;
use std::path::Path;

fn raddoppia_file<P: AsRef<Path>>(percorso_di_file: P) -> i32 {
    let mut file = File::open(percorso_di_file).unwrap(); // error 1
    let mut contenuto = String::new();
    file.read_to_string(&mut contenuto).unwrap(); // error 2
    let n: i32 = contenuto.trim().parse().unwrap(); // error 3
    2 * n
}

fn main() {
    let doubled = raddoppia_file("foobar");
    println!("{}", doubled);
}
```

(N.B. Il tratto `AsRef<Path>` viene usato perché i [medesimi legami sono usati
da `std::fs::File::open`](../std/fs/struct.File.html#method.open). Questo
rende ergonomico da usare qualunque tipo di stringa come percorso di file.)

Qui ci sono tre diversi errori che possono avvenire:

1. Un fallimento nell'apertura del file.
1. Un fallimento nella lettura dei dati dal file.
1. Un fallimento nella conversione del testo in un numero.

I primi due errori sono descritti tramite il tipo [`std::io::Error`]
(../std/io/struct.Error.html). Lo sappiamo a causa dei tipi dei valori
restituiti da [`std::fs::File::open`](../std/fs/struct.File.html#method.open)
 e da [`std::io::Read::read_to_string`]
(../std/io/trait.Read.html#method.read_to_string).
(Si noti che entrambi usano l'[idioma dell'alias del tipo `Result`]
(#the-result-type-alias-idiom) descritto prima. Se si clicca sul tipo `Result`,
si [vedrà l'alias del tipo](../std/io/type.Result.html), e di conseguenza,
il tipo `io::Error` soggiacente.)  Il terzo errore è descritto dal tipo
[`std::num::ParseIntError`](../std/num/struct.ParseIntError.html). Il tipo
`io::Error` in particolare è *pervasivo* nella libreria standard. Lo si rivedrà
più volte.

Iniziamo il procedimento di rifattorizzazione della funzione `raddoppia_file`.
Affinché questa funzione sia componibile con altri componenti del programma,
*non* dovrebbe andare in panico se si incontrano qualcune delle suddette
condizioni d'errore. Effettivamente, questo comporta che la funzione dovrebbe
*restituire un errore* se qualcuna delle sue operazioni fallisce. Il nostro
problema è che il tipo del valore restituito da `raddoppia_file` è `i32`,
che non ci dà nessun modo utile di riportare un errore. Così, dobbiamo iniziare
cambiando il tipo del valore restituito da `i32` a qualcos'altro.

La prima cosa che dobbiamo decidere: dovremmo usare `Option` o `Result`?
Certamente potremmo usare `Option` molto facilmente. Se avviene uno dei
tre errori, potremmo seplicemente restituire `None`. Questo funzionerà *ed è
meglio che andare in panico*, ma possiamo fare molto meglio. Invece, dovremmo
passare qualche dettaglio sull'errore che è avvenuto. Siccome vogliamo
esprimere la *possibilità di errore*, dovremmo usare `Result<i32, E>`. Ma
che cosa dovrebbe essere `E`? Siccome possono succedere due *diversi* tipi
di errori, dobbiamo convertirli a un tipo comune. Un tale tipo è `String`.
Vediamo che impatto ha sul nostro codice:

```rust
use std::fs::File;
use std::io::Read;
use std::path::Path;

fn raddoppia_file<P: AsRef<Path>>(percorso_di_file: P) -> Result<i32, String> {
    File::open(percorso_di_file)
         .map_err(|err| err.to_string())
         .and_then(|mut file| {
              let mut contenuto = String::new();
              file.read_to_string(&mut contenuto)
                  .map_err(|err| err.to_string())
                  .map(|_| contenuto)
         })
         .and_then(|contenuto| {
              contenuto.trim().parse::<i32>()
                      .map_err(|err| err.to_string())
         })
         .map(|n| 2 * n)
}

fn main() {
    match raddoppia_file("foobar") {
        Ok(n) => println!("{}", n),
        Err(err) => println!("Error: {}", err),
    }
}
```

Questo codice sembra un po' oscuro. Ci può volere un bel po' di pratica prima
che del codice come questo diventi facile da scrivere. Il modo in cui
lo scriviamo è *seguendo i tipi*. Non appena cambiamo il tipo del valore
restituito da `raddoppia_file` a `Result<i32, String>`, abbiamo dovuto
iniziare a cercare i giusti combinatori. In questo caso, abbiamo usato
solamente tre diversi combinatori: `and_then`, `map`, e `map_err`.

`and_then` viene usato per concatenare più elaborazioni dove ogni elaorazione
potrebbe restituire un errore. Dopo aver apert il file, ci sono die altre
elaborazioni che potrebbero fallire: leggere dal file e convertire il contenuto
in un numero. Di conseguenza, ci sono due chiamate a `and_then`.

`map` viene usato per applicare una funzione al valore `Ok(...)` di
un `Result`. Per esempio, l'ultimissima chiamata a `map` moltiplica per `2`
il valore `Ok(...)` (che è un `i32`). Se fosse successo un errore prima di quel
punto, questa operazione sarebbe stata saltata, per come è definito `map`.

`map_err` è il trucco che fa funzionare il tutto. `map_err` è come `map`,
eccetto che applica una funzione al valore `Err(...)` di un `Result`. In questo
caso, vogliamo convertire tutti i nostri errori a un solo tipo: `String`.
Siccome sia `io::Error` che `num::ParseIntError` implementano `ToString`,
possiamo chiamare il metodo `to_string()` per convertirli.

Annche avendo detto tutto ciò, il codice rimane oscuro. Impadronirsi dell'uso
dei combinatori è importante, ma hanno i loro limiti. Proviamo un approccio
diverso: le uscite precoci.

## Uscite precoci

Mi piacerebbe prendere il codice della sottosezione precedente e riscriverlo
usando delle *uscite precoci*. Le uscite precoci consentono di uscire da
una funzione prima della fine. Non si può uscire precocemente
da `raddoppia_file` quando si è dentro una chiusura, quindi dovremo ritornare
all'analisi esplicita dei casi.

```rust
use std::fs::File;
use std::io::Read;
use std::path::Path;

fn raddoppia_file<P: AsRef<Path>>(percorso_di_file: P) -> Result<i32, String> {
    let mut file = match File::open(percorso_di_file) {
        Ok(file) => file,
        Err(err) => return Err(err.to_string()),
    };
    let mut contenuto = String::new();
    if let Err(err) = file.read_to_string(&mut contenuto) {
        return Err(err.to_string());
    }
    let n: i32 = match contenuto.trim().parse() {
        Ok(n) => n,
        Err(err) => return Err(err.to_string()),
    };
    Ok(2 * n)
}

fn main() {
    match raddoppia_file("foobar") {
        Ok(n) => println!("{}", n),
        Err(err) => println!("Error: {}", err),
    }
}
```

Persone ragionevoli possono discordare sul fatto che questo codice
sia migliore o peggiore del codice che usa i combinatori, ma se non si è
familiari con l'approccio dei combinatori, questo codice appare più semplice
da leggere. Usa l'analisi esplicita dei casi con `match` e `if let`.
Se succede un errore, semplicemente smette di eseguire la funzione
e restituisce l'errore (convertendolo a una stringa).

Però, questo non è un passo indietro? Prima, abbiamo detto che la chiave per
una gestione ergonomica degli errori sta nel ridurre l'analisi esplicita
dei casi, però qui siamo ritornati l'analisi esplicita dei casi. Però, pare che
ci siano *più* modi di ridurre l'analisi esplicita dei casi. I combinatori
non sono l'unico modo.

## La macro `try!`

Una pietra angolare della gestione degli errori in Rust è la macro `try!`.
La macro `try!` astrae l'analisi dei casi come i combinatori, ma diversamente
dai combinatori, astrae anche il *flusso di controllo*. Propriamente, può
astrarre il pattern dell'*uscita precoce* visto prima.

Ecco una definizione semplificata di una macro `try!`:

<span id="code-try-def-simple"></span>

```rust
macro_rules! try {
    ($e:expr) => (match $e {
        Ok(val) => val,
        Err(err) => return Err(err),
    });
}
```

(La [vera definizione](../std/macro.try!.html) è un po' più sofisticata.
Ne parleremo più avanti.)

Usare la macro `try!` facilita molto la semplificazione del nostro ultimo
esempio. Siccome si occupa di eseguire l'analisi dei casi e l'uscita precoce,
otteniamo del codice più compatto, che è più facile da leggere:

```rust
use std::fs::File;
use std::io::Read;
use std::path::Path;

fn raddoppia_file<P: AsRef<Path>>(percorso_di_file: P) -> Result<i32, String> {
    let mut file = try!(File::open(percorso_di_file).map_err(|e| e.to_string()));
    let mut contenuto = String::new();
    try!(file.read_to_string(&mut contenuto).map_err(|e| e.to_string()));
    let n = try!(contenuto.trim().parse::<i32>().map_err(|e| e.to_string()));
    Ok(2 * n)
}

fn main() {
    match raddoppia_file("foobar") {
        Ok(n) => println!("{}", n),
        Err(err) => println!("Error: {}", err),
    }
}
```

Le chiamate a `map_err` sono ancora necessarie data [la nostra definizione
di `try!`](#code-try-def-simple). Questo perché i tipi di errori hanno ancora
bisogno di essere convertiti in `String`. La buona notizia è che presto
impareremo a togliere quelle chiamate a `map_err`! La cattiva notizia è che
dovremo imparare qualcos'altro di un paio di tratti importanti
della libreria standard prima di poter togliere le chiamate a `map_err`.

## Definire il proprio tipo di errori

Prima di tuffarci in alcuni dei tratti di errore della libreria standard,
concludiamo questa sottosezione togliendo l'uso di `String` come nostro tipo
di errore negli esempi precedenti.

Usare `String` come abbiamo fatto nei notri esempi precedenti è comodo perché
è facile convertire gli errori in stringhe, o perfino costruire all'occorrenza
i propri errori come stringhe. Però, usare `String` per i propri errori ha
alcuni svantaggi.

Il primo svantaggio è che i messaggi d'errore tendono ad ingombrare il codice.
È possibile definire altrove i messaggi d'errore, ma a meno di essere
insolitamente disciplinati, c'è la forte tentazione di incorporare i messaggi
d'errore nel proprio codice. In effetti, è proprio quello che abbiamo fatto
in un [esempio precedente](#code-error-double-string).

Il secondo e più importante svantaggio è che le stringhe *perdono
informazioni*. Cioè, se tutti gli errori sono convertiti in stringhe, allora
gli errori che passiamo al chiamante diventano completamente opachi sulla
causa dell'errore. L'unica cosa ragionevole che il chiamante può fare ricevendo
un errore di tipo `String` è mostrarlo all'utente. Certamente, ispezionare
la stringa per determinare il tipo di errore non è robusto. (Chiaramente,
questo svantaggio è molto peggiore dentro una libreria che dentro
un'applicazione.)

Per esempio, il tipo `io::Error` incorpora un [`io::ErrorKind`]
(../std/io/enum.ErrorKind.html), che è un *dato strutturato* che rappresenta
ciò che è andato storto durante un'operazione di I/O. Questo è importante
perché si potrebbe voler reagire diversamente a seconda dell'errore. (per es.,
un errore di tipo `BrokenPipe` potrebbe causare l'uscita pulita dal programma,
mentre un errore di tipo `NonTrovati` potrebbe causare solamente che
un messaggio d'errore viene mostrato all'utente.) Con `io::ErrorKind`,
il chiamante può esaminare il tipo di un errore con l'analisi dei casi,
che è strettamente migliore di provare a racimolare i dettagli di un errore
frugando all'interno di una stringa.

Invece di usare una `String` come tipo di errore nel nostro esempio precedente
di lettura di un intero da un file, possiamo definire il nostro tipo di errori
che rappresenta gli errori con dei *dati strutturati*. Cerchiamo di non perdere
informazioni dagli errori soggiacienti nel caso il chiamante volesse
ispezionare i dettagli.

Il modo ideale di rappresentare *una tra molte possibilità* è definire il
nostro tipo somma usando `enum`. Nel nostro caso, un errore è o un `io::Error`
o un `num::ParseIntError`, e quindi ne deriva una definizione naturale:

```rust
use std::io;
use std::num;

// Deriviamo da `Debug` perché tutti i tipi probabilmente dovrebbero derivare
// da `Debug`. Questo ci dà una ragionevole descrizione umanamente leggibile
// dei valori `CliError`.
#[derive(Debug)]
enum CliError {
    Io(io::Error),
    Parse(num::ParseIntError),
}
```

Ritoccare il nostro codice è molto facile. Invece di convertire gli errori
in stringhe, semplicemente li convertiamo al nostro tipo `CliError`,
usando il corrispondente costruttore di valore:

```rust
# #[derive(Debug)]
# enum CliError { Io(::std::io::Error), Parse(::std::num::ParseIntError) }
use std::fs::File;
use std::io::Read;
use std::path::Path;

fn raddoppia_file<P: AsRef<Path>>(percorso_di_file: P) -> Result<i32, CliError> {
    let mut file = try!(File::open(percorso_di_file).map_err(CliError::Io));
    let mut contenuto = String::new();
    try!(file.read_to_string(&mut contenuto).map_err(CliError::Io));
    let n: i32 = try!(contenuto.trim().parse().map_err(CliError::Parse));
    Ok(2 * n)
}

fn main() {
    match raddoppia_file("foobar") {
        Ok(n) => println!("{}", n),
        Err(err) => println!("Error: {:?}", err),
    }
}
```

Qui le uniche modifich sono passare da `map_err(|e| e.to_string())` (che
converte gli errori in stringhe) a `map_err(CliError::Io)` o
`map_err(CliError::Parse)`. Il *chiamante* può decidere il livello di dettaglio
da riferire all'utente. In effetti, usare una `String` come tipo di errore
toglie scelte al chiamante mentre usare un tipo di errore `enum` personalizzato
come `CliError` dà al chiamante tutte le comodità di prima più un *dato
strutturato* che descrive l'errore.

Una regola empirica è definire il proprio tipo di errore, ma un tipo di errore
`String` potrà anche bastare, particolarmente se si sta scrivendo
un'applicazione. Se invece si sta scrivendo una libreria, definire il proprio
tipo di errori dovrebbe essere fortemente preferito, così da non togliere
scelte al chiamante senza necessità.

# I tratti della libreria standard usati per la gestione degli errori

La libreria standard definisce due tratti completi per la gestione
degli errori: [`std::error::Error`](../std/error/trait.Error.html) e
[`std::convert::From`](../std/convert/trait.From.html). Mentre `Error`
è progettato specificamente per descrivere genericamente degli errori,
il tratto `From` ha lo scopo più generale di convertire valori fra due tipi
distinti.

## Il tratto `Error`

Il tratto `Error` è [definito nella libreria standard]
(../std/error/trait.Error.html):

```rust
use std::fmt::{Debug, Display};

trait Error: Debug + Display {
  /// Una breve descrizione dell'errore.
  fn description(&self) -> &str;

  /// La causa di basso livello di questo errore, se esiste.
  fn cause(&self) -> Option<&Error> { None }
}
```

Questo tratto è super-generico, perché è pensato per essere implementato
per *tutti* i tipi che rappresentano errori. Ciò si dimostrerà utile
per scrivere codice componibile come vedremo più avanti. Altrimenti, il tratto
permette di fare almeno le seguenti cose:

* Ottenere una rappresentazione dell'errore per il programmatore,
  tramite `Debug`.
* Ottenere una rappresentazione dell'errore da mostrare all'utente,
  tramite `Display`.
* Ottenere una breve descrizione dell'errore (tramite il metodo `description`).
* Ispezionare la catena causale dell'errore, se ne esiste una (tramite
  il metodo `cause`).

Le prime due cose sono dovute al fatto che `Error` richiede che siano
implementati sia `Debug` che `Display`. Le ultime due cose sono dovute
ai due metodi defined in `Error`. Il potere di `Error` deriva dal fatto che
tutti i tipi di errore implementano `Error`, che significa che gli errori
possono essere quantificati esistenzialmente come un [oggetto tratto]
(../book/trait-objects.html). Questo si manifesta come o un `Box<Error>`
o un `&Error`. In effetti, il metodo `cause` restituisce un `&Error`, che è
esso stesso un oggetto tratto. Più avanti ritorneremo sull'utilità del tratto
`Error` come oggetto tratto.

Per adesso, basta mostrare un esempio che implementa il tratto `Error`. Usiamo
il tipo di errore che abbiamo definito nella [sottosezione precedente]
(#defining-your-own-error-type):

```rust
use std::io;
use std::num;

// Deriviamo da `Debug` perché tutti i tipi probabilmente dovrebbero derivare
// da `Debug`. Questo ci dà una ragionevole descrizione umanamente leggibile
// dei valori `CliError`.
#[derive(Debug)]
enum CliError {
    Io(io::Error),
    Parse(num::ParseIntError),
}
```

Questo particolare tipo di errore rappresenta la possibilità che avvengano
due tipi di errore: un errore che riguarda l'I/O o un errore nel convertire
una stringa in un numero. L'errore potrebbe rappresentare tanti tipi di errore
quanti se ne vogliono, aggiungendo nuove varianti alla definizione dell'`enum`.

Implementare `Error` è abbastanza lineare. Per lo più si tratta di fare molte
analisi esplicite dei casi.

```rust,ignore
use std::error;
use std::fmt;

impl fmt::Display for CliError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        match *self {
            // Entrambi gli errori soggiacenti implementano già `Display`,
            // perciò rimandiamo alle loro implementazioni.
            CliError::Io(ref err) => write!(f, "IO error: {}", err),
            CliError::Parse(ref err) => write!(f, "Parse error: {}", err),
        }
    }
}

impl error::Error for CliError {
    fn description(&self) -> &str {
        // Entrambi gli errori soggiacenti implementano già `Error`,
        // perciò rimandiamo alle loro implementazioni.
        match *self {
            CliError::Io(ref err) => err.description(),
            CliError::Parse(ref err) => err.description(),
        }
    }

    fn cause(&self) -> Option<&error::Error> {
        match *self {
            // N.B. Entrambi convertono implicitamente `err` dai loro tipi
            // concreti (o `&io::Error` o `&num::ParseIntError`)
            // a un oggetto tratto `&Error`. Questo funziona perché entrambi
            // i tipi di errore implementano `Error`.
            CliError::Io(ref err) => Some(err),
            CliError::Parse(ref err) => Some(err),
        }
    }
}
```

Notiamo che questa è una implementazione molto tipica di `Error`: si applica
`match` ai propri diversi tipi di errore e si soddisfano i contratti
definiti per `description` e `cause`.

## Il tratto `From`

Il tratto `std::convert::From` è [definito nella libreria standard]
(../std/convert/trait.From.html):

<span id="code-from-def"></span>

```rust
trait From<T> {
    fn from(T) -> Self;
}
```

Deliziosamente semplice, no? `From` è molto utile perché ci dà us un modo
generico di parlare della conversione *da* un tipo particolare `T` a qualche
altro tipo (in questo caso, “qualche altro tipo” è il soggetto
dell'implementazione, ossia `Self`). Il punto cruciale di `From` è
l'[insieme di implementazioni fornite dalla libreria standard]
(../std/convert/trait.From.html).

Ecco alcuni semplici esempi che mostrano come funziona `From`:

```rust
let string: String = From::from("foo");
let bytes: Vec<u8> = From::from("foo");
let cow: ::std::borrow::Cow<str> = From::from("foo");
```

OK, quindi `From` serve a convertire fra stringhe. Ma, e per gli errori?
Risulta esistere un'implementazione cruciale:

```rust,ignore
impl<'a, E: Error + 'a> From<E> for Box<Error + 'a>
```

Questa implementazione dice che per *ogni* tipo che implementa `Error`,
possiamo convertirlo in un oggetto tratto `Box<Error>`. Questo può non sembrare
terribilmente sorprendente, ma è utile in un contesto generico.

Ricordiamo i due errori che stavamo trattando prima? Specificamente,
`io::Error` e `num::ParseIntError`. Dato che entrambi implementano `Error`,
funzionano con `From`:

```rust
use std::error::Error;
use std::fs;
use std::io;
use std::num;

// Dobbiamo saltare in alcuni cerchi per ottenere effettivamente
// i valori di errore.
let io_err: io::Error = io::Error::last_os_error();
let parse_err: num::ParseIntError = "non è un numero".parse::<i32>().unwrap_err();

// OK, ecco le conversioni.
let err1: Box<Error> = From::from(io_err);
let err2: Box<Error> = From::from(parse_err);
```

Qui c'è un pattern veramente importante da riconoscere. `err1` ed `err2` hanno
il *medesimo tipo*. Questo perché sono tipi quantificati esistenzialmente,
od oggetti tratto. In particolare, il loro tipo soggiaciente viene *eliminato*
dalla conoscenza del compilatore, in modo che veda realmente `err1` e `err2`
come esattamente del medesimo tipo. Inoltre, abbiamo costruito `err1` e `err2`
usando precisamente la medesima chiamata di funzione: `From::from`.
Questo perché `From::from` è sovraccaricata sia sul suo argomento che sul suo
tipo del valore restituito.

Questo pattern è importante perché risolve un problema che avevamo prima: ci dà
un modo di convertire affidabilmente gli erroi al medesimo stesso tipo usando
la medesima funzione.

È ora di rivedere un vecchio amico; la macro `try!`.

## La vera macro `try!`

Prima, abbiamo presentato questa definizione di `try!`:

```rust
macro_rules! try {
    ($e:expr) => (match $e {
        Ok(val) => val,
        Err(err) => return Err(err),
    });
}
```

Ma questa non è la sua vera definizione. La sua vera definizione
[nella libreria standard](../std/macro.try!.html) è:

<span id="code-try-def"></span>

```rust
macro_rules! try {
    ($e:expr) => (match $e {
        Ok(val) => val,
        Err(err) => return Err(::std::convert::From::from(err)),
    });
}
```

C'è un minuscolo ma potente cambiamento: il valore di errore viene passato
attraverso `From::from`. Questo rende la macro `try!` molto più potente perché
fornisce gratis la conversione di tipo automatica.

Armati con la nostra più potente macro `try!`, diamo un'occhiata al codice
che abbiamo scritto prima per leggere un file e convertire il suo contenuto
in un intero:

```rust
use std::fs::File;
use std::io::Read;
use std::path::Path;

fn raddoppia_file<P: AsRef<Path>>(percorso_di_file: P) -> Result<i32, String> {
    let mut file = try!(File::open(percorso_di_file).map_err(|e| e.to_string()));
    let mut contenuto = String::new();
    try!(file.read_to_string(&mut contenuto).map_err(|e| e.to_string()));
    let n = try!(contenuto.trim().parse::<i32>().map_err(|e| e.to_string()));
    Ok(2 * n)
}
```

Prima, abbiamo promesso che potevamo sbarazzarci delle chiamate `map_err`.
Infatti, quel che dobbiamo fare è prendere un tipo con cui `From` funziona.
Come abbiamo visto nella sottosezione precedente, `From` ha un'implementazione
che consente di convertire un errore di qualunque tipo in un `Box<Error>`:

```rust
use std::error::Error;
use std::fs::File;
use std::io::Read;
use std::path::Path;

fn raddoppia_file<P: AsRef<Path>>(percorso_di_file: P) -> Result<i32, Box<Error>> {
    let mut file = try!(File::open(percorso_di_file));
    let mut contenuto = String::new();
    try!(file.read_to_string(&mut contenuto));
    let n = try!(contenuto.trim().parse::<i32>());
    Ok(2 * n)
}
```

Ci stiamo avvicinando molto alla gestione ideale degli errori. Il nostro codice
ha pochissimo spreco dovuto alla gestione degli errori, perché la macro `try!`
incapsula tre cose simultaneamente:

1. L'analisi dei casi.
1. Il flusso di controllo.
1. La conversione del tipo di errore.

Quando si combinano tutte e tre le cose, otteniamo del codice che non è
ingombrato da combinatori, né da chiamate a `unwrap`, né da analisi dei casi.

È rimasto un piccolo neo: il tipo `Box<Error>` è *opaco*. Se restituiamo
un `Box<Error>` al chiamante, il chiamante non può (facilmente) ispezionare
il tipo di errore soggiacente. La situazione è certamente migliore dell'uso
di `String`, perché il chiamante può chiamare dei metodi come [`description`]
(../std/error/trait.Error.html#tymethod.description) e [`cause`]
(../std/error/trait.Error.html#method.cause), ma la limitazione rimane:
`Box<Error>` è opaco. (N.B. Ciò  non è del tutto vero, perché Rust ha
l'introspezione in fase di esecuzione, che serve in alcuni scenari che
[esulano dallo scopo di questa sezione](https://crates.io/crates/error).)

È ora di rivedere il nostro tipo `CliError` personalizzato e legare insieme
il tutto.

## Comporre tipi di errore personalizzati

Nell'ultima sezione, abbiamo visto la vera macro `try!` e come faccia
una conversione automatica dei tipi chiamando `From::from` sul valore
di errore. In particolare, abbiamo convertito degli errori in `Box<Error>`,
il che funziona, ma il tipo è opaco ai chiamanti.

Per correggerlo, usiamo il medesimo rimedio con cui abbiamo già familiarizzato:
Un tipo di errore personalizzato. Ancora un volta, ecco il codice che legge
il contenuto di un file e lo converte in un intero:

```rust
use std::fs::File;
use std::io::{self, Read};
use std::num;
use std::path::Path;

// Deriviamo da `Debug` perché tutti i tipi probabilmente dovrebbero derivare
// da `Debug`. Questo ci dà una ragionevole descrizione umanamente leggibile
// dei valori `CliError`.
#[derive(Debug)]
enum CliError {
    Io(io::Error),
    Parse(num::ParseIntError),
}

fn raddoppia_file_verbose<P: AsRef<Path>>(percorso_di_file: P)
        -> Result<i32, CliError> {
    let mut file = try!(File::open(percorso_di_file).map_err(CliError::Io));
    let mut contenuto = String::new();
    try!(file.read_to_string(&mut contenuto).map_err(CliError::Io));
    let n: i32 = try!(contenuto.trim().parse().map_err(CliError::Parse));
    Ok(2 * n)
}
```

Si noti che abbiamo ancora le chiamate a `map_err`. Perché? Beh, ripensiamo
alle definizioni di [`try!`](#code-try-def) e di [`From`](#code-from-def).
Il problema è che non ci spno implementazioni di `From` che consentono
di convertire dai tipi di errore come `io::Error` e `num::ParseIntError`
al nostro errore personalizzato `CliError`. Naturalmente, è facile correggerlo!
Dato che abbiamo definito `CliError`, possiamo implementare `From` per esso:

```rust
# #[derive(Debug)]
# enum CliError { Io(io::Error), Parse(num::ParseIntError) }
use std::io;
use std::num;

impl From<io::Error> for CliError {
    fn from(err: io::Error) -> CliError {
        CliError::Io(err)
    }
}

impl From<num::ParseIntError> for CliError {
    fn from(err: num::ParseIntError) -> CliError {
        CliError::Parse(err)
    }
}
```

Quello che fanno tutte queste implementazioni è insegnare a `From` come creare
un `CliError` da altri tipi di errore. Nel nostro caso, la costruzione è
semplice quanto invocare il corrispondente costruttore di valore. Infatti,
è *tipicamente* così facile.

Finalmente possiamo riscrivere `raddoppia_file`:

```rust
# use std::io;
# use std::num;
# enum CliError { Io(::std::io::Error), Parse(::std::num::ParseIntError) }
# impl From<io::Error> for CliError {
#     fn from(err: io::Error) -> CliError { CliError::Io(err) }
# }
# impl From<num::ParseIntError> for CliError {
#     fn from(err: num::ParseIntError) -> CliError { CliError::Parse(err) }
# }

use std::fs::File;
use std::io::Read;
use std::path::Path;

fn raddoppia_file<P: AsRef<Path>>(percorso_di_file: P)
        -> Result<i32, CliError> {
    let mut file = try!(File::open(percorso_di_file));
    let mut contenuto = String::new();
    try!(file.read_to_string(&mut contenuto));
    let n: i32 = try!(contenuto.trim().parse());
    Ok(2 * n)
}
```

L'unica cosa che abbiamo fatto qui è stato togliere le chiamate a `map_err`.
Non sono più necessarie perché la macro `try!` invoca `From::from` sul valore
di errore. Ciò funziona perché abbiamo fornito le implementazioni di `From`
per tutti i tipi di errore che potrebbero apparire.

Se modificassimo la nostra funzione `raddoppia_file` per eseguire qualche altra
operazione, diciamo, convertire una stringa in un numero a virgola mobile,
allora dovremmo aggiungere una nuova variante al nostro tipo di errore:

```rust
use std::io;
use std::num;

enum CliError {
    Io(io::Error),
    ParseInt(num::ParseIntError),
    ParseFloat(num::ParseFloatError),
}
```

E aggiungere una nuova implementazione di `From`:

```rust
# enum CliError {
#     Io(::std::io::Error),
#     ParseInt(num::ParseIntError),
#     ParseFloat(num::ParseFloatError),
# }

use std::num;

impl From<num::ParseFloatError> for CliError {
    fn from(err: num::ParseFloatError) -> CliError {
        CliError::ParseFloat(err)
    }
}
```

E questo è tutto!

## Consigli per autori di librerie

Se la propria libreria deve riportare errori personalizzati, probabilmente
si dovrebbe definire il proprio tipo di errore. Sta all'autore decidere se
esporre la sua rappresentazione (come [`ErrorKind`]
(../std/io/enum.ErrorKind.html)) o tenerla nascosta (come [`ParseIntError`]
(../std/num/struct.ParseIntError.html)). Indipendentemente da come lo si fa,
solitamente è una buona pratica fornire almeno qualche informazione sull'errore
oltre alla sua rappresentazione come `String`. Ma certamente, ciò varierà
a seconda dei casi d'uso.

Come minimo, probabilmente si dovrebbe implementare il tratto [`Error`]
(../std/error/trait.Error.html). Ciò darà agli utenti della propria libreria
un minimo di flessibilità per [comporre gli errori](#the-real-try-macro).
Implementare il tratto `Error` comporta anche che agli utenti è garantita
l'abilità di ottenere una rappresentazione in stringa di un errore (perché
obbliga a implementare sia `fmt::Debug` che `fmt::Display`).

Oltre a ciò, può anche essere utile fornire implementazioni di `From` sui
propri tipi di errore. Ciò consente all'autore della libreria e ai suoi utenti
di [comporre errori più dettagliati](#composing-custom-error-types).
Per esempio, [`csv::Error`](http://burntsushi.net/rustdoc/csv/enum.Error.html)
fornisce implementazioni di `From` sia per `io::Error` che per
`byteorder::Error`.

Infine, a seconda dei gusti, si può anche voler definire un [alias del tipo
`Result`](#the-result-type-alias-idiom), particolarmente se la propria
libreria definisce un singolo tipo di errore. Questo è usato nella libreria
standard per [`io::Result`](../std/io/type.Result.html) e per [`fmt::Result`]
(../std/fmt/type.Result.html).

# Studio di un caso: Un programma per leggere dati sulla popolazione

Questa sezione è stata lunga, e a seconda della propria formazione, potrebbe
essere stato piuttosto impegnativo. Mentre c'è molto codice d'esempio alternato
al testo, la maggior parte di tale codice era stato progettato specificamente
per essere pedagogico. Quindi, adesso faremo qualcosa di nuovo: lo studio
di un caso.

Per questo, costruiremo un programma a riga di comando che consente
di interrogare i dati sulla popolazione mondiale. L'obiettivo è semplice:
gli si dà un luogo e dirà la popolazione. Nonostante la sua semplicità,
ci sono molte cose che possono andare storte!

I dati che useremo provengono dal [Data Science Toolkit][11]. Abbiamo preparato
dei dati presi da esso per questo esercizio. Si può o scaricare
i [dati sulla popolazione mondiale][12] (41MB compressi con gzip,
145MB non compressi) o solamente i [dati sulla popolazione degli USA][13]
(2.2MB compressi con gzip, 7.2MB non compressi).

Finora, abbiamo mantenuto il codice limitato alla libreria standard di Rust.
Però, per un compito reale come questo, vorremo almeno usare qualcosa
per analizzare i dati CSV, analizzare gli argomenti del programma,
e decodificare automaticamente quella roba in tipi Rust. A tali scopi, useremo
i crate [`csv`](https://crates.io/crates/csv), e
[`rustc-serialize`](https://crates.io/crates/rustc-serialize).

## Impostazione iniziale

Non spenderemo molto tempo nell'impostare un progetto con Cargo, perché
questo argomento è già trattato bene nella [sezione su Cargo]
(getting-started.html#hello-cargo) e nella [documentazione di Cargo][14].

Per iniziare da zero, eseguiamo `cargo new --bin city-pop` e assicuriamoci
che il nostro `Cargo.toml` sia simile a questo:

```text
[package]
name = "city-pop"
version = "0.1.0"
authors = ["Andrew Gallant <jamslam@gmail.com>"]

[[bin]]
name = "city-pop"

[dependencies]
csv = "0.*"
rustc-serialize = "0.*"
getopts = "0.*"
```

Si dovrebbe già essere in grado di eseguire:

```text
cargo build --release
./target/release/city-pop
# Outputs: Hello, world!
```

## Analisi degli argomenti

Vediamo toglierci di mezzo l'analisi degli argomenti. Non andremo in troppi
dettagli sul crate "Getopts", ma c'è [della buona documentazione][15]
che lo descrive. La storia breve è che Getopts genera un analizzatore
di argomenti e un messaggio di aiuto da un vettore di opzioni (Il fatto che
sia un vettore è nascosto dietro una struct e un insieme di metodi).
Una volta che l'analisi è fatta, l'analizzatore parser restituisce una struct
che registra le corrispondenze per le opzioni definite, e i rimanenti argomenti
"liberi". Da lì, possiamo ottenre informazioni sulle opzioni, per esempio,
se sono state passate, e quali argomenti avevano. Ecco il nostro programma
con le appropriate istruzioni `extern crate`, e l'impostazione di base
degli argomenti per Getopts:

```rust,ignore
extern crate getopts;
extern crate rustc_serialize;

use getopts::Options;
use std::env;

fn stampa_utilizzo(programma: &str, opzioni: Options) {
    println!("{}", opzioni.usage(&format!(
        "Utilizzo: {} [opzioni] <percorso-dati> <comune>", programma)));
}

fn main() {
    let argomenti: Vec<String> = env::args().collect();
    let programma = &args[0];

    let mut opzioni = Options::new();
    opzioni.optflag("?", "aiuto", "Mostra questo messaggio di utilizzo.");

    let corrisp = match opzioni.parse(&args[1..]) {
        Ok(m)  => { m }
        Err(e) => { panic!(e.to_string()) }
    };
    if corrisp.opt_present("h") {
        stampa_utilizzo(&programma, opzioni);
        return;
    }
    let percorso_dati = &corrisp.free[0];
    let comune: &str = &corrisp.free[1];

    // Fa' qualcosa con le informazioni
}
```

Prima, otteniamo un vettore degli argomenti passati nel nostro programma.
Poi immagazziniamo il primo, sapendo che è il nome del nostro programma.
Una volta che è fatto, impostiamo le opzioni dei nostri argomenti; in questo
caso, l'opzione di un semplicistico messaggio d'aiuto. Una volta che abbiamo
impostato le opzioni degli argomenti, usiamo `Options.parse` per analizzare
il vettore degli argomenti (iniziando dall'indice uno, poiché l'indice 0 è
il nome del programma). Se questo ha avuto successo, assegnamo
le corrispondenze agli oggetti analizzati, se no, andiamo in panico. Una volta
passato quello, verifichiamo che l'utente ha passato l'opzione di aiuto, e
se è così stampiamo il messaggio di aiuto. I messaggi di aiuto sulle opzioni
sono costruiti da Getopts, quindi quello che ci rimane da fare per stampare
il messaggio di utilizzo è dirgli che cosa vogliamo che stampi come nome
del programma e come modello. Se l'utente non ha specificato l'opzione
di aiuto, assegnamo le variabili appropriate ai loro argomenti corrispondenti.

## Scrivere la logica

Noi tutti scriviamo il codice ognuno a modo suo, ma la gestione degli errori
è solitamente l'ultima cosa a cui vorremmo pensare. Ciò non è positivo per
la progettazione complessiva di un programma, ma può essere utile
per la prototipazione rapida. Siccome Rust ci costringe a essere espliciti
sulla gestione degli errori (facendoci chiamare `unwrap`), è facile vedere
quali parti del nostro programma possono provocare errori.

In questo studio di un caso, la logica è davvero semplice. Dobbiamo solamente
analizzare i dati CSV che ci vengono passati e stampare un campo nelle righe
che corrispondono. Facciamolo. (Assicuriamoci di aggiungere `extern crate csv;`
in cima al nostro file.)

```rust,ignore
use std::fs::File;

// Questa struct rappresenta i dati in ogni riga del file CSV.
// La decodifica basata sui tipi ci evita un sacco della gestione degli errori
// di basso livello, come convertire le stringhe in numeri.
#[derive(Debug, RustcDecodable)]
struct Riga {
    nazione: String,
    comune: String,
    accent_comune: String,
    regione: String,

    // Non tutte le righe hanno dati sulla popolazione, sulla latitudine
    // o sulla longitudine! Perciò li esprimiamo come tipi `Option`,
    // che ammettono la possibilità di assenza. L'analizzatore CSV assegnerà
    // i valori corretti.
    popolazione: Option<u64>,
    latitudine: Option<f64>,
    longitudine: Option<f64>,
}

fn stampa_utilizzo(programma: &str, opzioni: Options) {
    println!("{}", opzioni.usage(&format!(
        "Utilizzo: {} [opzioni] <percorso-dati> <comune>", programma)));
}

fn main() {
    let argomenti: Vec<String> = env::args().collect();
    let programma = &args[0];

    let mut opzioni = Options::new();
    opzioni.optflag("?", "aiuto", "Mostra questo messaggio di utilizzo.");

    let corrisp = match opzioni.parse(&args[1..]) {
        Ok(m)  => { m }
        Err(e) => { panic!(e.to_string()) }
    };

    if corrisp.opt_present("h") {
        stampa_utilizzo(&programma, opzioni);
        return;
    }

    let percorso_dati = &corrisp.free[0];
    let comune: &str = &corrisp.free[1];

    let file = File::open(percorso_dati).unwrap();
    let mut rdr = csv::Reader::from_reader(file);

    for riga in rdr.decode::<Riga>() {
        let riga = riga.unwrap();

        if riga.comune == comune {
            println!("{}, {}: {:?}",
                riga.comune, riga.nazione,
                riga.popolazione.expect("conteggio della popolazione"));
        }
    }
}
```

Delineiamo gli errori. Possiamo partire con quelli evidenti: i tre posti dove
viene chiamata `unwrap`:

1. [`File::open`](../std/fs/struct.File.html#method.open)
   può restituire [`io::Error`](../std/io/struct.Error.html).
1. [`csv::Reader::decode`]
   (http://burntsushi.net/rustdoc/csv/struct.Reader.html#method.decode)
   decodifica un record per volta, e [decodificare un record]
   (http://burntsushi.net/rustdoc/csv/struct.DecodedRecords.html)
   (si guardi il tipo associato `Item` nell'implementazione di `Iterator`)
   può produrre un [`csv::Error`]
   (http://burntsushi.net/rustdoc/csv/enum.Error.html).
1. Se `riga.popolazione` è `None`, allora chiamare `expect` andrà in panico.

Ce ne sono altri? Che succede se non si trova nessun comune corrispondente?
Strumenti come `grep` restituiscono un codice d'errore, e quindi probabilmente
dovremmo anche noi. Quindi abbiamo degli errori logici specifici per il nostro
problema, degli errori di I/O e degli errori di decodifica del formato CSV.
Esploreremo due diversi modi di approcciare la gestione di questi errori.

Iniziamo con `Box<Error>`. Più avanti, vedremo come può essere utile definire
il nostro tipo di errore.

## Usare `Box<Error>` per la gestione degli errori

`Box<Error>` è carino perché *in qualche modo, funziona*. Non è necessario
definire i propri tipi di errore e né implementazioni di `From`. Lo svantaggio
è che, siccome `Box<Error>` è un oggetto tratto, *cancella il tipo*,
che significa che il compilatore non può più ragionare sul suo
tipo soggiaciente.

[Prima](#the-limits-of-combinators) abbiamo iniziato a rifattorizzare
il nostro codice cambiando il tipo della nostra funzione da `T`
a `Result<T, OurErrorType>`. In questo caso, `OurErrorType` è solamente
`Box<Error>`. Ma cos'è `T`? E possiamo aggiungere un tipo del valore
restituito a `main`?

La risposta alla seconda domanda è: no, non possiamo. Ciò comporta che dovremo
scrivere una nuova funzione. Ma cos'è `T`? La cosa più semplice che possiamo
fare è restituire un elenco di valori `Riga` corrispondenti, in un `Vec<Riga>`.
(Sarebbe meglio restituire un iteratore, ma questo lo si lascia
come esercizio.)

Rifattorizziamo il nostro codice nella sua funzione, ma manteniamo le chiamate
a `unwrap`. Si noti che decidiamo di gestire la possibilità di un conteggio
di popolazione mancante semplicemente ignorando quella riga.

```rust,ignore
use std::path::Path;

struct Riga {
    // immutata
}

struct ConteggioPopolazione {
    comune: String,
    nazione: String,
    // Questa non è più una `Option` perché i valori di questo tipo
    // vengono costruiti solamente se hanno un conteggio della popolazione.
    conteggio: u64,
}

fn stampa_utilizzo(programma: &str, opzioni: Options) {
    println!("{}", opzioni.usage(&format!(
        "Utilizzo: {} [opzioni] <percorso-dati> <comune>", programma)));
}

fn cerca<P: AsRef<Path>>(percorso_di_file: P, comune: &str)
        -> Vec<ConteggioPopolazione> {
    let mut trovati = vec![];
    let file = File::open(percorso_di_file).unwrap();
    let mut rdr = csv::Reader::from_reader(file);
    for riga in rdr.decode::<Riga>() {
        let riga = riga.unwrap();
        match riga.popolazione {
            None => { } // saltalo
            Some(conteggio) => if riga.comune == comune {
                trovati.push(ConteggioPopolazione {
                    comune: riga.comune,
                    nazione: riga.nazione,
                    conteggio: conteggio,
                });
            },
        }
    }
    trovati
}

fn main() {
    let args: Vec<String> = env::args().collect();
    let programma = &args[0];

    let mut opzioni = Options::new();
    opzioni.optflag("?", "aiuto", "Mostra questo messaggio di utilizzo.");

    let corrisp = match opzioni.parse(&args[1..]) {
        Ok(m)  => { m }
        Err(e) => { panic!(e.to_string()) }
    };

    if corrisp.opt_present("h") {
        stampa_utilizzo(&programma, opzioni);
        return;
    }

    let percorso_dati = &corrisp.free[0];
    let comune: &str = &corrisp.free[1];

    for pop in cerca(percorso_dati, comune) {
        println!("{}, {}: {:?}", pop.comune, pop.nazione, pop.conteggio);
    }
}

```

Mentre ci siamo sbarazzati di un uso di `expect` (che è una variante più carina
di `unwrap`), dobbiamo ancora gestire l'assenza di risultati della ricerca.

Per convertire questo a un'appropriata gestione degli errori, dobbiamo:

1. Cambiare il tipo del valore restituito di `cerca` in
   `Result<Vec<ConteggioPopolazione>, Box<Error>>`.
1. Usare la [macro `try!`](#code-try-def) così che gli errori siano restituiti
   al chiamante, invece di mandare in panico il programma.
1. Gestire l'errore in `main`.

Proviamoci:

```rust,ignore
use std::error::Error;

// Il resto del codice prima di questo è immutato

fn cerca<P: AsRef<Path>>(percorso_di_file: P, comune: &str)
        -> Result<Vec<ConteggioPopolazione>, Box<Error>> {
    let mut trovati = vec![];
    let file = try!(File::open(percorso_di_file));
    let mut rdr = csv::Reader::from_reader(file);
    for riga in rdr.decode::<Riga>() {
        let riga = try!(riga);
        match riga.popolazione {
            None => { } // saltalo
            Some(conteggio) => if riga.comune == comune {
                trovati.push(ConteggioPopolazione {
                    comune: riga.comune,
                    nazione: riga.nazione,
                    conteggio: conteggio,
                });
            },
        }
    }
    if trovati.is_empty() {
        Err(From::from("Non sono stati trovati comuni corrispondenti \
            aventi una popolazione."))
    } else {
        Ok(trovati)
    }
}
```

Invece di `x.unwrap()`, adesso abbiamo `try!(x)`. Siccome la nostra funzione
restituisce un `Result<T, E>`, la macro `try!` uscirà precocemente
dalla funzione se avviene un errore.

Alla fine di `cerca` convertiamo anche una semplice stringa in un tipo
di errore usando le [corrispondenti implementazioni di `From`]
(../std/convert/trait.From.html):

```rust,ignore
// Stiamo facendo uso di questa implementazione nel codice di sopra, dato che
// chiamiamo `From::from` su una `&'static str`.
impl<'a> From<&'a str> for Box<Error>

// Ma questo è utile anche quando bisogna allocare una nuova stringa
// per un messaggio d'errore, solitamente con `format!`.
impl From<String> for Box<Error>
```

Siccome `cerca` adesso restituisce un `Result<T, E>`, `main` dovrebbe usare
l'analisi dei casi quando chiama `cerca`:

```rust,ignore
...
    match cerca(percorso_dati, comune) {
        Ok(pops) => {
            for pop in pops {
                println!("{}, {}: {:?}", pop.comune, pop.nazione, pop.conteggio);
            }
        }
        Err(err) => println!("{}", err)
    }
...
```

Adesso che abbiamo visto come fare l'appropriata gestione degli errori
con `Box<Error>`, proviamo un approccio diverso, usando il nostro tipo
di errore personalizzato. Ma prima, facciamo una breve pausa dalla gestione
degli errori e aggiungiamo il supporto per leggere da `stdin`.

## Leggere da stdin

Nel nostro programma, accettiamo un singolo file come input e facciamo
una sola passata sui dati. Ciò comporta che probabilmente dovremmo essere
in grado di accettare l'input da stdin. Ma può darsi che ci piaccia anche
l'attuale formato—e allora teniamoceli entrambi!

Aggiungere il supporto per stdin è effettivamente molto facile. Ci sono
solamente tre cose che dobbiamo fare:

1. Ritoccare gli argomenti del programma, in modo che possa essere accettato
   un singolo argomento—il comune—mentre i dati sulla popolazione vengono letti
   da stdin.
1. Modificare il programma in modo che un'opzione `-f` possa prendere il file,
   se non è passato da stdin.
1. Modificare la funzione `cerca` in modo che prenda un percorso di file
   *facoltativo*. Quando è `None`, dovrebbe sapere di leggere da stdin.

Prima, ecco il nuovo utilizzo:

```rust,ignore
fn stampa_utilizzo(programma: &str, opzioni: Options) {
    println!("{}", opzioni.usage(&format!(
        "Utilizzo: {} [options] <comune>", programma)));
}
```

Naturalmente dobbiamo adattare il codice di gestione degli argomenti:

```rust,ignore
...
    let mut opzioni = Options::new();
    opzioni.optopt("f", "file", "Scegli un file di input, invece di usare STDIN.", "NAME");
    opzioni.optflag("?", "aiuto", "Mostra questo messaggio di utilizzo.");
    ...
    let percorso_dati = corrisp.opt_str("f");

    let comune = if !corrisp.free.is_empty() {
        &corrisp.free[0]
    } else {
        stampa_utilizzo(&programma, opzioni);
        return;
    };

    match cerca(&percorso_dati, comune) {
        Ok(pops) => {
            for pop in pops {
                println!("{}, {}: {:?}", pop.comune, pop.nazione, pop.conteggio);
            }
        }
        Err(err) => println!("{}", err)
    }
...
```

Abbiamo reso l'esperienza dell'utente un po' più gradevole mostrando il
messaggio di utilizzo, invece di andare in panico per l'uso di un indice fuori
dai limiti, quando `comune`, il rimanente argomento libero, non è presente.

Modificare `cerca` è leggermente più complicato. Il crate `csv` può costruire
un analizzatore da [qualunque tipo che implementi `io::Read`]
(http://burntsushi.net/rustdoc/csv/struct.Reader.html#method.from_reader).
Ma come possiamo usare il medesimo codice su entrambi i tipi? Effettivamente
c'è un paio di strade che potremmo percorrere per questo. Una strada è scrivere
`cerca` in modo tale che sia generica su qualche parametro di tipo `R`, che
soddisfa `io::Read`. Un altro modo è usare gli oggetti tratto:

```rust,ignore
use std::io;

// Il resto del codice prima di questo è immutato

fn cerca<P: AsRef<Path>>(percorso_di_file: &Option<P>, comune: &str)
        -> Result<Vec<ConteggioPopolazione>, Box<Error>> {
    let mut trovati = vec![];
    let input: Box<io::Read> = match *percorso_di_file {
        None => Box::new(io::stdin()),
        Some(ref percorso_di_file) => Box::new(try!(File::open(percorso_di_file))),
    };
    let mut rdr = csv::Reader::from_reader(input);
    // Il resto rimane immutato!
}
```

## Usare un tipo personalizzato per la gestione degli errori

Prima, abbiamo imparato come [comporre errori usando un tipo di errore
personalizzato](#composing-custom-error-types). L'abbiamo fatto definendo
il nostro tipo di errore come un `enum`, e implementando `Error` e `From`.

Siccome abbiamo tre errori distinti (I/O, analisi del CSV, e non trovati),
definiamo un `enum` con tre varianti:

```rust,ignore
#[derive(Debug)]
enum CliError {
    Io(io::Error),
    Csv(csv::Error),
    NonTrovati,
}
```

E adesso le sue implementazioni di `Display` e di `Error`:

```rust,ignore
use std::fmt;

impl fmt::Display for CliError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        match *self {
            CliError::Io(ref err) => err.fmt(f),
            CliError::Csv(ref err) => err.fmt(f),
            CliError::NonTrovati => write!(f, "Non sono stati trovati \
                comuni corrispondenti aventi una popolazione."),
        }
    }
}

impl Error for CliError {
    fn description(&self) -> &str {
        match *self {
            CliError::Io(ref err) => err.description(),
            CliError::Csv(ref err) => err.description(),
            CliError::NonTrovati => "non trovati",
        }
    }

    fn cause(&self) -> Option<&Error> {
        match *self {
            CliError::Io(ref err) => Some(err),
            CliError::Csv(ref err) => Some(err),
            // Il nostro errore personalizzato non ha una causa soggiacente,
            // ma potremmo modificarlo in modo che ce l'abbia.
            CliError::NonTrovati => None,
        }
    }
}
```

Prima di poter usare il tipo `CliError` nella nostra funzione `cerca`, dobbiamo
fornire un paio di implementazioni di `From`. Come sappiamo quali
implementazioni fornire? Beh, dovremo convertire sia `io::Error`
che `csv::Error` in `CliError`. Quelli sono i soli errori esterni, e quindi
per adesso ci serviranno solamente due implementazioni di `From`:

```rust,ignore
impl From<io::Error> for CliError {
    fn from(err: io::Error) -> CliError {
        CliError::Io(err)
    }
}

impl From<csv::Error> for CliError {
    fn from(err: csv::Error) -> CliError {
        CliError::Csv(err)
    }
}
```

Le implementazioni di `From` sono importanti a causa di come [è definita
`try!`](#code-try-def). In particolare, se avviene un errore, `From::from`
viene chiamata sull'errore, che in questo caso, lo convertirà al nostro tipo
di errore `CliError`.

Avendo fatto le implementazioni di `From`, dobbiamo solamente fare due ritocchi
alla nostra funzione `cerca`: il tipo del valore restituito e l'errore
“non trovati”. Eccola integralmente:

```rust,ignore
fn cerca<P: AsRef<Path>>
         (percorso_di_file: &Option<P>, comune: &str)
         -> Result<Vec<ConteggioPopolazione>, CliError> {
    let mut trovati = vec![];
    let input: Box<io::Read> = match *percorso_di_file {
        None => Box::new(io::stdin()),
        Some(ref percorso_di_file) => Box::new(try!(File::open(percorso_di_file))),
    };
    let mut rdr = csv::Reader::from_reader(input);
    for riga in rdr.decode::<Riga>() {
        let riga = try!(riga);
        match riga.popolazione {
            None => { } // saltalo
            Some(conteggio) => if riga.comune == comune {
                trovati.push(ConteggioPopolazione {
                    comune: riga.comune,
                    nazione: riga.nazione,
                    conteggio: conteggio,
                });
            },
        }
    }
    if trovati.is_empty() {
        Err(CliError::NonTrovati)
    } else {
        Ok(trovati)
    }
}
```

Non sono necessarie altre modifiche.

## Aggiungere funzionalità

Scrivere del codice generico è  code is grandioso, perché generalizzare le cose
è bello, e in seguito può servire. Ma talvolta, il gioco non vale la candela.
Guardiamo ciò che abbiamo appena fatto al passo precedente:

1. Definito un nuovo tipo di errore.
1. Agiunte le implementazioni di `Error`, e di `Display`, e due di `From`.

Qui il grosso svantaggio è che il nostro programma non è migliorato moltissimo.
C'è un bel po' di spreco nel rappresentare gli errori con delle `enum`,
specialmente in programmi brevi come questo.

*Un* aspetto utile dell'usare un tipo di errore personalizzato come abbiamo
fatto qui è che la funzione `main` adesso può scegliere di gestire gli errori
diversamente. Prima, con `Box<Error>`, non aveva molte scelte: poteva solamente
stampare il messaggio. Qui stiamo facendo ancora quello, ma se volessimo,
diciamo, aggiungere un'opzione `--zitto`? L'opzione `--zitto` silenzierebbe
ogni output prolisso.

Il programma attuale, se non trova corrispondenze, emetterà un messaggio
d'errore. Questo può essere un po' maldestro, specialmente se si voleva usare
il programma in script di shell.

Allora iniziamo aggiungendo le opzioni. Come prima, dobbiamo ritoccare
la stringa id utilizzo e aggiungere un'opzione alla variabile Option. Una volta
che l'abbiamo fatto, Getopts fa il resto:

```rust,ignore
...
    let mut opzioni = Options::new();
    opzioni.optopt("f", "file",
        "Scegli un file di input, invece di usare STDIN.", "NAME");
    opzioni.optflag("?", "aiuto", "Mostra questo messaggio di utilizzo.");
    opzioni.optflag("z", "zitto", "Silenzia gli errori e gli avvertimenti.");
...
```

Adesso dobbiamo solamente implementare la nostra funzionalità “zitto”. Ciò
ci obblica a ritoccare l'analisi dei casi in `main`:

```rust,ignore
use std::process;
...
    match cerca(&percorso_dati, comune) {
        Err(CliError::NonTrovati) if corrisp.opt_present("q")
            => process::exit(1),
        Err(err) => panic!("{}", err),
        Ok(pops) => for pop in pops {
            println!("{}, {}: {:?}", pop.comune, pop.nazione, pop.conteggio);
        }
    }
...
```

Certamente, non vogliamo stare zitti se succede un errore di I/O, o se i dati
non erano decodificabili. Perciò, usiamo l'analisi dei casi per verificare
se il tipo di errore è `NonTrovati` *e* se `--zitto` è stato abilitato.
Se la ricerca è fallita, continuiamo a uscire con un codice di uscita
(seguendo la convenzione di `grep`).

Se ci fossimo bloccati con `Box<Error>`, allora sarebbe parecchio complicato
implementare la funzionalità `--zitto`.

Questo riassume abbastanza il nostro studio di un caso. Da qui, si dovrebbe
essere pronti ad andare nel mondo e scrivere i propri programmi e librerie
con un'appropriata gestione degli errori.

# La storia breve

Siccome questa sezione è lunga, è utile avere un breve riassunto
sulla gestione degli errori in Rust. Queste sono alcune buone
regole empiriche, ma assolutamente *non* sono dei comandamenti.
Probabilmente ci sono buone ragioni per violare ognuna di queste regole!

* Se si sta scrivendo un breve codice di esempio, che sarebbe appesantito
  dalla gestione degli errori, probabilmente va bene usare `unwrap` (che sia
  [`Result::unwrap`](../std/result/enum.Result.html#method.unwrap),
  [`Option::unwrap`](../std/option/enum.Option.html#method.unwrap)
  o preferibilmente
  [`Option::expect`](../std/option/enum.Option.html#method.expect)).
  I consumatori del proprio codice dovrebbero sapere usare l'appropriata
  gestione degli errori. (Altrimenti, mandali qui!)
* Se si sta scrivendo un programma abborracciato, non ci si deve vergognare
  a usare `unwrap`. Attenzione: se finisce nelle mani di qualcun altro, non
  ci si sorprenda se poi si agita per oscuri messaggi d'errore!
* Se si sta scrivendo un programma abborracciato, ma ci si vergogna di andare
  in panico, allora si usi o una `String` o un `Box<Error>` come tipo
  per i propri errori.
* Altrimenti, in un programma, si definisca i propri tipi di errore con
  le appropriate implementazioni di [`From`](../std/convert/trait.From.html)
  e di [`Error`](../std/error/trait.Error.html) per rendere più ergonomica
  la macro [`try!`](../std/macro.try!.html).
* Se si sta scrivendo una libreria, e il proprio codice può produrre errori,
  si definisca il proprio tipo di errori e si implementi il tratto
  [`std::error::Error`](../std/error/trait.Error.html). Dove è appropriato,
  si implementi [`From`](../std/convert/trait.From.html) per rendere più facile
  da scrivere sia il codice della propria libreria che il codice del chiamante.
  (A causa delle regole di coerenza di Rust, i chiamanti non potranno
  implementare `From` sul tipo di errori della libreria, e quindi la libreria
  dovrebbe farlo.
* Si imparino i combinatori definiti per [`Option`]
  (../std/option/enum.Option.html) e per [`Result`]
  (../std/result/enum.Result.html). Usarli esclusivamente può essere un po'
  stancante a volte, ma un buon miscuglio di `try!` e di combinatori è
  un parecchio attraente. I più tipici sono `and_then`, `map` e `unwrap_or`.

[1]: ../book/patterns.html
[2]: ../std/option/enum.Option.html#method.map
[3]: ../std/option/enum.Option.html#method.unwrap_or
[4]: ../std/option/enum.Option.html#method.unwrap_or_else
[5]: ../std/option/enum.Option.html
[6]: ../std/result/index.html
[7]: ../std/result/enum.Result.html#method.unwrap
[8]: ../std/fmt/trait.Debug.html
[9]: ../std/primitive.str.html#method.parse
[10]: ../book/associated-types.html
[11]: https://github.com/petewarden/dstkdata
[12]: http://burntsushi.net/stuff/worldcitiespop.csv.gz
[13]: http://burntsushi.net/stuff/uscitiespop.csv.gz
[14]: http://doc.crates.io/guide.html
[15]: http://doc.rust-lang.org/getopts/getopts/index.html
