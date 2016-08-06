% Tempi di vita

Questa è l'ultima delle tre sezioni che presentano il sistema di possesso
di Rust. Qui si assume che siano già state lette le altre due:

* Il [possesso][possesso], il concetto chiave
* I [prestiti][prestiti], e le loro caratteristiche associate, i ‘riferimenti’

[possesso]: ownership.html
[prestiti]: references-and-borrowing.html

# I tempi di vita

Prestare un riferimento a una risorsa posseduta da qualcun altro può essere
complicato. Per esempio, immaginiamo questa sequenza di operazioni:

1. Acquisisco un riferimento a una risorsa di qualche tipo.
2. Ti presto un riferimento a tale risorsa.
3. Decido di aver finito di lavorare con quella risorsa, e quindi la rilascio,
   mentre tu hai ancora il tuo riferimento a tale risorsa.
4. Tu decidi di usare quella risorsa.

Ahi, ahi! Il tuo riferimento sta puntando a una risorsa non più valida.
Questo difetto si chiama ‘puntatore penzolante‘ o ‘utilizzo dopo il rilascio’.

Per correggerlo, dobbiamo assicurarci che il passo 4 non avvenga mai dopo
il passo 3. Il sistema di possesso in Rust lo fa tramite un concetto chiamato
"tempo di vita" ["lifetime"], che descrive l'ambito in cui un riferimento
è valido. Nel nostro caso, o decidiamo che il tempo di vita vale solamente
fino al passo 3, e in tal caso il passo 4 darà errore di compilazione,
o decidiamo che il tempo di vita vale fino al passo 4,
e in tal caso la risorsa dovrà essere rilasciata al passo 5.

Quando abbiamo una funzione che prende un argomento per riferimento,
possiamo essere impliciti o espliciti riguardo al tempo di vita di tale
riferimento:

```rust
// implicito
fn foo(x: &i32) {
}

// esplicito
fn bar<'a>(x: &'a i32) {
}
```

L'`'a` si legge ‘il tempo di vita a’. Tecnicamente, ogni riferimento
ha qualche tempo di vita associato ad esso, ma il compilatore consente
di eliderlo (cioè ometterlo, si veda la sezione ["Elisione del tempo di vita"]
[elisione del tempo di vita] più avanti) nei casi più tipici.
Però, prima di arrivarci, scomponiamo l'esempio esplicito:

[lifetime-elision]: #lifetime-elision

```rust,ignore
fn bar<'a>(...)
```

Precedentemente abbiamo parlato un po' della [sintassi delle funzioni]
[funzioni], ma non abbiamo discusso dei `<>` dopo il nome della funzione.
Una funzione può avere dei ‘parametri generici’ fra le `<>`, dei quali
i tempi di vita sono un tipo. Discuteremo altri tipi di generici
[più avanti nel libro][generici], ma per adesso, focalizziamoci sull'aspetto
dei tempi di vita.

[funzioni]: functions.html
[generici]: generics.html

Usiamo le `<>` per dichiarare i nostri tempi di vita. Questo dice che `bar`
ha un solo tempo di vita, `'a`. Se avessimo dei parametri riferimento,
si presenterebbe così:


```rust,ignore
fn bar<'a, 'b>(...)
```

Poi nel nostro elenco di argomenti, usiamo i tempi di vita che abbiamo
nominato:

```rust,ignore
...(x: &'a i32)
```

Se avessimo voluto un riferimento `&mut`, avremmo scritto:

```rust,ignore
...(x: &'a mut i32)
```

Confrontando `&mut i32` con `&'a mut i32`, si nota che l'unica differenza
è che il tempo di vita `'a` si è intrufolato fra il `&` il `mut i32`.
La clausola `&mut i32` va letta come ‘un riferimento mutabile a un `i32`’,
mentre la clausola `&'a mut i32` va letta come ‘un riferimento mutabile
a un `i32` con tempo di vita `'a`’.

# Nelle `struct`

C'è bisogno dei tempi di vita espliciti anche quando si lavora come le
[`struct`][struct] che contengono riferimenti:

```rust
struct Foo<'a> {
    x: &'a i32,
}

fn main() {
    let y = &5; // questo è lo stesso che `let _y = 5; let y = &_y;`
    let f = Foo { x: y };
    println!("{}", f.x);
}
```

[struct]: structs.html

Come si vede, anche le `struct` possono avere tempi di vita. In modo simile
alle funzioni,

```rust
struct Foo<'a> {
# x: &'a i32,
# }
```

dichiara un tempo di vita, e

```rust
# struct Foo<'a> {
x: &'a i32,
# }
```

lo usa. Allora perché qui ci serve un tempo di vita? Ci serve per assicurare
che ogni riferimento a un `Foo` non possa sopravvivere al riferimento
a un `i32` che contiene.

## I blocchi `impl`

Implementiamo un metodo su `Foo`:

```rust
struct Foo<'a> {
    x: &'a i32,
}

impl<'a> Foo<'a> {
    fn x(&self) -> &'a i32 { self.x }
}

fn main() {
    let y = &5; // questo è lo stesso che `let _y = 5; let y = &_y;`
    let f = Foo { x: y };
    println!("x is: {}", f.x());
}
```

Come si vede, dobbiamo dichiarare un tempo di vita per `Foo` nella riga
di `impl`. `'a` viene ripetuto, come per le funzioni: `impl<'a>` definisce
un tempo di vita `'a`, e `Foo<'a>` lo usa.

## Tempi di vita multipli

Se si hanno riferimenti multipli, si può usare lo stesso tempo di vita
più volte:

```rust
fn x_o_y<'a>(x: &'a str, y: &'a str) -> &'a str {
#    x
# }
```

Questo dice che sia `x` che `y` sono vivi per lo stesso ambito, e che anche
il valore reso è vivo per lo stesso ambito. Se si voless che `x` e `y` avessero
tempi di vita diversi, si possono usare più parametri di tempo di vita:

```rust
fn x_o_y<'a, 'b>(x: &'a str, y: &'b str) -> &'a str {
#    x
# }
```

In questo esempio, `x` e `y` hanno diversi ambiti validi, ma il valore reso ha
lo stesso tempo di vita di `x`.

## Pensare agli ambiti

Un modo di pensare ai tempi di vita è visualizzare l'ambito per cui
un riferimento rimane valido. Per esempio:

```rust
fn main() {
    let y = &5;    // -+ y entra nell'ambito
                   //  |
    // roba        //  |
                   //  |
}                  // -+ y esce dall'ambito
```

Aggiungendo il nostro `Foo`:

```rust
struct Foo<'a> {
    x: &'a i32,
}

fn main() {
    let y = &5;           // -+ y entra nell'ambito
    let f = Foo { x: y }; // -+ f entra nell'ambito
    // roba               //  |
                          //  |
}                         // -+ prima f e poi y escono dall'ambito
```

Il nostro `f` vive entro l'ambito di `y`, perciò tutto funziona. E non fosse
così? Questo codice non funziona:

```rust,ignore
struct Foo<'a> {
    x: &'a i32,
}

fn main() {
    let x;                    // -+ x entra nell'ambito
                              //  |
    {                         //  |
        let y = &5;           // ---+ y entra nell'ambito
        let f = Foo { x: y }; // ---+ f entra nell'ambito
        x = &f.x;             //  | | errore qui
    }                         // ---+ prima f e poi y escono dall'ambito
                              //  |
    println!("{}", x);        //  |
}                             // -+ x esce dall'ambito
```

Whew! Come si vede, gli ambiti di `f` e `y` sono più piccoli dell'ambito
di `x`. Ma quando facciamo `x = &f.x`, rendiamo `x` un riferimento a qualcosa
che sta per uscire dal suo ambito.

I tempi di vita con nome sono un modo di dare un nome a questi ambiti.
Dare un nome a qualcosa è il primo passo verso l'essere capaci di parlarne.

## 'static

Il tempo di vita chiamato ‘static’ è un tempo di vita speciale. Segnala che
qualcosa ha il tempo di vita dell'intero programma. La maggior parte
dei programmatori Rust si imbatto per la prima volta in `'static`
quando trattano le stringhe:

```rust
let x: &'static str = "Ciao, mondo.";
```

I letterali di stringa sono di tipo `&'static str` perché il riferimento è
sempre vivo: vengono depositati nel segmento dati del file binario finale.
Un altro esempio sono i globali:

```rust
static FOO: i32 = 5;
let x: &'static i32 = &FOO;
```

Questo aggiunge un `i32` al segmento dati del file binario, e `x` è
un riferimento a esso.

## Elisione del tempo di vita

Rust supporta una potente inferenza di tipo locale nei corpi delle funzioni
ma non nelle firme dei loro elementi. È vietato consentire di ragionare
sui tipi a seconda della sola firma degli elementi.  Però, per ragioni
di comodità, un algoritmo di inferenza secondaria molto ristretto chiamato
“elisione del tempo di vita” si applica quando si giudicano i tempi di vita.
L'elisione dei tempi di vita viene considerata solamente per inferire
i parametri del tempo di vita usando tre regole facilmente memorizzabili e
non ambigue. Ciò significa che l'elisione del tempo di vita agisce
da abbreviazione per scrivere una firma di un elemento, mentre non nasconde
i tipi effettivamente coinvolti, come avverrebbe se fosse applicata una
una completa inferenza locale.

Quando si parla dell'elisione del tempo di vita, usiamo i termini
*tempo di vita di input* e *tempo di vita di output*. Un *tempo di vita
di input* è un tempo di vita associato a un argomento di una funzione, mentre
un *tempo di vita di output* è un tempo di vita associato a un valore
reso da una funzione. Per esempio, questa funzione ha un tempo di vita
di input:

```rust,ignore
fn foo<'a>(bar: &'a str)
```

Quest'altra ha un tempo di vita di output:

```rust,ignore
fn foo<'a>() -> &'a str
```

E questa ha un tempo di vita in entrambe le posizioni:

```rust,ignore
fn foo<'a>(bar: &'a str) -> &'a str
```

Ecco le tre regole:

* Ogni tempo di vita eliso tra gli argomenti di una funzione diventa un
  un distinto parametro tempo di vita.

* Se c'è esattamente un tempo di vita di input, eliso o no, quel tempo di vita
  è assegnato a tutti i tempi di vita elisi nei valori resi di quella funzione.

* Se ci sono più tempo di vita di input, ma uno di essi è `&self` o
  `&mut self`, il tempo di vita di `self` viene assegnato a tutti i tempi
  di vita di output elisi.

Altrimenti, è un errore elidere un tempo di vita di output.

### Esempi

Ecco alcuni esempi di funzioni con tempi di vita elisi. Abbiamo accoppiato
ogni esempio di un tempo di vita eliso con la sua forma espansa.

```rust,ignore
fn stampa(s: &str); // eliso
fn stampa<'a>(s: &'a str); // espanso

fn debug(lvl: u32, s: &str); // eliso
fn debug<'a>(lvl: u32, s: &'a str); // espanso
```

Nell'esempio precedente, `lvl` non ha bisogno di un tempo di vita, perché non è
un riferimento (`&`). Solamente oggetti riferiti a riferimenti (come
uno `struct` che contiene un riferimento) hanno bisogno di tempi di vita.

```rust,ignore
fn substr(s: &str, until: u32) -> &str; // eliso
fn substr<'a>(s: &'a str, until: u32) -> &'a str; // espanso

fn get_str() -> &str; // ILLEGALE, nessun input

fn frob(s: &str, t: &str) -> &str; // ILLEGALE, due input
// espanso: il tempo di vita di output è ambiguo
fn frob<'a, 'b>(s: &'a str, t: &'b str) -> &str;

fn get_mut(&mut self) -> &mut T; // eliso
fn get_mut<'a>(&'a mut self) -> &'a mut T; // espanso

fn argomenti<T: ToCStr>(&mut self, args: &[T]) -> &mut Command; // eliso
fn argomenti<'a, 'b, T: ToCStr>(&'a mut self, args: &'b [T])
-> &'a mut Command; // espanso

fn new(buf: &mut [u8]) -> BufWriter; // eliso
fn new<'a>(buf: &'a mut [u8]) -> BufWriter<'a>; // espanso
```
