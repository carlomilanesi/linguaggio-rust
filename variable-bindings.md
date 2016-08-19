% Legami di variabile

Praticamente qualunque programma in Rust più complesso di 'Hello World’ usa
i *legami di variabile* ["variable binding"]. Tali istruzioni legano qualche
valore ad un nome, in modo da poter essere usato in seguito. Per introdurre
un legame, si usa la parola-chiave `let`, così:

```rust
fn main() {
    let x = 5;
}
```

Mettere `fn main() {` in ogni esempio è un po' noioso, perciò in futuro
lo ometteremo. Nel prosieguo, ci si ricordi di editare il corpo della propria
funzione `main()`, invece di ometterla. Altrimenti, si otterrà un'errore.

# I patterns

In molti linguaggi, un legame di variabile verrebbe chiamato semplicemente
*variabile*, ma i legami di variabile di Rust hanno alcuni assi nella manica.
Per esempio, il lato sinistro di un'istruzione `let` è un ‘[pattern][pattern]’,
non un semplice nome di variabile. Ciò significa che si può fare:

```rust
let (x, y) = (1, 2);
```

Dopo che questa istruzione viene eseguita, `x` varrà uno, e `y` varrà due.
I pattern sono veramente vantaggiosi, e hanno [una loro sezione][pattern]
in questo libro. Per adesso non ci servono quelle caratteristiche, e quindi
le accantoneremo mentre proseguiamo.

[pattern]: patterns.html

# Annotazioni di tipo

Rust è un linguaggio tipizzato staticamente, il che significa che specifichiamo
subito i tipi, e questi vengono verificati in fase di compilazione. Allora
perché il nostro primo esempio compilava? Beh, Rust ha una cosa chiamata
‘inferenza di tipo’. Se riesce a desumere qual'è il tipo di qualche dato,
Rust non costringe a digitarlo esplicitamente.

Se vogliamo, possiamo aggiungere il tipo di dato. I tipi si mettono dopo
un punto-e-virgola (`:`):

```rust
let x: i32 = 5;
```

Se volessi leggere questa istruzione ad alta voce, direi “`x` è un legame
di tipo `i32` al valore `cinque`.”

In questo caso abbiamo scelto di rappresentare `x` come un intero con segno
a 32 bit. Rust ha molti tipi interi primitivi. I loro nomi cominciano con `i`
per gli interi con segno, e con `u` per gli interi senza segno (unsigned).
Le dimensioni intere possibili sono 8, 16, 32, e 64 bit.

Negli esempi futuri, potremmo annotare il tipo in un commento. Gli esempi si
presenteranno così:

```rust
fn main() {
    let x = 5; // x: i32
}
```

Si noti la somiglianza fra questa annotazione e la sintassi che si usa con
`let`. Scrivere questo genere di commenti è insolito in Rust, ma lo faremo
occasionalmente per aiutare a capire quali sono i tipi che Rust inferisce.

# Mutabilità

Di default, i legami sono *immutabili*. Questo codice non compilerà:

```rust,ignore
let x = 5;
x = 10;
```

Darà questo errore:

```text
error: re-assignment of immutable variable `x`
     x = 10;
     ^~~~~~~
```

Se si vuole che un legame sia mutabile, si deve usare `mut`:

```rust
let mut x = 5; // mut x: i32
x = 10;
```

Non c'è una singola ragione per cui i legami sono immutabili di default, ma
possiamo pensarci in base a uno degli obiettivi primari di Rust: la sicurezza.
Se ci si dimentica di dire `mut`, il compilatore se ne accorgerà, e farà
sapere che si ha mutato qualcosa che si potrebbe non aver inteso mutare.
Se i legami fossero mutabili di default, il compilatore non sarebbe in grado
di dirlo. Se si _intendesse_ proprio la mutazione, allora la soluzione è
facilissima: aggiungere `mut`.

Ci sono altre buone ragioni per evitare lo stato mutabile quando possibile,
ma non ne parleremo in questo libro. In generale, si può spesso evitare
la mutazione esplicita, e quindi in Rust è preferibile evitarla. Detto questo,
talvolta, la mutazione è quello che serve, e quindi non è proibita.

# Inizializzare i legami

I legami di variabile in Rust hanno un altro aspetto che differisce da altri
linguaggi: i legami devono essere inizializzati con un valore prima di poterli
usare.

Facciamo una prova. Modifichiamo il nostro file `src/main.rs` in modo che
si presenti così:

```rust
fn main() {
    let x: i32;

    println!("Hello world!");
}
```

Si può usare `cargo build` dalla riga di comando per costruirlo. Si otterrà
un avvertimento, ma stamperà ancora "Hello, world!":

```text
   Compiling hello_world v0.0.1 (file:///home/you/projects/hello_world)
src/main.rs:2:9: 2:10 warning: unused variable: `x`, #[warn(unused_variable)]
   on by default
src/main.rs:2     let x: i32;
                      ^
```

Rust ci avverte che non abbiamo mai usato il legame di variabile, ma dato che
non l'abbiamo mai usato, nessun danno, nessun fallo. Però le cose cambiano
se proviamo ad usare effettivamente questa `x`. Facciamolo. Modifichiamo
il programma in modo che si presenti così:

```rust,ignore
fn main() {
    let x: i32;

    println!("Il valore di x è: {}", x);
}
```

E proviamo a costruirlo. Otterremo un errore:

```bash
$ cargo build
   Compiling hello_world v0.0.1 (file:///home/you/projects/hello_world)
src/main.rs:4:39: 4:40 error: use of possibly uninitialized variable: `x`
src/main.rs:4     println!("Il valore di x è: {}", x);
                                                   ^
note: in expansion of format_args!
<std macros>:2:23: 2:77 note: expansion site
<std macros>:1:1: 3:2 note: in expansion of println!
src/main.rs:4:5: 4:41 note: expansion site
error: aborting due to previous error
Could not compile `hello_world`.
```

Rust non ci permetterà di usare un valore che non è stato inizializzato.

Prendiamo un minuto per parlare di questa cosa che abbiamo aggiunto a
`println!`.

Se si inseriscono due graffe (`{}`, alcuni li chiamano baffi...) nella nostra
stringa da stampare, Rust le interpreterà come una richiesta di interpolare
qualche sorta di valore. L'*interpolazione di stringa* è un termine
informatico che significa "inserire una o più stringhe dentro un'altra stringa,
al posto di altrettanti segnaposto." Dopo la nostra stringa, mettiamo
una virgola, e una `x`, per indicare che vogliamo che `x` sia il valore che
stiamo interpolando. La virgola serve a separare gli argomenti che passiamo
alle funzioni e alle macro.

Quando si usa la coppia di graffe, Rust tenterà di visualizzare il valore
in un modo significativo verificando il suo tipo. Se si vuole specificare
il formato in una maniera più dettagliata, c'è un [ampio numero di opzioni
disponibili][format]. Per adesso, ci limitiamo al default: gli interi non sono
molto complicati da stampare.

[format]: ../std/fmt/index.html

# Ambito ed oscuramento

Torniamo ai legami. I legami di variabile hanno un ambito - ossia sono
vincolati a risiedere nel blocco in cui sono stati definiti. Un blocco è una
collezione di istruzioni racchiuse da `{` e `}`. Anche le definizioni
di funzione sono blocchi! Nell'esempio seguente definiamo due legami
di variabile, `x` e `y`, che risiedono in blocchi diversi. Si può accedere
a `x` da tutto il blocco `fn main() {}`, mentre si può accedere a `y` solamente
dal blocco più interno:

```rust,ignore
fn main() {
    let x: i32 = 17;
    {
        let y: i32 = 3;
        println!("Il valore di x è {} e il valore di y è {}", x, y);
    }
    println!("Il valore di x è {} e il valore di y è {}", x, y); // Questo non funziona
}
```

La prima `println!` stamperebbe "Il valore di x è 17 e il valore di y è 3",
ma questo esempio non può essere compilato con successo, perché la seconda
`println!` non può accedere al valore di `y`, dato che non è più
nel suo ambito. Otteniamo invece questo errore:

```bash
$ cargo build
   Compiling hello v0.1.0 (file:///home/you/projects/hello_world)
main.rs:7:62: 7:63 error: unresolved name `y`. Did you mean `x`? [E0425]
main.rs:7     println!("Il valore di x è {} e il valore di y è {}", x, y); // Questo non funziona
                                                                       ^
note: in expansion of format_args!
<std macros>:2:25: 2:56 note: expansion site
<std macros>:1:1: 2:62 note: in expansion of print!
<std macros>:3:1: 3:54 note: expansion site
<std macros>:1:1: 3:58 note: in expansion of println!
main.rs:7:5: 7:65 note: expansion site
main.rs:7:62: 7:63 help: run `rustc --explain E0425` to see a detailed explanation
error: aborting due to previous error
Could not compile `hello`.

To learn more, run the command again with --verbose.
```

Inoltre, i legami di variabile possono venire oscurati ("shadowed"). Ciò
significa che un successivo legame di variabile con il medesimo nome di un
legame attualmente nel suo ambito scavalcherà il legame precedente.

```rust
let x: i32 = 8;
{
    println!("{}", x); // Stampa "8"
    let x = 12;
    println!("{}", x); // Stampa "12"
}
println!("{}", x); // Stampa "8"
let x =  42;
println!("{}", x); // Stampa "42"
```

L'oscuramento e la mutabilità dei legami possono apparire come due facce
della stessa medaglia, ma sono due concetti distinti che non sono sempre
intercambiabili. Per dirne una, l'oscuramento ci consente di rilegare un nome
ad un valore di tipo diverso. È anche possibile cambiare la mutabilità
di un legame. Si noti che oscurare un nome non altera né distrugge il valore
a cui era legato quel nome, e tale valore continuerà a esistere finché
non esce di ambito, anche se non è più accessibile in nessun modo.

```rust
let mut x: i32 = 1;
x = 7;
let x = x; // x adesso è immutabile ed è legato a 7

let y = 4;
let y = "Posso anche essere legato a un testo!"; // y adesso è di un altro tipo
```
