% Riferimenti e prestiti

Questa è la seconda delle tre sezioni che presentano il sistema di possesso
di Rust. Questa è una delle caratteristiche più distintive e avvincenti
di Rust, con la quale gli sviluppatori Rust dovrebbero diventare familiari.
Il possesso è il modo in cui Rust raggiunge il suo maggior obiettivo, la sicurezza
di accesso alla memoria. Ci sono alcuni concetti distinti, ognuno descritto
in una sezione distinta:


* [possesso][possesso], il concetto chiave
* i prestiti, che leggeremo adesso
* [tempi di vita][tempi di vita], un concetto avanzato di prestito

Queste tre sezioni sono correlate, e seguono un ordine. Bisognerà leggerli
tutti e tre per capire pienamente il sistema di possesso.

[possesso]: ownership.html
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
accade spesso perché il modello mentale del programmatore su come il possesso
dovrebbe funzionare non combacia con le regole effettivamente implementate
da Rust.
Dapprima tutti sperimentano cose simili. Però, c'è una buona notizia:
gli sviluppatori Rust più esperti riferiscono che una volta che lavorano con
le regole del sistema di possesso per un periodo di tempo, combattono sempre
meno con il verificatore dei prestiti.

Con questo in mente, vediamo di imparare riguardo il prestito.

# Borrowing

Alla fine della sezione sul [possesso][possesso], avevamo una brutta funzione
che si presentava così:

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

Però questo non è Rust tipico, dato che non sfrutta i prestiti.
Ecco il primo passo:

```rust
fn foo(v1: &Vec<i32>, v2: &Vec<i32>) -> i32 {
    // fa' qualcosa con v1 e con v2

    // restituisci la risposta
    42
}

let v1 = vec![1, 2, 3];
let v2 = vec![1, 2, 3];

let risposta = foo(&v1, &v2);

// qui possiamo usare v1 e v2!
```

Un esempio più concreto:

```rust
fn main() {
    // Non importa se non si capisce cosa fa `fold`, quello che importa
    // qui è che un riferimento immutabile viene preso in prestito.
    fn somma_vec(v: &Vec<i32>) -> i32 {
        return v.iter().fold(0, |a, &b| a + b);
    }
    // Prendi in prestito due vettori e sommane gli elementi.
    // Questo tipo di prestito non permette che gli oggetti siano mutati.
    fn foo(v1: &Vec<i32>, v2: &Vec<i32>) -> i32 {
        // fa' qualcosa con v1 e con v2
        let s1 = somma_vec(v1);
        let s2 = somma_vec(v2);
        // restituisci la risposta
        s1 + s2
    }

    let v1 = vec![1, 2, 3];
    let v2 = vec![4, 5, 6];

    let risposta = foo(&v1, &v2);
    println!("{}", risposta);
}
```

Invece di prendere dei `Vec<i32>` come argomenti, prendiamo dei riferimenti:
`&Vec<i32>`. E invece di passare `v1` e `v2` direttamente, passiamo `&v1` e
`&v2`. Il tipo `&T` viene chiamato ‘riferimento’, e invece di possedere
la risorsa, ne prende in prestito il possesso. Un legame che prende in prestito
qualche oggetto non dealloca quella risorsa quando esce dall'ambito.
Ciò significa che dopo la chiamata a `foo()`, possiamo usare ancora i nostri
legami originali.

I riferimenti sono immutabile, come i legami. Ciò significa che dentro `foo()`,
i due vettori non possono affatto essere modificati:

```rust,ignore
fn foo(v: &Vec<i32>) {
     v.push(5);
}

let v = vec![];

foo(&v);
```

ci darà questo errore:

```text
error: cannot borrow immutable borrowed content `*v` as mutable
v.push(5);
^
```

Aggiungere un valore (chiamando `push`) muterebbe il vettore, e quindi non
ci viene permesso.

# I riferimenti &mut

C'è un altro tipo di riferimento: `&mut T`. Un ‘riferimento mutabile’ permette
di mutare la risorsa che viene presa in prestito. Per esempio:

```rust
let mut x = 5;
{
    let y = &mut x;
    *y += 1;
}
println!("{}", x);
```

Questo stamperà `6`. Abbiamo creato `y` come riferimento mutabile a `x`, e poi
abbiamo incrementato l'oggetto a cui `y` punta. Si noterà che abbiamo dovuto
marcare anche `x` come `mut`.
Se non l'avessimo fatto, non avremmo potuto prendere in prestito mutabile
un valore immutabile.

Si noterà anche che abbiamo aggiunto un asterisco (`*`) prima di `y`,
rendendolo `*y`.  Questo è necessario perché `y` è un riferimento. Se deve
usare un asterisco per accedere al contenuto di un riferimento.

I riferimenti `&mut` somigliano ai riferimenti; però, c'_è_ una grossa
differenza tra i due, e su come interagiscono. Si potrebbe dire che
nell'esempio qui sopra ci sono due graffe di troppo, che racchiudono l'ambito
di `y`. Ma se le togliamo, otteniamo un errore:

```text
error: cannot borrow `x` as immutable because it is also borrowed as mutable
    println!("{}", x);
                   ^
note: previous borrow of `x` occurs here; the mutable borrow prevents
subsequent moves, borrows, or modification of `x` until the borrow ends
        let y = &mut x;
                     ^
note: previous borrow ends here
fn main() {

}
^
```

Da quel che risulta, ci sono regole da rispettare.

# Le regole

Ecco le regole per prendere a prestito in Rust:

Primo, ogni prestito deve durare per un ambito non più esteso di quello
del possessore. Secondo, si può avere uno o l'altro dei due seguenti generi
di prestiti, ma non entrambi allo stesso tempo:

* uno o più riferimenti non mutabili (`&T`) a un oggetto,
* esattamente un riferimento mutabile (`&mut T`) a un oggetto.

Questa regola è molto simile, anche se non esattamente uguale,
alla definizione di 'corsa ai dati' ["data race"]:

> C'è una ‘corsa ai dati’ quando due o più puntatori accedono alla medesima
> posizione di memoria nello stesso tempo, e almeno uno dei due sta scrivendo
> in memoria e le operazioni non sono sincronizzate.

Per quanto riguarda i riferimenti immutabili, se ne possono avere quanti se ne
vogliono, dato che nessuno di essi sta scrivendo. Però, dato possiamo avere
solamente un riferimento mutabili per volta, è impossibile avere una corsa
ai dati. Questa tecnica consente a Rust in fase di compilazione di prevenire
le corse ai dati: otterremmo degli errori se violiamo le regole.

Tenendo questo a mente, consideriamo ancora il nostro esempio.

## Pensare secondo gli ambiti

Ecco il codice:

```rust,ignore
fn main() {
    let mut x = 5;
    let y = &mut x;

    *y += 1;

    println!("{}", x);
}
```

Questo codice ci dà questo errore:

```text
error: cannot borrow `x` as immutable because it is also borrowed as mutable
    println!("{}", x);
                   ^
```

Questo perché abbiamo violate le regole: abbiamo un `&mut T` che punta a `x`,
e così non ci è permesso creare dei `&T` che puntino al medesimo oggetto.
È l'uno o l'altro. La nota suggerisce come pensare a questo problema:

```text
note: previous borrow ends here
fn main() {

}
^
```

In altre parole, il prestito mutabile viene tenuto per tutto il resto
del programma. Ciò che vogliamo è che il prestito mutable a `y` finisca, così
che la risorsa possa essere restituita al possessore, `x`. Poi `x` può fornire
un prestito immutabile a `println!`. In Rust, prendere in prestito è legato
all'ambito per cui il prestito è valido. E il nostro ambito si presenta così:

```rust,ignore
fn main() {
    let mut x = 5;

    let y = &mut x;    // -+ qui inizia il prestito mutabile (mut&) di x
                       //  |
    *y += 1;           //  |
                       //  |
    println!("{}", x); // -+ - qui prova a prendere in prestito x
}                      // -+ qui finisce il prestito mutabile di x

```

Gli ambiti sono in conflitto: non possiamo fare un `&x` mentre `y` è
nell'ambito.

Perciò quando aggiungiamo le graffe:

```rust
let mut x = 5;

{
    let y = &mut x; // -+ qui inizia il prestito mutabile
    *y += 1;        //  |
}                   // -+ ... e qui finisce

println!("{}", x);  // <- qui prova a prendere a prestito immutabile x
```

Non c'è problema. Il nostro prestito mutabile esce di ambito prima che venga
creato quello immutabile. Perciò l'ambito è la chiave per vedere quanto dura
un prestito.

## Problemi prevenuti dai prestiti

Perché ci sono queste regole restrittive? Beh, come abbiamo detto,
queste regole prevengono le corse ai dati. Che genere
di difetti provocano le corse ai dati? Eccone alcuni.

### Invalidazione degli iteratori

Un esempio è ‘l'invalidazione degli iteratori’, che avviene quando si prova
a mutare una collezione su cui si sta iterando. Il verificatore dei prestiti
di Rust previene che accada:

```rust
let mut v = vec![1, 2, 3];

for i in &v {
    println!("{}", i);
}
```

Questo codice stampa i numeri da uno a tre. Mentre iteriamo lungo il vettore,
ci vengono dati solamente dei riferimenti agli elementi. E `v` è esso stesso
preso in prestito come immutabile, il che significa che non possiamo cambiarlo
mentre stiamo iterando:

```rust,ignore
let mut v = vec![1, 2, 3];

for i in &v {
    println!("{}", i);
    v.push(34);
}
```

Ecco l'errore:

```text
error: cannot borrow `v` as mutable because it is also borrowed as immutable
    v.push(34);
    ^
note: previous borrow of `v` occurs here; the immutable borrow prevents
subsequent moves or mutable borrows of `v` until the borrow ends
for i in &v {
          ^
note: previous borrow ends here
for i in &v {
    println!(“{}”, i);
    v.push(34);
}
^
```

Non possiamo modificare `v` perché è preso in prestito dal ciclo.

### Uso dopo il rilascio

I riferimenti non devono vivere più a lungo dell'oggetto a cui fanno
riferimento. Rust verificherà gli ambiti dei riferimenti per assicurare
che sia così.

Se Rust non verificasse questa proprietà, potremmo accidentalmente usare
un riferimento ad un oggetto che è diventato invalido. Per esempio:

```rust,ignore
let y: &i32;
{
    let x = 5;
    y = &x;
}

println!("{}", y);
```

Otteniamo questo errore:

```text
error: `x` does not live long enough
    y = &x;
         ^
note: reference must be valid for the block suffix following statement 0 at
2:16...
let y: &i32;
{
    let x = 5;
    y = &x;
}

note: ...but borrowed value is only valid for the block suffix following
statement 0 at 4:18
    let x = 5;
    y = &x;
}
```

In altre parole, il valore di `y` è valido solamente per l'ambito dove `x`
esiste. Non appena `x` se ne va, diventa invalido fare riferimento ad esso.
Come tale, l'errore dice che il prestito ‘non vive abbastanza a lungo’ perché
non è valido per la giusta quantità di tempo.

Il medesimo problema avviene quando il riferimento è dichiarato _prima_
della variabile a cui si riferisce. Questo è dovuto al fatto che le risorse
entro lo stesso ambito vengono rilasciate nell'ordine inverso di quello
con cui sono state acquisite:

```rust,ignore
let y: &i32;
let x = 5;
y = &x;

println!("{}", y);
```

Otteniamo questo errore:

```text
error: `x` does not live long enough
y = &x;
     ^
note: reference must be valid for the block suffix following statement 0 at
2:16...
    let y: &i32;
    let x = 5;
    y = &x;

    println!("{}", y);
}

note: ...but borrowed value is only valid for the block suffix following
statement 1 at 3:14
    let x = 5;
    y = &x;

    println!("{}", y);
}
```

Nell'esempio qui sopra, `y` è dichiarato prima di `x`, il che comporta che `y`
vive (leggermente) più a lungo di `x`, il che non è consentito.
