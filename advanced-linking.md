% Link avanzato

I casi comuni di link con Rust sono stati trattati precedentemente,
ma supportare la gamma di possibilità di linking rese disponibili da altri
linguaggi è importante per Rust per raggiungere un interazione fluida
con le librerie native.

# Argomenti di link

C'è un altro modo di dire a `rustc` come personalizzare il link, e cioè tramite
l'attributo `link_args`. Questo attributo viene applicato ai blocchi `extern`
e specifica dei flags grezzi che devono essere passati al linker
quando si produce un artefatto. Ecco un esempio di utilizzo:

```rust,no_run
#![feature(link_args)]

#[link_args = "-foo -bar -baz"]
extern {}
# fn main() {}
```

Si noti che questa caratteristica attualmente è nascosta dietro il gate
`feature(link_args)` perché questo non è un modo ufficialmente accettato
di eseguire il link. Attualmente `rustc` lancia il linker di sistema
(`gcc` per la maggior parte delle configurazioni, `link.exe` per MSVC),
perciò ha senso fornire ulteriore argomenti di riga di comando,
ma potrebbe non essere così per sempre. In futuro `rustc` potrebbe usare
direttamente LLVM per linkare librerie native, nel qual caso `link_args`
non avrà significato. Si può ottenere lo stesso effetto dell'attributo
`link_args` con l'argomento `-C link-args` di `rustc`.

Si consiglia vivamente di *non* usare questo attributo, e invece usare
il più formale attributo `#[link(...)]` su blocchi `extern`.

# Link statico

Per "link statico" si intende il procedimento di creazione di un output
che contiene tutte le librerie necessarie, e quindi non ha bisogno che
delle librerie siano installate su ogni sistema in cui si vuole usare
il proprio progetto compilato. Le dipendenze in puro Rust sono linkate
staticamente di default, e quindi si possono usare i programmi e le librerie
creati senza dover installare Rust ovunque. Al contrario, le librerie native
(per es. `libc` e `libm`) vengono solitamente linkate dinamicamente,
ma è possibile fare diversamente, e linkare staticamente anche loro.

Linkare è un argomento molto dipendente dalla piattaforma, e il link statico
potrebbe persino non essere disponibile su qualche piattaforma! Questa sezione
assume una familiarità di base con il link nella propria piattaforma.

## Linux

Di default, tutti i programmi in Rust su Linux linkeranno con la libreria
`libc` di sistema, insieme a varie altre librerie. Guardiamo un esempio
su una macchina Linux a 64 bit avente GCC e `glibc` (nettamente la `libc`
più diffusa su Linux):

```text
$ cat example.rs
fn main() {}
$ rustc example.rs
$ ldd example
        linux-vdso.so.1 =>  (0x00007ffd565fd000)
        libdl.so.2 => /lib/x86_64-linux-gnu/libdl.so.2 (0x00007fa81889c000)
        libpthread.so.0 => /lib/x86_64-linux-gnu/libpthread.so.0 (0x00007fa81867e000)
        librt.so.1 => /lib/x86_64-linux-gnu/librt.so.1 (0x00007fa818475000)
        libgcc_s.so.1 => /lib/x86_64-linux-gnu/libgcc_s.so.1 (0x00007fa81825f000)
        libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007fa817e9a000)
        /lib64/ld-linux-x86-64.so.2 (0x00007fa818cf9000)
        libm.so.6 => /lib/x86_64-linux-gnu/libm.so.6 (0x00007fa817b93000)
```

Il link dinamico su Linux può essere indesiderabile se si vogliono usare
delle nuove caratteristiche di libreria su vecchi sistemi, o avere
come target dei sistemi che non hanno le dipendenze necessarie per eseguire
il proprio programma.

Il link statico è supportato tramite un `libc` alternativo,
[`musl`](http://www.musl-libc.org). Si può compilare la propria versione
di Rust con `musl` abilitato e installarlo in una directory personalizzata
seguendo queste istruzioni:

```text
$ mkdir musldist
$ PREFIX=$(pwd)/musldist
$
$ # Build musl
$ curl -O http://www.musl-libc.org/releases/musl-1.1.10.tar.gz
$ tar xf musl-1.1.10.tar.gz
$ cd musl-1.1.10/
musl-1.1.10 $ ./configure --disable-shared --prefix=$PREFIX
musl-1.1.10 $ make
musl-1.1.10 $ make install
musl-1.1.10 $ cd ..
$ du -h musldist/lib/libc.a
2.2M    musldist/lib/libc.a
$
$ # Build libunwind.a
$ curl -O http://llvm.org/releases/3.7.0/llvm-3.7.0.src.tar.xz
$ tar xf llvm-3.7.0.src.tar.xz
$ cd llvm-3.7.0.src/projects/
llvm-3.7.0.src/projects $ curl http://llvm.org/releases/3.7.0/libunwind-3.7.0.src.tar.xz | tar xJf -
llvm-3.7.0.src/projects $ mv libunwind-3.7.0.src libunwind
llvm-3.7.0.src/projects $ mkdir libunwind/build
llvm-3.7.0.src/projects $ cd libunwind/build
llvm-3.7.0.src/projects/libunwind/build $ cmake -DLLVM_PATH=../../.. -DLIBUNWIND_ENABLE_SHARED=0 ..
llvm-3.7.0.src/projects/libunwind/build $ make
llvm-3.7.0.src/projects/libunwind/build $ cp lib/libunwind.a $PREFIX/lib/
llvm-3.7.0.src/projects/libunwind/build $ cd ../../../../
$ du -h musldist/lib/libunwind.a
164K    musldist/lib/libunwind.a
$
$ # Build musl-enabled rust
$ git clone https://github.com/rust-lang/rust.git muslrust
$ cd muslrust
muslrust $ ./configure --target=x86_64-unknown-linux-musl --musl-root=$PREFIX --prefix=$PREFIX
muslrust $ make
muslrust $ make install
muslrust $ cd ..
$ du -h musldist/bin/rustc
12K     musldist/bin/rustc
```

Adesso abbiamo una build di un Rust abilitato a `musl`! Siccome l'abbiamo
installato a un prefisso personalizzato, dobbiamo assicurarci che il nostro
sistema  possa trovare i programmi e le appropriate librerie quando proviamo
a eseguirlo:

```text
$ export PATH=$PREFIX/bin:$PATH
$ export LD_LIBRARY_PATH=$PREFIX/lib:$LD_LIBRARY_PATH
```

Proviamolo!

```text
$ echo 'fn main() { println!("hi!"); panic!("failed"); }' > example.rs
$ rustc --target=x86_64-unknown-linux-musl example.rs
$ ldd example
        not a dynamic executable
$ ./example
hi!
thread 'main' panicked at 'failed', example.rs:1
```

Successo! Questo programma può essere copiato in quasi ogni macchina Linux
che abbia la stessa architettura di CPU ed essere eseguito senza problemi.

Anche `cargo build` permette l'opzione `--target`, e quindi si dovrebbe
poter costruire i propri crate normalmente. Però, potrebbe essere necessario
recompilare le librerie native con `musl` prima di poterle usare nel link.
