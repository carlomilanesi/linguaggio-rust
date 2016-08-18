% Gli elementi "lang"

> **Nota**: gli elementi "lang" sono spesso forniti da crate
> nella distribuzione di Rust, e gli stessi elementi lang hanno
> un'interfaccia instabile. Si consiglia di usare i crate distribuiti
> ufficialmente, invece di definire i propri elementi lang.

Per il compilatore `rustc` ci sono delle operazioni innestabili, cioè,
delle funzionalità che non sono cablate nel linguaggio, ma sono implementate
nelle librerie, con un marcatore speciale per dire al compilatore
che esistono. Il marcatore è l'attributo `#[lang = "..."]`, e ci sono
vari diversi valori di `...`, cioè vari diversi 'elementi lang'.

Per esempio, i puntatori `Box` richiedono due elementi lang, uno
per l'allocazione e uno per la deallocazione. Ecco un programma autonomo
che usa `Box` per l'allocazione dinamica tramite `malloc` e `free`:

```rust
#![feature(lang_items, box_syntax, start, libc)]
#![no_std]

extern crate libc;

extern {
    fn abort() -> !;
}

#[lang = "owned_box"]
pub struct Box<T>(*mut T);

#[lang = "exchange_malloc"]
unsafe fn allocate(size: usize, _align: usize) -> *mut u8 {
    let p = libc::malloc(size as libc::size_t) as *mut u8;

    // malloc ha fallito
    if p as usize == 0 {
        abort();
    }

    p
}

#[lang = "exchange_free"]
unsafe fn deallocate(ptr: *mut u8, _size: usize, _align: usize) {
    libc::free(ptr as *mut libc::c_void)
}

#[lang = "box_free"]
unsafe fn box_free<T>(ptr: *mut T) {
    deallocate(ptr as *mut u8, ::core::mem::size_of::<T>(),
        ::core::mem::align_of::<T>());
}

#[start]
fn main(argc: isize, argv: *const *const u8) -> isize {
    let x = box 1;

    0
}

#[lang = "eh_personality"] extern fn eh_personality() {}
#[lang = "panic_fmt"] fn panic_fmt() -> ! { loop {} }
# #[lang = "eh_unwind_resume"] extern fn rust_eh_unwind_resume() {}
# #[no_mangle] pub extern fn rust_eh_register_frames () {}
# #[no_mangle] pub extern fn rust_eh_unregister_frames () {}
```

Si noti l'uso di `abort`: si assume che l'elemento lang `exchange_malloc`
restituisca un puntatore valido, e quindi internamente deve fare la verifica.

Tra le caratteristiche fornite dagli elementi lang si sono:

- operatori sovraccaricabili tramite tratti: i tratti corrispondenti agli
  operatori `==`, `<`, dereferenziazione (`*`), e `+` (ecc.) sono tutti
  marcati con elementi lang; per questi specifici quattro, gli elementi lang
  sono, rispettivamente: `eq`, `ord`, `deref`, e `add`.
- svolgimento dello stacke fallimento generale; gli elementi lang
  `eh_personality`, `fail`, e `fail_bounds_checks`.
- i tratti in `std::marker` usati per indicare tipi di vari generi;
  gli elementi lang `send`, `sync` e `copy`.
- i tipi marcatori e gli indicatori di varianza trovati in `std::marker`;
  gli elementi lang `covariant_type`, `contravariant_lifetime`, ecc.

Gli elementi lang vengono caricati pigramente dal compilatore; per es. se
non si usa mai `Box`, allora non c'è bisogno di definire funzioni per
`exchange_malloc` e `exchange_free`. `rustc` emitterà un errore quando
un elemento serve ma non viene trovato nel crate attuale né in quelli
da cui dipende.
