% Chiusure ["closure"]

Talvolta è utile racchiudere una funzione e le _variabili libere_ per ottenere
maggiore chiarezza e riutilizzo. Le variabili libere che possono essere usate
vengono dall'ambito circostante, e vengono 'rinchiuse' quando vengono
usate nella funzione. Da ciò deriva il nome ‘chiusura’, e Rust ne fornisce
un'implementazione davvero ottima, come vedremo.

# Sintassi

Le chiusure si presentano così:

```rust
let piu_uno = |x: i32| x + 1;

assert_eq!(6, piu_uno(5));
```

Abbiamo creato un legame, `piu_uno`, e l'abbiamo assegnato a una chiusura.
Gli argomenti della chiusura vanno fra due caratteri 'pipe' (`|`);
mentre il corpo della chiusura è un'espressione, in questo caso, `x + 1`.
Si noti che `{ }` è un'espressione, e quindi si possono scrivere chiusure
che contengono più istruzioni, in questo modo:

```rust
let piu_due = |x| {
    let mut risultato: i32 = x;
    risultato += 1;
    risultato += 1;
    risultato
};
assert_eq!(9, piu_due(7));
```

Si notino alcune cose riguardo le chiusure che sono un po' diverse dalle
normali funzioni con nome definite tramite `fn`. La prima cosa è che
non abbiamo dovuto annotare i tipi degli argomenti che la chiusura prende
né il valore che restituisce. È consentito:

```rust
let piu_uno = |x: i32| -> i32 { x + 1 };
assert_eq!(6, piu_uno(5));
```

ma non è obbligatorio. Perché? Di base, è stato scelto per praticità. Mentre
specificare il tipo completo per le funzioni con nome è di aiuto con cose
come la documentazione e le interfacce dei tipi, le complete firme dei tipi
delle chiusure sono documentate di rado dato che sono anonime, e non
provocano il tipo di errori a distanza che possono essere provocati
dall'inferire i tipi delle funzioni con nome.

La seconda cosa è che la sintassi è simile, ma un po' diversa. Qui sono
stati aggiunti spazi per facilitare il confronto:

```rust
fn  piu_uno_v1   (x: i32) -> i32 { x + 1 }
let piu_uno_v2 = |x: i32| -> i32 { x + 1 };
let piu_uno_v3 = |x: i32|          x + 1  ;
```

Piccole differenze, ma sono simili.

# Le chiusure e il loro ambiente

L'ambiente per una chiusura può comprendere i legami del suo ambito
circostante oltre agli argomenti e ai legami locali. Si presenta così:

```rust
let numero = 5;
let piu_numero = |x: i32| x + numero;
assert_eq!(10, piu_numero(5));
```

Questa chiusura, `piu_numero`, fa riferimento a un legame `let`
nel suo ambito: `numero`. Più specificamente, prende in prestito il legame.
Se facciamo qualcosa che entrasse in conflitto con quel legame, otterremmo
un errore. Come questo:

```rust,ignore
let mut numero = 5;
let piu_numero = |x: i32| x + numero;
let y = &mut numero;
```

Che va in errore con:

```text
error: cannot borrow `numero` as mutable because it is also borrowed as immutable
    let y = &mut numero;
                 ^~~~~~
note: previous borrow of `numero` occurs here due to use in closure; the immutable
  borrow prevents subsequent moves or mutable borrows of `numero` until the borrow
  ends
    let piu_numero = |x| x + numero;
                   ^~~~~~~~~~~~~~~~
note: previous borrow ends here
fn main() {
    let mut numero = 5;
    let piu_numero = |x| x + numero;

    let y = &mut numero;
}
^
```

Un messaggio d'errore prolisso ma utile! Come dice, non si può prendere `num`
a prestito mutabile, perché la chiusura lo sta già tenendo a prestito.
Ma se lasciamo uscire di ambito la chiusura, lo possiamo fare:

```rust
let mut numero = 5;
{
    let piu_numero = |x: i32| x + numero;
} // piu_numero esce di ambito, e quindi il prestito di 'numero' finisce

let y = &mut numero;
```

Però, se la chiusura lo richiede, Rust invece prenderà il possesso del legame
e lo sposterà dall'ambiente. Quindi questo non funziona:

```rust,ignore
let numeri = vec![1, 2, 3];

let prende_numeri = || numeri;

println!("{:?}", numeri);
```

Otteniamo questo errore:

```text
note: `numeri` moved into closure environment here because it has type
  `[closure(()) -> collections::vec::Vec<i32>]`, which is non-copyable
let prende_numeri = || numeri;
                    ^~~~~~~~~
```

`Vec<T>` ha il possesso del suo contenuto, e quindi, quando
facciamo riferimento ad esso nella nostra chiusura, dobbiamo prendere
possesso di `numeri`. È lo stesso come se avessimo passato `numeri`
a una funzione che ne prendesse il possesso.

## Le chiusure `move`

Possiamo costringere la nostra chiusura a prendere possesso del suo ambiente
con la parola-chiave `move`:

```rust
let numero = 5;

let possiede_numero = move |x: i32| x + numero;
```

Adesso, anche se la parola-chiave è `move`, le variabili seguono
la normale semantica di spostamento. In questo caso, `5` implementa `Copy`,
e così `possiede_numero` prende possesso di una copia di `numero`.
E allora che differenza c'è?

```rust
let mut numero = 5;

{
    let mut aggiungi_numero = |x: i32| numero += x;

    aggiungi_numero(5);
}

assert_eq!(10, numero);
```

Quindi in questo caso, la nostra chiusura ha preso un riferimento mutabile
a `numero`, e poi, quando abbiamo chiamato `aggiungi_numero`, ha mutato
il valore soggiacente, come ci aspettavamo. Abbiamo anche dovuto dichiarare
`aggiungi_numero` come `mut`, perché stiamo mutando il suo ambiente.

Se lo trasformiamo in una chiusura `move`, sarà diverso:

```rust
let mut numero = 5;

{
    let mut aggiungi_numero = move |x: i32| numero += x;

    aggiungi_numero(5);
}

assert_eq!(5, numero);
```

Otteneniamo solamente `5`. Invece di prendere un prestito mutabile sul
nostro `numero`, abbiamo preso possesso di una sua copia.

Un altro modo di pensare alle chiusure `move`: danno a una chiusura
il suo stesso frame di stack. Senza `move`, una chiusura può essere
vincolata al frame di stack che l'ha creata, mentre una chiusura `move`
è autocontenuta. Ciò comporta, per esempio, che in generale una funzione
non può restituire una chiusura non-`move`.

Ma prima di parlare di come prendere e restituire chiusure, dovremmo
parlare ancora un po' del modo in cui le chiusure sono implementate.
Essendo un linguaggio di sistemi, Rust dà moltissimo controllo su ciò che fa
il codice, e le chiusure non sono da meno.

# Implementazione delle chiusure 

L'implementazione delle chiusure di Rust è un po' diversa dagli altri
linguaggi. Effettivamente sono un addolcimento sintattico dei tratti.
Prima di leggere questa sezione, ci si assicuri di aver letto la sezione
sui [tratti][traits], e anche quella sugli [oggetti-tratto][trait-objects].

[traits]: traits.html
[trait-objects]: trait-objects.html

Capito tutto? Bene.

La chiave per capire come funzionano sotto il cofano le chiusure è qualcosa
di un po' strano: Usare `()` per chiamare una funzione, come in `foo()`,
è un operatore sovraccaricabile. Da questo, tutto il resto scatta
al suo posto. In Rust, si usa il sistema dei tratti per sovraccaricare
gli operatori. Chiamare funzioni non è diverso. Ci sono tre tratti distinti
da sovraccaricare:

```rust
# mod foo {
pub trait Fn<Args> : FnMut<Args> {
    extern "rust-call" fn call(&self, args: Args) -> Self::Output;
}

pub trait FnMut<Args> : FnOnce<Args> {
    extern "rust-call" fn call_mut(&mut self, args: Args) -> Self::Output;
}

pub trait FnOnce<Args> {
    type Output;

    extern "rust-call" fn call_once(self, args: Args) -> Self::Output;
}
# }
```

Si noteranno alcune differenze fra questi tratti, ma una grossa è `self`:
`Fn` prende `&self`, `FnMut` prende `&mut self`, e `FnOnce` prende `self`.
Ciò considera tutte e tre i generi di `self` permessi dalla solita sintassi
di chiamata di metodo. Ma li abbiamo separati in tre tratti, invece
di averne uno solo. Questo ci consente di controllare con precisione
quali generi di chiusure possiamo prendere.

La sintassi `|| {}` per le chiusure è un addolcimento per questi tre tratti.
Rust genererà una struct che rappresenta l'ambiente, implementerà il tratto
appropriato per essa, e poi lo userà.

# Prendere chiusure come argomenti

Adesso che sappiamo che le chiusure sono tratti, sappiamo già come accettare
e restituire le chiusure: proprio come ogni altro tratto!

Ciò comporta anche che possiamo scegliere tra il dispatch statico e
quello dinamico. Prima, scriviamo una funzione che prende qualcosa
di chiamabile, lo chiamiamo, e restituiamo il risultato:

```rust
fn chiama_con_uno<F>(una_chiusura: F) -> i32
    where F : Fn(i32) -> i32 {
    una_chiusura(1)
}

let risposta = chiama_con_uno(|x| x + 2);

assert_eq!(3, risposta);
```

Passiamo la nostra chiusura, `|x| x + 2`, a `chiama_con_uno`, che fa quello
che dice il suo nome: chiama la chiusura, dandole `1` come argomento.

Esaminiamo la firma di `chiama_con_uno` più in profondità:

```rust
fn chiama_con_uno<F>(una_chiusura: F) -> i32
#    where F : Fn(i32) -> i32 {
#    una_chiusura(1) }
```

Prendiamo un argomente, che è di tipo `F`. Inoltre restituiamo un `i32`.
Questa parte non è interessante. La prossima parte è:

```rust
# fn chiama_con_uno<F>(una_chiusura: F) -> i32
    where F : Fn(i32) -> i32 {
#   una_chiusura(1) }
```

Siccome `Fn` è un tratto, possiamo vincolare ad esso il nostro generico.
In questo caso, la nostra chiusura prende un `i32` come argomento
e restituisce un `i32`, e quindi il vincolo generico che usiamo
è `Fn(i32) -> i32`.

Qui c'è un altro punto chiave: siccome stiamo vincolando un generico
con un tratto, questo diventerà monomorfizzato, e perciò, nella chiusura
faremo un dispatch statico. È piuttosto pulito. In molti linguaggi,
le chiusure sono inerentemente allocate su heap, e comporteranno sempre
un dispatch dinamico. In Rust, possiamo allocare su stack l'ambiente
della nostra chiusura, e il eseguire un dispatch statico della chiamata.
Ciò accade davvero spesso con gli iteratori e i loro adattatori,
che spesso prendono come argomenti delle chiusure.

Naturalmente, se vogliamo un dispatch dinamico, possiamo averlo.
Un oggetto-tratto gestisce questo caso, come al solito:

```rust
fn chiama_con_uno(una_chiusura: &Fn(i32) -> i32) -> i32 {
    una_chiusura(1)
}

let risposta = chiama_con_uno(&|x| x + 2);

assert_eq!(3, risposta);
```

Adesso prendiamo un oggetto-tratto, un `&Fn`. E dobbiamo creare
un riferimento alla nostra chiusura, quando la passiamo a `chiama_con_uno`,
e quindi usiamo `&||`.

Una nota veloce sulle chiusure che usano tempi di vita espliciti.
Talvolta si potrebbe avere una chiusura che prende un riferimento così:

```rust
fn chiama_con_riferimento<F>(una_chiusura:F) -> i32
    where F: Fn(&i32) -> i32 {

    let mut value = 0;
    una_chiusura(&value)
}
```

Normalmente si può specificare il tempo di vita dell'argomento alla chiusura.
Potremmo annotarlo nella dichiarazione della funzione:

```rust,ignore
fn chiama_con_riferimento<'a, F>(una_chiusura:F) -> i32
    where F: Fn(&'a i32) -> i32 {
```

Però ciò presenta un problema nel nostro caso. Quando si specifica
il tempo di vita esplicito in una funzione, questo lega quel tempo di vita
all'*intero* ambito della funzione invece che appena all'ambito
di invocazione della nostra chiusura. Ciò comporta che il verificatore
dei prestiti vedrà un riferimento mutabile nel medesimo tempo di vita
del nostro riferimento immutabile, non potrà compilarlo.

Per poter dire che ci serve solamente che il tempo di vita sia valido per
l'ambito di invocazione della chiusura, possiamo usare i vincoli dei tratti
di rango superiore ("Higher-Ranked Trait Bounds") con la sintassi `for<...>`:

```ignore
fn chiama_con_riferimento<F>(una_chiusura:F) -> i32
    where F: for<'a> Fn(&'a i32) -> i32 {
```

Ciò consente al compilatore Rust di trovare il tempo di vita minimo
per invocare la nostra chiusura, e soddisfare le regole del verificatore
dei prestiti. Quindi la nostra funzione potrà essere compilata ed eseguita
come ci aspettiamo.

```rust
fn chiama_con_riferimento<F>(una_chiusura:F) -> i32
    where F: for<'a> Fn(&'a i32) -> i32 {

    let mut value = 0;
    una_chiusura(&value)
}
```

# Puntatori a funzione e chiusure

Un puntatore a funzione è un po' come una chiusura che non ha nessun ambiente.
Come tale, si può passare un puntatore a funzione a qualunque funzione
che si aspetta come argomento una chiusura, e funzionerà:

```rust
fn chiama_con_uno(una_chiusura: &Fn(i32) -> i32) -> i32 {
    una_chiusura(1)
}

fn aggiungi_uno(i: i32) -> i32 {
    i + 1
}

let f = aggiungi_uno;

let risposta = chiama_con_uno(&f);

assert_eq!(2, risposta);
```

In questo esempio, non abbiamo strettamente bisogno della variabile
intermedia `f`; il nome della funzione va altrettanto bene:

```rust,ignore
let risposta = chiama_con_uno(&aggiungi_uno);
```

# Restituire chiusure

È molto tipico per il codice in stile funzionale restituire delle chiusure
in varie situazioni. Se si prova a restituire una chiusura, ci si può
imbattere in un errore. Dapprima, può sembrare strano, ma vedremo perché.
Ecco un tentativo plausibile di restituire una chiusura da una funzione:

```rust,ignore
fn factory() -> (Fn(i32) -> i32) {
    let num = 5;

    |x| x + num
}

let f = factory();

let risposta = f(1);
assert_eq!(6, risposta);
```

Questo ci dà questi lunghi errori correlati:

```text
error: the trait bound `core::ops::Fn(i32) -> i32 : core::marker::Sized` is not satisfied [E0277]
fn factory() -> (Fn(i32) -> i32) {
                ^~~~~~~~~~~~~~~~
note: `core::ops::Fn(i32) -> i32` does not have a constant size known at compile-time
fn factory() -> (Fn(i32) -> i32) {
                ^~~~~~~~~~~~~~~~
error: the trait bound `core::ops::Fn(i32) -> i32 : core::marker::Sized` is not satisfied [E0277]
let f = factory();
    ^
note: `core::ops::Fn(i32) -> i32` does not have a constant size known at compile-time
let f = factory();
    ^
```

Per poter restituire qualcosa da una funzione, Rust deve conoscere
la dimensione del tipo del valore restituito. Ma siccome `Fn` è un tratto,
potrebbe essere varie cose di varie dimensioni: molti tipi diversi possono
implementare `Fn`. Un modo facile per dare una dimensione a qualcosa è
prendere un riferimento ad esso, dato che i riferimenti hanno
una dimensione nota. Quindi scriveremmo questo:

```rust,ignore
fn factory() -> &(Fn(i32) -> i32) {
    let num = 5;

    |x| x + num
}

let f = factory();

let risposta = f(1);
assert_eq!(6, risposta);
```

Ma otteniamo un altro errore:

```text
error: missing lifetime specifier [E0106]
fn factory() -> &(Fn(i32) -> i32) {
                ^~~~~~~~~~~~~~~~~
```

Giusto. Siccome abbiamo un riferimento, dobbiamo dargli un tempo di vita.
Ma la nostra funzione `factory()` non prende argomenti, e quindi qui
l'[elisione](lifetimes.html#lifetime-elision) non entra in gioco.
Allora che scelte abbiamo? Possiamo provare `'static`:

```rust,ignore
fn factory() -> &'static (Fn(i32) -> i32) {
    let num = 5;

    |x| x + num
}

let f = factory();

let risposta = f(1);
assert_eq!(6, risposta);
```

Ma otteniamo un altro errore:

```text
error: mismatched types:
 expected `&'static core::ops::Fn(i32) -> i32`,
    found `[closure@<anon>:7:9: 7:20]`
(expected &-ptr,
    found closure) [E0308]
         |x| x + num
         ^~~~~~~~~~~

```

Questo errore ci fa sapere che non abbiamo una `&'static Fn(i32) -> i32`,
abbiamo una `[closure@<anon>:7:9: 7:20]`. Un attimo, cos'è?

Siccome ogni chiusura genera la sua `struct` ambiente e
la sua implementazione di `Fn` e compagni, questi tipo sono anonimi.
Esistono solamente per questa chiusura. Quindi Rust li mostra come
`closure@<anon>`, invece di mostrare un nome autogenerato.

L'errore fa notare anche che come tipo del valore restituito ci si
aspetta un riferimento, ma quello che stiamo provando a restituire non lo è.
Inoltre, non possiamo assegnare direttamente un tempo di vita `'static`
a un oggetto. Quindi sceglieremo un approccio diverso e restituiremo
un oggetto-tratto incapsulando la `Fn` in un `Box`. Questo _quasi_ funziona:

```rust,ignore
fn factory() -> Box<Fn(i32) -> i32> {
    let num = 5;

    Box::new(|x| x + num)
}
# fn main() {
let f = factory();

let risposta = f(1);
assert_eq!(6, risposta);
# }
```

C'è appena un ultimo difetto:

```text
error: closure may outlive the current function, but it borrows `num`,
which is owned by the current function [E0373]
Box::new(|x| x + num)
         ^~~~~~~~~~~
```

Beh, come abbiamo discusso prima, le chiusure prendono in prestito
il loro ambiente. E in questo caso, il nostro ambiente è basato su un `5`
allocato sullo stack, il legame di variabile `num`. Quindi il prestito
ha il tempo di vita del frame di stack. Perciò se restituissimo
questa chiusura, la chiamata di funzione finirebbe, il frame di stack
andrebbe via, e la nostra chiusura avrebbe catturato un ambiente di memoria
spazzatura! Con un'ultima correzione, lo possiamo far funzionare:

```rust
fn factory() -> Box<Fn(i32) -> i32> {
    let num = 5;

    Box::new(move |x| x + num)
}
fn main() {
let f = factory();

let risposta = f(1);
assert_eq!(6, risposta);
}
```

Rendendo la chiusura interna un `move Fn`, creiamo un nuovo frame di stack
per la nostra chiusura. Incapsulandolo in un `Box`, gli abbiamo dato
una dimensione nota, consentendogli di fuggire dal nostro frame di stack.
