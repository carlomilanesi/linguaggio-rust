% Indice della sintassi

## Parole-chiave

* `as`: cast dei tipi primitivi, o per disambiguare il tratto specifico contenente un elemento.  Si veda [cast fra tipi (`as`)], [forma con parentesi angolari], [tipi associati].
* `break`: salta fuori dal ciclo.  Si veda [uscita precoce dai cicli].
* `const`: elementi costanti e puntatori grezzi costanti.  Si veda [`const` and `static`], [puntatori grezzi].
* `continue`: procedi alla prossima iterazione del ciclo.  Si veda [uscita precoce dai cicli].
* `crate`: link con un crate esterno.  Si veda [importare crate esterni].
* `else`: alternativa per i costrutti `if` e `if let`.  Si veda [`if`], [`if let`].
* `enum`: definizione di un'enumerazione.  Si veda [enum].
* `extern`: crate, funzione, o variabile con link esterno.  Si veda [importare crate esterni], [interfaccia alle funzioni straniere].
* `false`: letterale boolean falso.  Si veda [booleani].
* `fn`: definizione di funzione e tipo di puntatore a funzione.  Si veda [funzioni].
* `for`: ciclo su iteratore, parte della sintassi di `impl` di un tratto, e sintassi dei tempi di vita di rango superiore.  Si veda [cicli `for`], [sintassi dei metodi].
* `if`: diramazione condizionale.  Si veda [`if`], [`if let`].
* `impl`: blocco di implementazione di strutture o di tratti.  Si veda [sintassi dei metodi].
* `in`: parte della sintassi del ciclo `for`.  Si veda [cicli `for`].
* `let`: legame di variabile.  Si veda [legami di variabili].
* `loop`: ciclo infinite.  Si veda [cicli `loop`].
* `match`: pattern matching.  Si veda [match].
* `mod`: dichiarazione di modulo.  Si veda [definire moduli].
* `move`: parte della sintassi di chiusura.  Si veda [chiusure `move`].
* `mut`: denota mutabilità nei tipi puntatore e nei legami di pattern.  Si veda [mutabilità].
* `pub`: denota visibilità pubblica nei campi delle `struct`, dei blocchi `impl`, e dei moduli.  Si veda [Crates and Modules (Exporting a Public Interface)].
* `ref`: legame per riferimento.  Si veda [Patterns (`ref` and `ref mut`)].
* `return`: uscita precoce da funzione.  Si veda [uscita precoce dalle funzioni].
* `Self`: alias del tipo che si sta implementando.  Si veda [tratti].
* `self`: suggetto del metodo corrente.  Si veda [chiamate di metodo].
* `static`: variabile globale.  Si veda [`const` and `static` (`static`)].
* `struct`: definizione di struttura.  Si veda [struct].
* `trait`: definizione di tratto.  Si veda [tratti].
* `true`: letterale boolean vero.  Si veda [booleani].
* `type`: alias di tipo, e definizione di tipo associato.  Si veda [`type`], [tipi associati].
* `unsafe`: denota codice, funzione, tratto, o implementazione non sicuro.  Si veda [Unsafe].
* `use`: importa simboli nell'ambito corrente.  Si veda [Crates and Modules (Importing Modules with `use`)].
* `where`: clausola di vincolo sul tipo generico.  Si veda [clausola `where` per i tratti)].
* `while`: ciclo condizionale.  Si veda [cicli `while`].

## Operatori e simboli

* `!` (`ident!(…)`, `ident!{…}`, `ident![…]`): denota una espansione di macro.  Si veda [macro].
* `!` (`!expr`): complemento bit-a-bit o negazione logica.  Sovraccaricabile (`Not`).
* `!=` (`var != expr`): diseguaglianza.  Sovraccaricabile (`PartialEq`).
* `%` (`expr % expr`): resto aritmetico.  Sovraccaricabile (`Rem`).
* `%=` (`var %= expr`): resto aritmetico e assegnmento. Sovraccaricabile (`RemAssign`).
* `&` (`expr & expr`): And bit-a-bit.  Sovraccaricabile (`BitAnd`).
* `&` (`&expr`): prestito.  Si veda [References and Borrowing].
* `&` (`&type`, `&mut type`, `&'a type`, `&'a mut type`): tipo di puntatore preso a prestito.  Si veda [riferimenti e prestiti].
* `&=` (`var &= expr`): And bit-a-bit e assegnmento. Sovraccaricabile (`BitAndAssign`).
* `&&` (`expr && expr`): And logico.
* `*` (`expr * expr`): Moltiplicazione aritmetica.  Sovraccaricabile (`Mul`).
* `*` (`*expr`): dereferenziazione.
* `*` (`*const type`, `*mut type`): puntatore grezzo.  Si veda [Raw Pointers].
* `*=` (`var *= expr`): moltiplicazione aritmetica e assegnmento. Sovraccaricabile (`MulAssign`).
* `+` (`expr + expr`): addizione aritmetica.  Sovraccaricabile (`Add`).
* `+` (`trait + trait`, `'a + trait`): vincolo composto su tipo.  Si veda [legami a tratti multipli].
* `+=` (`var += expr`): addizion aritmetica e assegnmento. Sovraccaricabile (`AddAssign`).
* `,`: separatore di argomenti e di elementi.  Si veda [attributi], [funzioni], [struct], [generici], [match], [chiusure], [importare moduli usando `use`)].
* `-` (`expr - expr`): sottrazione aritmetica.  Sovraccaricabile (`Sub`).
* `-` (`- expr`): negazione aritmetica.  Sovraccaricabile (`Neg`).
* `-=` (`var -= expr`): sottrazione aritmetica e assegnmento. Sovraccaricabile (`SubAssign`).
* `->` (`fn(…) -> type`, `|…| -> type`): tipo reso da funzione o chiusura.  Si veda [Functions], [Closures].
* `-> !` (`fn(…) -> !`, `|…| -> !`): funzione divergente o chiusura.  Si veda [Diverging Functions].
* `.` (`expr.ident`): accesso a membro.  Si veda [Structs], [Method Syntax].
* `..` (`..`, `expr..`, `..expr`, `expr..expr`): letterale di gamma esclusivo a destra.
* `..` (`..expr`): struct literal update syntax.  Si veda [Structs (Update syntax)].
* `..` (`variant(x, ..)`, `struct_type { x, .. }`): legame di pattern "tutto il resto".  Si veda [ignorare i legami].
* `...` (`...expr`, `expr...expr`) *in un'espressione*: espressione di gamma inclusiva.  Si veda [iteratori].
* `...` (`expr...expr`) *in un pattern*: pattern di gamma inclusivo.  Si veda [gamme].
* `/` (`expr / expr`): divisione aritmetica.  Sovraccaricabile (`Div`).
* `/=` (`var /= expr`): division aritmetica e assegnmento. Sovraccaricabile (`DivAssign`).
* `:` (`pat: type`, `ident: type`): vincoli.  Si veda [legami di variabili], [funzioni], [struct], [tratti].
* `:` (`ident: expr`): inizializzatore di campo di struct.  Si veda [struct].
* `:` (`'a: loop {…}`): etichetta di ciclo.  Si veda [etichette di cicli].
* `;`: istruzione e terminatore di elemento.
* `;` (`[…; len]`): parte della sintassi di un array.  Si veda [array].
* `<<` (`expr << expr`): scorrimento a sinistra.  Sovraccaricabile (`Shl`).
* `<<=` (`var <<= expr`): scorrimento a sinistra e assegnmento. Sovraccaricabile (`ShlAssign`).
* `<` (`expr < expr`): meno di.  Sovraccaricabile (`PartialOrd`).
* `<=` (`var <= expr`): meno di o uguale a.  Sovraccaricabile (`PartialOrd`).
* `=` (`var = expr`, `ident = type`): assegnamento/equivalenza.  Si veda [legami di variabili], [`type`], generic parameter defaults.
* `==` (`var == expr`): uguaglianza.  Sovraccaricabile (`PartialEq`).
* `=>` (`pat => expr`): parte della sintassi del braccio di match.  Si veda [Match].
* `>` (`expr > expr`): maggiore di.  Sovraccaricabile (`PartialOrd`).
* `>=` (`var >= expr`): maggiore di o uguale a.  Sovraccaricabile (`PartialOrd`).
* `>>` (`expr >> expr`): scorrimento a destra.  Sovraccaricabile (`Shr`).
* `>>=` (`var >>= expr`): scorrimento a destra e assegnmento. Sovraccaricabile (`ShrAssign`).
* `@` (`ident @ pat`): legame di pattern.  Si veda [Patterns (Bindings)].
* `^` (`expr ^ expr`): Xor bit-a-bit.  Sovraccaricabile (`BitXor`).
* `^=` (`var ^= expr`): Xor bit-a-bit e assegnmento. Sovraccaricabile (`BitXorAssign`).
* `|` (`expr | expr`): Or bit-a-bit.  Sovraccaricabile (`BitOr`).
* `|` (`pat | pat`): alternativa di un pattern.  Si veda [pattern multipli].
* `|` (`|…| expr`): chiusura.  Si veda [Closures].
* `|=` (`var |= expr`): Or bit-a-bit e assegnmento. Sovraccaricabile (`BitOrAssign`).
* `||` (`expr || expr`): Or logico.
* `_`: legame di pattern "ignorato".  Si veda [ignorare i legami].

## Altra sintassi

<!-- Vari pezzetti di roba autonoma. -->

* `'ident`: tempo di vita con nome o etichetta di ciclo.  Si veda [Lifetimes], [Loops (Loops Labels)].
* `…u8`, `…i32`, `…f64`, `…usize`, …: letterale numerico di tipo specifico.
* `"…"`: letterali di stringa.  Si veda [Strings].
* `r"…"`, `r#"…"#`, `r##"…"##`, …: letterali di stringa grezza, in cui i caratteri di escape non vengono elaborati.  Si veda [Reference (Raw String Literals)].
* `b"…"`: letterale di stringa di byte, che costruisce un `[u8]` invece di una stringa.  Si veda [Reference (Byte String Literals)].
* `br"…"`, `br#"…"#`, `br##"…"##`, …: letterale di stringa di byte grezza, combinazione dei letterali di stringa di byte e grezzi.  Si veda [Reference (Raw Byte String Literals)].
* `'…'`: letterale di carattere.  Si veda [`char`].
* `b'…'`: letterale di byte ASCII.
* `|…| expr`: chiusura.  Si veda [chiusure].

<!-- Sintassi correlata ai percorsi -->

* `ident::ident`: percorso.  Si veda [definire moduli].
* `::path`: percorso relativo alla radice del crate (*cioè* un percorso esplicitamente assoluto).  Si veda [Crates and Modules (Re-exporting with `pub use`)].
* `self::path`: percorso relativo al modulo corrente (*cioè* un percorso esplicitamente relativo).  Si veda [Crates and Modules (Re-exporting with `pub use`)].
* `super::path`: percorso relativo al genitore del modulo corrente.  Si veda [Crates and Modules (Re-exporting with `pub use`)].
* `type::ident`, `<type as trait>::ident`: costanti, funzioni, e tipi associati.  Si veda [tipi associati].
* `<type>::…`: elementi associati per un tipo che non può essere nominato direttamente (*per es.* `<&T>::…`, `<[T]>::…`, *ecc.*).  Si veda [tipi associati].
* `trait::method(…)`: disambiguare una chiamata di metodo nominando il tratto che lo definisce.  Si veda [sintassi universale di chiamata di funzione].
* `type::method(…)`: disambiguare una chiamata di metodo nominando il tipo per cui è definito.  Si veda [sintassi universale di chiamata di funzione].
* `<type as trait>::method(…)`: disambiguare una chiamata di metodo nominando il tratto _e_ il tipo.  Si veda [forma con parentesi angolari].

<!-- Generici -->

* `path<…>` (*e.g.* `Vec<u8>`): specifica i parametri a un tipo generico *in un tipo*.  Si veda [Generics].
* `path::<…>`, `method::<…>` (*e.g.* `"42".parse::<i32>()`): specifica i parametri a un tipo, funzione, o metodo generico *in un'espressione*.
* `fn ident<…> …`: definisce una funzione generica.  Si veda [Generics].
* `struct ident<…> …`: definisce una struttura generica.  Si veda [Generics].
* `enum ident<…> …`: definisce un'enumerazione generica.  Si veda [Generics].
* `impl<…> …`: definisce un'implementazione generica.
* `for<…> type`: limiti di tempo di vita di rango superiore.
* `type<ident=type>` (*per es.* `Iterator<Item=T>`): un tipo generico dove uno o più tipi associati hanno assegnamenti specifici.  Si veda [tipi associati].

<!-- Vincoli -->

* `T: U`: parametro generico `T` vincolatoa tipi che implementano `U`.  Si veda [tratti].
* `T: 'a`: il tipo generico `T` deve soppravvivere al tempo di vita `'a`. Quando diciamo che un tipo 'sopravvive' il tempo di vita, intendiamo che non può transitivamente contenere riferimenti con tempi di vita più brevi di `'a`.
* `T : 'static`: il tipo generico `T` non contiene riferimenti presi in prestito eccetto quelli `'static`.
* `'b: 'a`: il tempo di vita generico `'b` deve sopravvivere `'a`.
* `T: ?Sized`: consenti al parametro di tipo generico di essere un tipo dimensionato dinamicamente.  Si veda [`?Sized`].
* `'a + trait`, `trait + trait`: vincoli composito di tipo.  Si veda [legami a tratti multipli].

<!-- Macro e attributi -->

* `#[meta]`: attributo esterno.  Si veda [attributi].
* `#![meta]`: attributo interno.  Si veda [attributi].
* `$ident`: sostituzione di macro.  Si veda [macro].
* `$ident:kind`: cattura di macro.  Si veda [macro].
* `$(…)…`: ripetizione di macro.  Si veda [macro].

<!-- Commenti -->

* `//`: commento di riga.  Si veda [commenti].
* `//!`: commento di documentazione di riga interno.  Si veda [commenti].
* `///`: commento di documentazione di riga esterno.  Si veda [commenti].
* `/*…*/`: commento di blocco.  Si veda [commenti].
* `/*!…*/`: commento di documentazione di blocco interno.  Si veda [commenti].
* `/**…*/`: commento di documentazione di blocco esterno.  Si veda [commenti].

<!-- Varie cose che coinvolgono parentesi ed ennuple -->

* `()`: ennupla vuota (*detta anche* unità), sia come costante che come tipo.
* `(expr)`: espressione tra parentesi.
* `(expr,)`: espressione di ennupla di un solo elemento.  Si veda [ennuple].
* `(type,)`: tipo di ennupla di un solo elemento.  Si veda [ennuple].
* `(expr, …)`: espressione di ennupla.  Si veda [ennuple].
* `(type, …)`: tipo di ennupla.  Si veda [ennuple].
* `expr(expr, …)`: espressione di chiamata di funzione.  Usata anche per inizializzare le varianti di strutture ennuple e di enumerazioni ennuple.  Si veda [Functions].
* `ident!(…)`, `ident!{…}`, `ident![…]`: invocazione di macro.  Si veda [macro].
* `expr.0`, `expr.1`, …: indicizzazione di ennupla.  Si veda [indicizzazione di ennuple].

<!-- Cose relative alle graffe -->

* `{…}`: espressione di blocco.
* `Type {…}`: letterale di `struct`.  Si veda [struct].

<!-- Cose relative alle quadre -->

* `[…]`: letterale di array.  Si veda [array].
* `[expr; len]`: letterale di array contenente `len` copie di `expr`.  Si veda [array].
* `[type; len]`: tipo di array contenente `len` istanze di `type`.  Si veda [array].
* `expr[expr]`: indicizzazione di collezione.  Sovraccaricabile (`Index`, `IndexMut`).
* `expr[..]`, `expr[a..]`, `expr[..b]`, `expr[a..b]`: indicizzazione di collezione che fa finta di essere un'affettatura di collezione, usando `Range`, `RangeFrom`, `RangeTo`, `RangeFull` come "indice".

[`const` e `static` (`static`)]: const-and-static.html#static
[`const` e `static`]: const-and-static.html
[`if let`]: if-let.html
[`if`]: if.html
[`type`]: type-aliases.html
[tipi associati]: associated-types.html
[attributi]: attributes.html
[cast fra tipi (`as`)]: casting-between-types.html#as
[chiusure `move`]: closures.html#move-closures
[chiusure]: closures.html
[commenti]: comments.html
[definire moduli]: crates-and-modules.html#defining-modules
[esportare un'interfaccia pubblica]: crates-and-modules.html#exporting-a-public-interface
[importare crate esterni]: crates-and-modules.html#importing-external-crates
[importare moduli usando `use`)]: crates-and-modules.html#importing-modules-with-use
[riesportare usando `pub use`)]: crates-and-modules.html#re-exporting-with-pub-use
[funzioni divergenti]: functions.html#diverging-functions
[enum]: enums.html
[interfaccia alle funzioni straniere]: ffi.html
[uscita precoce dalle funzioni]: functions.html#early-returns
[funzioni]: functions.html
[generici]: generics.html
[iteratori]: iterators.html
[tempi di vita]: lifetimes.html
[cicli `for`]: loops.html#for
[cicli `loop`]: loops.html#loop
[cicli `while`]: loops.html#while
[uscita precoce dai cicli]: loops.html#ending-iteration-early
[etichette dei cicli]: loops.html#loop-labels
[macro]: macros.html
[match]: match.html
[chiamate dei metodi]: method-syntax.html#method-calls
[sintassi dei metodi]: method-syntax.html
[mutabilità]: mutability.html
[operatori e sovraccaricamento]: operators-and-overloading.html
[pattern con `ref` e `ref mut`]: patterns.html#ref-and-ref-mut
[legami dei pattern]: patterns.html#bindings
[ignorare i legami]: patterns.html#ignoring-bindings
[pattern multipli]: patterns.html#multiple-patterns
[gamme]: patterns.html#ranges
[`char`]: primitive-types.html#char
[array]: primitive-types.html#arrays
[booleani]: primitive-types.html#booleans
[indicizzazione di ennuple]: primitive-types.html#tuple-indexing
[ennuple]: primitive-types.html#tuples
[puntatori grezzi]: raw-pointers.html
[letterali di stringa di byte]: ../reference.html#byte-string-literals
[letterali di stringa di byte grezza]: ../reference.html#raw-byte-string-literals
[letterali di stringa grezza]: ../reference.html#raw-string-literals
[riferimenti e prestiti]: references-and-borrowing.html
[stringhe]: strings.html
[sintassi di aggiornamento delle struct]: structs.html#update-syntax
[struct]: structs.html
[clausola `where` dei tratti]: traits.html#where-clause
[legami a tratti multipli]: traits.html#multiple-trait-bounds
[tratti]: traits.html
[sintassi universale di chiamata di funzione]: ufcs.html
[forma con parentesi angolari]: ufcs.html#angle-bracket-form
[unsafe]: unsafe.html
[`?Sized`]: unsized-types.html#sized
[legami di variabili]: variable-bindings.html
