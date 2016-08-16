% Concorrenza

La concorrenza e il parallelismo sono argomenti incredibilmente importanti
in informatica teorica, e oggi sono anche un argomento attuale
nella tecnologia informatica. I computer hanno sempre più core, però molti
programmatori non sono in grado di utilizzarli pienamente.

La sicurezza nella gestione della memoria da parte di Rust si applica anche
ai problemi di concorrenza. Infatti, anche i programmi concorrenti scritti
in Rust devono garantire un accesso sicura alla memoria, evitando
le collisioni nell'accesso ai dati ["data race"]. Il sistema dei tipi di Rust
è all'altezza del compito, fornendo strumenti potenti per gestire il codice
concorrente in fase di compilazione.

Prima di parlare delle caratteristiche di concorrenza fornite da Rust, è
importante capire qualcosa: Rust è abbastanza a basso livello che la grande
maggioranza di cioò è fornita dalla libreria standard, non dal linguaggio.
Ciò significa che se non piace qualche aspetto del modo in cui Rust tratta
la concorrenza, si può implementare un altro modo di fare le cose.
[mio](https://github.com/carllerche/mio) è un esempio del mondo reale
di questo principio in azione.

## Background: `Send` e `Sync`

È difficile ragionare sulla concorrenza. In Rust, c'è un sistema dei tipi
forte, statico, che aiuta a ragionare sul proprio codice. Come tale, Rust
fornisce due tratti per aiutare a dar senso al codice che eventualmente può
diventare concorrente.

### `Send`

Il primo trattl di cui parleremo è [`Send`](../std/marker/trait.Send.html).
Quando un tipo `T` implementa `Send`, indicata che qualcosa di questo tipo
può avere la sua proprietà trasferita con sicurezza da un thread a un'altro.

Questo è importante per imporre certe restrizioni. Per esempio, se abbiamo
un canale che connette due thread, vorremmo poter mandare dei dati lungo
il canale fino all'altro thread. Perciò, ci assicureremmo che `Send`
fosse implementato per tale tipo.

All'opposto, se stessimo avvolgendo una libreria che usa una [FFI][ffi] che
non è sicuro per i thread, non dovremmo implementare `Send`, e quindi
il compilatore ci aiuterà a impedire che possa uscire dal thread corrente.

[ffi]: ffi.html

### `Sync`

Il secondo di questi tratti è chiamato [`Sync`](../std/marker/trait.Sync.html).
Quando un tipo `T` implementa `Sync`, indicata che qualcosa di questo tipo
non ha possibilità di introdurre insicurezze di memoria quando sia usato
da più thread concorrentemente tramite riferimenti condivisi. Ciò implica che
i tipi che non hanno la [mutabilità interna](mutability.html) sono
inerentemente `Sync`, il che comprende i tipi primitivi semplici (come `u8`)
e i tipi aggregati che li contengono.

Per condividere i riferimenti tra i thread, Rust fornisce un tipo ausiliario
chiamato `Arc<T>`. `Arc<T>` implementa `Send` e `Sync` se e solamente se `T`
implementa sia `Send` che `Sync`. Per esempio, un oggetto di tipo
`Arc<RefCell<U>>` non può essere trasferito fra thread perché [`RefCell`]
(choosing-your-guarantees.html#refcellt) non implementa `Sync`, e
di conseguenza `Arc<RefCell<U>>` non implementerebbe `Send`.

Questi due tratti consentono di usare il sistema dei tipi per dare forti
garanzie sulle proprietà del codice concorrente. Prima di mostrare perché,
in primo luogo, dobbiamo vedere come creare un programma Rust concorrente!

## I thread

La libreria standard di Rust contiene una libreria per i thread, che consente
di eseguire del codice Rust in parallelo. Ecco un esempio di base di come
usare `std::thread`:

```rust
use std::thread;

fn main() {
    thread::spawn(|| {
        println!("Ciao da un thread!");
    });
}
```

Il metodo `thread::spawn()` accetta una [chiusura](closures.html), che viene
eseguita in un nuovo thread. Tale metodo rende un handle che rappresenta
il thread, e tale handle può servire ad aspettare che il thread figlio
finisca, per poi estrarne il risultato:

```rust
use std::thread;

fn main() {
    let handle = thread::spawn(|| {
        "Ciao da un thread!"
    });

    println!("{}", handle.join().unwrap());
}
```

Dato che le chiusure possono catturare le variabili dal loro ambiente,
possiamo anche provare a portare dei dati nell'altro thread:

```rust,ignore
use std::thread;

fn main() {
    let x = 1;
    thread::spawn(|| {
        println!("x è {}", x);
    });
}
```

Però, questo ci dà un errore:

```text
5:19: 7:6 error: closure may outlive the current function, but it
                 borrows `x`, which is owned by the current function
...
5:19: 7:6 help: to force the closure to take ownership of `x` (and any other referenced variables),
          use the `move` keyword, as shown:
      thread::spawn(move || {
          println!("x is {}", x);
      });
```

È così perché di default le chiusure catturano le variabili per riferimento, e
così la chiusura cattura solamente un _riferimento a `x`_. Questo è un difetto,
perché il thread può sopravvivere l'ambito di `x`, conducendo ad avere
un puntatore penzolante.

Per correggerlo, usiamo una chiusura `move` come suggerito nel messaggio
d'errore. Le chiusure `move` sono spiegate approfonditamente [qui]
(closures.html#move-closures); di base, spostano le variabili dal loro
ambiente in sé stesse.

```rust
use std::thread;

fn main() {
    let x = 1;
    thread::spawn(move || {
        println!("x è {}", x);
    });
}
```

Molti linguaggi hanno la capacità di eseguire dei thread, ma in modo follemente
insicuro. Ci sono interi libri su come prevenire gli errori dovuti allo stato
mutabile condiviso. Anche qui, Rust aiuta a uscirne con il suo sistema
dei tipi, prevenendo le corse ai dati in fase di compilazione. Parliamo di come
si condividono effettivamente gli oggetti fra i thread.

## Lo stato mutabile condiviso in modo sicuro

A causa del sistema dei tipi di Rust, abbiamo un concetto che suona come una
bugia: "stato mutabile condiviso in modo sicuro." Molti programmatori
concordano sul fatto che lo stato mutabile condiviso sia una pessima cosa.

Qualcuno una volta ha detto:

> Lo stato mutabile condiviso è la radice di tutto il male. La maggior parte
> dei linguaggi tentano di affrontare questo problema tramite il concetto di
> 'mutabile', mentre Rust lo affronta risolvendo il concetto di 'condiviso'.

Lo stesso [sistema di possesso](ownership.html) che aiuta a prevenire l'uso
scorretto dei puntatori aiuta anche a escludere le corse ai dati, una dei
peggiori generi di difetti di concorrenza.

Come esempio, ecco un programma Rust che avrebbe una corsa ai dati in molti
linguaggi. Non compilerà:

```rust,ignore
use std::thread;
use std::time::Duration;

fn main() {
    let mut dati = vec![1, 2, 3];

    for i in 0..3 {
        thread::spawn(move || {
            dati[0] += i;
        });
    }

    thread::sleep(Duration::from_millis(50));
}
```

Questo dà un errore:

```text
8:17 error: capture of moved value: `dati`
        dati[0] += i;
        ^~~~
```

Rust sa che non sarebbe sicuro! Se avessimo un riferimento a `dati` in ogni
thread, e i thread prendessero il possesso del riferimento, avremmo tre
possessori! `dati` viene spostato fuori da `main` nella prima chiamata
a `spawn()`, e così le chiamate successive nel ciclo non possono più usare
questa variabile.

Perciò, ci serve qualche tipo che ci consente di avere più di un riferimento
che possiede un valore. Solitamente, per questo useremmo `Rc<T>`, che è
un tipo a conteggio dei riferimenti che fornisce il possesso condiviso.
Tiene una certa contabilità in fase di esecuzione che tiene traccia
del numero di riferimenti a esso, e da ciò deriva il suo nome
"reference count", cioè "conteggio dei riferimenti".

Chiamando `clone()` su un oggetto di tipo `Rc<T>`, si riceverà un nuovo
riferimento posseduto e si toccherà il conteggio dei riferimenti interno.
Creiamo uno di questi per ogni thread:


```rust,ignore
use std::thread;
use std::time::Duration;
use std::rc::Rc;

fn main() {
    let mut dati = Rc::new(vec![1, 2, 3]);

    for i in 0..3 {
        // crea un nuovo riferimento posseduto
        let rif_a_dati = dati.clone();

        // usalo in un thread
        thread::spawn(move || {
            rif_a_dati[0] += i;
        });
    }

    thread::sleep(Duration::from_millis(50));
}
```

Però, neanche questo funzionerà, e ci darà l'errore:

```text
13:9: 13:22 error: the trait bound `alloc::rc::Rc<collections::vec::Vec<i32>> : core::marker::Send`
            is not satisfied
...
13:9: 13:22 note: `alloc::rc::Rc<collections::vec::Vec<i32>>`
            cannot be sent between threads safely
```

Come dice il messaggio d'errore, `Rc` non può essere mandato fra thread in modo
sicuro. È così perché il conteggio dei riferimenti interno non è gestito in
un modo sicuro per i thread, e può avere una corsa ai dati.

Per risolvere questo problema, useremo `Arc<T>`, il tipo standard di Rust
per il conteggio dei riferimenti atomico.

La parola "atomico" significa che `Arc<T>` può essere acceduto in modo sicuro
da più threads. Per farlo, il compilatore garantisce che le mutazioni
del contateggio interno usano operazioni indivisibili, le quali non possono
avere corse ai dati.

In essenza, `Arc<T>` è un tipo che ci consente di condividere il possesso
dei dati _attraverso i thread_.

```rust,ignore
use std::thread;
use std::sync::Arc;
use std::time::Duration;

fn main() {
    let mut dati = Arc::new(vec![1, 2, 3]);

    for i in 0..3 {
        let dati = dati.clone();
        thread::spawn(move || {
            dati[0] += i;
        });
    }

    thread::sleep(Duration::from_millis(50));
}
```

Analogamente all'ultima volta, usiamo `clone()` per creare un nuovo handle
posseduto. Questo handle viene poi spostato dentro il nuovo thread.

E... ci dà ancora un errore.

```text
<anon>:11:24 error: cannot borrow immutable borrowed content as mutable
<anon>:11                    dati[0] += i;
                             ^~~~
```

`Arc<T>` di default ha dei contenuti immutabili. Consente la _condivisione_
dei dati fra thread, ma i dati mutabili condivisi sono insicuri, e quando
sono implicati dei thread, provocano delle corse ai dati!

Solitamente quando vogliamo rendere mutabile qualcosa che è in una posizione
immutabile, usiamo `Cell<T>` o `RefCell<T>` che permettono la mutazione
sicura tramite delle verifiche in fase di esecuzione o in altro modo (si veda
anche: [Scegliere le proprie garanzie](choosing-your-guarantees.html)).
Però, come gli `Rc`, anche questi non sono sicuri per i thread. Se proviamo
a usarli, otterremo un errore sul fatto che questi tipi non sono `Sync`,
e il codice non riuscirà a essere compilato.

Pare che ci serva qualche tipo che ci consenta di mutare in modo sicuro
un valore condiviso tra thread; per esempio un tipo che possa assicurare che
solamente un thread per volta sia in grado di mutare il valore al suo interno
in qualunque momento.

A tale scopo, possiamo usare il tipo `Mutex<T>`!

Ecco la versione funzionante:

```rust
use std::sync::{Arc, Mutex};
use std::thread;
use std::time::Duration;

fn main() {
    let dati = Arc::new(Mutex::new(vec![1, 2, 3]));

    for i in 0..3 {
        let dati = dati.clone();
        thread::spawn(move || {
            let mut dati = dati.lock().unwrap();
            dati[0] += i;
        });
    }

    thread::sleep(Duration::from_millis(50));
}
```

Si noti che il valore di `i` è legato (copiato) alla chiusura e non condiviso
fra i thread.

Qui stiamo usando "lock" sul mutex. Un mutex (abbreviazione di "mutua
esclusione"), come detto, consente a un solo thread alla volta di accedere
a un valore. Quando desideriamo accedere al valore, usiamo `lock()` su di esso.
Useremo "lock" sul mutex, e nessun altro thread potrà farlo (e quindi, fare
qualcosa con il valore) finché non abbiamo finito di usarlo. Se un thread
tenta si usare lock su un mutex che è già bloccato da lock, aspetterà finché
l'altro thread rilasci il blocco lock.

Qui il "rilascio" del lock è implicito; quando il risultato del lock (in questo
caso, `dati`) esce di scope, il lock viene automaticamente rilasciato.

Si noti che il metodo [`lock`](../std/sync/struct.Mutex.html#method.lock) di
[`Mutex`](../std/sync/struct.Mutex.html) ha questa firma:

```rust,ignore
fn lock(&self) -> LockResult<MutexGuard<T>>
```

e siccome `Send` non è implementato per `MutexGuard<T>`, la guardia non può
attraversare il confine del thread, assicurando località al thread delle
operazione di acquisizione e rilascio del lock.

Esaminiamo più da vicino il corpo del thread:

```rust
# use std::sync::{Arc, Mutex};
# use std::thread;
# use std::time::Duration;
# fn main() {
#     let dati = Arc::new(Mutex::new(vec![1, 2, 3]));
#     for i in 0..3 {
#         let dati = dati.clone();
thread::spawn(move || {
    let mut dati = dati.lock().unwrap();
    dati[0] += i;
});
#     }
#     thread::sleep(Duration::from_millis(50));
# }
```

Prima, chiamiamo `lock()`, che acquisisce il lock del mutex. Siccome questo
può fallire, rende un `Result<T, E>`, e siccome questo è appena un esempio,
eseguiamo `unwrap()` per ottenere un riferimento ai dati. Qui del codice reale
avrebbe una gestione degli errori più robusta. Poi siamo liberi di mutarlo,
dato che abbiamo il lock.

Infine, mentre i thread secondari sono in esecuzione, il thread principale
aspetta per 50 millisecondi. Ma questo non è certo l'ideale: possiamo aver
scelto di aspettare una quantità di tempo ragionevole, ma è più probabile che,
o aspetteremo più a lungo del necessario, o non abbastanza a lungo, a seconda
di quanto tempo richiedono effettivamente i thread per finire i loro calcoli.

Un'alternativa più precisa all'attesa temporizzata sarebbe usare uno
dei meccanismi forniti dalla libreria standard di Rust per sincronizzare
i thread tra di loro. Parlaimo di un di essi: i canali.

## I canali

Ecco una versione del nostro codice che usa i canali per la sincronizzazione,
invece di aspettare un tempo specifico:

```rust
use std::sync::{Arc, Mutex};
use std::thread;
use std::sync::mpsc;

fn main() {
    let dati = Arc::new(Mutex::new(0));

    // `tx` è il "trasmettitore" o "mittente"
    // `rx` è il "ricevitore"
    let (tx, rx) = mpsc::channel();

    for _ in 0..10 {
        let (dati, tx) = (dati.clone(), tx.clone());

        thread::spawn(move || {
            let mut dati = dati.lock().unwrap();
            *dati += 1;

            tx.send(()).unwrap();
        });
    }

    for _ in 0..10 {
        rx.recv().unwrap();
    }
}
```

Usiamo il metodo `mpsc::channel()` per costruire un nuovo canale.
Usiamo `send` per inviare un semplice `()` lungo il canale,
e poi aspettiamo che ne ritornino dieci.

Mentre adesso questo canale sta trasmettendo un segnale generico, lungo
il canale possiamo inviare qualunque dato che è `Send`!

```rust
use std::thread;
use std::sync::mpsc;

fn main() {
    let (tx, rx) = mpsc::channel();

    for i in 0..10 {
        let tx = tx.clone();

        thread::spawn(move || {
            let answer = i * i;

            tx.send(answer).unwrap();
        });
    }

    for _ in 0..10 {
        println!("{}", rx.recv().unwrap());
    }
}
```

Qui creiamo 10 thread, chiedendo a ognuno di calcolare il quadrato di
un numero (`i` al momento della `spawn()`), e poi facciamo ritornare
la risposta lungo il canale usando `send()`.

## Panico

Una chiamata a `panic!` manderà in crash il thread attualmente in esecuzione.
I thread di Rust si possono usare come semplice meccanismo di isolamento:

```rust
use std::thread;

let handle = thread::spawn(move || {
    panic!("oops!");
});

let result = handle.join();

assert!(result.is_err());
```

`Thread.join()` ci rende un `Result`, che ci consente di verificare se
il thread ha avuto un panico o no.
