% Come Iniziare

Questo primo capitolo del libro ci farà cominciare ad usare Rust e la sua
strumentazione. Per cominciare, installeremo Rust. Poi, il classico programma
‘Hello World’. Infine, parleremo di Cargo, il sistema di build per Rust e
del suo gestore di pacchetti.

# Installare Rust

Il primo passo per usare Rust è installarlo. In generale, servirà
una connessione ad Internet per eseguire i comandi di questa sezione, dato che
scaricheremo Rust da Internet.

Mostreremo vari comandi usando un terminale, e tutte quelle righe di comando
inizieranno con `$`. Non si devono digitare i `$`;
sono lì per indicare l'inizio di ogni comando. Vedremo molti tutorial ed
esempi in giro per il Web che seguono questa convenzione: `$` per i comandi
eseguiti da un utente normale, e `#` per i comandi che possono essere eseguiti
solamente da un amministratore.

## Supporto alle piattaforme

Il compilatore Rust funziona su, e genera codice per, un gran numero
di piattaforme; però non tutte le piattaforme sono egualmente supportate.
I livelli di supporto di Rust sono organizzati in tre livelli ["tier"],
ognuno con un diverso insieme di garanzie.

Le piattaforme sono identificate dalla loro "terna bersaglio" ("target triple")
che è la stringa che informa il compilatore su quale tipo di output dovrebbe
essere prodotto. Le colonne sotto indicano se il componente corrispondente
funziona sulla piattaforma specificata.

### Livello 1

Le piattaforme di livello 1 possono essere pensate come "con garanzia
di compilazione e funzionamento".
Specificamente ognuna di esse soddisferà i seguenti requisiti:

* Il collaudo automatizzato è impostato per eseguire i test per la piattaforma.
* L'approdo di modifiche al ramo master del repository `rust-lang/rust`
  dipende dal superamento di tutti i test.
* Per tale piattaforma sono forniti degli artefatti di rilascio ufficiale.
* Viene fornita la documentazione su come usare e come costruire il software
  per tale piattaforma.

|  Target                       | std |rustc|cargo| note                        |
|-------------------------------|-----|-----|-----|-----------------------------|
| `i686-apple-darwin`           |  ✓  |  ✓  |  ✓  | OSX (10.7+, Lion+) a 32 bit |
| `i686-pc-windows-gnu`         |  ✓  |  ✓  |  ✓  | MinGW (Windows 7+) a 32 bit |
| `i686-pc-windows-msvc`        |  ✓  |  ✓  |  ✓  | MSVC (Windows 7+) a 32 bit  |
| `i686-unknown-linux-gnu`      |  ✓  |  ✓  |  ✓  | Linux (2.6.18+) a 32 bit    |
| `x86_64-apple-darwin`         |  ✓  |  ✓  |  ✓  | OSX (10.7+, Lion+) a 64 bit |
| `x86_64-pc-windows-gnu`       |  ✓  |  ✓  |  ✓  | MinGW (Windows 7+) a 64 bit |
| `x86_64-pc-windows-msvc`      |  ✓  |  ✓  |  ✓  | MSVC (Windows 7+) a 64 bit  |
| `x86_64-unknown-linux-gnu`    |  ✓  |  ✓  |  ✓  | Linux (2.6.18+) a 64 bit    |

### Livello 2

Le piattaforme di livello 2 possono essere pensate come "con compilazione
garantita".
I collaudi automatizzati non vengono eseguiti, e quindi non è garantito che
si producano eseguibili che funzionino, ma queste piattaforme spesso funzionano
abbastanza bene e le migliorie sono sempre benvenute! Specificamente,
queste piattaforme devono presentare i seguenti requisiti:

* Viene eseguita la compilazione automatizzato, ma potrebbe non eseguire
  i collaudi.
* L'approdo di modifiche nel ramo master del repository `rust-lang/rust`,
  dipende dalla corretta **compilazione** per tale piattaforma.
  Si noti che ciò significa
  che per alcune piattaforme viene compilata solamente la libreria standard,
  mentre per altre viene eseguito il bootstrap completo.
* Per tale piattaforma sono forniti degli artefatti di rilascio ufficiale.

|  Target                       | std |rustc|cargo| note                            |
|-------------------------------|-----|-----|-----|---------------------------------|
| `aarch64-apple-ios`           |  ✓  |     |     | iOS per ARM64                   |
| `aarch64-unknown-linux-gnu`   |  ✓  |  ✓  |  ✓  | Linux (2.6.18+) per ARM64       |
| `arm-linux-androideabi`       |  ✓  |     |     | Android per ARM                 |
| `arm-unknown-linux-gnueabi`   |  ✓  |  ✓  |  ✓  | Linux (2.6.18+) per ARM         |
| `arm-unknown-linux-gnueabihf` |  ✓  |  ✓  |  ✓  | Linux (2.6.18+) per ARM         |
| `armv7-apple-ios`             |  ✓  |     |     | iOS per ARM                     |
|`armv7-unknown-linux-gnueabihf`|  ✓  |  ✓  |  ✓  | Linux (2.6.18+) per ARMv7       |
| `armv7s-apple-ios`            |  ✓  |     |     | iOS per ARM                     |
| `i386-apple-ios`              |  ✓  |     |     | iOS a 32 bit per x86            |
| `i586-pc-windows-msvc`        |  ✓  |     |     | Windows a 32 bit senza SSE      |
| `mips-unknown-linux-gnu`      |  ✓  |     |     | Linux (2.6.18+) per MIPS        |
| `mips-unknown-linux-musl`     |  ✓  |     |     | Linux con MUSL per MIPS         |
| `mipsel-unknown-linux-gnu`    |  ✓  |     |     | Linux (2.6.18+) per MIPS (LE)   |
| `mipsel-unknown-linux-musl`   |  ✓  |     |     | Linux con MUSL per MIPS (LE)    |
| `powerpc-unknown-linux-gnu`   |  ✓  |     |     | Linux (2.6.18+) per PowerPC     |
| `powerpc64-unknown-linux-gnu` |  ✓  |     |     | Linux (2.6.18+) per PPC64       |
|`powerpc64le-unknown-linux-gnu`|  ✓  |     |     | Linux (2.6.18+) per PPC64LE     |
| `x86_64-apple-ios`            |  ✓  |     |     | iOS a 64 bit per x86            |
| `x86_64-rumprun-netbsd`       |  ✓  |     |     | NetBSD a 64 bit con Kernel Rump |
| `x86_64-unknown-freebsd`      |  ✓  |  ✓  |  ✓  | FreeBSD a 64 bit                |
| `x86_64-unknown-linux-musl`   |  ✓  |     |     | Linux a 64 bit con MUSL         |
| `x86_64-unknown-netbsd`       |  ✓  |  ✓  |  ✓  | NetBSD a 64 bit                 |

### Livello 3

Le piattaforme di livello 3 sono quelle per cui Rust ha supporto, ma per
applicare delle modifiche non è necessario né eseguire la compilazione,
né passare i testi di collaudo.
Gli eseguibili funzionanti su tali piattaforme possono essere lacunosi,
dato che la loro affidabilità è spesso definita in termini di contributi
dalla comunità. In aggiunta, gli artefatti di rilascio e gli installatori
non sono forniti, ma ci possono essere infrastrutture comunitarie che
li producono in luoghi non ufficiali.

|  Target                       | std |rustc|cargo| note                     |
|-------------------------------|-----|-----|-----|--------------------------|
| `aarch64-linux-android`       |  ✓  |     |     | Android per ARM64        |
| `armv7-linux-androideabi`     |  ✓  |     |     | Android per ARM-v7a      |
| `i686-linux-android`          |  ✓  |     |     | Android a 32 bit per x86 |
| `i686-pc-windows-msvc` (XP)   |  ✓  |     |     | Windows XP support       |
| `i686-unknown-freebsd`        |  ✓  |  ✓  |  ✓  | FreeBSD a 32 bit         |
| `x86_64-pc-windows-msvc` (XP) |  ✓  |     |     | Windows XP support       |
| `x86_64-sun-solaris`          |  ✓  |  ✓  |     | Solaris/SunOS a 64 bit   |
| `x86_64-unknown-bitrig`       |  ✓  |  ✓  |     | Bitrig a 64 bit          |
| `x86_64-unknown-dragonfly`    |  ✓  |  ✓  |     | DragonFlyBSD a 64 bit    |
| `x86_64-unknown-openbsd`      |  ✓  |  ✓  |     | OpenBSD a 64 bit         |

Si noti che questa tabella può essere estesa col passare del tempo; questo non
è l'insieme completo di tutte le piattaforme di livello 3
che saranno supportate!

## Installare su Linux o Mac

Usando Linux o un Mac, si deve solamente aprire un terminale e digitare:

```bash
$ curl -sSf https://static.rust-lang.org/rustup.sh | sh
```

Questo comando scaricherà uno script, e avvierà l'installazione.
Se tutto va bene, si vedrà apparire:

```text
Rust is ready to roll.
```

Da qui, premere `y` per ‘yes’, e poi seguire il resto dei prompt.

## Installare su Windows

Usando Windows, si scarichi l'appropriato [installatore][install-page].

[install-page]: https://www.rust-lang.org/install.html

## Disinstallare

Disinstallare Rust è tanto facile quanto installarlo. Su Linux o Mac,
si lanci lo script di disintallazione:

```bash
$ sudo /usr/local/lib/rustlib/uninstall.sh
```

Se è stato usato l'installatore per Windows, si può rieseguire
il file `.msi`, che darà l'opzione di disinstallazione.

## Risoluzione difficoltà

Dopo aver installato Rust, si può aprire un terminale, e digitare:

```bash
$ rustc --version
```

Si dovrebbe vedere il numero di versione, l'hash di commit, e la data
di commit.

Se è così, Rust è stato installato con successo! Congratulazioni!

Se non è così, e si sta usando Windows, si verifichi che Rust sia
nella variabile di sistema %PATH%, digitando: `echo %PATH%`. Se non c'è,
si rilanci l'installatore, si selezioni "Change" nella pagina
"Change, repair, or remove installation" e ci si assicuri
che "Add to PATH" sia installato sul disco fisso locale.
Se c'è bisogno di configurare manualmente la variabile PATH,
gli eseguibili di Rust si trovano in una directory come
`"C:\Program Files\Rust stable GNU 1.x\bin"`.

Rust non esegue personalmente il link, e perciò c'è bisogno di avere un linker
installato. L'installazione del medesimo dipende dallo specifico sistema;
si consulti la sua documentazione per avere maggiori dettagli.

Se l'installazione non funziona, ci sono vari posti dove ottenere aiuto.
Il più facile è [il canale IRC #rust-beginners su irc.mozilla.org]
[irc-beginners] e per discussioni generali
[il canale IRC #rust su irc.mozilla.org][irc], a cui si può accedere tramite
[Mibbit][mibbit]. Lì si potrà chattare con altri Rustacean (uno sciocco
soprannome con cui ci chiamiamo) che possono aiutarci. Altre ottime risorse
sono [il forum degli utenti][users] e [Stack Overflow][stackoverflow].

[irc-beginners]: irc://irc.mozilla.org/#rust-beginners
[irc]: irc://irc.mozilla.org/#rust
[mibbit]: http://chat.mibbit.com/?server=irc.mozilla.org&channel=%23rust-beginners,%23rust
[users]: https://users.rust-lang.org/
[stackoverflow]: http://stackoverflow.com/questions/tagged/rust

Questo installatore installa anche localmente una copia della documentazione,
e quindi la si può leggere offline. Su sistemi UNIX si trova in
`/usr/local/share/doc/rust`. Su Windows, si trova nella directory `share\doc`,
all'interno della directory in cui Rust è stato installato.

# Hello, world!

Adesso che abbiamo installato Rust, vedremo come si scrive il primo programma
in Rust. Quando si impara un nuovo linguaggio di programmazione, c'è
la tradizione di scrivere programmino che stampi il testo “Hello, world!”
sullo schermo, e in questa sezione, seguiremo questa tradizione.

La cosa carina riguardo ad iniziare con un programma così semplice è che
si può verificare rapidamente che il compilatore sia installato
e funzioni correttamente.
Anche stampare informazioni sullo schermo è una cosa che si fa spesso,
e quindi è bene fare pratica da subito.

> Nota: Questo libro assume familiarità di base con la riga di comando. Rust
> stesso non pone specifici requisiti su quali strumenti usare, o su dove
> risieda il codice sorgente, e quindi se si preferisce un IDE alla riga
> di comando, si può fare. Si può provare [SolidOak], che è stato costruito
> specificamente per Rust. La comunità sta sviluppando varie estensioni, e
> la squadra di Rust consegna dei plugin per [vari editor].
> La configurazione dell'editor di testo o IDE non è trattata in questo
> tutorial; si consulti la documentazione del proprio ambiente di sviluppo.

[SolidOak]: https://github.com/oakes/SolidOak
[vari editor]: https://github.com/rust-lang/rust/blob/master/src/etc/CONFIGS.md

## Creare un file di progetto

Prima, creiamo un file per metterci dentro il codice Rust. A Rust
non interessa dove risieda il codice sorgente, a per questo libro, suggeriamo
di creare una directory *progetti* nell'area del proprio utente, e di tenerci
tutti i propri progetti. Si apra un terminal e si inseriscano i seguenti
comandi per creare una directory per questo progetto:

```bash
$ mkdir ~/projects
$ cd ~/projects
$ mkdir hello_world
$ cd hello_world
```

> Nota: Se si usa Windows ma non PowerShell, la `~` potrebbe non funzionare.
> Si consulti la documentazione della propria shell per avere più dettagli.

## Scrivere ed eseguire un programma in Rust

Poi, creiamo un nuovo file sorgente e lo chiamiamo *main.rs*. I file Rust
finiscono sempre con l'estensione *.rs*. Se si mettono più parole nel nome
del file, si usi un underscore per separarle; per esempio, si scriva
*hello_world.rs* invece di *helloworld.rs*.

Adesso si apra il file *main.rs* appena creato e si digiti:

```rust
fn main() {
    println!("Hello, world!");
}
```

Si salvi il file, e si torni alla finestra del terminale. Su Linux o OSX,
si digitino i seguenti comandi:

```bash
$ rustc main.rs
$ ./main
Hello, world!
```

In Windows, si sostituisca `./main` con `main`. Indipendentemente dal sistema
operativo, si dovrebbe vedere il testo `Hello, world!` sampato sul terminale.
Se è così, congratulazioni! Abbiamo scritto ufficialmente un programma in Rust.

## Anatomia di una programma in Rust

Adesso, esaminiamo ciò che è appena accaduto nel nostro programma
"Hello, world!" in dettaglio. Ecco il primo pezzo del puzzle:

```rust
fn main() {

}
```

Queste righe definiscono una *funzione* in Rust. La funzione `main` è speciale:
è l'inizio di ogni programma in Rust. La prima riga dice, “Sto dichiarando
una funzione chiamata `main` che non prende argomenti
e non restituisce niente.”
Se ci fossero argomenti, andrebbero fra le parentesi tonde (`(` e `)`),
e siccome non stiamo restituendo niente da questa funzione,
possiamo omettere il tipo di dato restituito.

Si noti anche che il corpo della funzione è racchiuso da parentesi graffe
(`{` e `}`). Rust le richiede intorno a tutti i corpi di funzione.
Si considera buon stile mettere la graffa aperta sulla stessa riga
della dichiarazione di funzione, separata da uno spazio.

Passiamo all'interno della funzione `main()`:

```rust
    println!("Hello, world!");
```

Questa riga fa tutto il lavoro di questo programmino: stampa il testo sullo
schermo. Qui ci sono vari dettagli importanti. Il primo è che è il codice
è rientrato di quattro spazi, senza usare tabulazioni.

La seconda parte importante è l'espressione `println!()`. Questa chiama
una *[macro]* di Rust, che è il modo usato da Rust per fare
la metaprogrammazione. Se invece stessimo chiamando una funzione,
si presenterebbe così: `println()` (senza il !). In seguito discuteremo
le macro di Rust in maggiore dettaglio, ma per ora basti sapere che quando c'è
un `!` significa che si sta chiamando una macro invece di una normale funzione.

[macro]: macros.html

Poi c'è `"Hello, world!"` che è una *stringa*. Le stringhe sono un argomento
sorprendentemente complicato in un linguaggio di programmazione di sistemi,
e questa è una stringa *[allocata staticamente]*. Passiamo questa stringa come
argumento a `println!`, che stampa la stringa sul terminale. Abbastanza facile!

[allocata staticamente]: the-stack-and-the-heap.html

La riga finisce con un punto-e-virgola (`;`). Rust è un *[linguaggio orientato
alle espressioni]*, che significa che la maggior parte delle cose sono
espressioni, invece che istruzioni. Il `;` indica che questa espressione
è finita, e la prossima sta per cominciare. La maggior parte delle righe
di codice Rust finiscono con un `;`.

[linguaggio orientato alle espressioni]:
glossary.html#expression-oriented-language

## Compilare ed eseguire sono passi separati

In "Scrivere ed eseguire un programma in Rust", abbiamo mostrato come eseguire
un programma appena creato. Adesso scomporremo quel procedimento e
ne esamineremo ogni passo.

Prima di eseguire un programma in Rust, bisogna compilarlo. Si può usare
il compilatore Rust digitando il comando `rustc` e passandogli il nome del file
sorgente, così:

```bash
$ rustc main.rs
```

Chi ha esperienza dei linguaggi C o C++, noterà che questo è simile all'uso di
`gcc` o `clang`. Dopo aver compilato con successo, Rust dovrebbe emettere un
programma eseguibile binario, che si può vedere su Linux o OSX digitando
il comando `ls` nel terminale come segue:

```bash
$ ls
main  main.rs
```

Su Windows, si dovrebbe digitare:

```bash
$ dir
main.exe
main.rs
```

Questo mostra che ci sono due file: il codice sorgente, con estensione `.rs`,
e l'eseguibile (`main.exe` su Windows, `main` altrove). Ciò che rimane da fare
è lanciare il file `main` o `main.exe`, così:

```bash
$ ./main  # o .\main.exe su Windows
```

Se *main.rs* fosse il nostro programma "Hello, world!", questo comando
stamperebbe `Hello, world!` sul nostro terminale.

Chi venisse da un linguaggio dinamico come Ruby, Python, o JavaScript, potrebbe
non essere abituato a compilare ed eseguire un programma in due passi separati.
Rust è un linguaggio *compilato in anticipo*, il che significa che si può
compilare un programma, darlo a qualcun altro, e questo lo può eseguire
anche se non ha Rust installato. Invece se si da a qualcuno un file `.rb`
o `.py` o `.js`, serve avere, rispettivamente, un'implementazione Ruby, Python,
o JavaScript installata, ma basta un solo comando per compilare ed eseguire
il programma. Ci sono pro e contro nella progettazione dei linguaggi.

Limitarsi a compilare con `rustc` va bene per programmi semplici, ma
nel momento in cui il progetto cresce, è utile poter gestire tutte le opzioni
del proprio progetto, e facilitare la condivisione del codice con altre persone
e altri progetti. Quindi, presenteremo uno strumento chiamato Cargo, che
aiuterà a scrivere programmi di dimensioni realistiche in Rust.

# Hello, Cargo!

Cargo è il sistema di build e gestore di pacchetti di Rust, e i Rustacean usano
Cargo per gestire i loro progetti in Rust. Cargo si occupa di tre attività:
compilare il proprio codice, scaricare le librerie che servono
al proprio codice, e costruire quelle librerie.
Chiameremo le librerie che servono al proprio codice ‘dipendenze’ dato che
il proprio codice dipende da loro.

I più semplici programmi in Rust non hanno dipendenze, e quindi per adesso
usiamo solamente la prima parte delle sue funzionalità. Man mano che scriveremo
programmi in Rust più complessi, si vorranno aggiungere dipendenze, e se si
comincia da subito ad usare Cargo, il compito sarà molto più facile.

Dato che la vastissima maggioranza di progetti in Rust usa Cargo, assumeremo
che sia usato per il resto di questo libro. Cargo viene installato insieme
a Rust, se vengono usati gli installatori ufficiali. Se Rust è stato installato
tramite qualche altro mezzo, si può verificare se Cargo è installato
digitando in un terminale:

```bash
$ cargo --version
```

Se si vede un numero di versione, ottimo! Se si vede un errore come
‘`command not found`’, allora si consiglia di guardare la documentazione
del sistema in cui è installato Rust, per determinare se Cargo deve essere
installato separatamente.

## Convertire a Cargo

Convertiamo in Cargo il programma Hello World. Per farlo, si devono fare
tre cose:

1. Mettere il file sorgente nella giusta directory.
1. Sbarazzarsi del vecchio eseguibile (`main.exe` su Windows, `main` altrove).
1. Creare un file di configurazione di Cargo.

Partiamo!

### Creare una directory del sorgente e rimuovere il vecchio eseguibile

Prima, torniamo al terminale, spostiamoci nella nostra directory *hello_world*,
e digitiamo i seguenti comandi:

```bash
$ mkdir src
$ mv main.rs src/main.rs # o 'move main.rs src\main.rs' su Windows
$ rm main  # o 'del main.exe' su Windows
```

Cargo si aspetta che i file sorgente risiedano dentro la directory *src*, e
quindi prima ce lo mettiamo. Questo lascia la directory principale del progetto
(in questo caso, *hello_world*) per i README, le informazioni di licenza, e
ogni altra cosa non correlata al codice. In questo modo, usare Cargo aiuta
a mantenere ordinati i propri progetti. C'è un posto per ogni cosa, e ogni cosa
sta al suo posto.

Adesso, si sposti *main.rs* nella directory *src*, e si elimini il file
compilato che abbiamo creato con `rustc`. In ambiente Windows, come al solito,
si sostituisca `main` con `main.exe`.

Questo esempio mantiene `main.rs` come nome del file sorgente, perché stiamo
creando un programma eseguibile. Se volessimo creare una libreria invece,
chiameremmo il file sorgente `lib.rs`. Questa convenzione è usata
da Cargo per compilare con successo il progetto, ma può essere modificata
se lo si desidera.

### Creare un file di configurazione

Poi, creiamo un nuovo file nella nostra directory *hello_world*, e lo chiamiamo
`Cargo.toml`.

Assicuriamoci di scrivere in maiuscolo la `C` in `Cargo.toml`, altrimenti
Cargo non saprà che farsene del file di configurazione.

Questo file è nel formato *[TOML]* (Tom's Obvious Minimal Language).
TOML è simile al formato INI, ma ha alcuni vantaggi, e viene usato
come formato di configurazione di Cargo.

[TOML]: https://github.com/toml-lang/toml

Dentro questo file, si digiti la seguente informazione:

```toml
[package]

name = "hello_world"
version = "0.0.1"
authors = [ "Il tuo nome <tu@esempio.it>" ]
```

La prima riga, `[package]`, indica che le direttive seguenti stanno
configurando un pacchetto. Man mano che aggiungiamo altre informazioni
a questo file, aggiungeremo altre sezioni, ma per adesso, abbiamo solamente
la configurazione del pacchetto.

Le altre tre righe impostano le tre informazioni che servono a Cargo
per compilare il nostro programma: il suo nome, a quale versione è, e chi
l'ha scritto.

Dopo aver aggiunto queste informazioni al file *Cargo.toml*, lo si salvi per
completare la creazione del file di configurazione.

## Costruire ed eseguire un progetto Cargo

Con il nostro file *Cargo.toml* al suo posto nella directory principale
del nostro progetto, siamo pronti a costruire ed eseguire
il programma Hello World!
Per farlo, si digitino i seguenti comandi:

```bash
$ cargo build
   Compiling hello_world v0.0.1 (file:///home/iltuonome/progetti/hello_world)
$ ./target/debug/hello_world
Hello, world!
```

Bam! Se tutto va bene, `Hello, world!` dovrebbe apparire sul terminale ancora
una volta.

Abbiamo appena costruito un progetto con il comando `cargo build` e l'abbiamo
eseguito con il comando `./target/debug/hello_world`, ma in realtà si possono
eseguire entrambi con il solo comando `cargo run`:

```bash
$ cargo run
     Running `target/debug/hello_world`
Hello, world!
```

Si noti che questo esempio non ha ricostruito il progetto. Cargo si è accorto
che il file sorgente non è cambiato, e perciò ha solamente lanciato
l'eseguibile. Se avessimo modificato il nostro file codice sorgente, Cargo avrebbe
ricostruito il progetto prima di eseguirlo, e avremmo visto qualcosa come:

```bash
$ cargo run
   Compiling hello_world v0.0.1 (file:///home/iltuonome/progetti/hello_world)
     Running `target/debug/hello_world`
Hello, world!
```

Cargo verifica se i file del progetto sono stati modificati, e
ricostruisce il progetto solamente se sono cambiati dall'ultima volta
che il progetto è stato costruito.

Con progetti semplici, Cargo non porta molti vantaggi rispetto ad usare
`rustc`, ma diventerà utile in futuro. Questo è vero specialmente
quando si iniziano ad usare i "crate"; questi, in altri linguaggi
di programmazione, sono chiamati ‘librerie’ o ‘pacchetti’.
Per progetti complessi, composti da più crate, è
molto più facile lasciare che Cargo coordini la costruzione. Usando Cargo,
si può lanciare `cargo build`, e tutto dovrebbe funzionare nel modo giusto.

### Costruire per il rilascio

Quando il proprio progetto è pronto per il rilascio, si può eseguire
`cargo build --release` per compilare il progetto con ottimizzazioni.
Queste ottimizzazioni velocizzano l'esecuzione del codice Rust, ma attivandole
ci vuole più tempo per compilare il programma. Ecco perché ci sono due diversi
profili, uno per lo sviluppo, e uno per costruire il programma finale
che verrà distribuito agli utenti.

### Cos'è quel `Cargo.lock`?

Eseguendo `cargo build` si fa anche in modo che Cargo crei un nuovo file
chiamato *Cargo.lock*, simile a questo:

```toml
[root]
name = "hello_world"
version = "0.0.1"
```

Cargo usa il file *Cargo.lock* per tener traccia delle dipendenze della nostra
applicazione. Questo è il file *Cargo.lock* del progetto Hello World. Questo
progetto non ha dipendenze, e quindi il contenuto del file è scarso.
Realisticamente, non ci sarà mai bisogno di toccare questo file;
lo si lascia gestire a Cargo.

Tutto qui! Dopo aver seguito tutti i passi, dovremmo aver costruito
con successo `hello_world` usando Cargo.

Anche se il progetto è semplice, usa gran parte degli strumenti che useremo
per il resto della nostra carriera come programmatore Rust.
Di fatto, ci si può aspettare di iniziare praticamente tutti
i progetti Rust con qualche variazione dei seguenti comandi:

```bash
$ git clone qualcheurl.com/foo
$ cd foo
$ cargo build
```

## Creare un nuovo progetto Cargo nel modo facile

Ma non si deve seguire quel procedimento precedente ogni volta che si vuole
iniziare un nuovo progetto! Cargo può creare rapidamente uno scheletro
di directory di progetto in cui si può iniziare a sviluppare subito.

Per iniziare un nuovo progetto usando Cargo, si digiti `cargo new` alla riga
di comando:

```bash
$ cargo new hello_world --bin
```

Questo comando passa l'opzione `--bin` perché l'obiettivo è di creare
direttamente un eseguibile, e non una libreria. I file eseguibili
sono spesso chiamati *binari* (come in `/usr/bin`, su sistemi Unix).

Cargo ha generato due file e una directory per noi: il file `Cargo.toml` e
la directory *src* con il file *main.rs* al suo interno. Questi dovrebbero
essere familiari, dato che sono esattamente quelli che prima abbiamo creato
manualmente.

Questo output è ciò che basta per iniziare. Prima si apre il file `Cargo.toml`.
Il suo contenuto dovrebbe essere simile a questo:

```toml
[package]

name = "hello_world"
version = "0.1.0"
authors = ["Il tuo nome <tu@esempio.it>"]

[dependencies]
```

Non ci si deve preoccupare della riga `[dependencies]`, ne parleremo dopo.

Cargo ha inserito in *Cargo.toml* dei valori di default ragionevoli,
che si basano sugli argomenti che gli abbiamo passato
e sulla configurazione globale di `git`.
Si potrà notare che Cargo ha anche inizializzato la directory `hello_world`
come un repository `git`.

Ecco cosa ci dovrebbe essere in `src/main.rs`:

```rust
fn main() {
    println!("Hello, world!");
}
```

Cargo ha generato un "Hello World!" per noi, e siamo pronti a iniziare
a programmare!

> Nota: Se si vuole esaminare Cargo in maggior dettaglio, si consulti
> la [guida ufficiale di Cargo], che tratta tutte le sue funzionalità.

[guida ufficiale Cargo]: http://doc.crates.io/guide.html

# Pensieri finali

Questo capitolo ha trattato le basi che serviranno per il resto del libro, e
il resto del tempo dedicato a Rust. Adesso che abbiamo visto gli strumenti,
passeremo a trattare il linguaggio Rust stesso.

Ci sono due strade: Tuffarsi in un progetto con
il ‘[Tutorial: gioco-indovina][guessinggame]’, o iniziare dal fondo
e poi risalire, con ‘[Sintassi e semantica][sintassi]’.
I programmatori di sistemi più esperti probabilmente preferiranno
‘Tutorial: gioco-indovina’, mentre quelli esperti di linguaggi dinamici
potrebbero preferire l'altro. Ognuno impara in modo diverso!
Si scelga quello che si preferisce.

[guessinggame]: guessing-game.html
[sintassi]: syntax-and-semantics.html
