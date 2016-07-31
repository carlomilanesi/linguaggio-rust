% Collaudo ["Testing"]

> Il collaudo dei programmi può essere un modo molto efficace per mostrare
> la presenza di difetti, ma è disperatamente inadeguato a mostrare
> la loro assenza.
> 
> Edsger W. Dijkstra, "The Humble Programmer" (1972)

Parliamo di come collaudare il codice Rust. Ciò di cui non parleremo è il modo
giusto di collaudare il codice Rust. Ci sono molte scuole di pensiero
riguardo il modo giusto o sbagliato di eseguire collaudi. Però, tutti questi
approcci usano gli stessi strumenti di base, e quindi mostreremo la sintassi
per usarli.

# L'attributo `test`

Nel caso più semplice, un collaudo in Rust ha un solo test, che è una funzione
annotata con l'attributo `test`. Usando Cargo, facciamo un nuovo progetto
chiamato `sommatore`:

```bash
$ cargo new sommatore
$ cd sommatore
```

Cargo genera automaticamente un semplice test quando si crea un nuovo progetto.
Ecco il contenuto di `src/lib.rs`:

```rust
# fn main() {}
#[test]
fn it_works() {
}
```

Si noti il `#[test]`. Questo attributo indica che questa è una funzione
di collaudo. Attualmente ha il corpo vuoto. È abbastanza buono per passare!
Possiamo eseguire i test con il comando `cargo test`:

```bash
$ cargo test
   Compiling sommatore v0.0.1 (file:///home/you/projects/sommatore)
     Running target/sommatore-91b3e234d4ed382a

running 1 test
test it_works ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured

   Doc-tests sommatore

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured
```

Cargo ha compilato ed eseguito il nostro test. Qui ci sono due insiemi
di output: uno per il test che abbiamo scritto, e un altro per i test
della documentazione. Di quelli ne parleremo dopo. Per adesso, vediamo
questa riga:

```text
test it_works ... ok
```

Si noti il `it_works` ("funziona"). Deriva dal nome della nostra funzione:

```rust
fn it_works() {
# }
```

Abbiamo anche una riga riassuntiva:

```text
test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured
```

Quindi perché il nostro test non-fare-niente è passato? Qualunque test
che non va in `panic!` passa, e qualunque test che va in `panic!` fallisce.
Facciamo fallire il nostro test:

```rust
# fn main() {}
#[test]
fn it_works() {
    assert!(false);
}
```

`assert!` è una macro fornita da Rust che prende un argomento: se l'argomento
è `true`, non succede niente. Se l'argomento è `false`, va in `panic!`.
Rieseguiamo i nostri test:

```bash
$ cargo test
   Compiling sommatore v0.0.1 (file:///home/you/projects/sommatore)
     Running target/sommatore-91b3e234d4ed382a

running 1 test
test it_works ... FAILED

failures:

---- it_works stdout ----
        thread 'it_works' panicked at 'assertion failed: false', /home/steve/tmp/sommatore/src/lib.rs:3



failures:
    it_works

test result: FAILED. 0 passed; 1 failed; 0 ignored; 0 measured

thread 'main' panicked at 'Some tests failed', /home/steve/src/rust/src/libtest/lib.rs:247
```

Rust indica che il nostro test è fallito:

```text
test it_works ... FAILED
```

E si riflette nella riga riassuntiva:

```text
test result: FAILED. 0 passed; 1 failed; 0 ignored; 0 measured
```

Otteniamo anche un codice di stato non-nullo. Possiamo usare `$?` su OS X
o su Linux:

```bash
$ echo $?
101
```

Su Windows, se si usa `cmd`:

```bash
> echo %ERRORLEVEL%
```

Mentre se si usa PowerShell:

```bash
> echo $LASTEXITCODE # il codice stesso
> echo $? # un booleano, fallimento o successo
```

Questo è utile se si vuole integrare `cargo test` con altre utility.

Possiamo invertire il fallimento del nostro test usando un altro attributo:
`should_panic`:

```rust
# fn main() {}
#[test]
#[should_panic]
fn it_works() {
    assert!(false);
}
```

Questo test adesso ha successo se va in `panic!` e fallisce
termina normalmente. Proviamolo:

```bash
$ cargo test
   Compiling sommatore v0.0.1 (file:///home/you/projects/sommatore)
     Running target/sommatore-91b3e234d4ed382a

running 1 test
test it_works ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured

   Doc-tests sommatore

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured
```

Rust fornisce un'altra macro, `assert_eq!`, che verifica l'uguaglianza tra
due argomenti:

```rust
# fn main() {}
#[test]
#[should_panic]
fn it_works() {
    assert_eq!("Ciao", "mondo");
}
```

Questo test passerà o fallirà? A causa dell'attribute `should_panic`, passerà:

```bash
$ cargo test
   Compiling sommatore v0.0.1 (file:///home/you/projects/sommatore)
     Running target/sommatore-91b3e234d4ed382a

running 1 test
test it_works ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured

   Doc-tests sommatore

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured
```

Però, i test `should_panic` possono essere fragili, dato che è difficile
garantire che il test non è fallito per una ragione inattesa. Per aiutare
in questa difficoltà, un argomento facoltativo `expected` può essere aggiunto
all'attributo `should_panic`. L'ambiente di collaudo assicuererà che
il messaggio di fallimento contenga il testo fornito. Una versione più sicura
dell'esempio precedente sarebbe:

```rust
# fn main() {}
#[test]
#[should_panic(expected = "asserzione fallita")]
fn it_works() {
    assert_eq!("Hello", "world");
}
```

E questo è tutto per i fondamenti! Scriviamo un test 'vero':

```rust,ignore
# fn main() {}
pub fn somma_due(a: i32) -> i32 {
    a + 2
}

#[test]
fn it_works() {
    assert_eq!(4, somma_due(2));
}
```

Questo è un uso molto comune di `assert_eq!`: chiamare una funzione
con alcuni argomenti noti, e confrontarne l'esito con l'output atteso.

# L'attributo `ignore`

Talvolta alcuni test specifici possono richiedere molto tempo per essere
eseguiti. Questi possono essere disabilitati di default usando l'attributo
`ignore`:

```rust
# fn main() {}
#[test]
fn it_works() {
    assert_eq!(4, somma_due(2));
}

#[test]
#[ignore]
fn test_costoso() {
    // codice che impiega un'ora
}
```

Adesso eseguiamo i nostri test e vediamo che `it_works` viene eseguito,
ma `test_costoso` no:

```bash
$ cargo test
   Compiling sommatore v0.0.1 (file:///home/you/projects/sommatore)
     Running target/sommatore-91b3e234d4ed382a

running 2 tests
test expensive_test ... ignored
test it_works ... ok

test result: ok. 1 passed; 0 failed; 1 ignored; 0 measured

   Doc-tests sommatore

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured
```

I test costosi possono essere eseguiti esplicitamente usando
`cargo test -- --ignored`:

```bash
$ cargo test -- --ignored
     Running target/sommatore-91b3e234d4ed382a

running 1 test
test expensive_test ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured

   Doc-tests sommatore

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured
```

L'argomento `--ignored` è un argomento per il programma di collaudo, e non
per Cargo, e per questo motivo il comando è `cargo test -- --ignored`.

# Il modulo `tests`

C'è un solo aspetto per cui il nostro esempio esistente non è tipico di Rust:
gli manca il modulo `tests`. Il modo tipico di scrivere il nostro esempio
è il seguente:

```rust,ignore
# fn main() {}
pub fn somma_due(a: i32) -> i32 {
    a + 2
}

#[cfg(test)]
mod tests {
    use super::somma_due;

    #[test]
    fn it_works() {
        assert_eq!(4, somma_due(2));
    }
}
```

Qui ci sono alcuni cambiamenti. Il primo è l'introduzione di un `mod tests`
con un attributo `cfg`. Il modulo ci consente di raggruppare insieme
tutti i nostri test, e anche di definire delle funzioni ausiliare,
se servono, le quali non diventano parte del resto del nostro crate.
L'attributo `cfg` compila il nostro codice di collaudo solamente se
attualmente stiamo provando a eseguire i test. Questo può far risparmiare
tempo di compilazione, e inoltre assicura che i nostri test siano
interamente lasciati fuori da una build normale.

Il secondo cambiamento è la dichiarazione `use`. Siccome siamo in un modulo
interno, dobbiamo portare la nostra funzione di test nell'ambito. Ciò può
essere seccante in un grande modulo, e perciò questo è un uso tipico
dei globali. Modifichiamo il nostro `src/lib.rs` per farne uso:

```rust,ignore
# fn main() {}
pub fn somma_due(a: i32) -> i32 {
    a + 2
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn it_works() {
        assert_eq!(4, somma_due(2));
    }
}
```

Si noti la dversa riga `use`. Adesso esuiamo i nostri test:

```bash
$ cargo test
    Updating registry `https://github.com/rust-lang/crates.io-index`
   Compiling sommatore v0.0.1 (file:///home/you/projects/sommatore)
     Running target/sommatore-91b3e234d4ed382a

running 1 test
test tests::it_works ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured

   Doc-tests sommatore

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured
```

Funziona!

La convenzione attuale è usare il modulo `tests` per tenere i propri test
di unità. Ogni cosa che collauda un pezzettino di funzionalità
ha senso che vada qui. Ma che dire invece dei test di integrazione?
Per quelli c'è la directory `tests`.

# La directory `tests`

Ogni file `*.rs` nella directory `tests` viene trattato come un crate
individuale. Perciò, per scrivere un test di integrazione, creiamo
la directory `tests`, e ci mettiamo dentro il file `integration_test.rs`,
avente il seguente contenuto:

```rust,ignore
extern crate sommatore;

# fn main() {}
#[test]
fn it_works() {
    assert_eq!(4, sommatore::somma_due(2));
}
```

Questo appare simile ai nostri test precedenti, ma è leggermente diverso.
Adesso abbiamo in cima l'istruzione `extern crate sommatore`. Questo perché
ogni test della directory `tests` è un crate completamente separato, e quindi
dobbiamo importare la nostra libreria.
Questo spiega anche perché `tests` è un posto adatto per metterci i test
di integrazione: tali test usano la libreria come lo farebbe qualunque altro
consumatore.

Eseguiamoli:

```bash
$ cargo test
   Compiling sommatore v0.0.1 (file:///home/you/projects/sommatore)
     Running target/sommatore-91b3e234d4ed382a

running 1 test
test tests::it_works ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured

     Running target/lib-c18e7d3494509e74

running 1 test
test it_works ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured

   Doc-tests sommatore

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured
```

Adesso abbiamo tre sezioni: anche il nostro test precedente viene eseguito,
così come quello nuovo.

Cargo ignorerà i file nelle sottodirectory della directory `tests`.
Perciò in tali sottodirectory si potranno mettere i moduli condivisi dai
test di integrazione.
Per esempio, il file `tests/common/mod.rs` non verrà compilato separatamente
da Cargo ma potrà essere importato in ogni test con l'istruzione `mod common;`

Questo è tutto quel che c'è da dire sulla directory `tests`. Qui non serve
il modulo `tests`, dato che il tutto si incentra sui test.

Infine andiamo a vedere quella terza sezione: i test della documentazione.

# I test della documentazione

Non c'è niente di meglio che della documentazione con degli esempi.
E non c'è niente di peggio che degli esempi che in realtà non funzionano,
perché il codice è cambiato da quando la documentazione è stata scritta.
A questo scopo, Rust supporta l'esecuzione automatica degli esempi contenuti
nella documentazione del codice (**nota:** questo funziona solamente
nei crate di libreria, non nei crate di programma). Ecco un `src/lib.rs`
concretizzato con degli esempi:

```rust,ignore
# fn main() {}
//! Il crate `sommatore` fornisce delle funzioni
//! che sommano numeri ad altri numeri.
//!
//! # Esempi
//!
//! ```
//! assert_eq!(4, sommatore::somma_due(2));
//! ```

/// Questa funzione somma due al suo argomento.
///
/// # Esempi
///
/// ```
/// use sommatore::somma_due;
///
/// assert_eq!(4, somma_due(2));
/// ```
pub fn somma_due(a: i32) -> i32 {
    a + 2
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn it_works() {
        assert_eq!(4, somma_due(2));
    }
}
```

Si noti che la documentazione a livello di modulo è marcata con `//!`, mentre
la documentazione a livello di funzione è marcata con `///`. La documentazione
di Rust supporta il linguaggio Markdown nei commenti, e quindi i blocchi
di codice nei commenti sono marcati da terne di accenti gravi.
Si usa aggiungere la sezione `# Esempi`, proprio come sopra, seguita
da uno o più esempi.

Eseguiamo nuovamente i test:

```bash
$ cargo test
   Compiling sommatore v0.0.1 (file:///home/steve/tmp/sommatore)
     Running target/sommatore-91b3e234d4ed382a

running 1 test
test tests::it_works ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured

     Running target/lib-c18e7d3494509e74

running 1 test
test it_works ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured

   Doc-tests sommatore

running 2 tests
test somma_due_0 ... ok
test _0 ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured
```

Adesso abbiamo eseguito realmente tutti e tre i tipi di test! Si notino i nomi
dei test di documentazione: il nome `_0` viene generato per il test di modulo,
e il nome `somma_due_0` per il test di funzione. Questi si autoincrementeranno,
producendo nomi come `somma_due_1` quando si aggiungono altri esempi.

Non abbiamo trattato tutti i dettagli della scrittura dei test
della documentazione. Per saperne di più, si veda
il [capitolo della documentazione](documentation.html).
