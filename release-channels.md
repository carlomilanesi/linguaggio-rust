% Canali di rilascio

Il progetto Rust usa un concetto chiamato ‘canali di rilascio’ per gestire
i rilasci. È importante capire questo procedimento per scegliere
quale versione di Rust è meglio usare.

# Panoramica

Ci sono tre canali per i rilasci di Rust:

* Notturno ["Nightly"]
* Beta
* Stabile ["Stable"]

I nuovi rilasci notturni vengono creati una volta al giorno. Ogni sei
settimane, l'ultimo rilascio notturno viene promosso a ‘Beta’. Da questo
momento, riceverà solamente pezze per correggere gravi errori. Sei settimane
dopo, la beta viene promossa a ‘Stabile’, e diventa la versione `1.x`.

Questo procedimento avviene in parallelo. Quindi ogni sei settimane,
nel medesimo giorno, la beta diventa la nuova stable, e la notturna diventa
la nuova beta. Quando la versione `1.x` viene rilasciata, nello stesso tempo,
anche la versione `1.(x + 1)-beta` viene rilasciata, e le ultime aggiunte al
sistema entrano a far parte della prima versione di `1.(x + 2)-nightly`.

# Scegliere una versione

In generale, a meno di avere una ragione specifica, si dovrebbe usare il canale
del rilascio stabile. Questi rilasci sono rivolti al pubblico generale.

Però, a seconda dell'interesse che si ha verso Rust, si può scegliere invece
di usare la versione notturna. I pro e contro di base sono questi: usando
il canale notturno, si possono usare caratteristiche nuove di Rust,
ma instabili. Queste caratteristiche instabili sono soggette a modifiche, e
così ogni nuovo rilascio notturno potrebbe essere incompatibile con il proprio
codice. Se si usa il rilascio stabile, non si possono usare le caratteristiche
sperimentali, ma il nuovo rilascio di Rust non provocherà problemi
significativi a causa di modifiche incompatibili.

# Aiutare l'ecosistema tramite CI

E che dire della beta? Incoraggiamo tutti gli utenti di Rust che usano
il canale di rilascio stabile a collaudare anche rispetto al canale beta
nei loro sistemi di integrazione continua.
Questo aiuterà ad allertare la squadra nel caso ci sia una regressione
accidentale.

In aggiunta, collaudare rispetto alla versione beta può rilevare
delle regressioni ancora prima, e quindi un terzo build non crea troppo
fastidio, apprezziamo il collaudo rispetto tutti i canali.

Come esempio, molti programmatori Rust, per collaudare i loro crate, usano
[Travis](https://travis-ci.org/) che è gratuito per progetti open source.
Travis [supporta Rust direttamente][travis], e per collaudare su tutti
i canali, si può usare un file `.travis.yml` come questo:

```yaml
language: rust
rust:
  - nightly
  - beta
  - stable

matrix:
  allow_failures:
    - rust: nightly
```

[travis]: http://docs.travis-ci.com/user/languages/rust/

Con questa configurazione, Travis collauderà tutte e tre i canali, ma se
qualcosa va storto sul canale notturno, non fallirà la costruzione.
Una configurazione simile è consigliata per ogni sistema CI. Si cosnulti
la documentazione di quella che si sta usando per avere più dettagli.
