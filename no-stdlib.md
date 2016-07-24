% Evitare l'uso di stdlib

La libreria standard di Rust fornisce molte utili funzionalità, ma assume che
il suo sistema ospitante supporti vari caratteristiche: i thread, la rete,
l'allocazione dinamica di memoria, e altro. Però, ci sono dei sistemi che
non hanno queste caratteristiche, e Rust può produrre del software anche
per loro! Per farlo, bisogna dire a Rust che non si vuole usare la libreria standard,
usando l'attributo: `#![no_std]`.

> Nota: Questa caratteristica è tecnicamente stabile, ma ci sono dei cavilli.
> Per dirne one, si può costruire una _libreria_ con `#![no_std]`
> con la versione stabile, ma non un _programma_.
> Per avere dettagli sulle librerie senza la libreria standard, si veda
> [il capitolo su `#![no_std]`](using-rust-without-the-standard-library.html)

Ovviamente nella vita c'è di più che le librerie: si può usare
`#[no_std]` anche con un programma.

### Usare libc

Per poter costruire un programma `#[no_std]`, avremo bisogno di una dipendenza
da libc. Lo possiamo specificare usando il nostro file `Cargo.toml`:

```toml
[dependencies]
libc = { version = "0.2.14", default-features = false }
```

Si noti che le caratteristiche di default ["default features"] sono state
disabilitate. Questo è un passo critico -
**le caratteristiche di default di libc comprendono la libreria standard e
quindi devono essere disabilitate.**

### Scrivere un programma senza stdlib

Controllare il punto di ingresso è possibile è possibile in due modi:
l'attributo `#[start]`, o scavalcare il shim di default per la funzione C
`main` con la propria.

The function marked `#[start]` is passed the command line parameters
in the same format as C:

```rust,ignore
#![feature(lang_items)]
#![feature(start)]
#![no_std]

// Tira dentro la libreria libc di sistema per avere ciò che probabilmente
// e' richiesto da crt0.o
extern crate libc;

// Punto d'ingresso in questo programma
#[start]
fn start(_argc: isize, _argv: *const *const u8) -> isize {
    0
}

// Queste funzioni sono usate dal compilatore, ma non
// per uno scheletrico "hello world". Queste sono normalmente
// fornite da libstd.
#[lang = "eh_personality"]
#[no_mangle]
pub extern fn eh_personality() {
}

#[lang = "panic_fmt"]
#[no_mangle]
pub extern fn rust_begin_panic(
    _msg: core::fmt::Arguments,
    _file: &'static str,
    _line: u32) -> ! {
    loop {}
}
```

Per scavalcare lo shim `main` inserito dal compilatore, lo si deve
disabilitare scrivendo `#![no_main]` e poi si deve creare l'appropriato simbolo
con l'ABI corretto il nome corretto, che richiede lo scavalcamento anche
della storpiatura ["mangling"] del nome da parte del compilatore:

```rust,ignore
#![feature(lang_items)]
#![feature(start)]
#![no_std]
#![no_main]

// Tira dentro la libreria libc di sistema per avere ciò che probabilmente
// e' richiesto da crt0.o
extern crate libc;

// Punto d'ingresso in questo programma
#[no_mangle] // assicurati che questo simbolo sia chiamato `main` nell'output
pub extern fn main(_argc: i32, _argv: *const *const u8) -> i32 {
    0
}

// Queste funczioni e questi tratti sono usati dal compilatore, ma non
// per uno scheletrico "ciao mondo". Normalmente sono
// forniti da libstd.
#[lang = "eh_personality"]
#[no_mangle]
pub extern fn eh_personality() {
}

#[lang = "panic_fmt"]
#[no_mangle]
pub extern fn rust_begin_panic(
    _msg: core::fmt::Arguments,
    _file: &'static str,
    _line: u32) -> ! {
    loop {}
}
```

## Altro sui "language item"

Il compilatore attualmente fa alcune assunzioni sui simboli che possono essere
chiamati nel programma. Normalmente queste funzioni sono fornite dalla libreria
standard, ma senza di essa bisogna definire i propri. Questi simboli
sono chiamati "language item" ("voci del linguaggio"), e ognuno di essi ha
un nome interno, e poi hanno una firma a cui ogni implementazione si deve
conformare.

La prima di queste due funzioni, `eh_personality`, è usata dai meccanismo di
fallimento del compilatore. Questa è spesso mappata alla funzione
di personalità del GCC (si veda l'[implementazione di libstd][unwind] per
avere altre informazioni), ma i crate che non fanno scattare un panico
possono stare tranquilli che questa funzione non verrà mai chiamata.
Sia i language item che i nomi di simboli sono `eh_personality`.
 
[unwind]: https://github.com/rust-lang/rust/blob/master/src/libpanic_unwind/gcc.rs

La seconda funzione, `panic_fmt`, viene pure usata dai meccanismi di fallimento
del compilatore. Quando avviene un panico, questo controlla il messaggio che
viene visualizzato sullo schermo. Mentre il nome del language item è
`panic_fmt`, il nome del simbolo è `rust_begin_panic`.
