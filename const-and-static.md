% `const` e `static`

In Rust si possono definire costanti usando la parola-chiave `const`:

```rust
const N: i32 = 5;
```

Diversamente dai legami [`let`][let], il tipo di un `const` deve essere
annotato.

[let]: variable-bindings.html

Le costanti vivono per tutto il tempo di vita del programma.
Più specificamente, le costanti in Rust non hanno indirizzi fissi in memoria.
Questo avviene perché di fatto vengono espanse inline in ogni posto in cui
sono usate. Per questa ragione, non è garantito che più riferimenti
alla stessa costante si riferiscano allo stesso indirizzo di memoria.

# `static`

Rust fornisce una sorta di funzionalità di ‘variabile globale’ sotto forma
di elementi statici. Sono simili a costanti, ma gli elementi statici non sono
espansi inline quando sono usati. Ciò significa che c'è solo un'istanza
per ogni valore, e si trova in una posizione fissa in memoria.

Ecco un esempio:

```rust
static N: i32 = 5;
```

Diveramente dai legami [`let`][let], si deve annotare il tipo di uno `static`.

Gli static vivono per tutta lo il tempo di vita del programma, e perciò
ogni riferimento immagazzinato in una costante è un [tempo di vita `'static`]
[lifetimes]:

```rust
static NAME: &'static str = "Stefano";
```

[lifetimes]: lifetimes.html

## Mutabilità

La mutabilità si introduce usando la parola-chiave `mut`:

```rust
static mut N: i32 = 5;
```

Dato che questo è mutabile, un thread potrebbe aggiornare `N` mentre un altro
lo sta leggendo, provocando insicurezza di memoria. Il fatto che entrambi
accedano e mutino uno `static mut` è [insicuro [`unsafe`]][unsafe], e quindi
tali accessi devono essere fatti in un blocco `unsafe`:

```rust
# static mut N: i32 = 5;

unsafe {
    N += 1;

    println!("N: {}", N);
}
```

[unsafe]: unsafe.html

Inoltre, ogni tipo immagazzinato in uno `static` deve essere `Sync`, non deve
implementare il tratto [`Drop`][drop].

[drop]: drop.html

# Inizializzazione

Sia `const` che `static` hanno requisiti per dargli un valore. Devono ricevere
un valore che sia un'espressione costante. In altre parole, non si può usare
il risultato di una chiamata di funzione o qualcosa di similmente complesso
eseguito in fase di esecuzione.

# Quale costrutto di dovrebbe usare?

Quasi sempre, se si può  scegliere fra i due, si scelga `const`. È piuttosto
raro voler effettivamente una posizione di memoria associato a tale costante,
e usare un `const` consente ottimizzazioni come la propagazione di costanti
non solo nel proprio crate ma anche nei crate da esso utilizzati.
