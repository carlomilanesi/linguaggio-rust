% Unsafe

La principale attrattiva di Rust è costituita dalle potenti garanzie statiche
sul suo comportamento. Ma le verifiche di sicurezza sono prudenti per natura:
ci sono alcuni programmi che sono effettivamente sicuri, ma il compilatore
non è in grado di verificarlo. Per compilare questo genere di programmi,
dobbiamo dire al compilatore di rilassare un po' le sue restrizioni. A questo
scopo, Rust ha una parola-chiave, `unsafe` (insicuro, in italiano).
Il codice che usa `unsafe` ha meno restrizioni del codice normale.

Andiamo a vedere la sintassi, e poi parleremo della semantica. `unsafe` si usa
in quattro contesti. Il primo è per marcare una funzione come insicura:

```rust
unsafe fn pericolo_di_morte() {
    // roba paurosa
}
```

Tutte le funzioni chiamate da [FFI][ffi] devono essere marcate come `unsafe`,
per esempio. Il secondo uso di `unsafe` è per marcare un blocco insicuro:

[ffi]: ffi.html

```rust
unsafe {
    // roba paurosa
}
```

Il terzo è per marcare tratti insicuri:

```rust
unsafe trait Pauroso { }
```

E il quarto è per `impl`ementare uno di quei tratti:

```rust
# unsafe trait Scary { }
unsafe impl Pauroso for i32 {}
```

È importante essere capaci di delimitare esplicitamente il codice che
può avere dei difetti che provocano grossi danni. Se un programma in Rust va
in segmentation fault, si può stare sicuri che la causa è correlata a qualcosa
marcato come `unsafe`.

# Che cosa significa ‘sicuro’?

Sicuro ["safe"], nel contesto di Rust, significa ‘non fa niente di insicuro
["unsafe"]’. È anche importante sapere che ci sono certi comportamenti che
probabilmente non sono desiderabili nel nostro codice, ma che sono
espressamente _non_ insicuri, tra i quali:

* i deadlock
* i leak di memoria o di altre risorse
* uscire senza chiamare i distruttori
* l'overflow di un intero

Rust non può prevenire tutti i tipi di difetti del software. In Rust si può
scrivere, e inevitabilmente si scrive, del codice bacato. Queste cose
ovviamente sono indesiderabili, ma non si possono chiamare specificamente
`unsafe`.

Inoltre, i seguenti sono tutti comportamenti indefiniti in Rust, e devono
essere evitati, anche quando si scrive del codice `unsafe`:

* Accedere simultaneamente ai dati ["data race"]
* Dereferenziare un puntatore grezzo nullo o penzolante
* Leggere memoria [non inizializzata][undef]
* Violare le [regole di aliasing dei puntatori][aliasing] con dei puntatori
  grezzi.
* `&mut T` e `&T` seguono il modello [noalias][noalias] di ambito di LLVM,
  tranne se il `&T` contiene un `UnsafeCell<U>`. Il codice unsafe non deve
  violare queste garanzie di aliasing.
* Mutare un valore/riferimento immutabile senza usare `UnsafeCell<U>`
* Invocare del comportamento indefinito tramite gli intrinseci del compilatore:
  * Indicizzare fuori dai limiti di un oggetto con `std::ptr::offset`
    (l'intrinseco `offset`), eccetto uscire di un solo byte il che è permesso.
  * Usare `std::ptr::copy_nonoverlapping_memory` (gli intrinseci `memcpy32`/
    `memcpy64`) su buffer che si sovrappongono
* Usare i seguenti valori non validi in tipi primitivi, anche in campi privati
  e in legami locali:
  * un riferimento o box nullo o penzolante
  * Un valore diverso da `false` (0) e da `true` (1) in un `bool`
  * Un discriminante in un `enum` non compreso nella definizione del suo tipo
  * Un valore in un `char` che è un surrogato, o è maggiore di `char::MAX`
  * Sequenze di byte non UTF-8 in un `str`
* Svolgere lo stack da codice straniero a codice Rust o da codice Rust a codice
  straniero.

[noalias]: http://llvm.org/docs/LangRef.html#noalias
[undef]: http://llvm.org/docs/LangRef.html#undefined-values
[aliasing]: http://llvm.org/docs/LangRef.html#pointer-aliasing-rules

# I superpoteri di unsafe

In both unsafe functions and unsafe blocks, Rust will let you do three things
that you normally can not do. Just three. Here they are:

1. Access or update a [static mutable variable][static].
2. Dereference a raw pointer.
3. Call unsafe functions. This is the most powerful ability.

That’s it. It’s important that `unsafe` does not, for example, ‘turn off the
borrow checker’. Adding `unsafe` to some random Rust code doesn’t change its
semantics, it won’t start accepting anything. But it will let you write
things that _do_ break some of the rules.

You will also encounter the `unsafe` keyword when writing bindings to foreign
(non-Rust) interfaces. You're encouraged to write a safe, native Rust interface
around the methods provided by the library.

Let’s go over the basic three abilities listed, in order.

## Access or update a `static mut`

Rust has a feature called ‘`static mut`’ which allows for mutable global state.
Doing so can cause a data race, and as such is inherently not safe. For more
details, see the [static][static] section of the book.

[static]: const-and-static.html#static

## Dereference a raw pointer

Raw pointers let you do arbitrary pointer arithmetic, and can cause a number of
different memory safety and security issues. In some senses, the ability to
dereference an arbitrary pointer is one of the most dangerous things you can
do. For more on raw pointers, see [their section of the book][rawpointers].

[rawpointers]: raw-pointers.html

## Call unsafe functions

This last ability works with both aspects of `unsafe`: you can only call
functions marked `unsafe` from inside an unsafe block.

This ability is powerful and varied. Rust exposes some [compiler
intrinsics][intrinsics] as unsafe functions, and some unsafe functions bypass
safety checks, trading safety for speed.

I’ll repeat again: even though you _can_ do arbitrary things in unsafe blocks
and functions doesn’t mean you should. The compiler will act as though you’re
upholding its invariants, so be careful!

[intrinsics]: intrinsics.html
