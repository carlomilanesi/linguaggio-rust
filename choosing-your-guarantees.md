% Scegliere le proprie garanzie

Una caratteristica importante di Rust è che permette di controllare i costi
e le garanzie del proprio programma.

Ci sono varie astrazioni "di tipo wrapper" nella libreria standard di Rust
che impersonano una moltitudine di pro e contro riguardo il costo, l'ergonomia,
e le garanzie. Molte permettono di scegliere se applicare le garanzie in fase
di compilazione o in fase di esecuzione. Questa sezione spiega in dettaglio
alcune astrazioni selezionate.

Prima di procedere, è fortemente consigliato aver letto le sezioni
sul [possesso][ownership] e sui [prestiti][borrowing] in Rust.

[ownership]: ownership.html
[borrowing]: references-and-borrowing.html

# I tipi puntatori di base

## `Box<T>`

[`Box<T>`][box] è un puntatore "posseduto", ossia un "box". Mentre può fornire
riferimenti ai dati contenuti, è l'unico possessore dei dati. In particolare,
di consideri questo:

```rust
let x = Box::new(1);
let y = x;
// qui x non è più accessibile
```

Qui, il box è stato _spostato_ in `y`. Siccome `x` non lo possiede più,
dopo di ciò il compilatore non consentirà più al programmatore di usare `x`.
Analogamente, un box può essere spostato _fuori_ da una funzione
restituendolo.

Quando un box (che non è stato spostato) esce di ambito, vengono eseguiti
i distruttori. Questi distruttori hanno cura di deallocare i dati interni.

Questa è un'astrazione a costo zero per l'allocazione dinamica.
se si vuole allocare della memoria dallo heap e mandare in giro in sicurezza
un puntatore a tale memoria, "box" è l'ideale. Si noti che verrà consentito
condividere riferimenti a questo oggetto solamente tramite le normali
regole di prestito, verificate in fase di compilazione.

[box]: ../std/boxed/struct.Box.html

## `&T` e `&mut T`

Questi sono, rispettivamente, un riferimento immutabile e
un riferimento mutabile. Seguono il pattern "lock di lettura-scrittura",
tale che si può o avere un solo riferimento mutabile ad alcuni dati,
o qualunque numero di riferimenti immutabile, ma non entrambi. Questa garanzia
è applicata in fase di compilazione, e non ha costi visibili
in fase di esecuzione. Nella maggior parte dei casi, questi due tipi
di puntatori bastando per condividere riferimenti poco costosi fra sezioni
di codice.

Questi puntatori non possono essere copiati in modo tale da sopravvivere
il tempo di vita associato ad essi.

## `*const T` e `*mut T`

Questi sono puntatori grezzi tipo-C, senza tempo di vita né possesso allegati.
Puntano ad alcune posizioni in memoria senza altre restrizioni. L'unica
garanzia che forniscono è che non possono essere dereferenziati eccetto
nel codice marcato `unsafe`.

Servono per costruire astrazioni sicure e a basso costo, come `Vec<T>`,
ma dovrebbero essere evitati nel codice sicuro.

## `Rc<T>`

Questo è il primo wrapper che tratteremo che ha un costo in fase di esecuzione.

[`Rc<T>`][rc] è un puntatore a conteggio di riferimenti. In altri termini,
consente di avere più puntatori "possessori" allo stesso dato, e il dato
verrà rilasciato (con l'esecuzione dei distruttori) quando tutti i puntatori
usciranno di ambito.

Internamente, contiene una "conteggio di riferimenti" condiviso (chiamato
anche "refcount"), che viene incrementato ogni volta che l'`Rc` viene clonato,
e decrementato ogni volta che uno degli `Rc` esce di ambito. La responsabilità
principale di `Rc<T>` è assicurarsi che siano chiamati i distruttori per
il dato condiviso.

Qui i dati interni sono immutabili, e se viene creato un ciclo
di riferimenti, i dati rimarranno sempre allocati ("leak").
Se vogliamo dei dati che potranno essere deallocati anche in presenza
di strutture cicliche, ci serve un garbage collector.

### Garanzie

La garanzia principale fornita qui è che i dati non saranno distrutti
finché tutti i riferimenti ad essi escano di ambito.

Questo si dovrebbe essere usato quando desideriamo allocare dinamicamente
e condividere dei dati (a sola lettura) fra varie porzioni del programma,
nel quale non è certo quale porzione finirà per ultima di usare tali dati.
È un'alternativa praticable a `&T`, quando o è impossibile verificare
staticamente la correttezza di `&T`, o usando `&T` si crea del codice
estremamente non ergonomico, per il quale non vale la pena investire
il tempo del programmatore.

Questo puntatore _non_ è sicuro per l'uso coi thread, e Rust non lo lascerà
inviare o condividere con altri thread. Ciò consente di evitare il costo
delle operazioni atomiche dove non sono necessarie.

C'è uno smart pointer fratello di questo, `Weak<T>`. Questo è
uno smart pointer che non possiede, ma non è neanche preso in prestito.
Anch'esso è simile a `&T`, ma non ha un tempo di vita limitato: un `Weak<T>`
può essere tenuto per sempre. Però, è possibile che un tentativo di accedere
al dato interno fallisca e restituisca `None`, dato che può sopravvivere
gli `Rc` che possiedono il dato. Ciò è utile, tra l'altro, per creare
strutture dati cicliche.

### Costo

Per quanto riguarda la memoria, `Rc<T>` è un'allocazione singola,
però allocherà due word in più (cioè due valori `usize`) rispetto
a un normale `Box<T>` (questo vale anche per i `Weak`).

`Rc<T>` ha il costo computazionale di incrementare/decrementare il conteggio,
rispettivamente ogni volta che viene clonato o che esce di ambito. Si noti
che un `clone` non farà una copia profonda, ma incrementerà semplicemente
il conteggio interno di riferimenti e restituirà una copia del `Rc<T>`.

[rc]: ../std/rc/struct.Rc.html

# I tipi Cell

I `Cell` forniscono la mutabilità interna. In altri termini, contengono
dei dati che possono essere manipolati anche se il tipo non può
essere ottenuto in una forma mutabile (per esempio, quando è dietro
un puntatore `&`, oppure un `Rc<T>`).

[La documentazione del modulo `cell` li spiega molto bene][cell-mod].

Questi tipo _solitamente_ si trovano nei campi di struct, ma si possono
trovare anche altrove.

## `Cell<T>`

[`Cell<T>`][cell] è un tipo che fornisce mutabilità interna a costo zero,
ma solamente per tipi `Copy`. Dato che il compilatore sa che tutti i dati
posseduto dal valore contenuto stanno sono sullo stack, non c'è il rischio
che la semplice sovrascrittura dei dati comporti mancate disallocazioni
di riferimenti (o peggio!).

Usando questo wrapper, è ancora possibile violare le proprie invarianti,
e quindi bisogna essere cauti nell'usarlo. Se un campo è contenuto
in un `Cell`, è una chiara indicazione che quel dato è mutabile e potrebbe
non rimanere invariato tra quando lo si legge e quando lo si vuole usare.

```rust
use std::cell::Cell;

let x = Cell::new(1);
let y = &x;
let z = &x;
x.set(2);
y.set(3);
z.set(4);
println!("{}", x.get());
```

Si noti che qui abbiamo potuto mutare il medesimo oggetto da vari
riferimenti immutabili.

Questo ha lo stesso costo in fase di esecuzione del seguente:

```rust,ignore
let mut x = 1;
let y = &mut x;
let z = &mut x;
x = 2;
*y = 3;
*z = 4;
println!("{}", x);
```

ma ha il beneficio non trascurabile di poter essere compilato.

### Garanzie

Questo allenta la restrizione del "nessun alias con la mutabilità" dove
non è necessaria. Però, questo allenta che le garanzie fornite
da tale restrizione; quindi se i propri invarianti dipendono da dati
memorizzati in un `Cell`, si dovrebbe essere cauti.

Ciò è utile per tipi primitivi mutabili e altri tipi `Copy`, quando
non ci sono modi facili di farlo conformemente alle regole statiche
di `&` e di `&mut`.

`Cell` non consente di ottenere riferimenti al dato interno, il che rende
sicuro mutarlo liberamente.

#### Costo

Non ci sono costi in fase di esecuzione a usare `Cell<T>`, però se lo
si sta usando per avvolgere struct (`Copy`) più grandi, potrebbe invece
essere opportuno avvolgere i singoli campi in `Cell<T>` dato che altrimenti
ogni scrittura copia l'intera struct.

## `RefCell<T>`

Anche [`RefCell<T>`][refcell] fornisce mutabilità interna, ma non è limitata
ai tipi `Copy`.

In compenso, ha un costo in fase di esecuzione. `RefCell<T>` impone in fase
di esecuzione il pattern del lock di lettura-scrittura (è come un mutex
a sngolo thread), diversamente da `&T`/`&mut T` che lo fanno in fase
di compilazione. Ciò viene fatto dalle funzioni `borrow()` e `borrow_mut()`,
che modificano un conteggio di riferimenti interno e restituiscono degli
smart pointers che possono essere dereferenziati, rispettivamente in modo
immutabile e mutabile. Il refcount viene ripristinato quando
gli smart pointer escono di ambito. Con questo sistema, possiamo assicurare
dinamicamente che non ci sono mai altri prestiti attivi quando un prestito
mutabile è attivo. Se il programmatore tenta di eseguire un tale prestito,
il thread andrà in panico.

```rust
use std::cell::RefCell;

let x = RefCell::new(vec![1,2,3,4]);
{
    println!("{:?}", *x.borrow())
}

{
    let mut mio_riferimento = x.borrow_mut();
    mio_riferimento.push(1);
}
```

Simile a `Cell`, serve principalmente in situazioni in cui è difficile
o impossibile soddisfare il verificatore dei prestiti. In generale sappiamo
che tali mutazioni non avverranno in una forma annidata, ma è bene verificare.

Per programmi grandi e complicati, diventa utile mettere alcuni oggetti
in `RefCell` per semplificare le cose. Per esempio, molte delle mappe
nella struct `ctxt` interna al compilatore Rust sono poste dentro
questo wrapper. Esse vengono modificate o una sola volta (durante
la creazione, che non è subito dopo l'inizializzazione) o un paio di volte
in luoghi bene separati. Però, siccome questa struct è usata dappertutto,
sarebbe difficile (e forse impossibile) destreggiarsi con puntatori mutabili
o immutabili, e probabilmente formare una zuppa di puntatori `&` che
poi sarebbe difficile estendere. D'altra parte, `RefCell` fornisce un modo
a basso costo di accedere con sicurezza a queste strutture. In futuro, se
qualcuno aggiungesse del codice che tenta di modificare la cella quando è
già stata prestata, questo provocherà un panico (solitamente deterministico)
che può esser fatto risalire al prestito erroneo.

Similmente, nel DOM di Servo ci sono molte mutazioni, per lo più locali
a un tipo DOM, ma alcune delle quali incrociano il DOM e modificano
varie cose. Usare `RefCell` e `Cell` per proteggere tutte le mutazioni
consente di evitare di preoccuparsi ovunque della mutabilità,
e simultaneamente evidenzia i posti dove la mutazione
sta _effettivamente_ avvenendo.

Si noti che `RefCell` dovrebbe essere evitato se è possibile
una soluzione più semplice utilizzando i puntatori `&`.

### Garanzie

`RefCell` allenta le restrizioni _statiche_ che impediscono le mutazioni
tramite alias, e le sostituisce con restrizioni _dinamiche_.
Però tali garanzie non sono cambiate.

#### Costo

`RefCell` non alloca, ma contiene, a fianco del dato, un indicatore
aggiuntivo di "stato di prestito" (grande una word).

In fase di esecuzione, ogni prestito provoca una modifica/verifica
di tale indicatore.

[cell-mod]: ../std/cell/index.html
[cell]: ../std/cell/struct.Cell.html
[refcell]: ../std/cell/struct.RefCell.html

# Tipi sincroni

Molti dei tipi di cui si è parlato non possono essere usati in modo sicuro
per l'uso coi thread. In particolare, `Rc<T>` e `RefCell<T>`, entrambi
i quali usano conteggi di riferimenti non atomici (i conteggi
di riferimenti _atomici_ sono quelli che possono essere incrementati da più
thread senza provocare una corsa ai dati), non si possono usare
in questo modo. Ciò li rende più efficienti da usare, ma ci servono anche
delle versioni sicure per l'uso coi thread. Esistono, sotto forma di `Arc<T>`
e di `Mutex<T>`/`RwLock<T>`

Si noti che i tipi non sicuri per l'uso coi thread _non possono_ essere
scambiati tra thread, e ciò viene verificato in fase di compilazione.

Nel modulo [sync][sync] ci sono molti utili wrapper per la programmazione
concorrente, ma qui sotto verranno trattati solo i principali.

[sync]: ../std/sync/index.html

## `Arc<T>`

[`Arc<T>`][arc] è una versione di `Rc<T>` che usa un conteggio di riferimenti
atomico (da cui il nome, "Arc" = "Atomic Reference Count").
Può essere scambiato liberamente fra thread.

Il tipo `shared_ptr` del linguaggio C++ è simile ad `Arc`, però nel caso
di C++ il dato interno è sempre mutabile. Per avere una semantica simile
a quella di `shared_ptr`, si dovrebbero usare `Arc<Mutex<T>>`,
`Arc<RwLock<T>>`, o `Arc<UnsafeCell<T>>`[^4] (`UnsafeCell<T>` è un tipo
di cella che può essere usato per tenere qualunque dato e non ha costi
in fase di esecuzione, ma è accessibile solamente da blocchi `unsafe`).
L'ultimo dovrebbe essere usato solamente se si è certi che l'utilizzo non
provocherà insicurezza nella gestione della memoria. Ricordiamo che
scrivere una struttura non è un'operazione atomica, e moste funzioni
come `vec.push()` possono riallocare internamente, e provocare
un comportamento insicuro, quindi perfino la monotonicità potrebbe
non bastare per giustificare `UnsafeCell`.

[^4]: `Arc<UnsafeCell<T>>` effettivamente non compilerà, siccome
`UnsafeCell<T>` non è `Send` né `Sync`, ma possiamo avvolgerlo in un tipo
e implementare manualmente `Send`/`Sync` per esso, così da ottenere
`Arc<Wrapper<T>>`, dove `Wrapper` è `struct Wrapper<T>(UnsafeCell<T>)`.

### Garanzie

Come `Rc`, anche questo tipo fornisce la garanzia (sicura per l'uso
coi thread) che il distruttore per i dati interni verrà eseguito quando
l'ultimo `Arc` esce di ambito (eccetto in presenza di strutture cicliche).

### Costo

Questo tipo ha il costo aggiuntivo di usare operazioni atomiche per modificare
il refcount (che accadrà ogni volta che è clonato o esce di ambito). Quando
si condividono dati da un `Arc` in un solo thread, è preferibile condividere
puntatori `&`, quando è possibile.

[arc]: ../std/sync/struct.Arc.html

## `Mutex<T>` e `RwLock<T>`

[`Mutex<T>`][mutex] e [`RwLock<T>`][rwlock] forniscono la mutua-esclusione
tramite guardie RAII (le guardie sono oggetti che mantengono un certo stato,
come un lock, fino a quando è chiamato il loro distruttore). Per entrambe,
il mutex è opaco finché chiamiamo `lock()` su di esso. A quel punto il thread
si bloccherà fino a quando si potrà acquisire un lock, e poi verrà
restituita un guardia. Questa guardia può essere usata per accedere
(mutabilmente) al dato interno, e il lock verrà rilasciato quando la guardia
esce di ambito.

```rust,ignore
{
    let guardia = mutex.lock();
    // guardia dereferenzia mutabilmente dando il tipo interno
    *guardia += 1;
} // lock rilasciato quando si esegue il distruttore
```

`RwLock` ha il beneficio aggiuntivo di essere efficiente per più letture.
È sempre sicuro avere più lettori a dati condivisi purché non ci
siano scrittori; e `RwLock` consente ai lettori di acquisire
un "lock di lettura". Tali lock possono essere acquisiti concorrentemente e
se ne tiene traccia tramite un conteggio di riferimenti.
Gli scrittori devono ottenere un "lock di scrittura" che può essere ottenuto
solamente quando tutti i lettori sono usciti di scope.

### Garanzie

Entrambi questi tipi forniscono mutabilità sicura condivisa fra thread,
però sono soggetti a deadlock. Qualche livello aggiuntivo di sicurezza
del protocollo può essere ottenuto tramite il sistema dei tipi.

### Costi

Questi tipi usano tipi interni di tipo atomico per mantenere i lock,
i quali sono parecchio costosi (possono bloccare tutte le letture in memoria
per tutti i processori fino a quando hanno finito). Anche attendere
questi lock può essere lento quando avvengono molti accessi concorrenti.

[rwlock]: ../std/sync/struct.RwLock.html
[mutex]: ../std/sync/struct.Mutex.html
[sessions]: https://github.com/Munksgaard/rust-sessions

# Composizione

Una tipica lamentela quando si legge del codice Rust è per i tipi come
`Rc<RefCell<Vec<T>>>` (o composizioni ancora più complicate di tali tipi).
Non è sempre chiaro che cosa faccia la composizione, o perché l'autore ne
ha scelta una così (e quando si dovrebbe usare tale composizione
nel proprio codice).

Solitamente, si tratta comporre insieme le garanzie che servono,
senza pagare per quello che non serve.

Per esempio, `Rc<RefCell<T>>` è una tale composizione. `Rc<T>` stesso
non può essere dereferenziato mutabilmente; siccome `Rc<T>` fornisce
la condivisione, e la mutabilità condivisa può condurre a comportamento
insicuro, mettiamo dentro `RefCell<T>` per ottenere mutabilità condivisa
verificata dinamicamente. Adesso abbiamo un dato mutable condiviso,
ma è condiviso in un modo che ci può essere un solo scrittore
(e nessun lettore), oppure più lettori.

Adesso, possiamo fare un passo avanti, e abbiamo `Rc<RefCell<Vec<T>>>` oppure
`Rc<Vec<RefCell<T>>>`. Questi sono entrambi vettori condivisibili e mtabili,
ma non sono la medesima cosa.

Con il primo, il `RefCell<T>` avvolge il `Vec<T>`, e così `Vec<T>`
nella sua interezza è mutabile. Al medesimo tempo, ci può essere
un solo prestito mutabile per volta dell'intero `Vec`.
Ciò comporta che il nostro codice nn può funzionare simultaneamente
su elementi distinti del vettore da diversi handle `Rc`. Però, possiamo
eseguire `push` e `pop` a volontà con il `Vec<T>`. Ciò è simile
a un `&mut Vec<T>` con i prestiti verificati in fase di esecuzione.

Con l'ultimo, il prestito è di elementi individuali, ma il vettore
complessivo è immutabile. Così, possiamo prendere in prestito
indipendentemente elementi distinti, ma non possiamo eseguire `push` né `pop`
con il vettore. Ciò  è simile a un `&mut [T]`[^3], ma, anche qui,
i prestiti sono verificati in fase di esecuzione.

Nei programmi concorrenti, abbiamo una situazione simile con `Arc<Mutex<T>>`,
che fornisce mutabilità e possesso condivisi.

Quando si legge del codice che li usa, si proceda passo per passo,
e si guardi alle garanzie e ai costi forniti.

Quando si sceglie un tipo composito, dobbiamo fare il contrario; decidere
quali garanzie vogliamo, e a quale punto della composizione ci servono.
Per esempio, se c'è una scelta fra `Vec<RefCell<T>>` e `RefCell<Vec<T>>`,
dovremmo decidere i pro e i contro come fatto prima, e sceglierne uno.

[^3]: `&[T]` e `&mut [T]` sono delle _slice_; consistono di un puntatore
e una lunghezza, e possono far riferimento a una porzione di un vettore
o di un array. `&mut [T]` può avere i suoi elementi mutati, però
la sua lunghezza con può essere toccata.
