% Il linguaggio di programmazione Rust

Benvenuti! Questo libro vi insegnerà il
[linguaggio di programmazione Rust][rust].
Rust è un linguaggio di programmazione di sistema incentrato su tre obiettivi:
sicurezza, velocità, e concorrenza. Questi tre obiettivi sono perseguiti senza
usare un garbage collector, il che lo rende un linguaggio utile
per vari ambiti applicativi in cui altri linguaggi non sono adeguati:
essere inserito in altri linguaggi, programmi con requisiti stringenti
di spazio o di tempo, e per scrivere codice a basso livello,
come device drivers e moduli di sistema operativo. È un miglioramento
rispetto ai linguaggi attualmente utilizzati in questo ambito, in quanto
introduce varie verifiche di correttezza in fase di compilazione
che non hanno alcun impatto in fase di esecuzione, e allo stesso tempo
eliminano tutti i conflitti di accesso ai dati (`data race`).
Inoltre Rust ha l'obiettivo di ottenre ‘astrazioni a costo zero’
anche se alcune di queste astrazioni somigliano a quelle di linguaggi
ad alto livello.
Ciononostante, Rust consente anche un controllo preciso,
come è richiesto per un linguaggio a basso livello.

[rust]: https://www.rust-lang.org

“Il linguaggio di programmazione Rust” è suddiviso in capitoli.
Questa introduzione è il primo. Dopo ci sono:

* [Iniziare][gs] - Come impostare il computer per lo sviluppo con Rust.
* [Tutorial: Indovinello][gg] - Imparare un po' di Rust con un piccolo progetto.
* [Sintassi e semantica][ss] - Ogni parte di Rust, un pezzetto per volta.
* [Rust efficace][er] - Concetti ad alto livello per scrivere dell'eccellente codice Rust.
* [Rust notturno][nr] - Funzionalità all'avanguardia che non sono ancora nella versione stabile.
* [Glossario][gl] - Riferimento ai termini usati nel libro.
* [Bibliografia][bi] - Articoli su Rust o che hanno ispirato lo sviluppo di Rust.

[gs]: getting-started.html
[gg]: guessing-game.html
[er]: effective-rust.html
[ss]: syntax-and-semantics.html
[nr]: nightly-rust.html
[gl]: glossary.html
[bi]: bibliography.html

### Contribuire

I file sorgente da cui questo libro è stato generato si trovano
su [GitHub][book].

[book]: https://github.com/rust-lang/rust/tree/master/src/doc/book
