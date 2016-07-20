% Assembly inline

Per manipolazioni di livello estremamente basso e per ragioni di prestazioni,
qualcuno potrebbe desiderare di controllare direttamente la CPU. Rust consente
di farlo scrivendo codice assembly inline, tramite la macro `asm!`.

```rust,ignore
asm!(template assembly
   : operandi di output
   : operandi di input
   : clobber
   : opzioni
   );
```

Any use of `asm` is feature gated (requires `#![feature(asm)]` on the
crate to allow) and of course requires an `unsafe` block.

> **Nota**: qui gli esempi sono dati nell'assembly x86/x86-64, però
> sono supportate tutte le piattaforme.

## Template assembly

The `template assembly` è l'unico parametro obbligatorio e deve essere
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

(The `feature(asm)` and `#[cfg]`s are omitted from now on.)

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

Gli operandi di input e di output hanno lo stesso formato: `:
"constraints1"(expr1), "constraints2"(expr2), ..."`. Le espressioni
degli operandi di output devono essere lvalue mutabili, o non ancora assegnati:

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
// Put the value 0x200 in eax
asm!("mov $$0x200, %eax" : /* no outputs */ : /* no inputs */ : "eax");
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

1. *volatile* - specifying this is analogous to
   `__asm__ __volatile__ (...)` in gcc/clang.
2. *alignstack* - certain instructions expect the stack to be
   aligned a certain way (i.e. SSE) and specifying this indicates to
   the compiler to insert its usual stack alignment code
3. *intel* - use intel syntax instead of the default AT&T.

```rust
# #![feature(asm)]
# #[cfg(any(target_arch = "x86", target_arch = "x86_64"))]
# fn main() {
let result: i32;
unsafe {
   asm!("mov eax, 2" : "={eax}"(result) : : : "intel")
}
println!("eax is currently {}", result);
# }
```

## More Information

The current implementation of the `asm!` macro is a direct binding to [LLVM's
inline assembler expressions][llvm-docs], so be sure to check out [their
documentation as well][llvm-docs] for more information about clobbers,
constraints, etc.

[llvm-docs]: http://llvm.org/docs/LangRef.html#inline-assembler-expressions
