% Allocatori personalizzati

Allocare memoria non è sempre la cosa più facile da fare, e mentre Rust
in generale se ne occupa di default, spesso diventa necessario personalizzare
come avviene l'allocazione. Il compilatore e la libreria standard attualmente
consentono di escludere l'uso dell'allocatore globale di default in fase
di compilazione. Il progetto è attualmente descritto nell'[RFC 1183][rfc],
ma qui vedremo i passi per impostare l'uso del proprio allocatore
personalizzato.

[rfc]: https://github.com/rust-lang/rfcs/blob/master/text/1183-swap-out-jemalloc.md

# L'allocatore di default

Il compilatore attualmente fornisce due allocatori di default: `alloc_system`
e `alloc_jemalloc` (però per alcuni target jemalloc non viene fornito). Questi
allocatori sono dei normali crate Rust e contengono un'implementazione
delle routine per allocare e deallocare memoria. La libreria standard non
viene compilata assumendo uno specifico, e il compilatore deciderà quale
allocatore è in uso in fase di compilazione a seconda del tipo degli artefatti
di output che vengono prodotti.

I binari generati dal compilatore useranno `alloc_jemalloc` di default (se
disponibile). In questa situazione il compilator "controlla il mondo"
nel senso che ha il potere di influenza il link finale. Primariamente ciò
significa che la decisione sull'allocatore si può lasciare al compilatore.

Però, le libererie dinamiche e statiche di default usano `alloc_system`.
In questo caso Rust è tipicamente un 'ospite' in un'altra applicazioneo
in un'altro mondo dove non può decidere autoritariamente quale allocatore
usare. Di conseguenza ricorre alle API standard (per esempio `malloc` e `free`)
per acquisire e rilasciare la memoria.

# Cambiare allocatori

Sebbene le scelte di default del compilatore possono funzionare la maggior
parte delle volte, è spesso necessario mettere a punto certi aspetti.
Scavalcare la decisione del compilatore su quale allocatore usare viene fatto
semplicemente eseguendo il link con l'allocatore desiderato:

```rust,no_run
#![feature(alloc_system)]

extern crate alloc_system;

fn main() {
    let a = Box::new(4); // alloca dall'allocatore di sistema
    println!("{}", a);
}
```

In questo esempio il binario generato non linkerà jemalloc di default ma invece
userà l'allocatore di sistema. Viceversa per generare una liberaria dinamica
che usa jemalloc di default si scriverebbe:

```rust,ignore
#![feature(alloc_jemalloc)]
#![crate_type = "dylib"]

extern crate alloc_jemalloc;

pub fn foo() {
    let a = Box::new(4); // alloca da jemalloc
    println!("{}", a);
}
# fn main() {}
```

# Scrivere un allocatore personalizzato

Talvolta perfino le scelte tra jemalloc e l'allocatore di sistema non bastano
e servirebbe un allocatore personalizzato completamente nuovo. In questa
sezione scriveremo il nostro crate che implementa l'API di allocatore
(per esempio la stessa di `alloc_system` e di `alloc_jemalloc`). Come esempio,
diamo un'occhiata a una versione semplificata e annotata di `alloc_system`:

```rust,no_run
# // necessario solamente per rustdoc --collaudo in seguito
# #![feature(lang_items)]
// Al compilatore si deve far sapere che questo crate è un allocatore affinché
// eviti di linkare un altro allocatore, come jemalloc
#![feature(allocator)]
#![allocator]

// Agli allocatori non è consentito dipendere dalla libreria standard,
// perché questa dipende da un allocatore, e quindi ci sarebbero dipendenze
// circolari. Questo crate, comunque, può usare tutto libcore.
#![no_std]

// Diamo un nome univoco al nostro allocatore personalizzato
#![crate_name = "il_mio_allocatore"]
#![crate_type = "rlib"]

// Il nostro allocatore personalizzato userà il crate libc in-albero per i legami FFI. Si noti
// che attuallmente il libc esterno (crates.io) non può essere usato perché linka
// alla libreria standard (per es. `#![no_std]` non è ancora stabile), quindi
// ecco perché questo richiede specificamente la versione in-albero.
#![feature(libc)]
extern crate libc;

// Elencati sotto ci sono cinque funzioni di allocazione attualmente richieste per gli allocatori personalizzati
// . Attualmente il compilatore non verifica il tipo delle loro firme e dei loro simboli
// , ma questa è un'estensione futura e devono corrispondere a ciò che si trova sotto.
//
// Si noti che le funzioni standard `malloc` e `realloc` non forniscono un modo
// per comunicare l'allineamento, e così questa implementazione avrebbe bisogno di essere migliorata
// rispetto all'allineamento.

#[no_mangle]
pub extern fn __rust_allocate(size: usize, _align: usize) -> *mut u8 {
    unsafe { libc::malloc(size as libc::size_t) as *mut u8 }
}

#[no_mangle]
pub extern fn __rust_deallocate(ptr: *mut u8, _old_size: usize, _align: usize) {
    unsafe { libc::free(ptr as *mut libc::c_void) }
}

#[no_mangle]
pub extern fn __rust_reallocate(ptr: *mut u8, _old_size: usize, size: usize,
                                _align: usize) -> *mut u8 {
    unsafe {
        libc::realloc(ptr as *mut libc::c_void, size as libc::size_t) as *mut u8
    }
}

#[no_mangle]
pub extern fn __rust_reallocate_inplace(_ptr: *mut u8, old_size: usize,
                                        _size: usize, _align: usize) -> usize {
    old_size // this api is not supported by libc
}

#[no_mangle]
pub extern fn __rust_usable_size(size: usize, _align: usize) -> usize {
    size
}

# // necessario solamente affinché rustdoc lo collaudi
# fn main() {}
# #[lang = "panic_fmt"] fn panic_fmt() {}
# #[lang = "eh_personality"] fn eh_personality() {}
# #[lang = "eh_unwind_resume"] extern fn eh_unwind_resume() {}
# #[no_mangle] pub extern fn rust_eh_register_frames () {}
# #[no_mangle] pub extern fn rust_eh_unregister_frames () {}
```

Dopo aver compilato questo crate, lo si può usare come segue:

```rust,ignore
extern crate il_mio_allocatore;

fn main() {
    let a = Box::new(8); // alloca la memoria tramite il crate del nostro allocatore personalizzato
    println!("{}", a);
}
```

# Limitazioni dell'allocatore personalizzato

Ci sono alcune restrizioni quando si lavora con gli allocatori personalizzati
che possono causare errori di compilazione:

* Ogni artefatto può essere linkato solamente a un allocatore al massimo. I binari,
  le dylib, e le staticlib devono linkare ad esattamente un allocatore, e se nessuno è
  stato esplicitamente scelto il compilatore ne sceglierà uno. D'altro canto le rlib
  non hanno bisogno di linkare a un allocatore (ma lo possono ancora).

* Un consumatore di un allocatore è etichettato con `#![needs_allocator]` (per es. attualmente
  il crate `liballoc`) e un crate `#[allocator]` non può transitivamente
  dipendere da un crate che ha bisogno di un allocatore (per es. le dipendenze circolari non sono ammesse).
  Questo sostanzialmente significa che gli allocatori attualmente devono restringersi to
  libcore.
