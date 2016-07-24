% Distruzione ["Drop"]

Adesso che abbiamo parlato dei tratti, parliamo di un particolare tratto
fornito dalla libreria standard di Rust, [`Drop`][drop], che letteralmente
significa "lasciar cadere", ma qui va inteso nel senso di "distruggere".
Il tratto `Drop` svolge la funzione che in altri linguaggi di programmazione
è svolta dai "distruttori"; infatti fornisce un modo di eseguire del codice
quando un oggetto esce dal suo ambito.

Per esempio:

[drop]: https://doc.rust-lang.org/std/ops/trait.Drop.html

```rust
struct DaDistruggere;

impl Drop for DaDistruggere {
    fn drop(&mut self) {
        println!("Distruzione!");
    }
}

fn main() {
    let x = DaDistruggere;
    
    // fai qualcosa

} // qui x esce di ambito e stampa "Distruzione!"
```

Quando `x` esce di ambito, alla fine della funzione `main()`, il metodo
`drop` verrà eseguito. Tale metodo, che è l'unico di `Drop`, prende
un riferimento mutabile a `self`.

Ecco fatto! La meccanica di `Drop` è molto semplice, ma ci sono alcune
sottigliezze. Per esempio, i valori vengono distrutti nell'ordine opposto
a quello in cui sono stati dichiarati. Ecco un altro esempio:

```rust
struct Esplosivo {
    energia: i32,
}

impl Drop for Esplosivo {
    fn drop(&mut self) {
        println!("BUM per {}!", self.energia);
    }
}

fn main() {
    let petardo = Esplosivo { energia: 1 };
    let granata = Esplosivo { energia: 100 };
    let bomba = Esplosivo { energia: 1000 };
}
```

Questo programma stamperà:

```text
BUM per 1000!
BUM per 100!
BUM per 1!
```

La `bomba` esplode prima della `granata`, che esplode prima del `petardo`,
perché sono stati dichiarati in quell'ordine invertito.
Ultimo dentro, primo fuori.

Dunque a cosa serve il tratto `Drop`? In generale, `Drop` viene usato
per distruggere le risorse associate a una `struct`. Per esempio,
[`Arc<T>`][arc] è un tipo che contiene un oggetto di tipo generico `T` e
un conteggio di riferimenti. Quando viene chiamato il suo metodo `drop`,
questo metodo decrementa il conteggio dei riferimenti,
e se tale conteggio è diventato zero, dealloca l'oggetto contenuto.

[arc]: https://doc.rust-lang.org/std/sync/struct.Arc.html
