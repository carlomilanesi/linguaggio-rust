% Documentazione

La documentazione è una parte importante di qualunque progetto software, e
in Rust è di prima classe. Parliamo della strumentazione fornita da Rust
per documentare i propri progetti.

## A proposito di `rustdoc`

La distribuzione di Rust include uno strumento, `rustdoc`, che genera
documentazione. `rustdoc` viene usato anche da Cargo con il comando
`cargo doc`.

La documentazione può essere generata in due modi: dal codice sorgente,
e da file Markdown autonomi.

## Documentare il codice sorgente

Il modo primario di documentare un progetto Rust è annotando il codice
sorgente. A questo scopo, si possono usare i commenti di documentazione:

```rust,ignore
/// Costruisce un nuovo `Rc<T>`.
///
/// # Esempi
///
/// ```
/// use std::rc::Rc;
///
/// let cinque = Rc::new(5);
/// ```
pub fn new(value: T) -> Rc<T> {
    // l'implementazione va qui
}
```

Questo codice genera della documentazione che si presenta [così][rc-new].
Ho escluso l'implementazione, mettendo al suo posto un normale commento.

La prima cosa da notare riguardo a questa annotazione è che usa
`///` invece di `//`. La tripla barra indica un commento di documentazione.

I commenti di documentazione sono scritti in Markdown.

Rust tiene traccia di questi commenti, e li usa quando genera
la documentazione. Questo è importante quando si documentano cose come
gli enums:

```rust
/// Il tipo `Option`. Si veda
// [la documentazione a livello di modulo](index.html) per saperne di più.
enum Option<T> {
    /// Nessun valore
    None,
    /// Qualche valore `T`
    Some(T),
}
```

Il codice sopra funziona, ma questo sotto no:

```rust,ignore
/// Il tipo `Option`. Si veda
// [la documentazione a livello di modulo](index.html) per saperne di più.
enum Option<T> {
    None, /// Nessun valore
    Some(T), /// Qualche valore `T`
}
```

Infatti si avrà l'errore:

```text
hello.rs:4:1: 4:2 error: expected ident, found `}`
hello.rs:4 }
           ^
```

Questo [errore sfortunato](https://github.com/rust-lang/rust/issues/22547) è
giusto; i commenti di documentazione si applicano a quello che li segue,
e non c'è niente dopo l'ultimo commento.

[rc-new]: ../std/rc/struct.Rc.html#method.new

### Scrivere i commenti di documentazione

Comunque, vediamo in dettaglio ogni parte di questo commento:

```rust
/// Costruisce un nuovo `Rc<T>`.
# fn foo() {}
```

La prima riga di un commento di documentazione dovrebbe essere un breve
riassunto della sua funzionalità che sta descrivendo. Una sola frase.
Solo le basi. Ad alto livello.

```rust
///
/// Altri dettagli sulla costruzione degli `Rc<T>`, eventualmente descrivendo
/// una semantica complicata, forse anche delle opzioni aggiuntive,
/// tutti gli aspetti
///
# fn foo() {}
```

Il nostro esempio originale aveva solo una riga riassuntiva, ma se avessimo
avuto più cose da dire, avremmo potuto aggiungere altre spiegazioni
in un altro paragrafo.

#### Sezioni speciali

Poi, ci sono le sezioni speciali. Queste sono indicate con un'intestazione,
`#`. Ci sono quattro tipi di intestazioni che vengono comunemente usate.
Per adesso, non hanno una sintassi speciale, ma solo convenzioni.

```rust
/// # Panico
# fn foo() {}
```

L'abuso irrecuperabile di una funzione (cioè un errore di programmazione)
in Rust è solitamente chiamato panico, che come minimo uccide l'intero thread
corrente. Se la propria funzione ha un contratto non banale, che se
rilevato/forzato produce un panico, è molto importante documentarlo.

```rust
/// # Errori
# fn foo() {}
```

Se la propria funzion o il proprio metodo rene un `Result<T, E>`, allora
descrivere le condizioni sotto le quali rende `Err(E)` è una cosa carina
da fare. Questo è leggermento meno importante del `Panics`,
perché tale fallimento è codificato nel sistema dei tipi,
ma è sempre una buona cosa da fare.

```rust
/// # Sicurezza
# fn foo() {}
```

Se la propria funzione è `unsafe`, si dovrebbe spiegare quali invarianti
il chiamante è tenuto a rispettare.

```rust
/// # Esempi
///
/// ```
/// use std::rc::Rc;
///
/// let cinque = Rc::new(5);
/// ```
# fn foo() {}
```

Quarto, `Esempi`. Aggiungere uno o più esempi di come usare la propria
funzione, sarà molto apprezzato dagli utenti di tale funzione. Questi esempi
vanno dentro annotazioni di blocchi di codice, di cui parleremo fra un momento,
e possono avere più di una sezione:

```rust
/// # Esempi
///
/// Semplici pattern di `&str`:
///
/// ```
/// let v: Vec<&str> = "Mary had a little lamb".split(' ').collect();
/// assert_eq!(v, vec!["Mary", "had", "a", "little", "lamb"]);
/// ```
///
/// Pattern più complessi, con una lambda:
///
/// ```
/// let v: Vec<&str> = "abc1def2ghi".split(|c: char| c.is_numeric()).collect();
/// assert_eq!(v, vec!["abc", "def", "ghi"]);
/// ```
# fn foo() {}
```

Discutiamo i dettagli di questi blocchi di codice.

#### Annotazioni dei blocchi di codice

Per scrivere del codice Rust dentro un commento, si usa
il triplo accento grave:

```rust
/// ```
/// println!("Ciao, mondo");
/// ```
# fn foo() {}
```

Se si vuol scrivere qualcosa che non è codice Rust, si può aggiungere
un'annotazione:

```rust
/// ```c
/// printf("Ciao, mondo\n");
/// ```
# fn foo() {}
```

Questo evidenzierà la sintassi del codice in base al linguaggio indicato.
Se si sta mostrando del semplice testo, si scelga `text`.

Qui è importante scegliere l'annotazione corretta, perché `rustdoc` la usa
in un modo interessante: può venire usata per collaudare effettivamente
gli esempi in un crate di libreria, così da assicurasi che rimangano
aggiornati. Se si ha del codice C ma `rustdoc` pensa che sia Rust perché non
si è aggiunta l'annotazione, `rustdoc` si lamenterà quando proverà a generare
la documentazione.

## Documentazione come test

Discutiamo il nostro esempio di documentazione:

```rust
/// ```
/// println!("Ciao, mondo");
/// ```
# fn foo() {}
```

Si noterà che qui non c'è bisogno di un `fn main()` né di altro. `rustdoc`
aggiungerà automaticamente una funzione `main()` intorno al nostro codice,
usando dell'euristica per tentare di metterlo al posto giusto. Per esempio:

```rust
/// ```
/// use std::rc::Rc;
///
/// let cinque = Rc::new(5);
/// ```
# fn foo() {}
```

Questo finirà per collaudare:

```rust
fn main() {
    use std::rc::Rc;
    let cinque = Rc::new(5);
}
```

Ecco l'algoritmo completo che rustdoc usa per preprocessare gli esempi:

1. Tutti gli attributi iniziali `#![foo]` sono lasciati intatt come
   attributi del crate.
2. Alcuni attributi `allow` tipici vengono inseriti, tra cui
   `unused_variables`, `unused_assignments`, `unused_mut`, `unused_attributes`,
   e `dead_code`. Dei piccoli esempi spesso fanno scattare questi lint.
3. Se l'esempio non contiene `extern crate`, allora `extern crate
   <mycrate>;` viene inserito (si noti l'assenza di `#[macro_use]`).
4. Infine, se l'esempio non contiene `fn main`, il resto del testo è
   avvolto in `fn main() { il_nostro_codice }`.

Questo `fn main` generato però può creare problemi! Se ci sono istruzioni
`extern crate` o `mod` nel codice d'esempio che sono riferite da istruzioni
`use`, non riusciranno a risolvere a meno che si includa almeno `fn main() {}`
per inibire il passo 4.
`#[macro_use] extern crate` also does not work except at the crate root, so when
testing macros an explicit `main` is always required. It doesn't have to clutter
up your docs, though -- keep reading!

Sometimes this algorithm isn't enough, though. For example, all of these code samples
with `///` we've been talking about? The raw text:

```text
/// Some documentation.
# fn foo() {}
```

looks different than the output:

```rust
/// Some documentation.
# fn foo() {}
```

Yes, that's right: you can add lines that start with `# `, and they will
be hidden from the output, but will be used when compiling your code. You
can use this to your advantage. In this case, documentation comments need
to apply to some kind of function, so if I want to show you just a
documentation comment, I need to add a little function definition below
it. At the same time, it's only there to satisfy the compiler, so hiding
it makes the example more clear. You can use this technique to explain
longer examples in detail, while still preserving the testability of your
documentation.

For example, imagine that we wanted to document this code:

```rust
let x = 5;
let y = 6;
println!("{}", x + y);
```

We might want the documentation to end up looking like this:

> First, we set `x` to five:
>
> ```rust
> let x = 5;
> # let y = 6;
> # println!("{}", x + y);
> ```
>
> Next, we set `y` to six:
>
> ```rust
> # let x = 5;
> let y = 6;
> # println!("{}", x + y);
> ```
>
> Finally, we print the sum of `x` and `y`:
>
> ```rust
> # let x = 5;
> # let y = 6;
> println!("{}", x + y);
> ```

To keep each code block testable, we want the whole program in each block, but
we don't want the reader to see every line every time.  Here's what we put in
our source code:

```text
    First, we set `x` to five:

    ```rust
    let x = 5;
    # let y = 6;
    # println!("{}", x + y);
    ```

    Next, we set `y` to six:

    ```rust
    # let x = 5;
    let y = 6;
    # println!("{}", x + y);
    ```

    Finally, we print the sum of `x` and `y`:

    ```rust
    # let x = 5;
    # let y = 6;
    println!("{}", x + y);
    ```
```

By repeating all parts of the example, you can ensure that your example still
compiles, while only showing the parts that are relevant to that part of your
explanation.

### Documenting macros

Here’s an example of documenting a macro:

```rust
/// Panic with a given message unless an expression evaluates to true.
///
/// # Examples
///
/// ```
/// # #[macro_use] extern crate foo;
/// # fn main() {
/// panic_unless!(1 + 1 == 2, “Math is broken.”);
/// # }
/// ```
///
/// ```rust,should_panic
/// # #[macro_use] extern crate foo;
/// # fn main() {
/// panic_unless!(true == false, “I’m broken.”);
/// # }
/// ```
#[macro_export]
macro_rules! panic_unless {
    ($condition:expr, $($rest:expr),+) => ({ if ! $condition { panic!($($rest),+); } });
}
# fn main() {}
```

You’ll note three things: we need to add our own `extern crate` line, so that
we can add the `#[macro_use]` attribute. Second, we’ll need to add our own
`main()` as well (for reasons discussed above). Finally, a judicious use of
`#` to comment out those two things, so they don’t show up in the output.

Another case where the use of `#` is handy is when you want to ignore
error handling. Lets say you want the following,

```rust,ignore
/// use std::io;
/// let mut input = String::new();
/// try!(io::stdin().read_line(&mut input));
```

The problem is that `try!` returns a `Result<T, E>` and test functions
don't return anything so this will give a mismatched types error.

```rust,ignore
/// A doc test using try!
///
/// ```
/// use std::io;
/// # fn foo() -> io::Result<()> {
/// let mut input = String::new();
/// try!(io::stdin().read_line(&mut input));
/// # Ok(())
/// # }
/// ```
# fn foo() {}
```

You can get around this by wrapping the code in a function. This catches
and swallows the `Result<T, E>` when running tests on the docs. This
pattern appears regularly in the standard library.

### Running documentation tests

To run the tests, either:

```bash
$ rustdoc --test path/to/my/crate/root.rs
# or
$ cargo test
```

That's right, `cargo test` tests embedded documentation too. **However,
`cargo test` will not test binary crates, only library ones.** This is
due to the way `rustdoc` works: it links against the library to be tested,
but with a binary, there’s nothing to link to.

There are a few more annotations that are useful to help `rustdoc` do the right
thing when testing your code:

```rust
/// ```rust,ignore
/// fn foo() {
/// ```
# fn foo() {}
```

The `ignore` directive tells Rust to ignore your code. This is almost never
what you want, as it's the most generic. Instead, consider annotating it
with `text` if it's not code, or using `#`s to get a working example that
only shows the part you care about.

```rust
/// ```rust,should_panic
/// assert!(false);
/// ```
# fn foo() {}
```

`should_panic` tells `rustdoc` that the code should compile correctly, but
not actually pass as a test.

```rust
/// ```rust,no_run
/// loop {
///     println!("Hello, world");
/// }
/// ```
# fn foo() {}
```

The `no_run` attribute will compile your code, but not run it. This is
important for examples such as "Here's how to start up a network service,"
which you would want to make sure compile, but might run in an infinite loop!

### Documenting modules

Rust has another kind of doc comment, `//!`. This comment doesn't document the next item, but the enclosing item. In other words:

```rust
mod foo {
    //! This is documentation for the `foo` module.
    //!
    //! # Examples

    // ...
}
```

This is where you'll see `//!` used most often: for module documentation. If
you have a module in `foo.rs`, you'll often open its code and see this:

```rust
//! A module for using `foo`s.
//!
//! The `foo` module contains a lot of useful functionality blah blah blah
```

### Crate documentation

Crates can be documented by placing an inner doc comment (`//!`) at the
beginning of the crate root, aka `lib.rs`:

```rust
//! This is documentation for the `foo` crate.
//!
//! The foo crate is meant to be used for bar.
```

### Documentation comment style

Check out [RFC 505][rfc505] for full conventions around the style and format of
documentation.

[rfc505]: https://github.com/rust-lang/rfcs/blob/master/text/0505-api-comment-conventions.md

## Other documentation

All of this behavior works in non-Rust source files too. Because comments
are written in Markdown, they're often `.md` files.

When you write documentation in Markdown files, you don't need to prefix
the documentation with comments. For example:

```rust
/// # Examples
///
/// ```
/// use std::rc::Rc;
///
/// let five = Rc::new(5);
/// ```
# fn foo() {}
```

is:

~~~markdown
# Examples

```
use std::rc::Rc;

let five = Rc::new(5);
```
~~~

when it's in a Markdown file. There is one wrinkle though: Markdown files need
to have a title like this:

```markdown
% The title

This is the example documentation.
```

This `%` line needs to be the very first line of the file.

## `doc` attributes

At a deeper level, documentation comments are syntactic sugar for documentation
attributes:

```rust
/// this
# fn foo() {}

#[doc="this"]
# fn bar() {}
```

are the same, as are these:

```rust
//! this

#![doc="this"]
```

You won't often see this attribute used for writing documentation, but it
can be useful when changing some options, or when writing a macro.

### Re-exports

`rustdoc` will show the documentation for a public re-export in both places:

```rust,ignore
extern crate foo;

pub use foo::bar;
```

This will create documentation for `bar` both inside the documentation for the
crate `foo`, as well as the documentation for your crate. It will use the same
documentation in both places.

This behavior can be suppressed with `no_inline`:

```rust,ignore
extern crate foo;

#[doc(no_inline)]
pub use foo::bar;
```

## Missing documentation

Sometimes you want to make sure that every single public thing in your project
is documented, especially when you are working on a library. Rust allows you to
to generate warnings or errors, when an item is missing documentation.
To generate warnings you use `warn`:

```rust
#![warn(missing_docs)]
```

And to generate errors you use `deny`:

```rust,ignore
#![deny(missing_docs)]
```

There are cases where you want to disable these warnings/errors to explicitly
leave something undocumented. This is done by using `allow`:

```rust
#[allow(missing_docs)]
struct Undocumented;
```

You might even want to hide items from the documentation completely:

```rust
#[doc(hidden)]
struct Hidden;
```

### Controlling HTML

You can control a few aspects of the HTML that `rustdoc` generates through the
`#![doc]` version of the attribute:

```rust
#![doc(html_logo_url = "https://www.rust-lang.org/logos/rust-logo-128x128-blk-v2.png",
       html_favicon_url = "https://www.rust-lang.org/favicon.ico",
       html_root_url = "https://doc.rust-lang.org/")]
```

This sets a few different options, with a logo, favicon, and a root URL.

### Configuring documentation tests

You can also configure the way that `rustdoc` tests your documentation examples
through the `#![doc(test(..))]` attribute.

```rust
#![doc(test(attr(allow(unused_variables), deny(warnings))))]
```

This allows unused variables within the examples, but will fail the test for any
other lint warning thrown.

## Generation options

`rustdoc` also contains a few other options on the command line, for further customization:

- `--html-in-header FILE`: includes the contents of FILE at the end of the
  `<head>...</head>` section.
- `--html-before-content FILE`: includes the contents of FILE directly after
  `<body>`, before the rendered content (including the search bar).
- `--html-after-content FILE`: includes the contents of FILE after all the rendered content.

## Security note

The Markdown in documentation comments is placed without processing into
the final webpage. Be careful with literal HTML:

```rust
/// <script>alert(document.cookie)</script>
# fn foo() {}
```
