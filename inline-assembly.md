% Assembly inline

Per manipolazioni di livello estremamente basso e per ragioni di prestazioni,
qualcuno potrebbe desiderare di controllare direttamente la CPU. Rust consente
di farlo scrivendo codice assembly inline, tramite la macro `asm!`.

```rust,ignore
asm!(template di assembly
   : operandi di output
   : operandi di input
   : clobber
   : opzioni
   );
```

Ogni uso di `asm` è una caratteristica attivata da un gate (cioè richiede
`#![feature(asm)]` sul crate per consentirla) e naturalmente richiede
un blocco `unsafe`.

> **Nota**: qui gli esempi sono dati nell'assembly x86/x86-64, però
> sono supportate tutte le piattaforme.

## Template di assembly

Il "template di assembly" è l'unico parametro obbligatorio e deve essere
una stringa letterale (per es. `""`)

```rust
#![feature(asm)]

#[cfg(any(target_arch = "x86", target_arch = "x86_64"))]
fn foo() {
    unsafe {
        asm!("NOP");
    }
}

// altre piattaforme
#[cfg(not(any(target_arch = "x86", target_arch = "x86_64")))]
fn foo() { /* ... */ }

fn main() {
    // ...
    foo();
    // ...
}
```

(Da qui in avanti, le istruzioni `feature(asm)` e `#[cfg]` saranno omesse.)

Gli operandi di output, gli operandi di input, i clobber e le opzioni sono
tutti facoltativi, ma si devono sempre mettere i relativi caratteri `:`:

```rust
# #![feature(asm)]
# #[cfg(any(target_arch = "x86", target_arch = "x86_64"))]
# fn main() { unsafe {
asm!("xor %eax, %eax"
    :
    :
    : "eax"
   );
# } }
```

Anche gli spazi di separazione non hanno importanza:

```rust
# #![feature(asm)]
# #[cfg(any(target_arch = "x86", target_arch = "x86_64"))]
# fn main() { unsafe {
asm!("xor %eax, %eax" ::: "eax");
# } }
```

## Operandi

Gli operandi di input e di output hanno lo stesso formato:
`: "constraints1"(expr1), "constraints2"(expr2), ..."`.
Le espressioni degli operandi di output devono essere l-value mutabili,
o non ancora assegnati:

```rust
# #![feature(asm)]
# #[cfg(any(target_arch = "x86", target_arch = "x86_64"))]
fn add(a: i32, b: i32) -> i32 {
    let c: i32;
    unsafe {
        asm!("add $2, $0"
             : "=r"(c)
             : "0"(a), "r"(b)
             );
    }
    c
}
# #[cfg(not(any(target_arch = "x86", target_arch = "x86_64")))]
# fn add(a: i32, b: i32) -> i32 { a + b }

fn main() {
    assert_eq!(add(3, 14159), 14162)
}
```

Tuttavia, se si desiderano usare veri operandi in questa posizione,
si devono mettere delle graffe `{}` intorno al registro che si desidera, e
si deve mettere la dimensione specifica dell'operando. Questo è utile
per la programmazione a bassissimo livello, in cui è importante
quale registro si usa:

```rust
# #![feature(asm)]
# #[cfg(any(target_arch = "x86", target_arch = "x86_64"))]
# unsafe fn read_byte_in(port: u16) -> u8 {
let result: u8;
asm!("in %dx, %al" : "={al}"(result) : "{dx}"(port));
result
# }
```

## Clobber

Alcune istruzioni modificano i valori di alcuni registri, i quali potrebbero
altrimenti contenere valori diversi, e quindi si usa la lista dei clobber
per indicate al compilatore di non assumere che i valori caricati in quei
registri rimangano validi.

```rust
# #![feature(asm)]
# #[cfg(any(target_arch = "x86", target_arch = "x86_64"))]
# fn main() { unsafe {
// Metti il valore 0x200 in eax
asm!("mov $$0x200, %eax" : /* nessun output */ : /* nessun input */ : "eax");
# } }
```

I registri di input e di output non devono essere elencati dato che
tale informazione è già comunicat dai relativi vincoli. Invece, ogni altro
registro usato implicitamente o esplicitamente dovrebbe essere elencato.

Se il codice assembly modifica il codice di condizione, il registro `cc`
dovrebbe essere specificato come uno dei clobber. Similmente, se il codice
assembly modifica la memoria, si dovrebbe specificare anche `memory`.

## Opzioni

L'ultima sezione, `opzioni` è specifica di Rust. Il formato è una sequenza
di stringhe letterali separate da virgole (per es. `:"foo", "bar", "baz"`).
Serve a specificare alcune informazioni aggiuntive riguardo l'assembly inline:

Le opzioni attualmente valide sono:

1. *volatile* - specificare questa è analogo a scrivere
   `__asm__ __volatile__ (...)` in gcc/clang.
2. *alignstack* - certe istruzioni si aspettano che lo stack sia
   allineato in un certo modo (per es. SSE) e specificare questa indica al
   compilatore di inserire il suo solito codice di allineamento dello stack
3. *intel* - usere la sintassi Intel invece di quella AT&T, che è il default.

```rust
# #![feature(asm)]
# #[cfg(any(target_arch = "x86", target_arch = "x86_64"))]
# fn main() {
let result: i32;
unsafe {
   asm!("mov eax, 2" : "={eax}"(result) : : : "intel")
}
println!("eax è attualmente {}", result);
# }
```

## Altre informazioni

L'attuale implementazione della macro `asm!` è un legame diretto alle
[espressioni assembler inline di LLVM][llvm-docs], quindi ci si deve
assicurare di leggere [anche la loro documentazione][llvm-docs] per avere
ulteriori informazioni sui clobbers, i vincoli, ecc.

[llvm-docs]: http://llvm.org/docs/LangRef.html#inline-assembler-expressions
