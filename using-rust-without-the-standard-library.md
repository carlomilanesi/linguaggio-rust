% Usare Rust senza la libreria standard

La libreria standard di Rust fornisce molte funzionalità utili, ma assume
che il sistema ospitante supporti varie caratteristiche: i thread, la rete,
l'allocazione su heap, e altre. Però, ci sono sistemi che non hanno
queste caratteristiche, e Rust può funzionare anche per tali sistemi!
Per farlo, diciamo a Rust che non vogliamo usare la libreria standard
tramite un attributo: `#![no_std]`.

> Nota: Questa caratteristica è tecnicamente stabile, ma ci sono alcune
> avvertenze. Per dirne una, con la versione stabile, si può costruire
> una _libreria_ `#![no_std]`, ma non un _programma_. Per vedere i dettagli
> su come creare programmi senza la libreria standard, si veda
> [il capitolo della versione notturna su `#![no_std]`](no-stdlib.html)

Per usare `#![no_std]`, lo si aggiunga alla radice del proprio crate:

```rust
#![no_std]

fn piu_uno(x: i32) -> i32 {
    x + 1
}
```

Molte delle funzionalità esposte nella libreria standard sono disponibili
anche tramire il [crate `core`](../core/index.html). Quando usiamo
la libreria standard, Rust automaticamente porta `std` nell'ambito corrente,
consentendo di usare le sue caratteristiche senza importazioni esplicite.

Per lo stesso motivo, quando si usa `#![no_std]`, Rust porterà `core`
nell'ambito corrente, come pure [il suo preludio]
(../core/prelude/v1/index.html). Ciò comporta che molto codice continuerà
a funzionare:

```rust
#![no_std]

fn puo_fallire(fallimento: bool) -> Result<(), &'static str> {
    if fallimento {
        Err("non ha funzionato!")
    } else {
        Ok(())
    }
}
```
