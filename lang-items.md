% Lang items

> **Nota**: lang items are often provided by crates in the Rust distribution,
> and lang items themselves have an unstable interface. It is recommended to use
> officially distributed crates instead of defining your own lang items.

Per il compilatore `rustc` ci sono delle operazioni innestabili, cioè,
delle funzionalità che non sono cablate nel linguaggio, ma sono implementate
nelle librerie, con un marcatore speciale per dire al compilatore
che esistono. Il marcatore è l'attributo `#[lang = "..."]` e ci sono
vari diversi valori di `...`, cioè vari diversi 'lang items'.

Per esempio, i puntatori `Box` richiedono due elementi linguistici, uno
per l'allocazione e uno per la deallocazione. Un programma autonomo che usa
lo zucchero `Box` per l'allocazione dinamica tramite `malloc` e `free`:

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

    // malloc failed
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
    deallocate(ptr as *mut u8, ::core::mem::size_of::<T>(), ::core::mem::align_of::<T>());
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

Note the use of `abort`: the `exchange_malloc` lang item is assumed to
return a valid pointer, and so needs to do the check internally.

Other features provided by lang items include:

- overloadable operators via traits: the traits corresponding to the
  `==`, `<`, dereferencing (`*`) and `+` (etc.) operators are all
  marked with lang items; those specific four are `eq`, `ord`,
  `deref`, and `add` respectively.
- stack unwinding and general failure; the `eh_personality`, `fail`
  and `fail_bounds_checks` lang items.
- the traits in `std::marker` used to indicate types of
  various kinds; lang items `send`, `sync` and `copy`.
- the marker types and variance indicators found in
  `std::marker`; lang items `covariant_type`,
  `contravariant_lifetime`, etc.

Lang items are loaded lazily by the compiler; e.g. if one never uses
`Box` then there is no need to define functions for `exchange_malloc`
and `exchange_free`. `rustc` will emit an error when an item is needed
but not found in the current crate or any that it depends on.
