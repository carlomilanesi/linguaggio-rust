% Le macro

Abbiamo già visto molti degli strumenti forniti da Rust per creare astrazioni e
per riutilizzare il codice. Queste unità di riuso del codice hanno una ricca
struttura semantica. Per esempio, le funzioni hanno una firma di tipo,
i parametri di tipo hanno dei legami di tratti, e le funzioni sovraccaricate
devono appartenere a un particolare tratto.

Questa struttura significa che le astrazioni del nucleo di Rust hanno
potenti verifiche di correttezza in fase di compilazione. Ma questo si paga
con una ridotta flessibilit. Se si identifica visivamente un pattern di codice
ripetuto, si può trovare difficile o contorto esprimere quel pattern come
funzione generica, tratto, qualcos'altro entro la semantica di Rust.

Le macro ci consentono di astrarre a livello sintattico. Un'invocazione
di macro è un'abbreviazione di una forma sintattica "espansa". Questa
espansione avviene nelle fasi iniziali della compilazione, prima di ogni
verifica statica. Di conseguenza, le macro possono catturare molti pattern
di riutilizzo del codice che le astrazioni del nucleo di Rust non possono.

Lo svantaggio è che il codice basato su macro può essere più difficile
da capire harder, perché poche delle regole incorporate si applicano. Come
le normali funzione, le macro che si comportano bene possono essere usate senza
dover capire come sono implementate. Però, può essere difficile progettare
una macro che si comporta bene! Inoltre, gli errori di compilazione nel codice
che usa macro sono più difficili da capire, perché descrivono errori nel codice
espanso, non nella forma sorgente scritta dagli sviluppatori.

Questi svantaggi rendono le macro qualcosa come "caratteristiche da ultima
spiaggia". Ciò non vuol dire che le macro vanno evitate; fanno parte di Rust
perché talvolta servono per scrivere del codice molto conciso e astratto.
Si deve solo tenere a mente di questi pro e contro.

# Definire una macro

Si avrà già visto la macro `vec!`, usata per inizializzare un [vettore]
[vettore] con una lista di elementi specificati.

[vettore]: vectors.html

```rust
let x: Vec<u32> = vec![1, 2, 3];
# assert_eq!(x, [1, 2, 3]);
```

Questa non può essere una normale funzione, perché prende un numero variabile
di argomenti. Ma la possiamo immaginare come abbreviazione sintattica di:

```rust
let x: Vec<u32> = {
    let mut temp_vec = Vec::new();
    temp_vec.push(1);
    temp_vec.push(2);
    temp_vec.push(3);
    temp_vec
};
# assert_eq!(x, [1, 2, 3]);
```

Possiamo implementare questa abbreviazione, usando una macro: [^actual]

[^actual]: L'effettiva definizione di `vec!` contenuta nella libreria
           collections differisce da quella presentata qui, per ragioni
           sia di efficienza che di riusabilità.

```rust
macro_rules! vec {
    ( $( $x:expr ),* ) => {
        {
            let mut temp_vec = Vec::new();
            $(
                temp_vec.push($x);
            )*
            temp_vec
        }
    };
}
# fn main() {
#     assert_eq!(vec![1,2,3], [1, 2, 3]);
# }
```

Urca, c'è un sacco di sintassi nuova! Analizziamola.

```rust,ignore
macro_rules! vec { ... }
```

Questo dice che stiamo definendo una macro chiamata `vec`, come `fn vec`
definirebbe una funzione chiamata `vec`. Quando si invoca una macro, il nome
della macro è seguito da un punto esclamativo, per es. `vec!`. Il punto
esclamativo fa parte della sintassi di invocazione e serve a distinguere
una macro da una normale funzione.

## Pattern-matching

La macro viene definita tramite una serie di regole, che sono casi
di pattern-matching. Sopra, avevamo

```rust,ignore
( $( $x:expr ),* ) => { ... };
```

Questo è simile a un braccio di un'espressione `match`, ma la corrispondenza
avviene con gli alberi sintattici di Rust, in fase di compilazione.
Il punto-e-virgola dopo l'ultimo caso (che qui è anche l'unico) è facoltativo.
Il "pattern" sul lato sinistro di `=>` è noto come ‘matcher’. Queste
espressioni hanno [la loro piccola grammatica] all'interno del linguaggio.

[la loro piccola grammatica]: ../reference.html#macros

Il matcher `$x:expr` combacierà con qualunque espressione Rust, legando
quell'albero sintattico alla ‘metavariabile’ `$x`. L'identificatore `expr` è
uno ‘specificatore di frammento’; le possibilità complete verranno elencate
più avanti in questa sezione.
Circondando il matcher con `$(...),*` significa che potranno combaciare
nessuna o alcune espressioni, separate da virgole.

A parte la speciale sintassi del matcher, ogni token Rust che appare in
un matcher deve combaciare esattamente. Per esempio,

```rust,ignore
macro_rules! foo {
    (x => $e:expr) => (println!("modalità X: {}", $e));
    (y => $e:expr) => (println!("modalità Y: {}", $e));
}

fn main() {
    foo!(y => 3);
}
```

stamperà

```text
modalità Y: 3
```

Con

```rust,ignore
foo!(z => 3);
```

otterremo un errore di compilazione

```text
error: no rules expected the token `z`
```

## Espansione

Il lato destro di una regola di macro è normale sintassi Rust, per lo più.
Ma possiamo innestare pezzetti della sintassi catturata dal matcher.
Dall'esempio originale:

```rust,ignore
$(
    temp_vec.push($x);
)*
```

Ogni espressione combaciante `$x` produrrà una singola istruzione `push`
nell'espansione della macro. La ripetizione dell'espansione procede di pari
passo con la ripetizione del matcher (fra poco ne diremo di più).

Siccome `$x` era già stata dichiarata come combaciante un'espressione, non si
ripete `:expr` sul lato destro. Inoltre, non si aggiunge una virgola
di separazione come parte dell'operatore di ripetizione. Invece, si mette
il punto-e-virgola di terminazione all'interno del blocco ripetuto.

Un altro dettaglio: la macro `vec!` ha *due* coppie di graffe sul lato destro.
Vengono spesso combinate così:

```rust,ignore
macro_rules! foo {
    () => {{
        ...
    }}
}
```

Le graffe esterne fanno parte della sintassi di `macro_rules!`. Di fatto,
si può usare `()` o `[]` invece. Delimitano semplicemente il lato destro
nel suo insieme.

Le graffe interne fanno parte della sintassi espansa. Si ricordi che la macro
`vec!` va usata in un contesto di espressione. Per scrivere un'espressione
contenente più istruzioni, tra cui delle istruzioni `let`, si deve usare
un blocco. Se invece si scrive una macro che si espande a una singola
espressione, non c'è bisogno di queste graffe ulteriori.

Si noti che non abbiamo mai *dichiarato* che la macro produce un'espressione.
Infatti, questo non è determinato fino a quando usiamo la macro
come espressione. Con attenzione, si può scrivere una macro la cui espansione
funziona in più contesti. Per esempio, un'abbreviazione di un tipo di dati
potrebbe essere valida sia come espressione che come pattern.

## Ripetizione

L'operatore di ripetizione segue due regole principali:

1. `$(...)*` percorre uno "strato" di ripetizioni, per ogni `$nome` che
   contiene, di pari passo, e
2. ogni `$nome` deve essere sotto almeno tanti `$(...)*` quanti sono quelli
   con cui ha combaciato. Se è sotto a un numero maggiore, verrà
   appropriatamente duplicato.

La seguente macro barocca illustra la duplicazione di variabili
da livelli esterni di ripetizione.

```rust
macro_rules! o_O {
    (
        $(
            $x:expr; [ $( $y:expr ),* ]
        );*
    ) => {
        &[ $($( $x + $y ),*),* ]
    }
}

fn main() {
    let a: &[i32]
        = o_O!(10; [1, 2, 3];
               20; [4, 5, 6]);

    assert_eq!(a, [11, 12, 13, 24, 25, 26]);
}
```

Questa è la sintassi più comune del matcher. Questi esempi usano `$(...)*`,
che è un combaciamento di tipo "nessuno o alcuni". Alternativamente si può
scrivere `$(...)+` per un combaciamento di tipo "uno o alcuni". Entrambe
le forme comprendono un separatore facoltativo, che può essere qualunque token
eccetto `+` o `*`.

Questo sistema si pasa su
"[Macro-by-Example](https://www.cs.indiana.edu/ftp/techreports/TR206.pdf)"
(collegamento a file PDF).

# Igiene

Alcuni linguaggi implementano macro usando semplici sostituzioni di testo,
il che comporta vari svantaggi. Per esempio, questo programma C stampa `13`
invece dell'atteso `25`.

```text
#define PER_CINQUE(x) 5 * x

int main() {
    printf("%d\n", PER_CINQUE(2 + 3));
    return 0;
}
```

Dopo l'espansione abbiamo `5 * 2 + 3`, e la moltiplicazione ha la precedenza
sull'addizione. Chi avesse usato molto le macro del C, probabilmente conosce
le tecniche standard per evitare questo difetto, e così pure cinque o sei
altri. In Rust, non dobbiamo preoccuparcene.

```rust
macro_rules! per_cinque {
    ($x:expr) => (5 * $x);
}

fn main() {
    assert_eq!(25, per_cinque!(2 + 3));
}
```

La metavariabile `$x` viene analizzata dal parser come singola espressione, e
mantiene il suo posto nell'albero sintattico anche dopo la sostituzione.

Un altro difetto tipico dei sistemi di macro è la ‘cattura di variabile’. Ecco
una macro C, che usa [una estensione del GNU C] per emulare i blocchi
di espressione di Rust.

[una estensione di GNU C]: https://gcc.gnu.org/onlinedocs/gcc/Statement-Exprs.html

```text
#define LOG(msg) ({ \
    int state = get_log_state(); \
    if (state > 0) { \
        printf("log(%d): %s\n", state, msg); \
    } \
})
```

Ecco un semplice caso d'uso che che funziona malissimo:

```text
const char *state = "reticulating splines";
LOG(state)
```

Questo si espande in

```text
const char *state = "reticulating splines";
{
    int state = get_log_state();
    if (state > 0) {
        printf("log(%d): %s\n", state, state);
    }
}
```

La seconda variabile chiamata `state` oscura la prima. Questo è un difetto
perché l'istruzione printf dovrebbe riferirsi a entrambe.

L'equivalente macro di Rust ha il comportamento desiderato.

```rust
# fn get_log_state() -> i32 { 3 }
macro_rules! log {
    ($msg:expr) => {{
        let state: i32 = get_log_state();
        if state > 0 {
            println!("log({}): {}", state, $msg);
        }
    }};
}

fn main() {
    let state: &str = "reticulating splines";
    log!(state);
}
```

Questa funziona perché Rust ha un [sistema igienico di macro]. Ogni espansione
di macro avviene in un ‘contesto sintattico’ distinto, e ogni variabile viene
etichettata con il contesto sintattico in cui è stata introdotta. È come se la
variabile `state` dentro `main` fosse dipinta di un diverso "colore" rispetto
alla variabile `state` interna alla macro, e perciò non entrano in conflitto.

[sistema igienico di macro]: https://en.wikipedia.org/wiki/Hygienic_macro

Ciò restringe anche la capacità delle macros di introdurre nuovi legami
al punto di invocazione. Del codice come l seguente non funzionerà:

```rust,ignore
macro_rules! foo {
    () => (let x = 3;);
}

fn main() {
    foo!();
    println!("{}", x);
}
```

Invece, bisogna passare il nome della variabile all'invocazione, così che
sia etichettata con il giusto contesto sintattico.

```rust
macro_rules! foo {
    ($v:ident) => (let $v = 3;);
}

fn main() {
    foo!(x);
    println!("{}", x);
}
```

Questo vale per i legami `let` e le etichette dei cicli, ma non per
gli [elementi][elementi]. Quindi il seguente codice compila:

```rust
macro_rules! foo {
    () => (fn x() { });
}

fn main() {
    foo!();
    x();
}
```

[elementi]: ../reference.html#items

# Macro ricorsive

Un'espansione di macro può comprendere altre invocazioni di macro, comprese
invocazioni proprio della stessa macro che si sta definendo. Queste macro
ricorsive sono utili per elaborare dell'input strutturato ad albero, come
esemplificato da questa (semplicistica) abbreviazione dell'HTML:

```rust
# #![allow(unused_must_use)]
macro_rules! write_html {
    ($w:expr, ) => (());

    ($w:expr, $e:tt) => (write!($w, "{}", $e));

    ($w:expr, $tag:ident [ $($inner:tt)* ] $($rest:tt)*) => {{
        write!($w, "<{}>", stringify!($tag));
        write_html!($w, $($inner)*);
        write!($w, "</{}>", stringify!($tag));
        write_html!($w, $($rest)*);
    }};
}

fn main() {
#   // FIXME(#21826)
    use std::fmt::Write;
    let mut out = String::new();

    write_html!(&mut out,
        html[
            head[title["Guida alle macro"]]
            body[h1["Le macro sono il meglio!"]]
        ]);

    assert_eq!(out,
        "<html><head><title>Guida alle macro</title></head>\
         <body><h1>Le macro sono il meglio!</h1></body></html>");
}
```

# Debug del codice delle macro

Per vedere il risultato dell'espansione delle macro, si esegua
`rustc --pretty expanded`. L'output rappresenta un interno crate, perciò lo
si può anche inoltrare a `rustc`, che talvolta produrrà dei messaggi d'errore
migliori di quelli della compilazione originale. Si noti che l'output
di `--pretty expanded` può avere un significato diverso se più variabili
con lo stesso nome (ma diversi contesti sintattici) sono in gioco nello stesso
ambito. In questo caso `--pretty expanded,hygiene` esporrà i contesti
sintattici.

`rustc` fornisce due estensioni sintattiche che aiutano il debug delle macro.
Per adesso, sono instabili e richiedono dei feature gate.

* `log_syntax!(...)` stamperà i suoi argomenti sullo standard output, in fase
  di compilazione, e si espanderà a niente.

* `trace_macros!(true)` abiliterà l'emissione di un messaggio di compilazione
  ogni volta che una macro viene espansa. Si usi `trace_macros!(false)`
  nel corso dell'espansione per disattivarla.

# Requisiti sintattici

Anche quando il codice Rust contiene macro non espanse, può essere analizzato
dal parser producendo un completo [albero sintattico][ast]. Questa proprietà
può essere molto utile per gli editor e per altri strumenti che elaborano
il codice. Ha anche alcune conseguenze per la progettazione del sistema
di macro di Rust.

[ast]: glossary.html#abstract-syntax-tree

Una conseguenza è che Rust deve determinare, quando analizza un'invocazione
di macro, se la macro si espanderà in

* nessuno o alcuni elementi,
* nessuno o alcuni metodi,
* un'espressione,
* un'istruzione, oppure
* un pattern.

Un'invocazione di macro entro un blocco potrebbe espandersi in alcuni elementi,
oppure in un'espressione / istruzione. Rust usa una semplice regola
per risolvere questa ambiguità. Un'invocazione di macro che si espande
in elementi deve essere o

* delimitata da graffe, per es. `foo! { ... }`, oppure
* terminata da un punto-e-virgola, per es. `foo!(...);`

Un'altra conseguenza dell'analisi pre-espansione è che l'invocazione
della macro deve consistere in token Rust validi. Inoltre, le parentesi tonde,
quadre, e graffe devono essere bilanciate internamente a un'invocazione
di macro. Per esempio, `foo!([)` è proibito. Questo consente a Rust si sapere
dove finisce l'invocazione della macro.

Più formalmente, il corpo dell'invocazione della macro deve essere una sequenza
di ‘alberi di token’. Un albero di token è definito ricorsivamente come o

* una sequenza di alberi di token circondati da `()`, `[]`, o `{}`, oppure
* qualunque altro token singolo.

Internamente a un matcher, ogni metavariabile ha uno ‘specificatore
di frammento’, che identifica a quale forma sintattica combacia. Eccoli:

* `ident`: un identificatore. Per es.: `x`, o `foo`.
* `path`: un nome qualificato. Per es.: `T::SpecialA`.
* `expr`: un'espressione. Per es.: `2 + 2`, o `if true { 1 } else { 2 }`,
  o `f(42)`.
* `ty`: un tipo. Per es.: `i32`, o `Vec<(char, String)>`, o `&T`.
* `pat`: un pattern. Per es.: `Some(t)`, o `(17, 'a')`, o `_`.
* `stmt`: una singola istruzione. Per es.: `let x = 3`.
* `block`: una sequenza di istruzioni delimitata da graffe, che eventualmente
  finisce con un'espressione. Per es.: `{ log(errore, "ciao"); return 12; }`.
* `item`: un [elemento][elemento]. Per es.: `fn foo() { }`, o `struct Bar;`.
* `meta`: un "meta elemento", come si trova negli attributi. Per es.:
  `cfg(target_os = "windows")`.
* `tt`: un singolo albero di token.

Ci sono altre regole che riguardano il token che segue una metavariabile:

* le variabili di tipo `expr` e `stmt` possono essere seguite solamente da uno
  dei seguenti token: `=> , ;`
* le variabili di tipo `ty` e `path` possono essere seguite solamente da uno
  di: `=> , = | ; : > [ { as where`
* le variabili di tipo `pat` possono essere seguite solamente da uno di:
  `=> , = | if in`
* le variabili di alri tipi possono essere seguite da qualunque token.

Queste regole forniscono alla sintassi di Rust un po' di flessibilità
di evolvere senza perdere compatibilità con le macro esistenti.

Il sistema delle macro non tratta affatto l'ambiguità sintattica. Per esempio,
la grammatica `$($i:ident)* $e:expr` non verrà mai accettato dal parser,
perché il parser sarebbe costretto a scegliere fra analizzare `$i`
e anlizzare `$e`. Cambiare la sintassi di invocazione per mettere
un token distintivo davanti può correggere il difetto. In questo caso,
si può scrivere `$(I $i:ident)* E $e:expr`.

[elemento]: ../reference.html#items

# Gli ambiti e l'import/export di macro

Le macro vengono espanse in una fase iniziale della compilazione, prima della
risoluzione dei nomi. Uno svantaggio è che gli ambiti funzionano diversamente
per le macro, rispetto agli altri costrutti del linguaggio.

Sia la definizione che l'espansione delle macro avvengono in un singolo
attraversamento lessicale del codice sorgente del crate. Quindi una macro
definita nell'ambito del modulo è visibile a tutto il codice seguente nello
stesso modulo, che comprende il corpo di ogni successivo elemendo `mod` figlio.

Una macro definita entro il corpo di un'unica `fn`, o da qualunque altra parte
non nell'ambito di un modulo, è visibile solamente entro quell'elemento.

Se un modulo ha l'attributo `macro_use`, le sue macro sono visibili anche nel
suo modulo genitore dopo l'elemento `mod` del figlio. Se anche il genitore
ha `macro_use` allora le macro saranno visibili nel nonno dopo l'elemento
`mod` del genitore, e così via.

L'attributo `macro_use` può apparire anche su `extern crate`. In questo
contesto controlla quali macro vengono caricate da crate esterno, per es.:

```rust,ignore
#[macro_use(foo, bar)]
extern crate baz;
```

Se l'attributo è dato semplicemente come `#[macro_use]`, tutte le macro vengono
caricate. Se non c'è nessun attributo `#[macro_use]` allora nessuna macro viene
caricata. Solamente le macro definite con l'attributo `#[macro_export]` possono
essere caricate.

Per caricare le macro di un crate senza linkarlo nell'output, si usi anche
l'attributo use `#[no_link]`.

Per esempio:

```rust
macro_rules! m1 { () => (()) }

// qui è visibile: m1

mod foo {
    // qui è visibile: m1

    #[macro_export]
    macro_rules! m2 { () => (()) }

    // qui sono visibili: m1, m2
}

// qui è visibile: m1

macro_rules! m3 { () => (()) }

// qui sono visibili: m1, m3

#[macro_use]
mod bar {
    // qui sono visibili: m1, m3

    macro_rules! m4 { () => (()) }

    // qui sono visibili: m1, m3, m4
}

// qui sono visibili: m1, m3, m4
# fn main() { }
```

Quando questa libreria viene caricata usando `#[macro_use] extern crate`,
verrà importata solo `m2`.

Il Rust Reference ha un [elenco di attributi relativi alle macro]
(../reference.html#macro-related-attributes).

# La variabile `$crate`

Un'ulteriore difficoltà avviene quando una macro viene usata in più crate.
Diciamo che `mylib` definisce

```rust
pub fn incrementa(x: u32) -> u32 {
    x + 1
}

#[macro_export]
macro_rules! inc_a {
    ($x:expr) => ( ::incrementa($x) )
}

#[macro_export]
macro_rules! inc_b {
    ($x:expr) => ( ::mylib::increment($x) )
}
# fn main() { }
```

`inc_a` funziona solamente entro `mylib`, mentre `inc_b` funziona solamente
al di fuori della libreria. Inoltre, `inc_b` non funzionerà se l'utente
importerà `mylib` sotto un altro nome.

Rust non ha (ancora) un sistema d'igiene per i riferimenti dei crate, ma
fornisce una semplice soluzione a questo problema. Entro una macro importata
da un crate chiamato `foo`, la speciale variabile di macro `$crate`
si espanderà in `::foo`. D'altra parte, quando una macro è definita e poi usata
nello stesso crate, `$crate` si espanderà a niente. Ciò significa
che possiamo scrivere

```rust
#[macro_export]
macro_rules! inc {
    ($x:expr) => ( $crate::incrementa($x) )
}
# fn main() { }
```

per definire una singola macro che funziona sia all'interno che all'esterno
della nostra libreria. Il nome della funzione si espanderà a `::increment` in
un caso, e a `::mylib::increment` nell'altro.

Per mantenere semplice e corretto questo sistema, `#[macro_use] extern crate
...` può apparire solamente alla radice del nostro crate, non dentro un `mod`.

# L'estremità profonda

La sezione introduttiva ha accennato alle macro ricorsive, ma non ha raccontato
tutta la storia. Le macro ricorsive sono utili per un'altra ragione: ogni
invocazione ricorsiva dà un'altra opportunità di far combaciare gli argomenti
della macro.

Come esempio estremo, è possibile, per quanto poco consigliabile, implementare
l'automa del linguaggio esoterico di programmazione [Bitwise_Cyclic_Tag]
(https://esolangs.org/wiki/Bitwise_Cyclic_Tag) entro il sistema delle macro
di Rust.

```rust
macro_rules! bct {
    // cmd 0:  d ... => ...
    (0, $($ps:tt),* ; $_d:tt)
        => (bct!($($ps),*, 0 ; ));
    (0, $($ps:tt),* ; $_d:tt, $($ds:tt),*)
        => (bct!($($ps),*, 0 ; $($ds),*));

    // cmd 1p:  1 ... => 1 ... p
    (1, $p:tt, $($ps:tt),* ; 1)
        => (bct!($($ps),*, 1, $p ; 1, $p));
    (1, $p:tt, $($ps:tt),* ; 1, $($ds:tt),*)
        => (bct!($($ps),*, 1, $p ; 1, $($ds),*, $p));

    // cmd 1p:  0 ... => 0 ...
    (1, $p:tt, $($ps:tt),* ; $($ds:tt),*)
        => (bct!($($ps),*, 1, $p ; $($ds),*));

    // arrestati quando la stringa di dati è vuota
    ( $($ps:tt),* ; )
        => (());
}
```

Esercizio: usa delle macro per ridurre la duplicazione nella definizione
della macro `bct!` scritta qui sopra.

# Macro tipiche

Ecco alcune macro che si incontrano spesso nel codice Rust.

## panic!

Questa macro provoca il panico nel thread corrente. Le si può dare un messaggio
da emettere:

```rust,no_run
panic!("oh no!");
```

## vec!

La macro `vec!` è usata spesso in questo libro. Crea agevolmente dei `Vec<T>`:

```rust
let v = vec![1, 2, 3, 4, 5];
```

Consente anche di creare un vettore ripetendo un valore. Per esempio,
per avere un vettore di cento zeri:

```rust
let v = vec![0; 100];
```

## assert! and assert_eq!

Queste due macro sono usate nei test. `assert!` prende un booleano.
`assert_eq!` prende due valori e verifica che siano uguali. `true` passa,
`false` va in `panic!`. Così:

```rust,no_run
// Ok!

assert!(true);
assert_eq!(5, 3 + 2);

// non va :(

assert!(5 < 3);
assert_eq!(5, 3);
```

## try!

`try!` serve alla gestione degli errori. Prende qualcosa che può rendere
un `Result<T, E>`, e dà il `T` se il risultato è un `Ok<T>`, e altrimenti esce
rendendo `Err(E)`. Così:

```rust,no_run
use std::fs::File;

fn foo() -> std::io::Result<()> {
    let f = try!(File::create("foo.txt"));

    Ok(())
}
```

Questo è l'equivalente, più pulito, di questo:

```rust,no_run
use std::fs::File;

fn foo() -> std::io::Result<()> {
    let f = match File::create("foo.txt") {
        Ok(t) => t,
        Err(e) => return Err(e),
    };

    Ok(())
}
```

## unreachable!

Questa macro serve quando si ritiene che un punto del codice non dovrebbe mai
essere raggiunto:

```rust
if false {
    unreachable!();
}
```

Talvolta, il compilatore può condurre a una diramazione che si è convintissimi
che non dovrebbe mai essere presa. In questi casi, si usi questa macro, così
che se ci si sbaglia, si otterrà un bel `panic!`.

```rust
let x: Option<i32> = None;

match x {
    Some(_) => unreachable!(),
    None => println!("Lo sapevo che x era None!"),
}
```

## unimplemented!

La macro `unimplemented!` può servire quando si sta provando a scrivere
una funzione del tipo giusto, ma che non ha ancora un corpo. Un esempio
di questa situazione è quando si implementa un tratto che ha più metodi
da definire, e si vuole affrontarne uno per volta. Si definiscano gli altri
come `unimplemented!` finché non se ne scrive un corpo corretto.

# Macro procedurali

Se il sistema delle macro di Rust non può fare quello di cui si ha bisogno,
si potrebbe voler scrvere invece un [plugin per il compilatore]
(compiler-plugins.html). Rispetto alle macro `macro_rules!`, ciò richiede
sostanzialmente più lavoro, le interfacce sono molto meno stabili,
e i difetti possono essere molto più difficili da risolvere. In cambio
si ottiene la flessibilità  di eseguire del codice Rust arbitrario all'interno
del compilatore. Per questa ragione, i plugin di estensione sintattica
sono talvolta chiamati ‘macro procedurali’.
