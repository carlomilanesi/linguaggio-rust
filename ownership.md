% Possesso

Questa è la prima delle tre sezioni che presentano il sistema di possesso
di Rust. Questa è una delle caratteristiche più distintive e avvincenti
di Rust, con la quale gli sviluppatori Rust dovrebbero diventare familiari.
Il possesso è il modo in cui Rust raggiunge il suo maggior obiettivo, la sicurezza
di accesso alla memoria. Ci sono alcuni concetti distinti, ognuno descritto
in una sezione distinta:

* il possesso, che è la sezione attuale
* i [prestiti][prestiti], e le caratteristiche a loro associate,
  i ‘riferimenti’
* i [tempi di vita][tempi di vita], un avanzato concetto di prestito

Queste tre sezioni sono correlate, e seguono un ordine. Bisognerà leggerli
tutti e tre per capire pienamente il sistema di possesso.

[prestiti]: references-and-borrowing.html
[tempi di vita]: lifetimes.html

# Meta

Prima di passare ai dettagli, due appunti importanti sul sistema di possesso.

Rust ha un'attenzione particolare sulla sicurezza e sulla velocità.
Raggiunge questi obiettivi tramite molte ‘astrazioni a costo zero’, il che
significa che in Rust, le astrazioni costano il meno possibile
al fine di farle funzionare. Il sistema di possesso è un esempio primario
di astrazione a costo zero. Tutta l'analisi di cui parleremo in questa guida
viene _fatta in fase di compilazione_. Non si paga nessun costo in fase
di esecuzione per queste funzionalità.

Però, questo sistema ha un certo costo: il tempo di apprendimento. Molti nuovi
utenti di Rust sperimentano qualcosa che ci piace chiamare ‘combattere
con il verificatore dei prestiti’, che è la parte del compilatore Rust che
si rifiuta di compilare un programma che l'autore pensa essere valido. Ciò
accade spesso perché il modello mentale del programmatore di come il possesso
dovrebbe funzionare non combacia con le regole effettivamente implementate
da Rust.
Dapprima tutti sperimentano cose simili. Però, c'è una buona notizia:
gli sviluppatori Rust più esperti riferiscono che una volta che lavorano con
le regole del sistema di possesso per un periodo di tempo, combattono sempre
meno con il verificatore dei prestiti.

Con questo in mente, vediamo in cosa consiste il possesso.

# Possesso

I [legami di variabili][legami] hanno una proprietà in Rust: ‘possiedono’
quello a cui sono legati. Ciò significa che quando un legame esce di ambito,
Rust libererà le risorse legate. Per esempio:

```rust
fn foo() {
    let v = vec![1, 2, 3];
}
```

Quando `v` viene nell'ambito, viene creato un nuovo [vettore][vettori]
sullo [stack][stack], e alloca spazio sullo [heap][heap] per i suoi elementi.
Quando `v` esce di ambito alla fine di `foo()`, Rust ripulirà ogni cosa
correlata al vettore, anche la memoria allocata sullo heap. Questo avviene
deterministicamente alla fine dell'ambito.

Tratteremo i [vettori] in dettaglio più avanti in questo capitolo; li usiamo
qui solamente come esempio di un tipo che alloca spazio sullo heap in fase
di esecuzione. Si comportano come [array], eccetto che la loro dimensione
può cambiare chiamando `push()` per aggiungere loro altri elementi.

I vettori hanno un [tipo generico][generici] `Vec<T>`, perciò in questo esempio
`v` sarà di tipo `Vec<i32>`. Tratteremo i generici in dettaglio più avanti
in questo capitolo.

[array]: primitive-types.html#arrays
[vettori]: vectors.html
[heap]: the-stack-and-the-heap.html#the-heap
[stack]: the-stack-and-the-heap.html#the-stack
[legami]: variable-bindings.html
[generici]: generics.html

# Semantica di spostamento

Però qui c'è qualche altra sottigliezza: Rust assicura che ci sia _esattamente
un_ legame a ogni data risorsa. Per esempio, se abbiamo un vettore, possiamo
assegnarlo a un altro legame:

```rust
let v = vec![1, 2, 3];
let v2 = v;
```

Ma, se dopo proviamo a usare `v`, otteniamo un  errore:

```rust,ignore
let v = vec![1, 2, 3];
let v2 = v;
println!("v[0] vale: {}", v[0]);
```

L'errore si presenta così:

```text
error: use of moved value: `v`
println!("v[0] vale: {}", v[0]);
                          ^
```

Una cosa simile accade se definiamo una funzione che prende possesso
dell'argomento, e proviamo a usare qualcosa dopo che l'abbiamo passato
come argomento:

```rust,ignore
fn prendi(v: Vec<i32>) {
    // ciò che accade qui dentro non è importante.
}

let v = vec![1, 2, 3];
prendi(v);
println!("v[0] vale: {}", v[0]);
```

Stesso errore: ‘use of moved value’. Quando si trasferisce il possesso
di un oggetto da un legame a un altro, si dice che l'oggetto a cui si fa
riferimento è stato ‘spostato’. Qui non ci vuole qualche sorta di annotazione
speciale, è il normale comportamento di Rust.

## I dettagli

La ragione per cui non si può più usare un legame dopo che l'oggetto è stato
spostato è sottile, ma importante.

Quando scriviamo del codice come questo:

```rust
let x = 10;
```

Rust alloca sullo [stack][sh] della memoria per un intero [i32], copia i bit
che rappresentano il valore 10 alla memoria allocata, e lega il nome
della variabile x a questa regione di memoria per poterna riferire in seguito.

[i32]: primitive-types.html#numeric-types

Adesso consideriamo il seguente frammento di codice:

```rust
let v = vec![1, 2, 3];
let mut v2 = v;
```

La prima riga alloca sullo stack della memoria per l'oggetto vettore `v`, come
ha fatto per `x` precedentemente. Ma in aggiunta a ciò, alloca anche
della memoria sullo [heap][sh] per i dati effettivi (`[1, 2, 3]`). Rust copia
l'indirizzo di questa allocazione sullo heap al puntatore interno, che fa parte
dell'oggetto vettore posto sullo stack (chiamiamolo "puntatore ai dati").

Vale la pena evidenziare (anche al rischio di affermare l'ovvio) che l'oggetto
vettore e i suoi dati vivono in regioni di memoria separate, invece di essere
un'unica allocazione di memoria contigua (a causa di ragioni che non
approfondiremo in questo momento). Queste due parti del vettore (quella sullo
stack e quella sullo heap) devono accordarsi l'un l'altra in ogni momento
riguardo a cose come la lunghezza, la capacità, ecc.

Quando si sposta `v` in `v2`, Rust effettivamente fa una copia bit-a-bit
dell'oggetto vettore `v` nell'allocazione sullo stack rappresentata da `v2`.
Questa copia superficiale non crea una copia dell'allocazione sullo heap
contenente i dati effettivi.
Il che significa che ci sarebbero due puntatori al contenuto del vettore
entrambi che puntano alla stessa allocazione di memoria sullo heap.
Se si potesse accedere sia a `v` che a `v2` nello stesso tempo, si violerebbe
la garanzia di sicurezza di Rust, introducendo un'accesso concorrente ai dati.

Per esempio, se troncassimo il vettore ad appena due elementi tramite `v2`:

```rust
# let v = vec![1, 2, 3];
# let mut v2 = v;
v2.truncate(2);
```

e `v` fosse ancora accessibile, finiremmo con un vettore non valido, dato che
`v` non saprebbe che i dati sullo heap sono stati troncati. Adesso, la parte
del vettore `v` sullo stack non concorda con la parte corrispondente sullo
heap. `v` pensa ancora che ci siano tre elementi nel vettore e permetterebbe
di accedere all'elemento non esistente `v[2]`, ma, come potremmo già sapere,
questa è una ricetta per il disastro. Specialmente perché potrebbe condurre
a un segmentation fault o peggio consentire a un utente non autorizzato
di leggere da un'area di memoria a cui non dovrebbe aver accesso.

Questa è la ragione per cui Rust proibisce di usare `v` dopo che l'abbiamo
spostato.

[sh]: the-stack-and-the-heap.html

È anche importante notare che le ottimizzazioni possono rimuovere la copia
effettiva dei byte sullo stack, a seconda delle circostanza. Perciò potrebbe
non essere così inefficiente come come sembra inizialmente.

## I tipi `Copy`

Abbiamo stabilito che quando il possesso viene trasferito a un altro legame,
non si può più usare il legame originale. Però, c'è un [tratto][tratto] che
cambia questo comportamento, e si chiama `Copy`. Non abbiamo ancora parlato
dei tratti, ma per ora, si può pensare ad essi come annotazioni a tipi
particolari che aggiungono ulteriori comportamenti. Per esempio:

```rust
let v = 1;
let v2 = v;
println!("v vale: {}", v);
```

In questo caso, `v` è un `i32`, tipo che implementa il tratto `Copy`. Ciò
significa che, proprio come uno spostamento, quando si assegna `v` a `v2`,
viene fatta una copia dei dati.
Ma, diversamente da uno spostamento, dopo, possiamo ancora usare `v`.
Infatti un `i32` non ha puntatori che puntano a dati da qualche
altra parte, e quindi spostandolo si fa una copia completa.

Tutti i tipi primitivi implementano il tratto `Copy` e perciò il loro possesso
non viene spostato come si potrebbe immaginare, seguendo le ‘regole
del possesso’. Per fare un esempio, i due seguenti frammenti di codice
compilano solamente perché i tipi `i32` e `bool` implementano il tratto `Copy`.

```rust
fn main() {
    let a = 5;
    let _y = raddoppia(a);
    println!("{}", araddoppia
}

fn raddoppia(x: i32) -> i32 {
    x * 2
}
```

```rust
fn main() {
    let a = true;
    let _y = cambia_verita(a);
    println!("{}", a);
}

fn cambia_verita(x: bool) -> bool {
    !x
}
```

Se avessimo usato dei tipi che non implementano il tratto `Copy`,
avremmo ottenuto un errore di compilazione perché abbiamo provato a usare
un valore spostato.

```text
error: use of moved value: `a`
println!("{}", a);
               ^
```

Discuteremo come aggiungere il tratto `Copy` ai propri tipi nella sezione
[tratti][tratti].

[tratti]: traits.html

# Oltre al possesso

Naturalmente, se ogni nostra funzione dovesse restituire il possesso,
scriveremmo:

```rust
fn foo(v: Vec<i32>) -> Vec<i32> {
    // fa' qualcosa con v

    // restituisci il possesso
    v
}
```

Ciò diventerebbe molto noioso. E peggiora più sono gli oggetti di cui vogliamo
prendere possesso:

```rust
fn foo(v1: Vec<i32>, v2: Vec<i32>) -> (Vec<i32>, Vec<i32>, i32) {
    // fa' qualcosa con v1 e con v2

    // restituisci il possesso di v1 e v2, e restituisci anche
    // il risultato della nostra funzione
    (v1, v2, 42)
}

let v1 = vec![1, 2, 3];
let v2 = vec![1, 2, 3];
let (v1, v2, risposta) = foo(v1, v2);
```

Mah! Il tipo reso, la riga finale della funzione, e la chiamata della funzione
diventano parecchio più complicati.

Fortunatamente, Rust offre una caratteristica che aiuta a risolvere questo
problema. Si chiama "prestito" ed è l'argomento della prossima sezione!
