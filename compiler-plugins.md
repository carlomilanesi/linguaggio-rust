% Estensioni del compilatore

# Introduzione

`rustc` può caricare estensioni del compilatore, che sono librerie
fornite da utenti che estendono il comportamento del compilatore con nuove
estensioni sintattiche, verifiche di correttezza, ecc.

Un'estensione ("plugin") è un crate di libreria dinamica con una funzione
designata come *archivista* che registra le estensioni associate a `rustc`.
Altri crate possono caricare queste estensioni usando l'attributo di crate
`#![plugin(...)]`. Si veda la documentazione di `rustc_plugin` per saperne
di più sulla meccanica della definizione e del caricamento delle estensioni.

Se presenti, gli argomenti passati come `#![plugin(foo(... args ...))]`
non sono interpretati da rustc stesso. Sono forniti all'estensione tramite
il metodo `args` del `Registry`.

Nella gran maggioranza dei casi, un'estensione dovrebbe essere usata
*solamente* tramite `#![plugin]` e non tramite un elemento `extern crate`.
Eseguire il link con un'estensione tirerebbe dentro libsyntax e librustc
come dipendenze del proprio crate. In gerale questo non è desiderabile
a meno che si stia costruendo un'altra estensione. L'opzione di correttezza
`plugin_as_library` verifica queste linne guida.

La pratica più usata è mettere le estensioni del compilatore nel loro crate,
separate da ogni macro `macro_rules!` e dal codice Rust ordinario rivolto
ai consumatori di una libreria.

# Estensioni di sintassi

Le estensioni possono estendere la sintassi di Rust in vari modi. Un genere
di estensione di sintassi è la macro procedurale. Queste si invocano
nel medesimo modo delle [macro ordinarie](macros.html), ma l'espansione
viene eseguita da codice Rust arbitrario che manipola l'albero sintattico
in fase di compilazione.

Scriviamo un'estensione [`roman_numerals.rs`]
(https://github.com/rust-lang/rust/blob/master/src/test/run-pass-fulldeps/auxiliary/roman_numerals.rs)
che implementa come costanti intere i numeri romani.

```rust,ignore
#![crate_type="dylib"]
#![feature(plugin_registrar, rustc_private)]

extern crate syntax;
extern crate rustc;
extern crate rustc_plugin;

use syntax::parse::token;
use syntax::ast::TokenTree;
use syntax::ext::base::{ExtCtxt, MacResult, DummyResult, MacEager};
use syntax::ext::build::AstBuilder;  // tratto per expr_usize
use syntax_pos::Span;
use rustc_plugin::Registry;

fn espandi_rn(cx: &mut ExtCtxt, sp: Span, args: &[TokenTree])
        -> Box<MacResult + 'static> {

    static NUMERALS: &'static [(&'static str, usize)] = &[
        ("M", 1000), ("CM", 900), ("D", 500), ("CD", 400),
        ("C",  100), ("XC",  90), ("L",  50), ("XL",  40),
        ("X",   10), ("IX",   9), ("V",   5), ("IV",   4),
        ("I",    1)];

    if args.len() != 1 {
        cx.span_err(
            sp,
            &format!("argument should be a single identifier, but got {} arguments", args.len()));
        return DummyResult::any(sp);
    }

    let text = match args[0] {
        TokenTree::Token(_, token::Ident(s, _)) => s.to_string(),
        _ => {
            cx.span_err(sp, "argument should be a single identifier");
            return DummyResult::any(sp);
        }
    };

    let mut text = &*text;
    let mut total = 0;
    while !text.is_empty() {
        match NUMERALS.iter().find(|&&(rn, _)| text.starts_with(rn)) {
            Some(&(rn, val)) => {
                total += val;
                text = &text[rn.len()..];
            }
            None => {
                cx.span_err(sp, "invalid Roman numeral");
                return DummyResult::any(sp);
            }
        }
    }

    MacEager::expr(cx.expr_usize(sp, total))
}

#[plugin_registrar]
pub fn plugin_registrar(reg: &mut Registry) {
    reg.register_macro("rn", expand_rn);
}
```

Then we can use `rn!()` like any other macro:

```rust,ignore
#![feature(plugin)]
#![plugin(roman_numerals)]

fn main() {
    assert_eq!(rn!(MMXV), 2015);
}
```

I vantaggi rispetto a un semplice `fn(&str) -> u32` sono:

* La conversione (arbitrariamente complessa) è fatta in fase di compilazione.
* Anche la convalida dell'input è eseguita in fase di compilazione.
* Si può estendere per consentirne l'uso nei pattern, che effettivamente
  dà un modo di definire una nuova sintassi delle costanti per qualunque
  tipo di dati.

Oltre alle macro procedurali, si possono definire nuovi attributi
tipo-[`derive`](../reference.html#derive) e altri generi di estensioni.
Si veda `Registry::register_syntax_extension` e l'enum `SyntaxExtension`.
Per un esempio di macro più complicato, si veda [`regex_macros`]
(https://github.com/rust-lang/regex/blob/master/regex_macros/src/lib.rs).

## Suggerimenti e trucchi

Alcuni dei [suggerimenti sul debug delle macro]
(macros.html#debugging-macro-code) sono ancora validi.

Si può usare `syntax::parse` per trasformare gli alberi di token in
elementi sintattici di livello più alto come espressioni:

```rust,ignore
fn expand_foo(cx: &mut ExtCtxt, sp: Span, args: &[TokenTree])
        -> Box<MacResult+'static> {

    let mut parser = cx.new_parser_from_tts(args);

    let expr: P<Expr> = parser.parse_expr();
```

Guardare nel [codice del parser `libsyntax`]
(https://github.com/rust-lang/rust/blob/master/src/libsyntax/parse/parser.rs)
darà una sensazione di come funziona l'infrastruttura di parsing.

Si mantengano gli `Span` di tutto ciò che viene analizzato, per produrre
migliori messaggi d'errore. Si può avvolgere `Spanned` intorno alle proprie
strutture dati personalizzate.

Chiamare `ExtCtxt::span_fatal` abortirà immediatamente la compilazione.
È meglio invece chiamare `ExtCtxt::span_err` e restituire `DummyResult`,
così che il compilatore possa proseguire e trovare altri errori.

Per stampare frammenti di sintassi per debug, si può usare `span_note`
insieme a `syntax::print::pprust::*_to_string`.

L'esempio sopra produceva un letterale intero usando `AstBuilder::expr_usize`.
Come alternativa al tratto `AstBuilder`, `libsyntax` fornisce un insieme
di macro quasiquote. Non sono documentate e sono molto spigolose.
Però, la loro implementazione può essere un buon punto di partenza
per un quasiquote migliorato come un'ordinaria estensione di libreria.


# Le estensioni di correttezza

Le estensioni possono estendere [l'infrastruttura di correttezza di Rust]
(../reference.html#lint-check-attributes) con verifiche aggiuntive relative
allo stile del codice, alla sicurezza, ecc. Adesso scriviamo un'estensione
[`lint_plugin_test.rs`]
(https://github.com/rust-lang/rust/blob/master/src/test/run-pass-fulldeps/auxiliary/lint_plugin_test.rs)
che avverte riguardo della presenza di elementi chiamati `lintme`.

```rust,ignore
#![feature(plugin_registrar)]
#![feature(box_syntax, rustc_private)]

extern crate syntax;

// Carica rustc come un'estensione per ottenere le macro
#[macro_use]
extern crate rustc;
extern crate rustc_plugin;

use rustc::lint::{EarlyContext, LintContext, LintPass, EarlyLintPass,
	EarlyLintPassObject, LintArray};
use rustc_plugin::Registry;
use syntax::ast;

declare_lint!(TEST_LINT, Warn,
    "Avverti la presenza di elementi di nome 'lintme'");

struct Pass;

impl LintPass for Pass {
    fn get_lints(&self) -> LintArray {
        lint_array!(TEST_LINT)
    }
}

impl EarlyLintPass for Pass {
    fn check_item(&mut self, cx: &EarlyContext, it: &ast::Item) {
        if it.ident.name.as_str() == "lintme" {
            cx.span_lint(TEST_LINT, it.span, "elemento si chiama 'lintme'");
        }
    }
}

#[plugin_registrar]
pub fn plugin_registrar(reg: &mut Registry) {
    reg.register_early_lint_pass(box Pass as EarlyLintPassObject);
}
```

Poi il codice come

```rust,ignore
#![plugin(lint_plugin_test)]

fn lintme() { }
```

produrrà un avvertimento del compilatore:

```txt
foo.rs:4:1: 4:16 warning: elemento si chiama 'lintme', #[warn(test_lint)] on by default
foo.rs:4 fn lintme() { }
         ^~~~~~~~~~~~~~~
```

I componenti di un'estensione di correttezza sono:

* uno o più invocazioni di `declare_lint!`, che definiscono
  le struct statiche `Lint`;

* una struct che tiene ogni stato che serve alla passata di analisi
  di correttezza (nel nostro caso, nessuno);

* una implementazione di `LintPass` che definisce come verificare
  ogni elemento sintattico. Un unico `LintPass` può chiamare `span_lint`
  per più diversi `Lint`, ma dovrebbe registrarli tutti tramite il metodo
  `get_lints`.

Le passate di Lint sono attraversamenti sintattici, ma vengono eseguite
in una fase tardiva della compilazione, dove sono disponibili le informazioni
sui tipi. [I lint incorporati in `rustc`]
(https://github.com/rust-lang/rust/blob/master/src/librustc/lint/builtin.rs)
per lo più usano la medesima infrastruttura delle estensioni di correttezza,
e forniscono esempi di come accedere alle informazioni sui tipi.

I Lint definiti dalle estensioni sono controlleti dai soliti [attributi
e opzioni del compilatore](../reference.html#lint-check-attributes),
per es. `#[allow(test_lint)]` o `-A test-lint`. Questi identificatori
sono derivati dal primo argomento a `declare_lint!`, con le appropriate
conversioni di maiuscolizzazione e di punteggiatura.

Si può eseguire `rustc -W help foo.rs` per vedere un elenco dei lint noti
a `rustc`, compresi quelli forniti dalle estensioni caricate da `foo.rs`.
