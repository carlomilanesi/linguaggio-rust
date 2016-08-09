% Lo stack e lo heap

Come linguaggio di sistema, Rust opera a basso livello. Per chi viene da
un linguaggio ad alto livello, ci sono alcuni aspetti della programmazione di
sistema con cui potrebbe non essere familiare. Quello più importante è come
funziona la memoria, con lo stack e lo heap. Per chi è familiare con
i linguaggi simili al C, e con il modo in cui usano l'allocazione su stack,
questa sezione sarà un ripasso. Chi non lo fosse, imparerà questo concetto
generale, ma focalizzato su Rust.

Come quando si imparano molte altre cose, per iniziare useremo un modello
semplificato. Ciò consente di afferrare le basi, senza essere sopraffati
dai dettagli. Gli esempi che useremo non saranno perfettamente accurati, ma
rappresentativi di quello che adesso vogliamo imparare. Dopo aver appreso
le basi, con questa particolare astrazione, se ne potranno rimuovere le pecche
quando si impareranno gli allocatori, la memoria virtuale e
altri argomenti avanzati.

# La gestione della memoria

Questi due termini riguardano la gestione della memoria. Lo stack e lo heap
sono astrazioni che aiutano a determinare quando allocare e deallocare memoria.

Ecco un confronto ad alto livello:

Lo stack è molto veloce, ed è dove la memoria è allocata di default in Rust.
Ma l'allocazione è locale a una chiamata di funzione, ed è di dimensione
limitata. Lo heap, d'altro canto, è più lento, ed è esplicitamente allocato
dal programma. Ma è di dimensione illimitata, ed è accessibile a tutto
il programma.

# Lo stack

Parliamo di questo programma in Rust:

```rust
fn main() {
    let x = 42;
}
```

Questo programma ha un solo legame di variabile, `x`. Questa memoria deve
essere allocata da qualche parte. Rust di default ‘alloca sullo stack’, il che
significa che i valori di base ‘vanno sullo stack’. Che cosa significa?

Beh, quando una funzione viene chiamata, viene allocata della memoria per tutte
le sue variabili locali e alcune altre informazioni. Questa memoria si chiama
‘stack frame’, e per lo scopo di  questa sezione, ignoreremo le informazioni
aggiuntive e considereremo solamente le variabili locali che stiamo allocando.
Perciò in questo caso, quando viene eseguito `main()`, allocheremo un solo
intero a 32 bit per il nostro stack frame. Questo viene gestito
automaticamente, come si vede; non abbiamo dovuto scrivere nessun codice Rust
speciale.

Quando la funzione esce, il suo stack frame viene deallocato. Anche questo
avviene automaticamente.

Questo è tutto per questo semplice programma. La cosa chiave da capire qui
è che l'allocazione su stack è velocissima. Dato che il compilatore conosce
tutte le variabili locali, può afferrare la memoria necessaria in un colpo
solo. E dato che le butterà via tutte nello stesso istante, può sbarazzarsene
pue in un colpo solo.

Lo svantaggio è che non si possono tenere in giro valori se ci servono per
un tempo più lungo di una singola funzione. Inoltre non abbiamo ancora detto
che cosa significa la parola ‘stack’. Per farlo, ci serve un esempio
leggermente più complicato:

```rust
fn foo() {
    let y = 5;
    let z = 100;
}

fn main() {
    let x = 42;

    foo();
}
```

Questo programma ha in totale tre variabili: due in `foo()`, e una in `main()`.
Come prima, quando `main()` viene chiamata, un solo intero viene allocato
per il suo stack frame. Ma prima di poter mostrare ciò che accade quando
`foo()` viene chiamato, dobbiamo visualizzare cosa succede con la memoria.
Il sistema operativo presenta al programma un'immagine molto semplice
della memoria: un'enorme array di byte, accessibile tramite il loro indirizzo,
da 0 a un grande numero, che rappresenta quanta RAM il sistema operativo
concede al processo. Per esempio, se il processo ha un gigabyte di spazio di
indirizzamento, gli indirizzi vanno da `0` a `1,073,741,823`. Quel numero
equivale a 2<sup>30</sup>, cioè il numero di byte in un gigabyte. [^gigabyte]

[^gigabyte]: La parola ‘gigabyte’ viene usata con due possibili significati:
    10^9, o 2^30. Lo standard SI ha risolto l'ambiguità affermando che
    ‘gigabyte’ è 10^9, e ‘gibibyte’ è 2^30. Però, pochissima gente usa
    quest'ultima parola, e si affida al contesto per disambiguare.
    Qui seguiamo la tradizione.

Perciò ecco un diagramma del nostro primo stack frame:

| Indirizzo | Nome | Valore |
|-----------|------|--------|
| 0         | x    | 42     |

Abbiamo che `x` si trova all'indirizzo `0`, con il valore `42`.

Quando `foo()` viene chiamata, viene allocato un nuovo stack frame:

| Indirizzo | Nome | Valore |
|-----------|------|--------|
| 2         | z    | 100    |
| 1         | y    | 5      |
| 0         | x    | 42     |

Siccome `0` è stato preso dal primo frame, `1` e `2` sono usati dallo stack
frame di `foo()`. Cresce in su, man mano che chiamiamo funzioni.

Qui ci sono alcune cose importanti di cui dobbiamo prendere nota. I numeri 0,
1, e 2 sono a solo scopo illustrativo, e non hanno relazione con i valori degli
indirizzi che il computer userà in realtà. In particolare, le serie
di indirizzi in realtà saranno separati da un certo numero di byte che separano
ogni indirizzo dal successivo, e quella separazione può superare persino
da dimensione del valore memorizzato.

Dopo che `foo()` è finito, il suo frame viene deallocato:

| Indirizzo | Nome | Valore |
|-----------|------|--------|
| 0         | x    | 42     |

E poi, dopo `main()`, perfino quest'ultimo valore va via. Facile!

Si chiama ‘stack’ (che in italiano significa, "pila", "catasta") perché
funziona come una catasta di vassoi da cena: il primo vassoio che sia appoggia
sarà l'ultimo vassoio che si riprende. Per questa ragione, gli stack sono
talvolta chiamati ‘code di tipo ultimo dentro, primo fuori‘ ["Last In First
Out"], dato che l'ultimo valore messo sullo stack è il primo che ne verrà
preso.

Proviamo un esempio di profondità tre:

```rust
fn corsivo() {
    let i = 6;
}

fn grassetto() {
    let a = 5;
    let b = 100;
    let c = 1;

    corsivo();
}

fn main() {
    let x = 42;

    grassetto();
}
```

Abbiamo dato alle funzioni alcuni nomi strani per rendere più chiaro lo schema.

Bene, prima chiamiamo `main()`:

| Indirizzo | Nome | Valore |
|-----------|------|--------|
| 0         | x    | 42     |

Poi, `main()` chiama `grassetto()`:

| Indirizzo | Nome | Valore |
|-----------|------|--------|
| **3**     | **c**|**1**   |
| **2**     | **b**|**100** |
| **1**     | **a**|**5**   |
| 0         | x    | 42     |

E poi `grassetto()` chiama `corsivo()`:

| Indirizzo | Nome | Valore |
|-----------|------|--------|
| *4*       | *i*  | *6*    |
| **3**     | **c**|**1**   |
| **2**     | **b**|**100** |
| **1**     | **a**| **5**  |
| 0         | x    | 42     |

Urca! Il nostro stack sta diventando alto.

Dopo che `italic()` è finita, il suo frame viene deallocato, lasciando
solamente `grassetto()` e `main()`:

| Indirizzo | Nome | Valore |
|-----------|------|--------|
| **3**     | **c**|**1**   |
| **2**     | **b**|**100** |
| **1**     | **a**| **5**  |
| 0         | x    | 42     |

E poi `grassetto()` finisce, lasciando solamente `main()`:

| Indirizzo | Nome | Valore |
|-----------|------|--------|
| 0         | x    | 42     |

E poi abbiamo finito. Ci state prendendo mano? È come impilare dei piatti:
si aggiunge in cima, si toglie dalla cima.

# Lo Heap

Adesso, questo funziona bene, ma non tutto può funzionare così. Talvolta,
si deve passare della memoria fra diverse funzioni, o tenerla viva più a lungo
dell'esecuzione di una sola funzione. Per questo, si può usare lo heap.

In Rust, si può allocare memoria sullo heap con il [tipo `Box<T>`][box].
Ecco un esempio:

```rust
fn main() {
    let x = Box::new(5);
    let y = 42;
}
```

[box]: ../std/boxed/index.html

Ecco quello che succede in memoria quando si chiama `main()`:

| Indirizzo | Nome | Valore |
|-----------|------|--------|
| 1         | y    | 42     |
| 0         | x    | ?????? |

Allochiamo spazio per due variabili sullo stack. `y` vale `42`, come sempre,
ma che dire di `x`? Beh, `x` è un `Box<i32>`, e i box allocano memoria sullo
heap. L'effettivo valore del box è una struttura che ha un puntatore allo
‘heap’. Quando iniziamo a eseguire la funzione, e `Box::new()` viene chiamato,
alloca della memoria dallo heap, e ci mette `5`. La memoria adesso appare così:

| Indirizzo            | Nome | Valore                 |
|----------------------|------|------------------------|
| (2<sup>30</sup>) - 1 |      | 5                      |
| ...                  | ...  | ...                    |
| 1                    | y    | 42                     |
| 0                    | x    | → (2<sup>30</sup>) - 1 |

Nel nostro ipotetico processo con 1GB of RAM abbiamo (2<sup>30</sup>) - 1
possibili indirizzi. E dato che il nostro stack cresce da zero, il luogo
più facile per allocare memoria è dall'altra estremità. Perciò il nostro primo
valore è nel posto più alto della memoria. E il valore della struttura
legata a `x` ha un [puntatore grezzo][rawpointer] che punta al posto che
abbiamo allocato sullo heap, perciò il valore di `x` è (2<sup>30</sup>) - 1,
cioè la posizione che abbiamo richiesto.

[rawpointer]: raw-pointers.html

In realtà non abbiamo parlato troppo di ciò che effettivamente significa
allocare e deallocare memoria in questi contesti. In questo libro non si
scenderà in dettagli molto approfonditi, ma ciò che è importante
evidenziare qui è che lo heap non è uno stack che cresce dall'estremità
opposta. Più avanti ne faremo un esempio, ma siccome lo heap può essere
allocato e deallocato in qualunque ordine, può finire per avere dei ‘buchi’.
Ecco uno schema della disposizione di memoria di un programma che è stato
in esecuzione per un po' di tempo:


| Indirizzo            | Nome | Valore                 |
|----------------------|------|------------------------|
| (2<sup>30</sup>) - 1 |      | 5                      |
| (2<sup>30</sup>) - 2 |      |                        |
| (2<sup>30</sup>) - 3 |      |                        |
| (2<sup>30</sup>) - 4 |      | 42                     |
| ...                  | ...  | ...                    |
| 2                    | z    | → (2<sup>30</sup>) - 4 |
| 1                    | y    | 42                     |
| 0                    | x    | → (2<sup>30</sup>) - 1 |

In questo caso, abbiamo allocato quattro oggetti sullo heap, ma ne abbiamo poi
deallocati due di essi. Fra (2<sup>30</sup>) - 1 e (2<sup>30</sup>) - 4 c'è
dello spazio di memoria che al momento non è utilizzato. I dettagli specifici
di come e perché questo accada dipendono dalla strategia usata per gestire
lo heap. Diversi programmi possono usare diversi ‘allocatori di memoria’, che
sono librerie che implementano queste strategie. Normalmente, i programmi
in Rust usano [jemalloc][jemalloc] a questo scopo.

[jemalloc]: http://www.canonware.com/jemalloc/

Comunque, torniamo al nostro esempio. Dato che questa memoria sta nello heap,
può rimanere viva più a lungo della funzione che alloca il box. In questo caso,
però, non lo fa.[^moving] Quando la funzione è finita, dobbiamo deallocare
lo stack frame di `main()`. `Box<T>`, però, ha un asso nella manica:
[Drop][drop]. L'implementazione di `Drop` per `Box` dealloca la memoria che
è stata allocata quando è stato creato. Ottimo! Perciò, quando `x` se ne va,
prima dealloca la memoria che ha allocato sullo heap:

| Indirizzo | Nome | Valore |
|-----------|------|--------|
| 1         | y    | 42     |
| 0         | x    | ?????? |

[drop]: drop.html
[^moving]: Possiamo fare in modo che la memoria viva più a lungo tramite
           il traferimento del possesso, talvolta chiamato ‘spostare fuori
           dal box’. Esempi più complessi verranno trattati più avanti.

E poi lo stack frame se ne va, rilasciando tutta la sua memoria.

# Argomenti e prestiti

Abbiamo visto alcuni semplici esempi che usano lo stack e lo heap, ma che dire
degli argomenti di funzione e dei prestiti? Ecco un piccolo programma in Rust:

```rust
fn foo(i: &i32) {
    let z = 42;
}

fn main() {
    let x = 5;
    let y = &x;

    foo(y);
}
```

Quando entriamo in `main()`, la memoria si presenta così:

| Indirizzo | Nome | Valore |
|-----------|------|--------|
| 1         | y    | → 0    |
| 0         | x    | 5      |

`x` è un semplice `5`, mentre `y` è un riferimento a `x`. Perciò il suo valore
è la posizione di memoria in cui risiede `x`, che in questo caso è `0`.

E che succede quando chiamiamo `foo()`, passandogli `y` come argomento?

| Indirizzo | Nome | Valore |
|-----------|------|--------|
| 3         | z    | 42     |
| 2         | i    | → 0    |
| 1         | y    | → 0    |
| 0         | x    | 5      |

Gli stack frame non servono solamente ai legami locali, servono anche
agli argomenti. Perciò in questo caso, dobbiamo avere sia `i`, che è il nostro
argomento, che `z`, che è il nostro legame locale a variabile. `i` è una copia
dell'argomento, `y`. Dato che il valore di `y` è `0`, lo è anche di `i`.

Questa è una ragione per cui prestare una variabile non dealloca memoria:
il valore di un riferimento è un puntatore a una posizione in memoria. Se ci
fossimo sbarazzati della memoria soggiaciente, le cose non potrebbero
funzionare.

# Un esempio complesso

Bene, adesso esaminiamo passo per passo questo programma complesso:

```rust
fn foo(x: &i32) {
    let y = 10;
    let z = &y;

    baz(z);
    bar(x, z);
}

fn bar(a: &i32, b: &i32) {
    let c = 5;
    let d = Box::new(5);
    let e = &d;

    baz(e);
}

fn baz(f: &i32) {
    let g = 100;
}

fn main() {
    let h = 3;
    let i = Box::new(20);
    let j = &h;

    foo(j);
}
```

Prima, chiamiamo `main()`:

| Indirizzo            | Nome | Valore                 |
|----------------------|------|------------------------|
| (2<sup>30</sup>) - 1 |      | 20                     |
| ...                  | ...  | ...                    |
| 2                    | j    | → 0                    |
| 1                    | i    | → (2<sup>30</sup>) - 1 |
| 0                    | h    | 3                      |

Allochiamo memoria per `j`, `i`, e `h`. `i` è sullo heap, e quindi ha un valore
che punta là.

Poi, alla fine di `main()`, viene chiamata `foo()`:

| Indirizzo            | Nome | Valore                 |
|----------------------|------|------------------------|
| (2<sup>30</sup>) - 1 |      | 20                     |
| ...                  | ...  | ...                    |
| 5                    | z    | → 4                    |
| 4                    | y    | 10                     |
| 3                    | x    | → 0                    |
| 2                    | j    | → 0                    |
| 1                    | i    | → (2<sup>30</sup>) - 1 |
| 0                    | h    | 3                      |

si alloca dello spazio per `x`, `y`, e `z`. L'argomento `x` ha lo stesso valore
di `j`, dato che è quello che gli abbiamo passato. È un puntatore all'indirizzo
`0`, dato che `j` punta ad `h`.

Poi, `foo()` chiama `baz()`, passandogli `z`:

| Indirizzo            | Nome | Valore                 |
|----------------------|------|------------------------|
| (2<sup>30</sup>) - 1 |      | 20                     |
| ...                  | ...  | ...                    |
| 7                    | g    | 100                    |
| 6                    | f    | → 4                    |
| 5                    | z    | → 4                    |
| 4                    | y    | 10                     |
| 3                    | x    | → 0                    |
| 2                    | j    | → 0                    |
| 1                    | i    | → (2<sup>30</sup>) - 1 |
| 0                    | h    | 3                      |

Abbiamo allocato memoria per `f` e per `g`. `baz()` è molto breve, perciò
quando è finita, ci sbarazziamo del suo stack frame:

| Indirizzo            | Nome | Valore                 |
|----------------------|------|------------------------|
| (2<sup>30</sup>) - 1 |      | 20                     |
| ...                  | ...  | ...                    |
| 5                    | z    | → 4                    |
| 4                    | y    | 10                     |
| 3                    | x    | → 0                    |
| 2                    | j    | → 0                    |
| 1                    | i    | → (2<sup>30</sup>) - 1 |
| 0                    | h    | 3                      |

Poi, `foo()` chiama `bar()` con `x` e `z`:

| Indirizzo            | Nome | Valore                 |
|----------------------|------|------------------------|
| (2<sup>30</sup>) - 1 |      | 20                     |
| (2<sup>30</sup>) - 2 |      | 5                      |
| ...                  | ...  | ...                    |
| 10                   | e    | → 9                    |
| 9                    | d    | → (2<sup>30</sup>) - 2 |
| 8                    | c    | 5                      |
| 7                    | b    | → 4                    |
| 6                    | a    | → 0                    |
| 5                    | z    | → 4                    |
| 4                    | y    | 10                     |
| 3                    | x    | → 0                    |
| 2                    | j    | → 0                    |
| 1                    | i    | → (2<sup>30</sup>) - 1 |
| 0                    | h    | 3                      |

Finiamo con l'allocare un altro valore sullo heap, e quindi dobbiamo sottrarre
uno da (2<sup>30</sup>) - 1. È più facile da scrivere che `1.073.741.822`.
In ogni caso, impostiamo le variabili come al solito.

`bar()`, alla fine, chiama `baz()`:

| Indirizzo            | Nome | Valore                 |
|----------------------|------|------------------------|
| (2<sup>30</sup>) - 1 |      | 20                     |
| (2<sup>30</sup>) - 2 |      | 5                      |
| ...                  | ...  | ...                    |
| 12                   | g    | 100                    |
| 11                   | f    | → 9                    |
| 10                   | e    | → 9                    |
| 9                    | d    | → (2<sup>30</sup>) - 2 |
| 8                    | c    | 5                      |
| 7                    | b    | → 4                    |
| 6                    | a    | → 0                    |
| 5                    | z    | → 4                    |
| 4                    | y    | 10                     |
| 3                    | x    | → 0                    |
| 2                    | j    | → 0                    |
| 1                    | i    | → (2<sup>30</sup>) - 1 |
| 0                    | h    | 3                      |

Con questo, siamo arrivati al punto più profondo! Urca! Congratulazioni
essere arrivati fino a questo punto.

Dopo che `baz()` è finita, ci sbarazziamo di `f` e di `g`:

| Indirizzo            | Nome | Valore                 |
|----------------------|------|------------------------|
| (2<sup>30</sup>) - 1 |      | 20                     |
| (2<sup>30</sup>) - 2 |      | 5                      |
| ...                  | ...  | ...                    |
| 10                   | e    | → 9                    |
| 9                    | d    | → (2<sup>30</sup>) - 2 |
| 8                    | c    | 5                      |
| 7                    | b    | → 4                    |
| 6                    | a    | → 0                    |
| 5                    | z    | → 4                    |
| 4                    | y    | 10                     |
| 3                    | x    | → 0                    |
| 2                    | j    | → 0                    |
| 1                    | i    | → (2<sup>30</sup>) - 1 |
| 0                    | h    | 3                      |

Poi, ritorniamo da `bar()`. `d` in questo caso è un `Box<T>`, quindi dealloca
anche quello a cui punta: (2<sup>30</sup>) - 2.

| Indirizzo            | Nome | Valore                 |
|----------------------|------|------------------------|
| (2<sup>30</sup>) - 1 |      | 20                     |
| ...                  | ...  | ...                    |
| 5                    | z    | → 4                    |
| 4                    | y    | 10                     |
| 3                    | x    | → 0                    |
| 2                    | j    | → 0                    |
| 1                    | i    | → (2<sup>30</sup>) - 1 |
| 0                    | h    | 3                      |

E dopo di quello, `foo()` termina:

| Indirizzo            | Nome | Valore                 |
|----------------------|------|------------------------|
| (2<sup>30</sup>) - 1 |      | 20                     |
| ...                  | ...  | ...                    |
| 2                    | j    | → 0                    |
| 1                    | i    | → (2<sup>30</sup>) - 1 |
| 0                    | h    | 3                      |

e poi, finalmente, anche `main()`, che ripulisce tutto lo stack. Quando `i`
esegue `Drop`, ripulirà anche tutto lo heap.

# Che cosa fanno gli altri linguaggi?

La maggior parte dei linguaggi che usano un garbage collector allocano
sullo heap di default. Ciò significa che ogni valore è racchiuso in un boxed.
Ci sono varie ragioni per cui viene fatto, ma non ne parlermo. Però sono
possibili alcune ottimizzazioni che non lo rendono sempre vero. Invece
di affidarsi allo stack ed eseguire `Drop` per rilasciare la memoria,
il garbage collector gestisce direttamente lo heap.

# Quale usare?

Dunque se lo stack è più veloce e più facile da gestire, che bisogno abbiamo
dello heap? Una grossa ragione è che avere solamente l'allocazione su stack
significa avere solmente una semantica di tipo 'ultimo dentro primo fuori'
per recuperare la memoria. L'allocazione su heap è più generale, consentendo
di rilasciare pezzi di memoria in ordine arbitrario, ma al costo di maggiore
complessità.

In generale, si dovrebbe preferire l'allocazione su stack, e quindi, Rust
alloca su stack di default. Il modello LIFO dello stack è più semplice, ad
un livello fondamentale. Ciò ha due grossi impatti: efficienza in fase
di esecuzione e impatto semantico.

## Efficienza in fase di esecuzione

Gestire la memoria sullo stack è banale: la macchina incrementa o decrementa
un singolo valore, il cosiddetto “puntatore allo stack”.
Gestire la memoria sullo heap non è affatto banale: la memoria allocata
sullo heap viene rilasciata in punti arbitrari, e ogni blocco di memoria
allocata sullo heap può essere di dimensione arbitraria, e quindi il gestore
della memoria in generale deve fare un lavoro molto più difficile per trovare
gli spazi di memoria da riutilizzare.

Per chi volesse approfondire questo argomento, [questo articolo][wilson]
è un'ottima introduzione.

[wilson]: http://citeseerx.ist.psu.edu/viewdoc/summary?doi=10.1.1.143.4688

## Impatto semantico

L'allocazione su stack ha un impatto sullo stesso linguaggio Rust, e così
sul modello mentale dello sviluppatore. La semantica LIFO è ciò che guida
come il linguaggio Rust gestisce automaticamente la memoria. Perfino
la deallocazione di un box allocato sullo heap e avente un solo possessore
può essere guidata dalla semantica LIFO basata su stack, come ampiamente
discusso in questa sezione. La flessibilità (cioè l'espressività) della
semantica LIFO significa che in generale il compilatore non può automaticamente
inferire in fase di compilazione dove dovrebbe essere deallocata la memoria;
deve affidarsi su protocolli dinamici, potenzialmente esterni al linguaggio
stesso, per guidare la deallocazione (il conteggio dei riferimenti, usato da
`Rc<T>` e da `Arc<T>`, ne è un esempio).

Se portato all'estremo, l'accresciuto potere espressivo dell'allocazione
su heap allocation comporta o il costo di un significativo supporto di fase
di esecuzione (per es. sotto forma di un garbage collector) o un significativo
sforzo del programmatore (sotto forma di chiamate esplicite di gestione
della memoria che richiedono verifiche non fornite dal compilatore).
