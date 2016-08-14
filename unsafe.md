% Unsafe

La principale attrattiva di Rust è costituita dalle potenti garanzie statiche
sul suo comportamento. Ma le verifiche di sicurezza sono prudenti per natura:
ci sono alcuni programmi che sono effettivamente sicuri, ma il compilatore
non è in grado di verificarlo. Per compilare questo genere di programmi,
dobbiamo dire al compilatore di rilassare un po' le sue restrizioni. A questo
scopo, Rust ha una parola-chiave, `unsafe` (insicuro, in italiano).
Il codice che usa `unsafe` ha meno restrizioni del codice normale.

Andiamo a vedere la sintassi, e poi parleremo della semantica. `unsafe` si usa
in quattro contesti. Il primo è per marcare una funzione come insicura:

```rust
unsafe fn pericolo_di_morte() {
    // roba paurosa
}
```

Tutte le funzioni chiamate da [FFI][ffi] devono essere marcate come `unsafe`,
per esempio. Il secondo uso di `unsafe` è per marcare un blocco insicuro:

[ffi]: ffi.html

```rust
unsafe {
    // roba paurosa
}
```

Il terzo è per marcare tratti insicuri:

```rust
unsafe trait Pauroso { }
```

E il quarto è per `impl`ementare uno di quei tratti:

```rust
# unsafe trait Scary { }
unsafe impl Pauroso for i32 {}
```

È importante essere capaci di delimitare esplicitamente il codice che
può avere dei difetti che provocano grossi danni. Se un programma in Rust va
in segmentation fault, si può stare sicuri che la causa è correlata a qualcosa
marcato come `unsafe`.

# Che cosa significa ‘sicuro’?

Sicuro ["safe"], nel contesto di Rust, significa ‘non fa niente di insicuro
["unsafe"]’. È anche importante sapere che ci sono certi comportamenti che
probabilmente non sono desiderabili nel nostro codice, ma che sono
espressamente _non_ insicuri, tra i quali:

* i deadlock
* i leak di memoria o di altre risorse
* uscire senza chiamare i distruttori
* l'overflow di un intero

Rust non può prevenire tutti i tipi di difetti del software. In Rust si può
scrivere, e inevitabilmente si scrive, del codice bacato. Queste cose
ovviamente sono indesiderabili, ma non si possono chiamare specificamente
`unsafe`.

Inoltre, i seguenti sono tutti comportamenti indefiniti in Rust, e devono
essere evitati, anche quando si scrive del codice `unsafe`:

* Accedere simultaneamente ai dati ["data race"]
* Dereferenziare un puntatore grezzo nullo o penzolante
* Leggere memoria [non inizializzata][undef]
* Violare le [regole di aliasing dei puntatori][aliasing] con dei puntatori
  grezzi.
* `&mut T` e `&T` seguono il modello [noalias][noalias] di ambito di LLVM,
  tranne se il `&T` contiene un `UnsafeCell<U>`. Il codice unsafe non deve
  violare queste garanzie di aliasing.
* Mutare un valore/riferimento immutabile senza usare `UnsafeCell<U>`
* Invocare del comportamento indefinito tramite gli intrinseci del compilatore:
  * Indicizzare fuori dai limiti di un oggetto con `std::ptr::offset`
    (l'intrinseco `offset`), eccetto uscire di un solo byte il che è permesso.
  * Usare `std::ptr::copy_nonoverlapping_memory` (gli intrinseci `memcpy32`/
    `memcpy64`) su buffer che si sovrappongono
* Usare i seguenti valori non validi in tipi primitivi, anche in campi privati
  e in legami locali:
  * un riferimento o box nullo o penzolante
  * Un valore diverso da `false` (0) e da `true` (1) in un `bool`
  * Un discriminante in un `enum` non compreso nella definizione del suo tipo
  * Un valore in un `char` che è un surrogato, o è maggiore di `char::MAX`
  * Sequenze di byte non UTF-8 in un `str`
* Svolgere lo stack da codice straniero a codice Rust o da codice Rust a codice
  straniero.

[noalias]: http://llvm.org/docs/LangRef.html#noalias
[undef]: http://llvm.org/docs/LangRef.html#undefined-values
[aliasing]: http://llvm.org/docs/LangRef.html#pointer-aliasing-rules

# I superpoteri del codice insicuro

Sia nelle funzioni insicure che nei blocchi insicuri, Rust consentirà di fare
tre cose che normalmente non consente. Appena tre. Eccole:

1. Leggere o scrivere una [variabile mutabile statica][static].
2. Dereferenziare un puntatore grezzo.
3. Chiamare funzioni insicure. Questa è l'abilità più potente.

Ecco. È importante `unsafe`, per esempio, non ‘spenga il verificatore
dei prestiti’. Aggiungere `unsafe` a qualche porzione di codice Rust non cambia
la sua semantica; non inizierà ad accettare di tutto. Ma permetterà
di scrivere cose che _violano_ ancune delle regole.

Si incontrerà la parola-chiave `unsafe` anche quando si scrivono dei legami
a interfacce straniere (cioè non in Rust). Usando i metodi forniti dalla
libreria, si è incoraggiati a scrivere interfacce sicure e native in Rust.

Esaminiamo le tre abilità di base elencate, in ordine.

## Lettura o scrittura di un `static mut`

Rust ha una caratteristica chiamata ‘`static mut`’ che permette uno stato
global mutabile. Farlo può causare una corsa ai dati, e come tale è
inerentemente non sicura. Per maggiori dettagli, si veda la sezione
[static][static] del libro.

[static]: const-and-static.html#static

## Dereferenziare un puntatore grezzo

I puntatori grezzi consentono di fare dell'aritmetica di puntatori arbitraria,
e possono provocare vari diversi difetti nella sicurezza e nella vulnerabilità
della memoria. In alcuni sensi, l'abilità di dereferenziare un puntatore
arbitrario è una delle cose più pericolose che si può fare. Per maggiori
informazioni sui puntatori grezzi, i veda [la loro sezione][rawpointers]
del libro.

[rawpointers]: raw-pointers.html

## Chiamare funzioni insicure

Quest'ultima abilità funziona con entrambi gli aspetti di `unsafe`: si possono
chiamare le funzioni marcate `unsafe` solamente dall'interno di un blocco
insicuro.

Quest'abilità è potente e variegata. Rust espone alcuni [intrinseci
del compilatore][intrinsics] come funzioni insicure, e alcune funzioni
insicure eludono le verifiche di sicurezza in fase di esecuzione,
barattando la sicurezza con la velocità.

Lo ripeto ancora: anche se si _può_ fare cose arbitrarie in blocchi e funzioni
insicuri, ciò non significa che si dovrebbe. Il compilatore agirà come se
si stessero mantenendo le sue invarianti, perciò bisogna fare attenzione!

[intrinsics]: intrinsics.html
