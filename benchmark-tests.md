% Collaudo delle prestazioni (benchmark)

Rust supporta la creazione di benchmark, che possono collaudare
le prestazioni del proprio codice. Rendiamo il file `src/lib.rs` come questo
(senza commenti):

```rust,ignore
#![feature(test)]

extern crate test;

pub fn aggiungi_due(a: i32) -> i32 {
    a + 2
}

#[cfg(test)]
mod tests {
    use super::*;
    use test::Bencher;

    #[test]
    fn funziona() {
        assert_eq!(4, aggiungi_due(2));
    }

    #[bench]
    fn bench_aggiungi_due(b: &mut Bencher) {
        b.iter(|| aggiungi_due(2));
    }
}
```

Si noti il feature gate `test`, che abilita questa caratteristica instabile.

Abbiamo importato il crate `test`, che contiene il nostro supporto
ai benchmark. Abbiamo anche una nuova funzione, con l'attributo `bench`.
Diversamente dai testi normali, che non prendono argomenti, i benchmark
prendono un `&mut Bencher`. Tale `Bencher` fornisce un metodo `iter`,
che prende una chiusura. Tale chiusura contiene il codice di cui vorremmo
collaudare le prestazioni.

I benchmark possono essere eseguiti con il comando `cargo bench`:

```bash
$ cargo bench
   Compiling adder v0.0.1 (file:///home/steve/tmp/adder)
     Running target/release/adder-91b3e234d4ed382a

running 2 tests
test tests::funziona ... ignored
test tests::bench_aggiungi_due ... bench:         1 ns/iter (+/- 0)

test result: ok. 0 passed; 0 failed; 1 ignored; 1 measured
```

Il nostro test che non è un benchmark è stato ignorato. Si potrebbe
aver notato che `cargo bench` impiega un po' di più di `cargo test`. Questo
è dovuto al fatto che Rust esegue i nostri benchmark numerose volte,
e poi ne fa la media. Siccome facciamo pochissimo lavoro in questo esempio,
otteniamo `1 ns/iter (+/- 0)`, ma verrebe mostrata la varianza,
se ce ne fosse una.

Consiglio sulla scrittura dei benchmark:


* Spostare il codice di impostazione fuori dal ciclo `iter`;
  mettere dentro solamente la parte che si vuole misurare.
* Fare in modo che il codice faccia "le stesse cose" ad ogni iterazione;
  non accumulare né cambiare stato.
* Rendere idempotente anche la funzione esterna; l'esecutore del benchmark
  è probabile che lo esegua molte volte.
* Rendere breve e veloce il ciclo `iter` interno, così che le esecuzioni
  del benchmark siano veloci, e il calibratore possa regolare finemente
  la lunghezza dell'esecuzione.
* Fare in modo che il codice nel ciclo `iter` faccia qualcosa di semplice,
  per aiutare a individuare i miglioramenti (o i peggioramenti)
  delle prestazioni.

## Tranello: ottimizzazioni

C'è un'altra parte delicata nella scrittura dei benchmark: i benchmark
compilati attivando le ottimizzazioni possono essere talmente
cambiati dall'ottimizzatore che il benchmark non sta più collaudando
quello che ci si aspetta. Per esempio, il compilatore potrebbe riconoscere
che alcuni calcoli non hanno effetti esterni, e quindi toglierli del tutto.

```rust,ignore
#![feature(test)]

extern crate test;
use test::Bencher;

#[bench]
fn bench_xor_1000_int(b: &mut Bencher) {
    b.iter(|| {
        (0..1000).fold(0, |vecchio, nuovo| vecchio ^ nuovo);
    });
}
```

dà il seguente risultato

```text
running 1 test
test bench_xor_1000_int ... bench:         0 ns/iter (+/- 0)

test result: ok. 0 passed; 0 failed; 0 ignored; 1 measured
```

L'esecutore del benchmark offre due modi per evitarlo. O la chiusura
ricevuta dal metodo `iter` può restituire un valore arbitrario
che costringe l'ottimizzatore a considerare usato il risultato, e assicura
che non possa togliere l'intera elaborazione. Ciò potrebbe essere fatto,
nell'esempio sopra, modificando la chiamata a `b.iter` come:

```rust
# struct X;
# impl X { fn iter<T, F>(&self, _: F) where F: FnMut() -> T {} } let b = X;
b.iter(|| {
    // Si noti la mancanza di `;`. Si poteva anche usare un `return` esplicito.
    (0..1000).fold(0, |vecchio, nuovo| vecchio ^ nuovo)
});
```

Oppure, l'altra opzione è chiamare la funzione generica `test::black_box`,
che è una "scatola nera" opaca all'ottimizzatore, e quindi lo costringe
a considerare usato ogni argomento.

```rust
#![feature(test)]

extern crate test;

# fn main() {
# struct X;
# impl X { fn iter<T, F>(&self, _: F) where F: FnMut() -> T {} } let b = X;
b.iter(|| {
    let n = test::black_box(1000);

    (0..n).fold(0, |a, b| a ^ b)
})
# }
```

Nessuna di queste legge né scrive il valore, e costano pochissimo
per piccoli valori. I valori più grandi possono essere passati
indirettamente per ridurre lo spreco (per es. `black_box(&huge_struct)`).

Eseguire una o l'altra delle suddette modifiche dà i seguenti risultati
di benchmark:

```text
running 1 test
test bench_xor_1000_ints ... bench:       131 ns/iter (+/- 3)

test result: ok. 0 passed; 0 failed; 0 ignored; 1 measured
```

Però, l'ottimizzatore può ancora modificare un test in maniera
indesiderabile perfino quando si usa una delle tecniche suddette.
