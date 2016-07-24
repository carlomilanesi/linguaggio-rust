% Enumerazioni ["enum"]

Una enumerazione in Rust è un tipo, dichiarato usando la parola-chiave `enum`,
che rappresenta un dato che ha un valore scelto in un elenco di varianti.
Ogni variante dell'enumerazione può avere dei dati associati ad essa.
Per esempio:

```rust
enum Message {
    Abbandona,
    CambiaColore(i32, i32, i32),
    Muovi { x: i32, y: i32 },
    Scrivi(String),
}
```

La sintassi per definire le varianti somiglia alle sintassi usate per definire
le strutture: si possono avere varianti senza dati (come le strutture
simili a unità), varianti con dati con nome, e varianti con dati senza nome
(come le strutture ennupla). Diversamente dalle definizioni di strutture
distinte, però, un `enum` è un singolo tipo. Un valore di un'enumerazione
può corrispondere a una qualunque delle sue varianti. Per questa ragione,
un'enumerazione è talvolta chiamata un ‘tipo somma’: l'insieme dei possibili
valori dell'enumerazione è la somma degli insiemi dei possibili valori
di ogni variante.

Usiamo la sintassi `::` per usare il nome di ogni variante: il loro nome
può essere usato solo se qualificato dal nome dell'`enum` stesso. Ciò
consente di avere le seguenti dichiarazioni di `x` e `y`:

```rust
# enum Messaggio {
#     Mossa { x: i32, y: i32 },
# }
let x: Messaggio = Messaggio::Mossa { x: 3, y: 4 };

enum TurnoGiocoDaTavolo {
    Mossa { quadrati: i32 },
    Passo,
}

let y: TurnoGiocoDaTavolo = TurnoGiocoDaTavolo::Mossa { quadrato: 1 };
```

Entrambe le varianti si chiamano `Mossa`, ma dato che sono qualificate dal nome
dell'enumerazione, possono essere usate entrambe senza creare ambiguità.

Un valore di un tipo `enum` contiene l'informazione su quale variante è,
oltre a ogni dato associato a quella variante. Questo fatto è talvolta
indicato come ‘unione etichettata’ ["tagged union"], dato che i dati
comprendono un'’etichetta’ ["tag"], che indica qual'è il tipo. Il compilatore
usa questa informazione per garantire che si acceda in modo sicuro ai dati
dell'enumerazione. Per esempio, non si può semplicemente provare
a destrutturare un valore come se fosse una delle possibile varianti:

```rust,ignore
fn elabora_cambio_colore(mess: Messaggio) {
    let Messaggio::CambiaColore(r, g, b) = mess; // errore di compilazione
}
```

Non supportare queste operazioni può sembrare piuttosto limitativo, ma è
una limitazione che possiamo superare. Ci sono due modi: o implementando
l'uguaglianza noi stessi, o eseguendo un pattern matching delle varianti
con espressioni [`match`][match], come vedremo nella prossima sezione.
Non abbiamo ancora visto abbastanza Rust per implementare l'uguaglianza, ma
lo vedremo nella sezione sui [`tratti`][tratti].

[match]: match.html
[traits]: traits.html

# Costruttori come funzioni

Un costruttore di un `enum` può anche essere usato come una funzione.
Per esempio:

```rust
# enum Messaggio {
# Scrivi(String),
# }
let m = Messaggio::Scrivi("Ciao, mondo".to_string());
```

è lo stesso che

```rust
# enum Messaggio {
# Scrivi(String),
# }
fn foo(x: String) -> Messaggio {
    Messaggio::Scrivi(x)
}

let x = foo("Ciao, mondo".to_string());
```

Questo non ci è immediatamente utile, ma quando arriveremo alle [`chiusure`]
[chiusure], parleremo di come passare le funzioni come argomenti ad altre
funzioni. Per esempio, con gli [`iteratori`][iteratori], possiamo fare questo
per convertire un vettore di `String` in un vettore di `Messaggio::Scrivi`:

```rust
# enum Messaggio {
# Scrivi(String),
# }

let v = vec!["Ciao".to_string(), "Mondo".to_string()];

let v1: Vec<Messaggio> = v.into_iter().map(Messaggio::Scrivi).collect();
```

[chiusure]: closures.html
[iteratori]: iterators.html
