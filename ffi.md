% Interfaccia alle funzioni straniere ["Foreign Function Interface"]

# Introduzione

Questa guida userà la libreria di compressione/decompressione [Snappy]
(https://github.com/google/snappy) come introduzione alla scrittura di legami
per il codice straniero. Rust attualmente non è in grado di chiamare
direttamente funzioni di una libreria C++, ma Snappy comprende un'interfaccia
per il linguaggio C (documentata in [`snappy-c.h`]
(https://github.com/google/snappy/blob/master/snappy-c.h)).

## Una nota su 'libc'

Molti di questi esempi usano [il crate `libc`][libc], il quale, tra le altre
cose, fornisce varie definizioni di tipi del linguaggio C. Per provare questi
esempi, si dovrà aggiungere il riferimento a `libc` nel file `Cargo.toml`:

```toml
[dependencies]
libc = "0.2.0"
```

[libc]: https://crates.io/crates/libc

e aggiungere `extern crate libc;` alla radice del proprio crate.

## Chiamare funzioni straniere

Il seguente è un esempio minimale di come chiamare una funzione straniera
che compilerà se Snappy è installato:

```rust,no_run
# #![feature(libc)]
extern crate libc;
use libc::size_t;

#[link(name = "snappy")]
extern {
    fn snappy_max_compressed_length(source_length: size_t) -> size_t;
}

fn main() {
    let x = unsafe { snappy_max_compressed_length(100) };
    println!("lunghezza massima compressa di un'area di 100: {}", x);
}
```

Il blocco `extern` è un elenco di firme di funzioni di una libreria straniera,
che in questo caso usa l'ABI del linguaggio della piattaforma corrente.
L'attributo `#[link(...)]` serve a istruire il linker a collegare la libreria
Snappy in modo da risolvere i simboli.

Le funzioni straniere si presume siano insicure, e quindi le loro chiamate
hanno bisogno di essere avvolte da `unsafe {}` come promessa
per il compilatore che ogni cosa contenuta entro di essa è veramente sicura.
Le librerie C espongono spesso interacce che non sono sicure
da usare coi thread, e quasi ogni funzione che prende un argomento puntatore
non è valida per tutti i possibili input, dato che il puntatore potrebbe
essere penzolante, e i puntatori grezzi cadono fuori dal modello
di memoria sicuro di Rust.

Quando si dichiarano i tipi degli argomenti a una funzione straniera,
il compilatore Rust non può verificare se la dichiarazione è corretta, quindi
specificarlo correttamente fa parte del mantenimento di un legame corretto
in fase di esecuzione.

Il blocco `extern` può venire esteso così da coprire l'intera API di Snappy:

```rust,no_run
# #![feature(libc)]
extern crate libc;
use libc::{c_int, size_t};

#[link(name = "snappy")]
extern {
    fn snappy_compress(
        input: *const u8,
        input_length: size_t,
        compressed: *mut u8,
        compressed_length: *mut size_t) -> c_int;
    fn snappy_uncompress(
        compressed: *const u8,
        compressed_length: size_t,
        uncompressed: *mut u8,
        uncompressed_length: *mut size_t) -> c_int;
    fn snappy_max_compressed_length(source_length: size_t) -> size_t;
    fn snappy_uncompressed_length(
        compressed: *const u8,
        compressed_length: size_t,
        result: *mut size_t) -> c_int;
    fn snappy_validate_compressed_buffer(
        compressed: *const u8,
        compressed_length: size_t) -> c_int;
}
# fn main() {}
```

# Creare un'interfaccia sicura

L'API C grezza ha bisogno di essere avvolta per fornire sicurezza di memoria
e fornire concetti di livello più alto, come i vettori. Una libreria può
scegliere di esporre solamente l'interfaccia sicura, ad alto livello, e
di nascondere i dettagli interni insicuri.

Avvolgere le funzioni che si aspettano di ricevere delle aree di memoria
richiede l'uso del modulo `slice::raw` per manipolare i vettori Rust
come puntatori alla memoria.
I vettori di Rust sono garantiti essere un blocco contiguo di memoria
(virtuale).
La lunghezza di tale blocco è il numero di elementi attualmente contenuti, e
la capacità è il numero totale di elementi che può essere contenuto nella
memoria già allocata. La lunghezza è minore o uguale alla capacità.

```rust
# #![feature(libc)]
# extern crate libc;
# use libc::{c_int, size_t};
# unsafe fn snappy_validate_compressed_buffer(_: *const u8, _: size_t) -> c_int { 0 }
# fn main() {}
pub fn convalida_area_compressa(sorgente: &[u8]) -> bool {
    unsafe {
        snappy_validate_compressed_buffer(src.as_ptr(), src.len() as size_t) == 0
    }
}
```

L'avvolgimento `convalida_area_compressa` qui sopra fa uso di un blocco
`unsafe`, ma garantisce che chiamarlo è sicuro per tutti gli input
escludendo la parola `unsafe` dalla firma della funzione.

Le funzioni `snappy_compress` e `snappy_uncompress` sono più complesse,
dato che si deve anche allocata un'area di memoria per contenere l'output.

La funzione `snappy_max_compressed_length` può essere usata per allocare
un vettore con la capacità massima necessaria per contenere l'output compresso.
Il vettore poi può essere passato alla funzione `snappy_compress`
come argomento di output. Un argomento di output viene passato anche
per recuperare la lunghezza risultante dei dati compressi.

```rust
# #![feature(libc)]
# extern crate libc;
# use libc::{size_t, c_int};
# unsafe fn snappy_compress(
#     a: *const u8, b: size_t, c: *mut u8,
#     d: *mut size_t) -> c_int { 0 }
# unsafe fn snappy_max_compressed_length(a: size_t) -> size_t { a }
# fn main() {}
pub fn comprimi(src: &[u8]) -> Vec<u8> {
    unsafe {
        let srclen = src.len() as size_t;
        let psrc = src.as_ptr();

        let mut dstlen = snappy_max_compressed_length(srclen);
        let mut dst = Vec::with_capacity(dstlen as usize);
        let pdst = dst.as_mut_ptr();

        snappy_compress(psrc, srclen, pdst, &mut dstlen);
        dst.set_len(dstlen as usize);
        dst
    }
}
```

La decompressione è simile, perché Snappy immagazzina la dimensione
non compressa come parte del formato di compressione, e
`snappy_uncompressed_length` recupererà la dimensione esatta
necessaria per l'area.

```rust
# #![feature(libc)]
# extern crate libc;
# use libc::{size_t, c_int};
# unsafe fn snappy_uncompress(
#     compressed: *const u8,
#     compressed_length: size_t,
#     uncompressed: *mut u8,
#     uncompressed_length: *mut size_t) -> c_int { 0 }
# unsafe fn snappy_uncompressed_length(
#     compressed: *const u8,
#     compressed_length: size_t,
# 	  result: *mut size_t) -> c_int { 0 }
# fn main() {}
pub fn decomprimi(src: &[u8]) -> Option<Vec<u8>> {
    unsafe {
        let srclen = src.len() as size_t;
        let psrc = src.as_ptr();

        let mut dstlen: size_t = 0;
        snappy_uncompressed_length(psrc, srclen, &mut dstlen);

        let mut dst = Vec::with_capacity(dstlen as usize);
        let pdst = dst.as_mut_ptr();

        if snappy_uncompress(psrc, srclen, pdst, &mut dstlen) == 0 {
            dst.set_len(dstlen as usize);
            Some(dst)
        } else {
            None // SNAPPY_INVALID_INPUT
        }
    }
}
```

Poi, possiamo aggiungere dei test per mostrare come usare queste funzioni.

```rust
# #![feature(libc)]
# extern crate libc;
# use libc::{c_int, size_t};
# unsafe fn snappy_compress(
#     input: *const u8,
#     input_length: size_t,
#     compressed: *mut u8,
#     compressed_length: *mut size_t)
#     -> c_int { 0 }
# unsafe fn snappy_uncompress(
#     compressed: *const u8,
#     compressed_length: size_t,
#     uncompressed: *mut u8,
#     uncompressed_length: *mut size_t)
#     -> c_int { 0 }
# unsafe fn snappy_max_compressed_length(
#     source_length: size_t) -> size_t { 0 }
# unsafe fn snappy_uncompressed_length(
#     compressed: *const u8,
#     compressed_length: size_t,
#     result: *mut size_t)
#     -> c_int { 0 }
# unsafe fn snappy_validate_compressed_buffer(
#     compressed: *const u8,
#     compressed_length: size_t)
#     -> c_int { 0 }
# fn main() { }

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn valido() {
        let d = vec![0xde, 0xad, 0xd0, 0x0d];
        let c: &[u8] = &comprimi(&d);
        assert!(convalida_area_compressa(c));
        assert!(decomprimi(c) == Some(d));
    }

    #[test]
    fn invalido() {
        let d = vec![0, 0, 0, 0];
        assert!(!convalida_area_compressa(&d));
        assert!(decomprimi(&d).is_none());
    }

    #[test]
    fn vuoto() {
        let d = vec![];
        assert!(!convalida_area_compressa(&d));
        assert!(decomprimi(&d).is_none());
        let c = comprimi(&d);
        assert!(convalida_area_compressa(&c));
        assert!(decomprimi(&c) == Some(d));
    }
}
```

# Distruttori

Le librerie straniere spesso cedono la proprietà delle risorse
al codice chiamante. Quando questo accade, si devono usare i distruttori
di Rust per fornire sicurezza e garantire il rilascio di queste risorse
(specialmente nel caso di panico).

Per maggiori informazioni sui distruttori, si veda il tratto [Drop]
(../std/ops/trait.Drop.html).

# Callback da codice C a funzioni Rust

Alcune librerie esterne richiedono l'utilizzo di callback per comunicare
al chiamante il loro stato attuale o qualche dato intermedio.
È possibile passare a una libreria esterna funzioni definite in Rust.
Il requisito per poterlo fare è che la funzione callback sia marcata come
`extern` con la corretta convenzione di chiamata, così da renderla chiamabile
da codice C.

Le funzioni callback possono poi essere comunicate alla libreria C mediante
una chiamata di registrazione, per poi essere invocate dalla libreria.

Un semplice esempio è:

Codice Rust:

```rust,no_run
extern fn la_mia_callback(a: i32) {
    println!("Sono chiamata da C con il valore {0}", a);
}

#[link(name = "extlib")]
extern {
   fn registra_callback(riferimento_callback: extern fn(i32)) -> i32;
   fn invoca_callback();
}

fn main() {
    unsafe {
        registra_callback(la_mia_callback);
        invoca_callback(); // Fa invocare la callback
    }
}
```

Codice C:

```c
typedef void (*callback_di_rust)(int32_t);
callback_di_rust puntatore_cb;

int32_t registra_callback(callback_di_rust callback) {
    puntatore_cb = callback;
    return 1;
}

void invoca_callback() {
  puntatore_cb(7); // Chiamerà callback(7) in Rust
}
```

In questo esempio, la `main()` di Rust chiamerà `invoca_callback()` in C,
che, a sua volta, richiamerà `callback()` in Rust.

## Applicare le callback a oggetti Rust

L'esempio precedente ha mostrato come una funzione globale Rust possa
essere chiamata da codice C. Però spesso si desidera che la callback
sia applicata ad un particolare oggetto Rust. Questo potrebbe essere
l'oggetto che rappresenta l'avvolgimento per il rispettivo oggetto C.

Ciò può essere fatto passando alla libreria C un puntatore grezzo
all'oggetto. Poi la libreria C può includere nella notifica il puntatore
all'oggetto Rust. Ciò consentirà alla callback di accedere,
in modo _non_ sicuro, all'oggetto Rust referenziato.

Codice Rust:

```rust,no_run
#[repr(C)]
struct OggettoRust {
    a: i32,
    // altri membri
}

extern "C" fn callback(target: *mut OggettoRust, a: i32) {
    println!("Sono chiamata da C con valore {0}", a);
    unsafe {
        // Aggiorna il valore in OggettoRust
        // con il valore ricevuto dalla callback
        (*target).a = a;
    }
}

#[link(name = "extlib")]
extern {
    fn registra_callback(target: *mut OggettoRust,
        cb: extern fn(*mut OggettoRust, i32)) -> i32;
    fn invoca_callback();
}

fn main() {
    // Crea l'oggetto che verrà referenziato nella callback
    let mut oggetto_rust = Box::new(OggettoRust { a: 5 });

    unsafe {
        registra_callback(&mut *oggetto_rust, callback);
        invoca_callback();
    }
}
```

Codice C:

```c
typedef void (*callback_di_rust)(void*, int32_t);
void* cb_target;
callback_di_rust cb;

int32_t registra_callback(void* callback_target, callback_di_rust callback) {
    cb_target = callback_target;
    cb = callback;
    return 1;
}

void invoca_callback() {
  cb(cb_target, 7); // Chiamerà callback(&oggettoRust, 7) in Rust
}
```

## Callback asincrone

Negli esempi precedenti, le callback sono invocate come reazione diretta
a una chiamata di funzione alla libreria C esterna.
Il controllo nel thread corrente passa da Rust a C e di nuovo a Rust
per l'esecuzione della callback, ma alla fine la callback viene eseguita
nel medesimo thread che ha chiamato la funzione
che ha fatto invocare la callback.

Le cose si complicano quando la libreria esterna genera i propri thread
e invoca delle callback da tali thread.
In questi casi, l'accesso alle strutture dati di Rust dall'interno
delle callback è particolarmente insicuro, e si deve usare un appropriato
meccanismo di sincronizzazione.
Oltre ai classici meccanismi di sincronizzazione, come i mutex, Rust offre
la possibilità di usare i canali (in `std::sync::mpsc`) per inoltrare dati
al thread in Rust, dal thread in C che ha invocato la callback.

Anche se una callback asincrona viene applicata a un oggetto particolare
nello spazio degli indirizzi di Rust, è assolutamente necessario
che nessun'altra callback sia eseguita dalla libreria C dopo che
il rispettivo oggetto Rust sia stato distrutto.
Ciò si può ottenere deregistrando la callback nel distruttore dell'oggetto,
e progettando la libreria in modo che garantisca che nessuna
callback sia eseguita dopo la deregistrazione.

# Eseguire il link

L'attributo `link` sui blocchi `extern` fornisce il blocco di costruzione
di base per istruire rustc su come collegare librerie native. Oggi ci sono
due forme accettate dell'attributo 'link':

* `#[link(name = "foo")]`
* `#[link(name = "foo", kind = "bar")]`

In entrambi questi casi, `foo` è il nome della libreria nativa che a cui
ci stiamo collegando, e nel secondo caso `bar` è il tipo della libreria nativa
a cui il compilatore si sta collegando. Attualmente ci sono tre tipi noti
di librerie native:

* Dinamiche - `#[link(name = "readline")]`
* Statiche - `#[link(name = "my_build_dependency", kind = "static")]`
* Framework - `#[link(name = "CoreFoundation", kind = "framework")]`

Si noti che i framework sono disponibili solamente su target OSX.

I diversi valori di `kind` sono pensati per differenziare come
la libreria nativa partecipa al collegamento. Per quanto riguarda
il collegamento, il compilatore Rust crea due varietà di artefatti:
parziale (rlib/staticlib) e finale (dylib/binary).
Le dipendenze delle librerie dinamiche native e dei framework vengono
propagate fino all'artefatto finale, mentre le dipendenze delle librerie
statiche non vengono propagate affatto, perché le librerie statiche vengono
direttamente integrate nell'artefatto prodotto.

Ecco alcuni esempi di come si può usare questo modello:

* Un dipendenza di una compilazione nativa. Talvolta della colla C/C++
  è necessaria quando si scrive del codice Rust, ma distribuire il codice
  C/C++ in formato di libreria è una zavorra. In questo caso, il codice
  verrà incapsulato in un archivio `libfoo.a` e poi il crate Rust
  dichiarerà una dipendenza tramite `#[link(name = "foo", kind = "static")]`.

  Indipendentemente dalla varietà dell'output del crate, la libreria statica
  nativa verrà inclusa nell'output, nel senso che non sarà necessario
  distribuire la libreria statica nativa.

* Una dipendenza dinamica normale. Le tipiche librerie di sistema
  (come `readline`) sono disponibili su un gran numero di sistemi, e spesso
  non si trova una copia statica di queste librerie. Quando questa dipendenza
  viene inclusa in un crate Rust, i target parziali (come rlibs)
  non verranno collegati alla libreria, ma quando la rlib viene inclusa
  in un target finale (come un programma), la libreria nativa verrà collegata.

Con OSX, i framework si comportano con la medesima semantica
delle librerie dinamiche.

# Blocci `unsafe`

Alcune operazioni, come dereferenziare i puntatori grezzi o chiamare funzioni
che sono state marcate `unsafe`, sono consentiti all'interno
di blocchi `unsafe`. I blocchi `unsafe` isolano l'insicurezza e sono promesse
al compilatore che l'insicurezza non travalicherà il blocco.

Le funzioni `unsafe`, d'altra parte, lo annunciano a tutto mondo.
Una funzione `unsafe` è scritta così:

```rust
unsafe fn kaboom(ptr: *const i32) -> i32 { *ptr }
```

Questa funzione può essere chiamata solamente da un blocco `unsafe` o
da un'altra funzione `unsafe`.

# Accedere a variabili globali straniere

Le API straniere spesso esportano una variabile globale che potrebbe fare
qualcosa come tener traccia di uno stato globale. Per poter accedere
a queste variabili, le si dichiara in blocchi `extern`
con la parola-chiave `static`:

```rust,no_run
# #![feature(libc)]
extern crate libc;

#[link(name = "readline")]
extern {
    static rl_readline_version: libc::c_int;
}

fn main() {
    println!("La versione di readline che hai installato è la {}.",
        rl_readline_version as i32);
}
```

Alternativamente, si può dover alterare lo stato globale fornito
da un'interfaccia straniera. Per farlo, gli statici possono essere
dichiarati con `mut`, così da poterli mutare.

```rust,no_run
# #![feature(libc)]
extern crate libc;

use std::ffi::CString;
use std::ptr;

#[link(name = "readline")]
extern {
    static mut rl_prompt: *const libc::c_char;
}

fn main() {
    let prompt = CString::new("[my-awesome-shell] $").unwrap();
    unsafe {
        rl_prompt = prompt.as_ptr();

        println!("{:?}", rl_prompt);

        rl_prompt = ptr::null();
    }
}
```

Si noti che ogni interazione con una `static mut` è `unsafe`, sia
in lettura che in scrittura. Trattare uno stato mutabile globale necessita
di moltissima attenzione.

# Convenzioni di chiamata straniera

La maggior parte del codice straniero espone una ABI per C, e Rust,
di default, usa la convenzione di chiamata del C della piattaforma, quando
chiama funzioni straniere. Alcune funzioni straniere, in particolare l'API
di Windows, usa altre convenzioni di chiamata. Rust fornisce un modo
di dire al compilatore quale convenzione usare:

```rust
# #![feature(libc)]
extern crate libc;

#[cfg(all(target_os = "win32", target_arch = "x86"))]
#[link(name = "kernel32")]
#[allow(non_snake_case)]
extern "stdcall" {
    fn SetEnvironmentVariableA(n: *const u8, v: *const u8) -> libc::c_int;
}
# fn main() { }
```

Ciò si applica all'intero blocco `extern`. I vincoli ABI supportati sono:

* `stdcall`
* `aapcs`
* `cdecl`
* `fastcall`
* `vectorcall`: attualmente questo è nascosto dietro il gate `abi_vectorcall`
   ed è soggetto a cambiamenti.
* `Rust`
* `rust-intrinsic`
* `system`
* `C`
* `win64`

La maggior parte delle ABI in questo elenco si spiegano da sole, ma
l'ABI `system` può sembrare un po' strana. Questo vincolo seleziona
l'ABI appropriata per comunicare con le librerie della piattaforma target.
Per esempio, su Win32 con architettura x86, risulta che l'ABI usata sarebbe
`stdcall`. Però su x86_64, Windows usa la convenzione di chiamata `C`,
e quindi verrebbe usata `C`. Ciò comporta che nel nostro esempio precedente,
avremmo potuto usare `extern "system" { ... }` per definire un blocco
per tutti i sistemi Windows, non solo quelli per x86.

# Comunicazione con codice straniero

Rust garantisce che il layout di una `struct` è compatibile
con la rappresentazione della piattaforma in C solamente se le è applicato
l'attributo `#[repr(C)]`. Si può usare `#[repr(C, packed)]` per disporre
i membri della struct senza padding. `#[repr(C)]` può essere applicato
anche a una enum.

I box posseduti da Rust (`Box<T>`) usano puntatori che non sono mai nulli
come handle che puntano all'oggetto contenuto. Però, non dovrebbero essere
creati manualmente perché sono gestiti da allocatori interni.
Normalmente si può avere la certezza che ogni riferimento punti a un oggetto
valido. Però, se si viola la verifica dei prestiti o le regole di mutabilità,
tale certezza non è più garantita, e allora è meglio usare i puntatori grezzi
(`*`), se è necessario, perché il compilatore non può fare altrettante
assunzioni su di essi.

I vettori e le stringhe condividono im medesimo layout di memoria di base,
e sono disponibili delle utility nei moduli `vec` e `str` per lavorare con
API in C. Però, le stringhe non sono terminate da un carattere `\0`.
Se serve una stringa terminata da NUL per comunicare con C, si dovrebbe
usare il tipo `CString` nel modulo `std::ffi`.

Il [crate `libc` su crates.io][libc] comprende nel modulo `libc` degli alias
di tipo e delle definizioni di funzioni per la libreria standard di C,
e Rust di default collega con le librerie `libc` e `libm`.

# L'"ottimizzazione del puntatore annullabile"

Certi tipi sono definiti in modo da non essere mai NULL. Tra di essi ci sono
i riferimenti (`&T`, `&mut T`), i box (`Box<T>`), e i puntatori a funzione
(`extern "abi" fn()`). Quando ci si interfaccia con C, si usano spesso
dei puntatori che potrebbero essere NULL. Come caso particolare,
a una `enum` generica che contiene esattamente due varianti, una delle quali
non contiene dati e l'altra contiene un solo campo, è applicabile
l'"ottimizzazione del puntatore annullabile". Quando una tale enum
viene istanziata con uno dei tipi non annullabili, viene rappresentata come
un singolo puntatore, e la variante senza dato viene rappresentata come
un puntatore NULL. Quindi `Option<extern "C" fn(c_int) -> c_int>` è il modo
di rappresentare un puntatore annullabile a funzione usando l'ABI di C.

# Chiamare codice Rust da C

Si può desiderare di compilare del codice Rust in un modo che possa
essere chiamato da codice C. Ciò è abbastanza facile, ma richiede alcune cose:

```rust
#[no_mangle]
pub extern fn hello_rust() -> *const u8 {
    "Hello, world!\0".as_ptr()
}
# fn main() {}
```

L'`extern` fa aderire questa funzione alla convenzione di chiamata di C, come
discusso sopra in "[Convenzioni di chiamata straniera]
(ffi.html#foreign-calling-conventions)". L'attributo `no_mangle`
spegne la storpiatura dei nomi fatta da Rust, così che sia più facile
da collegare.

# FFI e il panico

È importante riflettere sui `panic!` quando si lavora con l'FFI. Un `panic!`
che attraversa un confine di FFI ha un comportamento indefinito. Se stiamo
scrivendo del codice che può andare in panico, lo dovremmo eseguire in
un altro thread, in modo che il panico non emerga fino a C:

```rust
use std::thread;

#[no_mangle]
pub extern fn oh_no() -> i32 {
    let h = thread::spawn(|| {
        panic!("Ops!");
    });

    match h.join() {
        Ok(_) => 1,
        Err(_) => 0,
    }
}
# fn main() {}
```

# Rappresentare struct opache

Talvolta, una libreria C vuole fornire un puntatore a qualche oggetto, ma non
far sapere i dettagli interni di tale oggetto. Il modo più semplice è usare
un argomento di tipo `void *`:

```c
void foo(void *arg);
void bar(void *arg);
```

Possiamo rappresentare questo in Rust con il tipo `c_void`:

```rust
# #![feature(libc)]
extern crate libc;

extern "C" {
    pub fn foo(arg: *mut libc::c_void);
    pub fn bar(arg: *mut libc::c_void);
}
# fn main() {}
```

Questo è un modo perfettamente valido di gestire la situazione. Però,
possiamo fare di meglio. In casi del genere, alcune librerie C creeranno
una `struct`, nella quale i dettagli e la disposizione in memoria
sono lasciati privati. Questo dà una certa dose di sicurezza di tipo.
Queste strutture sono chiamate ‘opache’. Ecco un esempio, in C:

```c
/* Foo è una struttura, ma il suo contenuto
non fa parte dellla sua interfaccia pubblica */
struct Foo;
struct Bar;
void foo(struct Foo *arg);
void bar(struct Bar *arg);
```

Per farlo in Rust, creiamo i nostri tipi opachi usando `enum`:

```rust
pub enum Foo {}
pub enum Bar {}

extern "C" {
    pub fn foo(arg: *mut Foo);
    pub fn bar(arg: *mut Bar);
}
# fn main() {}
```

Usando un `enum` senza varianti, creiamo un tipo opaco che non possiamo
instanziare, dato che non ha nessuna variante. Ma siccome i nostri tipi `Foo`
e `Bar` sono  diversi, otterremo sicurezza di tipo fra loro due, perciò non
possiamo passare accidentalmente a `bar()` un puntatore a `Foo`.
