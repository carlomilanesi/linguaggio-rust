% Interfaccia alle funzioni straniere ["Foreign Function Interface"]

# Introduzione

Questa guida userà la libreria di compressione/decompressione [Snappy]
(https://github.com/google/snappy) come introduzione alla scrittura di legami
per il codice straniero. Rust attualmente non è in grado di chiamare
direttamente funzioni di una libreria C++, ma Snappy comprende una interfaccia
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

Le funzioni straniere si presume siano insicure, e quindi le chiamate a loro
hanno bisogno di essere avvolte da `unsafe {}` come promessa per il compilatore
che ogni cosa contenuta entro di essa è veramente sicura. Le libreria C
espongono spesso interacce che non sono thread-safe, e quasi ogni funzione che
prende un argomento puntatore non è valida per tutti i possibili input
dato che il puntatore potrebbe essere penzolante ["dangling"], e i puntatori
grezzi cadono fuori dal modello di memoria sicuro di Rust.

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
    fn snappy_compress(input: *const u8,
                       input_length: size_t,
                       compressed: *mut u8,
                       compressed_length: *mut size_t) -> c_int;
    fn snappy_uncompress(compressed: *const u8,
                         compressed_length: size_t,
                         uncompressed: *mut u8,
                         uncompressed_length: *mut size_t) -> c_int;
    fn snappy_max_compressed_length(source_length: size_t) -> size_t;
    fn snappy_uncompressed_length(compressed: *const u8,
                                  compressed_length: size_t,
                                  result: *mut size_t) -> c_int;
    fn snappy_validate_compressed_buffer(compressed: *const u8,
                                         compressed_length: size_t) -> c_int;
}
# fn main() {}
```

# Creare un'interfaccia sicura

L'API C grezza ha bisogno di essere avvolta per fornire sicurezza di memoria
e fornire concetti di livello più alto, come i vettori. Una libreria può
scegliere di esporre solamente l'interfaccia sicura, ad alto livello, e
di nascondere i dettagli interni insicuri.

Avvolgere le funzioni che si aspettano delle aree comporta usare il modulo
`slice::raw` per manipolare i vettori Rust come puntatori alla memoria.
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
`unsafe`, ma garantisce che chiamarlo è sicuro per tutti gli inputs
escludendo la parola `unsafe` dalla firma della funzione.

Le funzioni `snappy_compress` e `snappy_uncompress` sono più complesse,
dato che un'area deve anche essere allocata per tenere l'output.

La funzione `snappy_max_compressed_length` può essere usata per allocare
un vettore con la capacità massima necessaria per tenere l'output compresso.
Il vettore poi può essere passato alla funzione `snappy_compress`
come argomento di output. Un argomento di output viene passato anche
per recuperare la vera lunghezza dopo la compressione per impostare
la lunghezza.

```rust
# #![feature(libc)]
# extern crate libc;
# use libc::{size_t, c_int};
# unsafe fn snappy_compress(a: *const u8, b: size_t, c: *mut u8,
#                           d: *mut size_t) -> c_int { 0 }
# unsafe fn snappy_max_compressed_length(a: size_t) -> size_t { a }
# fn main() {}
pub fn compress(src: &[u8]) -> Vec<u8> {
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
non compressa come parte del formato di compressione e
`snappy_uncompressed_length` recupererà la dimensione esatta dimensione
necessaria per l'area.

```rust
# #![feature(libc)]
# extern crate libc;
# use libc::{size_t, c_int};
# unsafe fn snappy_uncompress(compressed: *const u8,
#                             compressed_length: size_t,
#                             uncompressed: *mut u8,
#                             uncompressed_length: *mut size_t) -> c_int { 0 }
# unsafe fn snappy_uncompressed_length(compressed: *const u8,
#                                      compressed_length: size_t,
#                                      result: *mut size_t) -> c_int { 0 }
# fn main() {}
pub fn uncompress(src: &[u8]) -> Option<Vec<u8>> {
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

Poi, possiamo aggiungere dei collaudi per mostrare come usare queste funzioni.

```rust
# #![feature(libc)]
# extern crate libc;
# use libc::{c_int, size_t};
# unsafe fn snappy_compress(input: *const u8,
#                           input_length: size_t,
#                           compressed: *mut u8,
#                           compressed_length: *mut size_t)
#                           -> c_int { 0 }
# unsafe fn snappy_uncompress(compressed: *const u8,
#                             compressed_length: size_t,
#                             uncompressed: *mut u8,
#                             uncompressed_length: *mut size_t)
#                             -> c_int { 0 }
# unsafe fn snappy_max_compressed_length(source_length: size_t) -> size_t { 0 }
# unsafe fn snappy_uncompressed_length(compressed: *const u8,
#                                      compressed_length: size_t,
#                                      result: *mut size_t)
#                                      -> c_int { 0 }
# unsafe fn snappy_validate_compressed_buffer(compressed: *const u8,
#                                             compressed_length: size_t)
#                                             -> c_int { 0 }
# fn main() { }

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn valid() {
        let d = vec![0xde, 0xad, 0xd0, 0x0d];
        let c: &[u8] = &compress(&d);
        assert!(validate_compressed_buffer(c));
        assert!(uncompress(c) == Some(d));
    }

    #[test]
    fn invalid() {
        let d = vec![0, 0, 0, 0];
        assert!(!validate_compressed_buffer(&d));
        assert!(uncompress(&d).is_none());
    }

    #[test]
    fn empty() {
        let d = vec![];
        assert!(!validate_compressed_buffer(&d));
        assert!(uncompress(&d).is_none());
        let c = compress(&d);
        assert!(validate_compressed_buffer(&c));
        assert!(uncompress(&c) == Some(d));
    }
}
```

# Distruttori

Le librerie straniere spesso cedono la proprietà delle risorse
al codice chiamante. Quando questo accade, si devono usare i disrruttori
di Rust per fornire sicurezza e garantire il rilascio di queste risorse
(specialmente nel caso di panico).

Per maggiori informazioni sui distruttori, si veda il tratto [Drop]
(../std/ops/trait.Drop.html).

# Callback da codice Ca funzioni Rust

Alcune librerie esterne richiedono l'utilizzo di callback per riferire
al chiamante il loro stato attuale o dei dati intermedi.
È possibile passare a una libreria esterna funzioni definite in Rust.
Il requisito per poterlo fare è che la funzione callback sia marcata come
`extern` con la corretta convenzione di chiamata così da renderla chiamabile
da codice C.

Le funzioni callback possono poi essere mandate attraverso una chiamata
di registrazione alla libreria C e poi invocate dalla libreria.

Un semplice esempio è:

Codice Rust:

```rust,no_run
extern fn la_mia_callback(a: i32) {
    println!("Sono chiamata dal C con il valore {0}", a);
}

#[link(name = "extlib")]
extern {
   fn registra_callback(riferimento_callback: extern fn(i32)) -> i32;
   fn scatta_callback();
}

fn main() {
    unsafe {
        registra_callback(la_mia_callback);
        scatta_callback(); // Fa invocare la callback
    }
}
```

Codice C:

```c
typedef void (*callback_di_rust)(int32_t);
callback_di_rust puntatore_cb;

int32_t registra_callback(rust_callback callback) {
    puntatore_cb = callback;
    return 1;
}

void scatta_callback() {
  puntatore_cb(7); // Will call callback(7) in Rust
}
```

In this example Rust's `main()` will call `trigger_callback()` in C,
which would, in turn, call back to `callback()` in Rust.


## Targeting callbacks to Rust objects

The former example showed how a global function can be called from C code.
However it is often desired that the callback is targeted to a special
Rust object. This could be the object that represents the wrapper for the
respective C object.

This can be achieved by passing an raw pointer to the object down to the
C library. The C library can then include the pointer to the Rust object in
the notification. This will allow the callback to unsafely access the
referenced Rust object.

Rust code:

```rust,no_run
#[repr(C)]
struct RustObject {
    a: i32,
    // other members
}

extern "C" fn callback(target: *mut RustObject, a: i32) {
    println!("I'm called from C with value {0}", a);
    unsafe {
        // Update the value in RustObject with the value received from the callback
        (*target).a = a;
    }
}

#[link(name = "extlib")]
extern {
   fn register_callback(target: *mut RustObject,
                        cb: extern fn(*mut RustObject, i32)) -> i32;
   fn trigger_callback();
}

fn main() {
    // Create the object that will be referenced in the callback
    let mut rust_object = Box::new(RustObject { a: 5 });

    unsafe {
        register_callback(&mut *rust_object, callback);
        trigger_callback();
    }
}
```

C code:

```c
typedef void (*rust_callback)(void*, int32_t);
void* cb_target;
rust_callback cb;

int32_t register_callback(void* callback_target, rust_callback callback) {
    cb_target = callback_target;
    cb = callback;
    return 1;
}

void trigger_callback() {
  cb(cb_target, 7); // Will call callback(&rustObject, 7) in Rust
}
```

## Asynchronous callbacks

In the previously given examples the callbacks are invoked as a direct reaction
to a function call to the external C library.
The control over the current thread is switched from Rust to C to Rust for the
execution of the callback, but in the end the callback is executed on the
same thread that called the function which triggered the callback.

Things get more complicated when the external library spawns its own threads
and invokes callbacks from there.
In these cases access to Rust data structures inside the callbacks is
especially unsafe and proper synchronization mechanisms must be used.
Besides classical synchronization mechanisms like mutexes, one possibility in
Rust is to use channels (in `std::sync::mpsc`) to forward data from the C
thread that invoked the callback into a Rust thread.

If an asynchronous callback targets a special object in the Rust address space
it is also absolutely necessary that no more callbacks are performed by the
C library after the respective Rust object gets destroyed.
This can be achieved by unregistering the callback in the object's
destructor and designing the library in a way that guarantees that no
callback will be performed after deregistration.

# Linking

The `link` attribute on `extern` blocks provides the basic building block for
instructing rustc how it will link to native libraries. There are two accepted
forms of the link attribute today:

* `#[link(name = "foo")]`
* `#[link(name = "foo", kind = "bar")]`

In both of these cases, `foo` is the name of the native library that we're
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
* `vectorcall`
This is currently hidden behind the `abi_vectorcall` gate and is subject to change.
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
