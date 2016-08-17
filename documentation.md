% Documentazione

La documentazione è una parte importante di qualunque progetto software, e
in Rust è di prima classe. Parliamo della strumentazione fornita da Rust
per documentare i propri progetti.

## A proposito di `rustdoc`

La distribuzione di Rust include uno strumento, `rustdoc`, che genera
documentazione. `rustdoc` viene usato anche da Cargo con il comando
`cargo doc`.

La documentazione può essere generata in due modi: dal codice sorgente,
e da file Markdown autonomi.

## Documentare il codice sorgente

Il modo primario di documentare un progetto Rust è annotando il codice
sorgente. A questo scopo, si possono usare i commenti di documentazione:

```rust,ignore
/// Costruisce un nuovo `Rc<T>`.
///
/// # Esempi
///
/// ```
/// use std::rc::Rc;
///
/// let cinque = Rc::new(5);
/// ```
pub fn new(value: T) -> Rc<T> {
    // l'implementazione va qui
}
```

Questo codice genera della documentazione che si presenta [così][rc-new].
Ho escluso l'implementazione, mettendo al suo posto un normale commento.

La prima cosa da notare riguardo a questa annotazione è che usa
`///` invece di `//`. La tripla barra indica un commento di documentazione.

I commenti di documentazione sono scritti in Markdown.

Rust tiene traccia di questi commenti, e li usa quando genera
la documentazione. Questo è importante quando si documentano cose come
gli enums:

```rust
/// Il tipo `Option`. Si veda
// [la documentazione a livello di modulo](index.html) per saperne di più.
enum Option<T> {
    /// Nessun valore
    None,
    /// Qualche valore `T`
    Some(T),
}
```

Il codice sopra funziona, ma questo sotto no:

```rust,ignore
/// Il tipo `Option`. Si veda
// [la documentazione a livello di modulo](index.html) per saperne di più.
enum Option<T> {
    None, /// Nessun valore
    Some(T), /// Qualche valore `T`
}
```

Infatti si avrà l'errore:

```text
hello.rs:4:1: 4:2 error: expected ident, found `}`
hello.rs:4 }
           ^
```

Questo [errore sfortunato](https://github.com/rust-lang/rust/issues/22547) è
giusto; i commenti di documentazione si applicano a quello che li segue,
e non c'è niente dopo l'ultimo commento.

[rc-new]: ../std/rc/struct.Rc.html#method.new

### Scrivere i commenti di documentazione

Comunque, vediamo in dettaglio ogni parte di questo commento:

```rust
/// Costruisce un nuovo `Rc<T>`.
# fn foo() {}
```

La prima riga di un commento di documentazione dovrebbe essere un breve
riassunto della sua funzionalità che sta descrivendo. Una sola frase.
Solo le basi. Ad alto livello.

```rust
///
/// Altri dettagli sulla costruzione degli `Rc<T>`, eventualmente descrivendo
/// una semantica complicata, forse anche delle opzioni aggiuntive,
/// tutti gli aspetti
///
# fn foo() {}
```

Il nostro esempio originale aveva solo una riga riassuntiva, ma se avessimo
avuto più cose da dire, avremmo potuto aggiungere altre spiegazioni
in un altro paragrafo.

#### Sezioni speciali

Poi, ci sono le sezioni speciali. Queste sono indicate con un'intestazione,
`#`. Ci sono quattro tipi di intestazioni che vengono comunemente usate.
Per adesso, non hanno una sintassi speciale, ma solo convenzioni.

```rust
/// # Panico
# fn foo() {}
```

L'abuso irrecuperabile di una funzione (cioè un errore di programmazione)
in Rust è solitamente chiamato panico, che come minimo uccide l'intero thread
corrente. Se la propria funzione ha un contratto non banale, che se
rilevato/forzato produce un panico, è molto importante documentarlo.

```rust
/// # Errori
# fn foo() {}
```

Se la propria funzion o il proprio metodo rene un `Result<T, E>`, allora
descrivere le condizioni sotto le quali restituisce `Err(E)` è una cosa carina
da fare. Questo è leggermento meno importante del `Panics`,
perché tale fallimento è codificato nel sistema dei tipi,
ma è sempre una buona cosa da fare.

```rust
/// # Sicurezza
# fn foo() {}
```

Se la propria funzione è `unsafe`, si dovrebbe spiegare quali invarianti
il chiamante è tenuto a rispettare.

```rust
/// # Esempi
///
/// ```
/// use std::rc::Rc;
///
/// let cinque = Rc::new(5);
/// ```
# fn foo() {}
```

Quarto, `Esempi`. Aggiungere uno o più esempi di come usare la propria
funzione, sarà molto apprezzato dagli utenti di tale funzione. Questi esempi
vanno dentro annotazioni di blocchi di codice, di cui parleremo fra un momento,
e possono avere più di una sezione:

```rust
/// # Esempi
///
/// Semplici pattern di `&str`:
///
/// ```
/// let v: Vec<&str> = "Mary had a little lamb".split(' ').collect();
/// assert_eq!(v, vec!["Mary", "had", "a", "little", "lamb"]);
/// ```
///
/// Pattern più complessi, con una lambda:
///
/// ```
/// let v: Vec<&str> = "abc1def2ghi".split(|c: char| c.is_numeric()).collect();
/// assert_eq!(v, vec!["abc", "def", "ghi"]);
/// ```
# fn foo() {}
```

Discutiamo i dettagli di questi blocchi di codice.

#### Annotazioni dei blocchi di codice

Per scrivere del codice Rust dentro un commento, si usa
il triplo accento grave:

```rust
/// ```
/// println!("Ciao, mondo");
/// ```
# fn foo() {}
```

Se si vuol scrivere qualcosa che non è codice Rust, si può aggiungere
un'annotazione:

```rust
/// ```c
/// printf("Ciao, mondo\n");
/// ```
# fn foo() {}
```

Questo evidenzierà la sintassi del codice in base al linguaggio indicato.
Se si sta mostrando del semplice testo, si scelga `text`.

Qui è importante scegliere l'annotazione corretta, perché `rustdoc` la usa
in un modo interessante: può venire usata per collaudare effettivamente
gli esempi in un crate di libreria, così da assicurasi che rimangano
aggiornati. Se si ha del codice C ma `rustdoc` pensa che sia Rust perché non
si è aggiunta l'annotazione, `rustdoc` si lamenterà quando proverà a generare
la documentazione.

## Documentazione come test

Discutiamo il nostro esempio di documentazione:

```rust
/// ```
/// println!("Ciao, mondo");
/// ```
# fn foo() {}
```

Si noterà che qui non c'è bisogno di un `fn main()` né di altro. `rustdoc`
aggiungerà automaticamente una funzione `main()` intorno al nostro codice,
usando dell'euristica per tentare di metterlo al posto giusto. Per esempio:

```rust
/// ```
/// use std::rc::Rc;
///
/// let cinque = Rc::new(5);
/// ```
# fn foo() {}
```

Questo finirà per collaudare:

```rust
fn main() {
    use std::rc::Rc;
    let cinque = Rc::new(5);
}
```

Ecco l'algoritmo completo che rustdoc usa per preprocessare gli esempi:

1. Tutti gli attributi iniziali `#![foo]` sono lasciati intatt come
   attributi del crate.
2. Alcuni attributi `allow` tipici vengono inseriti, tra cui
   `unused_variables`, `unused_assignments`, `unused_mut`, `unused_attributes`,
   e `dead_code`. Dei piccoli esempi spesso fanno scattare questi lint.
3. Se l'esempio non contiene `extern crate`, allora `extern crate
   <mycrate>;` viene inserito (si noti l'assenza di `#[macro_use]`).
4. Infine, se l'esempio non contiene `fn main`, il resto del testo è
   avvolto in `fn main() { il_nostro_codice }`.

Però questo `fn main` generato può creare problemi! Se ci sono istruzioni
`extern crate` o `mod` nel codice d'esempio, che sono riferite da istruzioni
`use`, non potranno essere risolte a meno che si includa almeno `fn main() {}`
per inibire il passo 4. Anche l'istruzione `#[macro_use] extern crate`
non funziona eccetto che alla radice del crate, e quindi, quando si collaudano
delle macro, è sempre obbligatorio inserire un `main` esplicito. Però,
non deve ingombrare la documentazione -- andiamo avanti a leggere!

Però, talvolta questo algoritmo non basta. Per esempio, che dire di tutti
questi esempi di codice con `///` di cui abbiamo parlato? Il testo grezzo:

```text
/// Un po' di documentazione.
# fn foo() {}
```

appare diverso dall'output:

```rust
/// Un po' di documentazione.
# fn foo() {}
```

Sì, è giusto: si possono aggiungere righe che iniziano con `# `, e verranno
nascoste dall'output, ma verranno usate quando si compila il proprio codice.
Lo si può usare a proprio vantaggio. In questo caso, i commenti
di documentazione devono essere applicati a qualche tipo di funzione, e quindi
se voglio mostrare appena un commento di documentazione, devo aggiungere
una piccola definizione di funzione sotto di esso. Al medesimo tempo, è lì
solamente per soddisfare il compilatore, e quindi nasconderla rende l'esempio
più chiaro. Si può usare questa tecnica per spiegare in dettaglio esempi
più lunghi, pur conservando la collaudabilità della propria documentazione.

Per esempio, si imagini che volessimo documentare questo codice:

```rust
let x = 5;
let y = 6;
println!("{}", x + y);
```

Potremmo volere che la documentazione finisca per apparire così:

> Prima, impostiamo `x` a cinque:
>
> ```rust
> let x = 5;
> # let y = 6;
> # println!("{}", x + y);
> ```
>
> Poi, impostiamo `y` a sei:
>
> ```rust
> # let x = 5;
> let y = 6;
> # println!("{}", x + y);
> ```
>
> E infine, stampiamo la somma di `x` e `y`:
>
> ```rust
> # let x = 5;
> # let y = 6;
> println!("{}", x + y);
> ```

Per mantenere collaudabile ogni blocco di codice, vogliamo l'intero programma
in ogni blocco, ma non vogliamo che il lettore veda tutte le linee ogni volta.
Ecco cosa abbiamo messo nel nostro codice sorgente:

```text
    Prima, impostiamo `x` a cinque:

    ```rust
    let x = 5;
    # let y = 6;
    # println!("{}", x + y);
    ```

    Poi, impostiamo `y` a sei:

    ```rust
    # let x = 5;
    let y = 6;
    # println!("{}", x + y);
    ```

    E infine, stampiamo la somma di `x` e `y`:

    ```rust
    # let x = 5;
    # let y = 6;
    println!("{}", x + y);
    ```
```

Ripetendo tutte le parti dell'esempio, possiamo assicurarci che
il nostro esempio compili ancora, mentre mostriamo solamente le parti che
sono rilevanti a quella parte della nostra spiegazione.

### Macro di documentazione

Ecco un esempio di come si documenta una macro:

```rust
/// Va in panico con un dato messaggio, a meno che un'espressione valga true.
///
/// # Esempi
///
/// ```
/// # #[macro_use] extern crate foo;
/// # fn main() {
/// panic_unless!(1 + 1 == 2, “La matematica è scassata.”);
/// # }
/// ```
///
/// ```rust,should_panic
/// # #[macro_use] extern crate foo;
/// # fn main() {
/// panic_unless!(true == false, “Sono scassato.”);
/// # }
/// ```
#[macro_export]
macro_rules! panic_unless {
    ($condition:expr, $($rest:expr),+) => ({ if ! $condition { panic!($($rest),+); } });
}
# fn main() {}
```

Si noteranno tre cose: dobbiamo aggiungere la nostra riga `extern crate`, per
poter aggiungere l'attributo `#[macro_use]`. Secondo, dovremo aggiungere anche
la nostra `main()` (per la ragione detta prima). Infine, un uso giudizioso
di `#` per escludere quelle due cose, così che non appaiano nell'output.

Un altro caso dove l'suo di `#` è comodo è quando si vuole ignorare
la gestione degli errori. Diciamo che vogliamo il codice seguente,

```rust,ignore
/// use std::io;
/// let mut input = String::new();
/// try!(io::stdin().read_line(&mut input));
```

Il problema è che `try!` restituisce un `Result<T, E>`, e dato che le funzioni
di test non devono restituire niente, questo codice genererà un errore di tipo.

```rust,ignore
/// Un test di documentazione che usa "try!"
///
/// ```
/// use std::io;
/// # fn foo() -> io::Result<()> {
/// let mut input = String::new();
/// try!(io::stdin().read_line(&mut input));
/// # Ok(())
/// # }
/// ```
# fn foo() {}
```

Questo problema si può aggirare avvolgendo il codice in una funzione.
Questa prende e inghiotte il `Result<T, E>` quando si eseguono i test
sui documenti. Questo pattern appare regolarmente nella libreria standard.

### Eseguire i test della documentazione

Per eseguire i test, o si esegue:

```bash
$ rustdoc --test percorso/al/mio/crate/radice.rs
# oppure si esegue
$ cargo test
```

Proprio così, `cargo test` collauda anche la documentazione incorporata
nei sorgenti. **Però, `cargo test` non collauderà i crate di programma, ma
solamente quelli di libreria.** Questo è dovuto al modo in cui funziona
`rustdoc`: esegue il link con la libreria da collaudare, ma con un programma,
non c'è niente con cui eseguire il link.

Ci sono alcune altre annotazione che servono ad aiutare `rustdoc` a fare
la cosa giusta quando collauda il codice:

```rust
/// ```rust,ignore
/// fn foo() {
/// ```
# fn foo() {}
```

La direttiva `ignore` dice a Rust di ignorare il codice. Questo non è
quasi mai quello che si vuole, dato che è il più generico. Invece, si prenda
in considerazione l'annotarlo con `text` se non è codice, o l'usare i `#` per
ottenere un esempio funzionante che mostra solamente la parte che interessa.

```rust
/// ```rust,should_panic
/// assert!(false);
/// ```
# fn foo() {}
```

`should_panic` dice a `rustdoc` che il codice dovrebbe compilare correttamente,
ma non passare effettivamente come test.

```rust
/// ```rust,no_run
/// loop {
///     println!("Hello, world");
/// }
/// ```
# fn foo() {}
```

L'attributo `no_run` compilerà il codice, ma non lo eseguirà. Questo è
importante per gli esempi come "Ecco come avviare un servizio di rete," che si
vorrebbe assicurarsi che compili, ma che potrebbe eseguire un ciclo infinito!

### Documentare i moduli

Rust ha un altro tipo di commento di documentazione, `//!`. Questo commento
non documenta l'elemento successivo, ma quello che lo racchiude. In altre
parole:

```rust
mod foo {
    //! Questa è documentazione per il modulo `foo`.
    //!
    //! # Esempi

    // ...
}
```

Qui è dove si vedrà `//!` usato più spesso: per la documentazione dei moduli.
Se si ha un modulo in `foo.rs`, spesso, quando si apre il suo codice, si vedrà:

```rust
//! Un modulo per usare i `foo`.
//!
//! Il modulo `foo` contiene molte funzionalità utili, bla bla bla ...
```

### Documentazione dei crate

I crate possono essere documentati collocando un commento interno
di documentazione (`//!`) all'inizio della radice del crate, ossia di `lib.rs`:

```rust
//! Questa è documentazione per il crate `foo`.
//!
//! Il crate `foo` è pensato per essere usate per `bar`.
```

### Stile dei commenti di documentazione

Si veda la [RFC 505][rfc505] per le convezioni complete sullo stile
e il formato della documentazione.

[rfc505]: https://github.com/rust-lang/rfcs/blob/master/text/0505-api-comment-conventions.md

## Altra documentazione

Tutto questo comportamento funziona anche in file sorgente non in Rust.
Siccome i commenti sono scritti in Markdown, sono spesso dei file `.md`.

Quando si scrive della documentazione nei file Markdown, non serve marcare
la documentazione con i prefissi dei commenti. Per esempio:

```rust
/// # Esempi
///
/// ```
/// use std::rc::Rc;
///
/// let five = Rc::new(5);
/// ```
# fn foo() {}
```

è:

~~~markdown
# Esempi

```
use std::rc::Rc;

let five = Rc::new(5);
```
~~~

quando è in un file Markdown. Però c'è un neo: i file Markdown devono avere
un titolo così:

```markdown
% Il titolo

Questa è la documentazione d'esempio.
```

Questa riga che inizia con `%` deve essere la primissima riga del file.

## Attributi `doc`

A un livello più profondo, i commenti di documentazione sono addolcimento
sintattico per gli attributi di documentazione:

```rust
/// questo
# fn foo() {}

#[doc="questo"]
# fn bar() {}
```

sono equivalenti, così come lo sono questi:

```rust
//! questo

#![doc="questo"]
```

Non si vedrà spesso questo attributo usato per scrivere documentazione, ma può
essere utile quando si cambiamo alcune opzioni, o quando si scrive una macro.

### Ri-esportazioni

`rustdoc` mostrerà la documentazione per una ri-esportazione pubblica
in entrambi i posti:

```rust,ignore
extern crate foo;

pub use foo::bar;
```

Questo creerà la documentazione per `bar` sia dentro la documentazione
del crate `foo`, che nella documentazione del crate corrente. Userà la medesima
documentazione in entrambi i posti.

Questo comportamento può venire soppresso usando `no_inline`:

```rust,ignore
extern crate foo;

#[doc(no_inline)]
pub use foo::bar;
```

## Documentazione mancante

Talvolta ci si vuole assicurare che ogni singola cosa public nel proprio
progetto sia documentata, specialmente quando si sta lavorando a una libreria.
Rust consente di generare avvertimenti o errori, quando un elemento è privo
di documentazione. Per generare degli avvertimenti, si usa `warn`:

```rust
#![warn(missing_docs)]
```

E per generare errore si usa `deny`:

```rust,ignore
#![deny(missing_docs)]
```

Ci sono casi in cui si vogliono disabilitare questi avvertimenti/errori
per lasciare esplicitamente qualcosa di non documentato. Questo si fa
usando `allow`:

```rust
#[allow(missing_docs)]
struct NonDocumentata;
```

Si potrebbe perfino voler nascondere completamente degli elementi
dalla documentazione:

```rust
#[doc(hidden)]
struct Nascosta;
```

### Controllare l'HTML

Si possono controllare alcuni aspetti del codice HTML generato da `rustdoc`,
tramite la versione `#![doc]` dell'attributo:

```rust
#![doc(html_logo_url = "https://www.rust-lang.org/logos/rust-logo-128x128-blk-v2.png",
       html_favicon_url = "https://www.rust-lang.org/favicon.ico",
       html_root_url = "https://doc.rust-lang.org/")]
```

Quest imposta alcune diverse opzioni, con un logo, un'icona per i preferiti
(favicon), e un URL radice.

### Configurare il collaudo della documentazione

Si può anche configurare il modo in cui `rustdoc` collauda gli esempi
nella documentazione tramite l'attributo `#![doc(test(..))]`.

```rust
#![doc(test(attr(allow(unused_variables), deny(warnings))))]
```

Ciò consente che gli esempi contengano variabili inutilizzate, ma fallirà
il test per ogni altro avvertimento generato.

## Opzioni di generazione

`rustdoc` contiene anche alcune altre opzioni da riga di comando,
per un'ulteriore personalizzazione:

- `--html-in-header FILE`: inserisce il contenuto del file FILE alla fine
  della sezione `<head>...</head>`.
- `--html-before-content FILE`: inserisce il contenuto del file FILE appena
  dopo `<body>`, prima del contenuto mostrato (compresa la barra di ricerca).
- `--html-after-content FILE`: inserisce il contenuto del file FILE dopo
  tutto il contenuto mostrato.

## Nota sulla vulnerabilità

Il codice Markdown nei commenti di documentazione viene collocato nella pagina
web finale senza essere elaborato. Si presti attenzione all'HTML esplicito:

```rust
/// <script>alert(document.cookie)</script>
# fn foo() {}
```
