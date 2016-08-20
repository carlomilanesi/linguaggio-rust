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
    println!("Sono chiamata dal C con il valore {0}", a);
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

The `link` attribute on `extern` blocks provides the basic building block for
instructing rustc how it will link to native libraries. There are two accepted
forms of the link attribute today:

* `#[link(name = "foo")]`
* `#[link(name = "foo", kind = "bar")]`

In both of these cases, `foo` is the name of the native library that we'' re
linking to, and in the second case `bar` is the type of native library that the
compiler is linking to. There are currently three known types of native
libraries:

* Dynamic - `#[link(name = "readline")]`
* Static - `#[link(name = "my_build_dependency", kind = "static")]`
* Frameworks - `#[link(name = "CoreFoundation", kind = "framework")]`

Note that frameworks are only available on OSX targets.

The different `kind` values are meant to differentiate how the native library
participates in linkage. From a linkage perspective, the Rust compiler creates
two flavors of artifacts: partial (rlib/staticlib) and final (dylib/binary).
Native dynamic library and framework dependencies are propagated to the final
artifact boundary, while static library dependencies are not propagated at
all, because the static libraries are integrated directly into the subsequent
artifact.

A few examples of how this model can be used are:

* A native build dependency. Sometimes some C/C++ glue is needed when writing
  some Rust code, but distribution of the C/C++ code in a library format is
  a burden. In this case, the code will be archived into `libfoo.a` and then the
  Rust crate would declare a dependency via `#[link(name = "foo", kind =
  "static")]`.

  Regardless of the flavor of output for the crate, the native static library
  will be included in the output, meaning that distribution of the native static
  library is not necessary.

* A normal dynamic dependency. Common system libraries (like `readline`) are
  available on a large number of systems, and often a static copy of these
  libraries cannot be found. When this dependency is included in a Rust crate,
  partial targets (like rlibs) will not link to the library, but when the rlib
  is included in a final target (like a binary), the native library will be
  linked in.

On OSX, frameworks behave with the same semantics as a dynamic library.

# Unsafe blocks

Some operations, like dereferencing raw pointers or calling functions that have been marked
unsafe are only allowed inside unsafe blocks. Unsafe blocks isolate unsafety and are a promise to
the compiler that the unsafety does not leak out of the block.

Unsafe functions, on the other hand, advertise it to the world. An unsafe function is written like
this:

```rust
unsafe fn kaboom(ptr: *const i32) -> i32 { *ptr }
```

This function can only be called from an `unsafe` block or another `unsafe` function.

# Accessing foreign globals

Foreign APIs often export a global variable which could do something like track
global state. In order to access these variables, you declare them in `extern`
blocks with the `static` keyword:

```rust,no_run
# #![feature(libc)]
extern crate libc;

#[link(name = "readline")]
extern {
    static rl_readline_version: libc::c_int;
}

fn main() {
    println!("You have readline version {} installed.",
             rl_readline_version as i32);
}
```

Alternatively, you may need to alter global state provided by a foreign
interface. To do this, statics can be declared with `mut` so we can mutate
them.

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

Note that all interaction with a `static mut` is unsafe, both reading and
writing. Trattare uno stato mutabile globale necessita di moltissima attenzione.

# Foreign calling conventions

Most foreign code exposes a C ABI, and Rust uses the platform's C calling convention by default when
calling foreign functions. Some foreign functions, most notably the Windows API, use other calling
conventions. Rust provides a way to tell the compiler which convention to use:

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

This applies to the entire `extern` block. The list of supported ABI constraints
are:

* `stdcall`
* `aapcs`
* `cdecl`
* `fastcall`
* `vectorcall` This is currently hidden behind the `abi_vectorcall`
   gate and is subject to change.
* `Rust`
* `rust-intrinsic`
* `system`
* `C`
* `win64`

Most of the abis in this list are self-explanatory, but the `system` abi may
seem a little odd. This constraint selects whatever the appropriate ABI is for
interoperating with the target's libraries. For example, on win32 with a x86
architecture, this means that the abi used would be `stdcall`. On x86_64,
however, windows uses the `C` calling convention, so `C` would be used. This
means that in our previous example, we could have used `extern "system" { ... }`
to define a block for all windows systems, not only x86 ones.

# Interoperability with foreign code

Rust guarantees that the layout of a `struct` is compatible with the platform's
representation in C only if the `#[repr(C)]` attribute is applied to it.
`#[repr(C, packed)]` can be used to lay out struct members without padding.
`#[repr(C)]` can also be applied to an enum.

Rust's owned boxes (`Box<T>`) use non-nullable pointers as handles which point
to the contained object. However, they should not be manually created because
they are managed by internal allocators. References can safely be assumed to be
non-nullable pointers directly to the type.  However, breaking the borrow
checking or mutability rules is not guaranteed to be safe, so prefer using raw
pointers (`*`) if that's needed because the compiler can't make as many
assumptions about them.

Vectors and strings share the same basic memory layout, and utilities are
available in the `vec` and `str` modules for working with C APIs. However,
strings are not terminated with `\0`. If you need a NUL-terminated string for
interoperability with C, you should use the `CString` type in the `std::ffi`
module.

The [`libc` crate on crates.io][libc] includes type aliases and function
definitions for the C standard library in the `libc` module, and Rust links
against `libc` and `libm` by default.

# The "nullable pointer optimization"

Certain types are defined to not be NULL. This includes references (`&T`,
`&mut T`), boxes (`Box<T>`), and function pointers (`extern "abi" fn()`).
When interfacing with C, pointers that might be NULL are often used.
As a special case, a generic `enum` that contains exactly two variants, one of
which contains no data and the other containing a single field, is eligible
for the "nullable pointer optimization". When such an enum is instantiated
with one of the non-nullable types, it is represented as a single pointer,
and the non-data variant is represented as the NULL pointer. So
`Option<extern "C" fn(c_int) -> c_int>` is how one represents a nullable
function pointer using the C ABI.

# Calling Rust code from C

You may wish to compile Rust code in a way so that it can be called from C. This is
fairly easy, but requires a few things:

```rust
#[no_mangle]
pub extern fn hello_rust() -> *const u8 {
    "Hello, world!\0".as_ptr()
}
# fn main() {}
```

The `extern` makes this function adhere to the C calling convention, as
discussed above in "[Foreign Calling
Conventions](ffi.html#foreign-calling-conventions)". The `no_mangle`
attribute turns off Rust's name mangling, so that it is easier to link to.

# FFI e il panico

È importante riflettere sui `panic!` quando si lavora con l'FFI. Un `panic!`
che attraversa un confine di FFI ha un comportamento indefinito. Se stiamo
scrivendo del codice che può andare in panico, lo dovremmo eseguire in
un altro thread, in modo che il panico non emerga fino al C:

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
far i dettagli interni di tale oggetto. Il modo più semplice è usare
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

Questo è un modo perfettamente valido di gestire la situazione. Tuttavia,
possiamo fare un po' meglio. Per risolverlo, alcune librerie C creeranno invece
una `struct`, nella quale i dettagli e la disposizione in memoria sono privati.
Questo dà una certa dose di sicurezza di tipo. Queste strutture sono chiamate
‘opache’. Ecco un esempio, in C:

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
