% Drop [caduta]

Adesso che abbiamo parlato dei tratti, parliamo di un particolare tratto
fornito dalla libreria standard di Rust, [`Drop`][drop]. Il tratto `Drop`
fornisce un modo di eseguire del codice quando un valore esce dal suo ambito.
Per esempio:

[drop]: ../std/ops/trait.Drop.html

```rust
struct HaDrop;

impl Drop for HaDrop {
    fn drop(&mut self) {
        println!("Caduta!");
    }
}

fn main() {
    let x = HaDrop;

    // fai qualcosa

} // qui x esce di ambito
```

Quando `x` esce di ambito alla fine della funzione `main()`, il codice
di `Drop` verrà eseguito. `Drop` ha un solo metodo, che pure si chiama
`drop()`. Prende un riferimento mutabile a `self`.

Ecco fatto! La meccanica di `Drop` è molto semplice, ma ci sono alcune
sottigliezze. Per esempio, i valori vengono fatti cadere nell'ordine opposto
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
    let petardo = Esplosivo { energia: 2 };
    let tnt = Esplosivo { energia: 100 };
}
```

Questo stamperà:

```text
BUM per 100!
BUM per 2!
```

Il `tnt` esplode prima del `petardo`, perché è stato dichiarato dopo.
Ultimo dentro, primo fuori.
Dunque a cosa serve il tratto `Drop`? In generale, `Drop` viene usato
per distruggere le risorse associate a uno `struct`. Per esempio,
il [tipo `Arc<T>`][arc] è un tipo con conteggio di riferimenti. Quando
viene chiamato `drop`, decrementa il conteggio dei riferimenti, e se il numero
totale dei riferimenti è diventato zero, distrugge il valore contenuto.

[arc]: ../std/sync/struct.Arc.html
