% Puntatori grezzi

Rust ha vari diversi tipi di smart pointer nella sua libreria standard, ma
ci sono due tipi che sono molto speciali. Molto della sicurezza di Rust deriva
dalle verifiche in fase di compilazione, ma i puntatori grezzi non hanno
tali garanzie, e quindi sono [insicuri][unsafe] da usare.

`*const T` e `*mut T` sono chiamati ‘puntatori grezzi’ ["raw pointer"] in Rust.
Talvolta, quando si scrivono certi tipi di librerie, c'è bisogno di aggirare
le garanzie di sicurezza di Rust, per varie ragioni. In tali casi, si possono
usare i puntatori grezzi nell'implementazione della propria libreria, pur
esponendo un'interfaccia sicura ai propri utenti. Per esempio, i puntatori `*`
sono consentiti eseguire degli alias, consentendogli di essere usati
per scrivere dei tipi di possesso condiviso, e perfino dei tipi di memoria
condivisa thread-safe (i tipi `Rc<T>` e `Arc<T>` sono entrambi implementati
interamente in Rust).

Ecco alcune cose da ricordare sui puntatori grezzi, che sono diverse
dagli altri tipi di puntatori. Tali puntatori: 

- non è garantito che puntino a memoria valida, e non è nemmeno garanito
  che non siano NULL (diversamente sia da `Box` che da `&`);
- non fanno nessuna pulizia automatica, diversamente da `Box`, e perciò
  richiedono una gestione manuale delle risorse;
- sono dei POD ("plain-old-data"), cioè non spostano il possesso, ancora
  diversamente da `Box`, e perciò il compilatore Rust non può proteggere
  da difetti come l'uso dopo da deallocazione;
- non hanno nessuna forma di tempo di vita, diversamente da `&`, e quindi
  il compilatore non può rilevare i puntatori penzolanti;
- non danno garanzie sull'aliasing né sulla mutabilità, a parte il fatto che
  la mutazione non è consentita direttamente tramite un `*const T`.

# Fondamenti

Creare un puntatore grezzo è perfettamente sicuro:

```rust
let x = 5;
let grezzo = &x as *const i32;

let mut y = 10;
let grezzo_mut = &mut y as *mut i32;
```

Però, dereferenziarne uno non lo è. Questo non funziona:

```rust,ignore
let x = 5;
let grezzo = &x as *const i32;

println!("grezzo punta a {}", *grezzo);
```

Dà quessto errore:

```text
error: dereference of raw pointer requires unsafe function or block [E0133]
println!("grezzo punta a {}", *grezzo);
                              ^~~~
```

Quando si dereferenza un puntatore grezzo, si sta assumendo la responsabilità
che non stia puntando da qualche luogo dove non dovrebbe. Pertanto, serve
`unsafe`:

```rust
let x = 5;
let grezzo = &x as *const i32;

let punta_a = unsafe { *grezzo };

println!("grezzo punta a {}", *grezzo);
```

Per vedere altre operazioni sui puntatori grezzi, si veda la documentazione
della loro [API][rawapi].

[unsafe]: unsafe.html
[rawapi]: ../std/primitive.pointer.html

# FFI

I puntatori grezzi sono utili per l'FFI: i `*const T` e i `*mut T` di Rust
sono simili, rispettivamente, ai `const T*` e ai `T*` del C. Per maggiori
informazioni su questo utilizzo, si consulti il capitolo [FFI][ffi].

[ffi]: ffi.html

# Riferimenti e puntatori grezzi

In fase di esecuzione, un puntatore grezzo `*` e un riferimento che puntano
allo stesso dato hanno un'identica rappresentazione. Di fatto, un riferimento
`&T` sarà implicitamente forzato a un  puntatore grezzo `*const T` nel codice
sicuro e similmente avverrà per le varianti `mut` (entrambe le forzature
possono essere eseguite esplicitamente, rispettivamente, condivisa
`value as *const T` e con `value as *mut T`).

Invece, andare nella direzione opposta, da un `*const` a un riferimento `&`,
non è sicuro. Un `&T` è sempre valido, e così, come minimo, il puntatore grezzo
`*const T`deve puntare a un'istanza valida del tipo `T`. Inoltre, il puntatore
risultante deve soddisfare le leggi di aliasing e mutabilità laws
dei riferimenti. Il compilatore assume che queste proprietà siano vere
per ogni riferimento, indipendentemente da come sono stati creati,
e così ogni conversione da puntatori grezzi sta asserendo che valgano.
Il programmatore *deve* garantirlo.

Il metodo consigliato per questa conversione è:

```rust
// cast esplicito
let i: u32 = 1;
let p_imm: *const u32 = &i as *const u32;

// forzatura implicita
let mut m: u32 = 2;
let p_mut: *mut u32 = &mut m;

unsafe {
    let ref_imm: &u32 = &*p_imm;
    let ref_mut: &mut u32 = &mut *p_mut;
}
```

Lo stile di dereferenziazione `&*x` è preferibile rispetto a usare `transmute`.
L'ultimo stile è molto più potente del necessario, e l'operazione più ristretta
è più difficile da usare scorrettamente; per esempio, richiede che `x`
sia un puntatore, diversamente da `transmute`.
