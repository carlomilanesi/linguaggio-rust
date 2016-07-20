% Intrinseci

> **Nota**: gli intrinseci avranno sempre un'interfaccia instabile, e quindi
> si consiglia di usare l'interfaccia stabile di libcore piuttosto che usare
> direttamente gli intrinseci.

Questi sono importati come se fossero funzioni FFI, con la speciali ABI
`rust-intrinsic`. Per esempio, se si fosse in un contesto indipendente,
ma si desiderasse poter trasmutare un tipo in un altro, ed eseguire
dell'aritmetica efficiente con i puntatori, si importerebbero quelle funzioni
tramite una dichiarazione come

```rust
#![feature(intrinsics)]
# fn main() {}

extern "rust-intrinsic" {
    fn transmute<T, U>(x: T) -> U;
    fn offset<T>(dst: *const T, offset: isize) -> *const T;
}
```

Come con ogni altra funzione FFI, questi sono sempre `unsafe` da chiamare.
