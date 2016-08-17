% Rust notturno

Rust fornisce tre canali di distribuzione per Rust: notturno, beta, e stabile.
Le caratteristiche instabili sono disponibli solamente su Rust notturno.
Per altri dettagli su questo procedimento, si veda ‘[La stabilità
come prodotto consegnabile][stability]’.

[stability]: http://blog.rust-lang.org/2014/10/30/Stability.html

Per installare Rust notturno, si può usare `rustup.sh`:

```bash
$ curl -s https://static.rust-lang.org/rustup.sh | sh -s -- --channel=nightly
```

Se si è preoccupati per la [potenziale vulnerabilità][insecurity] dell'uso
di `curl | sh`, si continui a leggere e si veda il disclaimer sotto. E ci
si senta liberi di usare una versione in due passi dell'installazione
e si esamini lo script di installazione:

```bash
$ curl -f -L https://static.rust-lang.org/rustup.sh -O
$ sh rustup.sh --channel=nightly
```

[insecurity]: http://curlpipesh.tumblr.com

Se si è su Windows, si scarichi o il [pacchetto di installazione a 32 bit]
[win32] o il [pacchetto di installazione a 64 bit][win64], e lo si esegua.

[win32]: https://static.rust-lang.org/dist/rust-nightly-i686-pc-windows-gnu.msi
[win64]: https://static.rust-lang.org/dist/rust-nightly-x86_64-pc-windows-gnu.msi

## Disinstallazione

Se si decide di non voler più Rust, basta lanciare lo script
di disintallazione:

```bash
$ sudo /usr/local/lib/rustlib/uninstall.sh
```

Se è stato usato un pacchetto di installazione di Windows, si ri-esegua
il file `.msi`, e si scelga l'opzione di disintallazione che compare.

Alcune persone, e un po' giustamente, si sconvolgono quando le diciamo
di eseguire `curl | sh`. Di base, quando lo si fa, ci si sta fidando che
le persone che mantengono Rust non stiano per acquisire il controllo
del computer per fare brutte cose.
È un buon istinto! Se si è una di tali persone, si consulti la documentazione
su [costruire Rust dai sorgenti][from-source], oppure
[i download binari ufficiali][install-page].

[from-source]: https://github.com/rust-lang/rust#building-from-source
[install-page]: https://www.rust-lang.org/install.html

Le piattaforme di sviluppo ufficialmente supportate sono:

* Windows (7, 8, Server 2008 R2)
* Linux (2.6.18 o successivo, varie distribuzioni), x86 e x86-64
* OSX 10.7 (Lion) o maggiore, x86 e x86-64

Rust è ampiamente testato su queste piattaforme, e anche su alcune altre,
come Android. Ma queste sono quelle che più probabilmente funzionano,
dato che hanno il collaudo più intenso.

Infine, un commento su Windows. Rust considera Windows una piattaforma
di prima-classe dopo il rilascio, ma onestamente, l'utilizzo in ambiente
Windows non è così integrato come negli ambienti Linux/OS X. Ci si sta
lavorando! Se qualcosa non funziona, è un difetto. Se accade, dovrebbe essere
comunicato. Ogni commit viene collaudata su Windows come ogni
altra piattaforma.

Se Rust è installato, si può aprire un terminale (o "shell"), e digitare:

```bash
$ rustc --version
```

Si dovrebbe vedere il numero di versione, lo hash di commit, la data
di commit, e la data di build. Per esempio:

```bash
rustc 1.0.0-nightly (f11f3e7ba 2015-01-04) (built 2015-01-06)
```

Se appaiono, allora Rust è stato installato correttamente!

Questo pacchetto di installazione installa anche localmente una copia
della documentazione, così da poterla leggere offline. Sui sistemi tipo-UNIX,
si trova in `/usr/local/share/doc/rust`. Su Windows, è in una cartella
`share/doc`, dentro la cartella dove è stato installato Rust.

Se non ha funzionato, ci sono vari posti dove si può ottenere aiuto. Il più
facile è [il canale IRC #rust on irc.mozilla.org][irc], a cui si può
accedere tramite [Mibbit][mibbit]. Cliccando quel link, si potrà chattare
in inglese con altri Rustacei (un soprannome per i programmatori in Rust),
e si potrà ricevere consigli per risolvere i problemi. Altre ottime
risorse in inglese sono [il forum degli utenti][users],
e [Stack Overflow][stackoverflow].

[irc]: irc://irc.mozilla.org/#rust
[mibbit]: http://chat.mibbit.com/?server=irc.mozilla.org&channel=%23rust
[users]: https://users.rust-lang.org/
[stackoverflow]: http://stackoverflow.com/questions/tagged/rust
