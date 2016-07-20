% Iteratori

Parliamo dei cicli.

Avevamo già parlato dei cicli `for` di Rust. Ecco un esempio:

```rust
for x in 0..10 {
    println!("{}", x);
}
```

Adesso che abbiamo visto vari aspetti di Rust, possiamo descrivere
in dettaglio come funzionano.
I range (lo `0..10`) sono degli 'iteratori'. Un iteratore è qualcosa su cui
possiamo chiamare ripetutamente il metodo `.next()`, e ottenere ogni volta
un elemento diverso, appartenente una sequenza di oggetti.

(Tra l'altro, un range con due punti, come `0..10` include il limite sulla
sinistra (e quindi, in questo caso, inizia da 0), ed esclude il limite sulla
destra (e quindi, in questo caso, finisce a 9). Un matematico scriverebbe
"[0, 10)". Per ottenere un range che arriva a 10 si può scrivere `0..11`.)

Ecco come usarlo:

```rust
let mut range = 0..10;

loop {
    match range.next() {
        Some(x) => {
            println!("{}", x);
        },
        None => { break }
    }
}
```

Abbiamo creato una variabile mutabile al range, che è il nostro iteratore.
Poi abbiamo eseguito un `loop`, contenente un `match`. Questo `match`
si applica al risultato dell'espressione `range.next()`, che ci dà
un riferimento al prossimo valofe dell'iteratore. In questo caso, il metodo
`next` rende un oggetto di tipo `Option<i32>`, il quale sarà un `Some(i32)`
quando viene preso un valore dalla sequenza, e un `None` quando non ci sono
più valori nella sequenza. Se otteniamo un `Some(i32)`, lo stampiamo,
mentre se otteniamo `None`, saltiamo fuori dal ciclo usando `break`.

Questa porzione di codice fa sostanzialmente lo stesso della nostra versione
del ciclo `for`. Il ciclo `for` è un modo comodo di scrivere questo costrutto
`loop`/`match`/`break`.

I cicli `for` non sono l'unica cosa che usa iteratori, comunque. Scrivere
il proprio iteratore comporta implementare il tratto `Iterator`. Mentre farlo
esulerebbe da questa guida, Rust fornisce vari utili iteratori
per compiere vari compiti. Ma prima, vediamo alcuni limitazioni dei range.

I range sono molto primitivi, e spesso si possono usare alternative migliori.
Si consideri il seguente anti-pattern di Rust: usare i range per emulare
un ciclo `for` in stile-C. Supponiamo di aver bisogno di iterare sul contenuto
di un vettore. Si può essere tentati di scrivere così:

```rust
let numeri = vec![1, 2, 3];

for i in 0..numeri.len() {
    println!("{}", numeri[i]);
}
```

Questo è nettamente peggio che usare un iteratore. Infatti, si può, e si
dovrebbe, iterare sui vettori direttamente, e quindi scrivere:

```rust
let numeri = vec![1, 2, 3];

for numero in &numeri {
    println!("{}", numero);
}
```

Ci sono due ragioni per farlo. Primo, questa notazione esprime più
esplicitamente cosa si intende fare. Stiamo iterando su tutti gli elementi
del vettore, e non iterando sulle posizioni degli elementi del vettore,
e poi accedendo agli elementi del vettore in base alla loro posizione.
Secondo, questa versione è più efficiente: la prima versione dovrà
verificare che gli indici non sforino i limiti, nell'espressione `numeri[i]`.
Invece nel secondo esempio, non c'è bisogno di verificare i limiti,
dato che l'iteratore si occupa di fornire un riferimento a ogni elemento
del vettore. Questo effetto è molto comune con gli iteratori: si possono
evitare le verifiche dei limiti, ma avere comunque la garanzia che non
vengano oltrepassati.

Qui c'è un altro dettaglio che non è chiaro al 100% a causa di come funziona
`println!`. `num` è effettivamente di tipo `&i32`. Cioè, è un riferimento
a un `i32`, non è un `i32` esso stesso. `println!` si occupa di gestire
la dereferenziazione, e quindi non la vediamo. Anche il seguente codice
funziona bene:

```rust
let numeri = vec![1, 2, 3];

for numero in &numeri {
    println!("{}", *numero);
}
```

Adesso stiamo dereferenziando `numero` esplicitamente. Perché otteniamo
dei riferimenti da `&nums`? In primo luogo, perché glielo abbiamo chiesto
esplicitamente scrivendo `&`. E in secondo luogo, se ci avesse dato
il dato stesso, noi saremmo diventati i suoi proprietari, il che
avrebbe comportato copiare i dati e darci la copia. Con i riferimenti,
stiamo solamente prendendo in prestito un riferimento al dato,
e quindi ci sta passando solamente un riferimento, senza bisogno di spostarlo.

Perciò, adesso che abbiamo stabilito che spesso i range non sono ciò che
si vorrebbe, parliamo invece di ciò si vorrebbe.

Ci sono tre ampie classi di cose che sono rilevanti qui: gli iteratori,
gli *adattatori di iteratore*, e i *consumatori*. Ecco alcune definizioni:

* Gli *iteratori* producono una sequenza di valori.
* Gli *adattatori di iteratore* operano su un iteratore, producendo un nuovo
  iteratore con una diversa sequenza in uscita.
* I *consumatori* operano su un iteratore, producendo un insieme finale
  di valori.

Dapprima parliamo dei consumatori, dato che abbiamo già visto un tipo
di iteratori, i range.

## Consumatori

Un *consumatore* opera su un iteratore, rendendo qualche tipo di valore
o di valori.
Il consumatore più comune è `collect()`. Il seguente codice non è proprio
corretto, ma rende l'idea:

```rust,ignore
let da_uno_a_cento = (1..101).collect();
```

Come si vede, si chiama `collect()` sul nostro iteratore. `collect()`  prende
tanti valori quanti gliene da l'iterator, e rende una collezione dei risultati.
Perciò questo non compila? Rust non riesce a determinare quale tipo collezione
creare, e perciò bisogna farglielo sapere.
Ecco la versione che compila:

```rust
let da_uno_a_cento = (1..101).collect::<Vec<i32>>();
```

Come già detto, la sintassi `::<>` consente di suggerire il tipo,
e perciò indichiamo che vogliamo un vettore di interi. Però non c'è sempre
bisogno di specificare l'intero tipo. Usando un `_` si può fornire
un suggerimento parziale:

```rust
let da_uno_a_cento = (1..101).collect::<Vec<_>>();
```

Quest codice dice "Raccogli in un `Vec<T>`, ma capiscilo da solo il tipo `T`."
`_` è talvolta chiamato "segnaposto di tipo" ("type placeholder") per questa
ragione.

`collect()` è il consumatore più comune, ma ce ne sono anche altri.
Uno è `find()`:

```rust
let maggiore_di_quaranta_due = (0..100).find(|x| *x > 42);

match maggiore_di_quaranta_due {
    Some(_) => println!("Trovata una corrispondenza!"),
    None => println!("Nessuna corrispondenza trovata :("),
}
```

`find` prende una chiusura, e lavora su un riferimento a ogni elemento
di un iteratore. Questa chiusura rende `true` se l'elemento è l'elemento
che stiamo cercando, e `false` altrimenti. `find` rende il primo elemento
che soddisfa il predicato specificato. Siccome potremmo non trovare
nessun elemento corrispondente, `find` rende un `Option` invece
dell'elemento stesso.

Un altro consumatore importante è `fold`. Ecco come si presenta:

```rust
let somma = (1..4).fold(0, |somma_parziale, x| somma_parziale + x);
```

`fold()` è un consumatore che si presenta così:
`fold(base, |accumulatore, elemento| ...)`. Prende due argomenti: il primo
è un elemento chiamato *base*. Il secondo è una chiusura che a sua volta
prende due argomenti: il primo è chiamato *accumulator*, e il second è un
*elemento*. Ad ogni iterazione, la chiusura viene chiamata, e il risultato
diventa il valore dell'accumulatore allìiterazione successiva. Alla prima
iterazione, la base diventa il valore dell'accumulatore.

Va be', non è chiarissimo. Esaminiamo i valori di tutte queste cose
in questo iteratore:

| base | accumulatore | elemento | risultato della chiusura |
|------|--------------|----------|--------------------------|
| 0    | 0            | 1        | 1                        |
| 0    | 1            | 2        | 3                        |
| 0    | 3            | 3        | 6                        |

Abbiamo chiamato `fold()` con questi argomenti:

```rust
# (1..4)
.fold(0, |somma_parziale, x| somma_parziale + x);
```

Così, `0` è la nostra base, `somma_parziale` è il nostro accumulatore,
e `x` è il nostro elemento.  Alla prima iterazione, abbiamo impostato
`somma_parziale` a `0`, e `x` è il primo elemento di `numeri`, cioè
`1`. Poi abbiamo sommato `somma_parziale` e `x`, ottenendo `0 + 1 = 1`.
Alla seconda iterazione, quel valore è diventato il nostro accumulatore,
`somma_parziale`, e l'elemento è il secondo elemento dell'array, `2`.
`1 + 2 = 3`, e così quello è diventato il valore dell'accumulatore
per l'ultima iterazione. In quell'iterazione, `x` è l'ultimo elemento, `3`,
e `3 + 3 = 6`, che è il risultato finale della nostra somma. `1 + 2 + 3 = 6`,
e quello è il risultato che abbiamo ottenuto.

Accidenti! `fold` può sembrare un po' strano le prime volte che lo si vede,
ma quando lo si afferra, lo si può usare un po' dappertutto. Ogni volta che
che si ha una lista di oggetti, e se ne vuole ottenere un risultato singolo,
`fold` può essere appropriato.

I consumatori sono importanti a causa di una proprietà aggiuntiva
degli iteratori di cui non abbiamo ancora parlato: la pigrizia ("laziness").
Parliamo ancora un po' degli iteratori, e vedremo perché i consumatori
sono così importanti.

## Iteratori

Come abbiamo detto prima, un iteratore è qualcosa di cui possiamo chiamare
ripetutamente il metodo `.next()`, e ci dà una sequenza di oggetti.
Siccome per ottenere l'oggetto si deve chiamare un metodo, questo significa
che gli iteratori possono essere *pigri*  ("lazy") e non generare subito
tutti i valori. Questo codice, per esempio, non genera effettivamente
i numeri da `1` a `99`, bensì crea un oggetto che si limita a rappresentare
tale sequenza:

```rust
let nums = 1..100;
```

E dato che non abbiamo fatto niente con tale range, non ha proprio generato
la sequenza. Aggiungiamo il consumatore:

```rust
let numeri = (1..100).collect::<Vec<i32>>();
```

Adesso, `collect()` richiederà che il range gli dia tutti i numeri, e quindi
farà il lavoro di generare la sequenza.

I range sono uno dei due iteratori di base che vedremo. L'altro è `iter()`.
`iter()` può trasformare un vettore in un semplice iteratore che ci dà
ogni elemento, uno alla volta:

```rust
let numeri = vec![1, 2, 3];

for numero in numeri.iter() {
   println!("{}", numero);
}
```

Questi due iteratori di base dovrebbero servirci bene. Ci sono alcuni
iteratori più avanzati, tra cui alcuni che sono infiniti.

Abbiamo parlato abbastanza degli iteratori. Gli adattatori degli iteratori
sono l'ultimo concetto di cui dobbiamo parlare a proposito degli iteratori.
Avanti!

## Adattatori degli iteratori

Gli *adattatori degli iteratori* prendono un iteratore e lo modificano
in qualche modo, producendo un nuovo iteratore. Quello più semplice
si chiama `map`:

```rust,ignore
(1..100).map(|x| x + 1);
```

`map` is called upon another iterator, and produces a new iterator where each
element reference has the closure it's been given as an argument called on it.
So this would give us the numbers from `2-100`. Well, almost! If you
compile the example, you'll get a warning:

```text
warning: unused result which must be used: iterator adaptors are lazy and
         do nothing unless consumed, #[warn(unused_must_use)] on by default
(1..100).map(|x| x + 1);
 ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
```

Laziness strikes again! That closure will never execute. This example
doesn't print any numbers:

```rust,ignore
(1..100).map(|x| println!("{}", x));
```

If you are trying to execute a closure on an iterator for its side effects,
use `for` instead.

There are tons of interesting iterator adaptors. `take(n)` will return an
iterator over the next `n` elements of the original iterator. Let's try it out
with an infinite iterator:

```rust
for i in (1..).take(5) {
    println!("{}", i);
}
```

This will print

```text
1
2
3
4
5
```

`filter()` is an adapter that takes a closure as an argument. This closure
returns `true` or `false`. The new iterator `filter()` produces
only the elements that the closure returns `true` for:

```rust
for i in (1..100).filter(|&x| x % 2 == 0) {
    println!("{}", i);
}
```

This will print all of the even numbers between one and a hundred.
(Note that, unlike `map`, the closure passed to `filter` is passed a reference
to the element instead of the element itself. The filter predicate here uses
the `&x` pattern to extract the integer. The filter closure is passed a
reference because it returns `true` or `false` instead of the element,
so the `filter` implementation must retain ownership to put the elements
into the newly constructed iterator.)

You can chain all three things together: start with an iterator, adapt it
a few times, and then consume the result. Check it out:

```rust
(1..)
    .filter(|&x| x % 2 == 0)
    .filter(|&x| x % 3 == 0)
    .take(5)
    .collect::<Vec<i32>>();
```

This will give you a vector containing `6`, `12`, `18`, `24`, and `30`.

This is just a small taste of what iterators, iterator adaptors, and consumers
can help you with. There are a number of really useful iterators, and you can
write your own as well. Iterators provide a safe, efficient way to manipulate
all kinds of lists. They're a little unusual at first, but if you play with
them, you'll get hooked. For a full list of the different iterators and
consumers, check out the [iterator module documentation](../std/iter/index.html).
