# Il linguaggio di programmazione Rust (anno 2016)

Questo libro è una traduzione della [documentazione ufficiale in inglese][book-html]
del linguaggio Rust. La versione originale di riferimento è la 1.12.0-dev,
risalente al 17 luglio 2016.
Tale documentazione comprende una dettagliata bibliografia in inglese.

Benvenuti! Questo libro insegnerà il [linguaggio di programmazione Rust][rust].
Rust è un linguaggio di programmazione di sistemi incentrato su tre obiettivi:
correttezza, velocità, e concorrenza. Il linguaggio raggiunge questi tre
obiettivi senza utilizzare un garbage collector, rendendosi utile in vari
ambiti applicativi in cui altri linguaggi non sono adeguati:
essere integrato con altri linguaggi, scrivere programmi con requisiti
stringenti di spazio e di tempo, e scrivere codice a basso livello,
come device driver e sistemi operativi. Il linguaggio è migliore di altri
attualmente utilizzati in questo ambito, in quanto
introduce varie verifiche di correttezza in fase di compilazione
che non hanno alcun impatto in fase di esecuzione, e allo stesso tempo
eliminano tutti i conflitti di accesso ai dati.
Rust ha anche l'obiettivo di ottenere ‘astrazioni a costo zero’
anche se alcune di queste astrazioni somigliano a quelle di linguaggi
ad alto livello.
Ciononostante, Rust consente ancora un controllo preciso,
tipico di un linguaggio di basso livello.

[book-html]: https://doc.rust-lang.org/book/
[rust]: https://www.rust-lang.org

“Il linguaggio di programmazione Rust” è suddiviso in capitoli.
Questa introduzione è il primo. Poi vengono:

* [Come iniziare][gs] - Settare il computer per sviluppare con Rust.
* [Tutorial: Gioco-indovina][gg] - Imparare un po' di Rust, con un piccolo
  progetto.
* [Sintassi e semantica][ss] - Ogni parte di Rust, un pezzetto per volta.
* [Rust efficace][er] - Concetti ad alto livello per scrivere del codice Rust
  eccellente .
* [Rust notturno][nr] - Funzionalità all'avanguardia che non sono ancora
  nella versione stabile.
* [Glossario][gl] - Un riferimento ai termini usati nel libro
  e la loro traduzione in inglese.

[gs]: getting-started.html
[gg]: guessing-game.html
[er]: effective-rust.html
[ss]: syntax-and-semantics.html
[nr]: nightly-rust.html
[gl]: glossary.html

### Contribuire

I file sorgente da cui questo libro è stato generato si trovano
su [GitHub][libro].

[libro]: https://github.com/carlomilanesi/linguaggio-rust
