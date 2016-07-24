% Glossario

ownership=proprietà
borrow=prendere a prestito
scope=ambito
range=gamma
closure=chiusura
tuple=ennupla
structure=struttura
array=schiera
generics=costrutti generici
lifetime=tempo di vita
drop=distruzione
trait=tratto
reference-count=conteggio di riferimenti
executable=programma
binary=programma
default=comportamento predefinito
entry point=punto di ingresso
library=libreria
overriding=scavalcare

Non tutti i Rustacei hanno un curriculum di programmazione di sistema,
né in informatica, e quindi abbiamo aggiunto le spiegazioni dei termini
che potrebbero risultare poco chiari.

### Albero sintattico astratto ("Abstract Syntax Tree")

Quando un compilatore sta compilando un programma, fa varie diverse cose.
Una di queste è trasformare il testo del programma in un ‘albero sintattico
astratto', o ‘AST’ da ("Abstract Syntax Tree"). Questo albero è
una rappresentazione della struttura del programma. Per esempio, `2 + 3`
può essere trasformato nel seguente albero:

```text
  +
 / \
2   3
```

E `2 + 3 * 4` si presenterebbe così:

```text
  +
 / \
2   *
   / \
  3   4
```

### Arietà [Arity]

L'arietà di una funzione è il numero di argomenti che richiede.

```rust
let x = (2, 3);
let y = (4, 6);
let z = (8, 2, 6);
```

Nell'esempio sopra, `x` e `y` hanno arietà 2, mentre `z` ha arietà 3.

### Limiti [Bounds]

I limiti sono vincoli di un tipo [tratto][tratti]. Per esempio, se un limite
viene posto sull'argomento che una funzione prende, i tipi passati
a quella funzione devono conformarsi a quel vincolo.

[tratti]: traits.html

### Combinatori [Combinators]

I combinatori sono funzioni di livello superiore [higher-order functions]
che applicano solamente funzioni e combinatori già definiti per fornire
un risultato tra i loro argomenti.
Possono servire per gestire il flusso di controllo in modo modulare.

### Tipo dimensionato dinamicamente [Dynamically Sized Type] (DST)

Un tipo la cui dimensione o allineamento non sono noti in fase di compilazione,
cioè staticamente. ([altre info][link])

[link]: ../nomicon/exotic-sizes.html#dynamically-sized-types-dsts

### Espressione

Nella programmazione dei computer, un'espressione è una combinazione di valori,
costanti, variabili, operatori e funzioni, la cui valutazione rende un singolo
valore. Per esempio, `2 + 3 * 4` è un'espressione che vale 14. Vale la pena
notare che le espressioni possono avere effetti collaterali. Per esempio,
una chiamata di funzione all'interno di una espressione potrebbe eseguire
delle azioni, oltre a rendere un valore.

### Linguaggio orientato alle espressioni

Nei primi linguaggi di programmazione, [espressioni][espressione] e
[istruzioni][istruzione] erano due categorie sintattiche distinte:
le espressioni avevano un valore e le istruzioni facevano delle cose.
Tuttavia, i linguaggi successivi hanno sfumato questa distinzione, consentendo
alle espressioni di fare delle cose e alle istruzioni di avere un valore.
In un linguaggio orientato alle espressioni, (quasi) ogni istruzione è
un'espressione e perciò rende un valore. Di conseguenza, queste
istruzioni-espressioni possono esse stesse far parte di espressioni più grandi.

[espressione]: glossary.html#expression
[istruzione]: glossary.html#statement

### Istruzione

Nella programmazione dei computer, un'istruzione è il più piccolo elemento
autonomo di un linguaggio di programmazione che ordina a un computer
di eseguire un'azione.
