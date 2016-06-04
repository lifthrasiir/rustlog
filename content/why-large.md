Title: Why is a Rust executable large?
Date: 2016-06-02 23:58

> Do you want tl;dr? [Go to the end of post.](#takeaway)

Suppose that you are a programmer primarily working with compiled languages. Somehow you’ve got tired of those languages, there may be multiple valid reasons, and heard of a trendy new programming language called [Rust](https://rust-lang.org/). Looking at some webpages and the [official forum](https://user.rust-lang.org/), it looks great and you decide to try it out. It seems that Rust was a bit cumbersome to install in the past, but thanks to [rustup](https://rustup.rs/) the problem seems gone by now. Cargo seems to be great, so you follow the [first sections of the Book](https://doc.rust-lang.org/book/) and put a small greeting to the new language:

```rust
fn main() {
    println!("Hello, world!");
}
```

Amazingly `cargo run` runs without a hassle. It is kind of a miracle as you used to configure the build script, Makefile, projects or whatever before building things. Impressed, you realize that the executable is available in `target/debug/hello`. You instinctively type `ls -al` out (or is it `dir`?) and you cannot believe your eyes:

```console
$ ls -al target/debug/hello
-rwxrwxr-x 1 lifthrasiir 650711 May 31 20:00 target/debug/hello*
```

650 *kilobytes* to print anything?! You remember that Rust is probably a sole language that may possibly displace C++, and C++ is noted of the code bloat; would that mean Rust failed to fix one of C++’s big problems? Out of curiosity, you make the same program in C and compile it. The result is eye-opening:

```console
$ cat hello-c.c
#include <stdio.h>
int main() {
    printf("Hello, World!\n");
}
$ make hello-c
$ ls -al hello-c
-rwxrwxr-x 1 lifthrasiir 8551 May 31 20:03 hello-c*
```

*Maybe C has a benefit of having bare-metal libraries*, you think. This time you try a C++ program using `iostream`, which should be much safer than C’s naive `printf`. But surprisingly it still seems tiny compared to Rust:

```console
$ cat hello-cpp.cpp
#include <iostream>
using namespace std;
int main() {
    cout << "Hello, World!" << endl;
}
$ make hello-cpp
$ ls -al hello-cpp
-rwxrwxr-x 1 lifthrasiir 9168 May 31 20:06 hello-cpp*
```

What is wrong with Rust?

----

It seems that the surprisingly large size of Rust binary is a massive concern for many. This question is by no means new; there is a well-known, albeit year-old, [question](https://stackoverflow.com/questions/29008127/why-are-rust-executables-so-huge) on StackOverflow, and searching for [“why is rust binary large”](https://duckduckgo.com/?q=why+is+rust+binary+large) gives several more. Given the frequency of such questions, it is a bit surprising that we don’t yet have a definitive article or page dealing with them. So this is my attempt to provide one.

Just to be cautious: Is it a valid question to ask after all? We have hundreds of gigabytes of storage, if not some terabytes, and people should be using decent ISPs nowadays, so the binary size should not be a concern, right? The answer is that it still may matter (though not as much as before):

* [Akamai State of the Internet](https://www.akamai.com/us/en/our-thinking/state-of-the-internet-report/state-of-the-internet-connectivity-visualization.jsp) shows that, while more than 80% of users enjoy 4Mbps or more in developed countries much less users do in developing countries. The average connection has been improved much (almost every country is now past 1Mbps average), but the entire distribution is still stagnating. I was fortunate that I’m in the country where gigabit ethernet only costs $30/mo (!), but many others may not.

* Ordinary consumers have only shallow understanding of computing, and they likely to relate any problem with anything they know of: one of the common sentiments is that the executable bloat causes slowdown. That’s unfortunate but true, and you would want to avoid that sentiment.

For wondering readers: All examples are tested in Rust 1.9.0 and 1.11.0-nightly (`a967611d8` 2016-05-30). Unless noted, the primary operating system used is Linux 3.13.0 on x86-64. Your mileage may vary.

## Optimization Level

If one were asked about the above, virtually every experienced Rust user would ask you back:

> Have you enabled the release build?

It turns out that Cargo distinguishes the debug build (default) from the release build (`--release`). The [Cargo documentation](http://doc.crates.io/manifest.html#the-profile-sections) explains the exact differences between them, but in general the release build gets rid of development-only routines and data and enables tons of optimization. It is not default because, well, the debug build is more frequently requested than the release build.

Note that Jorge Aparicio has correctly [pointed out](https://www.reddit.com/r/rust/comments/4m7kha/rustlog_why_is_a_rust_executable_large/d3t9yuj) that the release build does not produce the smallest binary possible. That’s because the release build defaults to the optimization level 3 (`-C opt-level=3`), which may sacrifice the size for performance. The size-optimizing level (`-C opt-level=s` or `-C opt-level=z`) has recently [landed](https://github.com/rust-lang/rust/pull/32386), so you may instead use them later. For now, however, we will stick to the default.

Let’s try the release build!

```console
$ cargo build --release
$ ls -al target/release/hello
-rwxrwxr-x 1 lifthrasiir 646467 May 31 20:10 target/release/hello*
```

And that didn’t really make a difference! This is because the optimization is only run over the user code, and we don’t have much user code. Almost all of the binary is from the standard library, and that doesn’t seem to be what we can do anything...

## Link-time Optimization (LTO)

…except that we can. Enter the world of link-time optimization.

So the story is as follows: We can individually optimize each crate, and in fact all standard libraries ship in the optimized form. Once the compiler produces an optimized binary, it gets assembled to a single executable by a program called the “linker”. But we don’t need the entirety of standard library: a simple “Hello, world” program definitely does not need [`std::net`](https://doc.rust-lang.org/std/net/) for example. Yet, the linker is so stupid that it won’t try to remove unused parts of crates; it will just paste them.

There is actually a good reason that the traditional linker behaves so. The linker is commonly used in C and C++ languages among others, and each file is compiled individually. This is a sharp difference from Rust where the entire crate is compiled altogether. Unless required functions are scattered throughout files, the linker can fairly easily get rid of unused files at once. It’s not perfect, but reasonably approximate what we want—removing unused functions. One disadvantage is that the compiler is unable to optimize function calls pointing to other files; it simply lacks a required information.

C and C++ folks had been fine with that approximation for decades, but in the recent decades they had enough and started to provide an option to enable the *link-time optimization* (LTO). In this scheme the compiler produces optimized binaries without looking at others, and the linker actively looks at them and tries to optimize binaries. It is much harder than working with (internally simplified) sources, and it blows the compilation time up, but it is worth trying if the smaller and/or faster executable is needed.

So far we have talked about C and C++, but the LTO is much more beneficial for Rust. `Cargo.toml` has an option to enable LTO:

```toml
[profile.release]
lto = true
```

Did that work? Well, sort of:

```console
$ ls -al target/release/hello
-rwxrwxr-x 1 lifthrasiir 615725 May 31 20:17 target/release/hello*
```

It had a larger effect than the optimization level, but not much. Maybe it is time to look at the executable itself.

## So what’s in my executable?

There are several tools directly working with the executable, but probably the most useful one is [GNU binutils](https://www.gnu.org/software/binutils/). It is available to every Unix-like systems, and also in the Windows ([MinGW](https://sourceforge.net/projects/mingw/files/MinGW/Base/binutils/) has a standalone install for example).

There are many utilities in binutils, but `strings` is probably the simplest. It simply crawls the binary to find a sequence of printable characters terminated by a zero byte, a typical representation of C string. Thus it tries to extract readable strings out of the binary, quite helpful for us. So let’s try that, and prepare for the scroll:

```console
$ strings target/release/hello | head -n 10
/lib64/ld-linux-x86-64.so.2
bki^
 B ,
libpthread.so.0
_ITM_deregisterTMCloneTable
_Jv_RegisterClasses
_ITM_registerTMCloneTable
write
pthread_mutex_destroy
pthread_self
```

And, wow, it already has something we haven’t expect, pthread. (More on that later, though.) There are indeed *tons* of strings in our executable:

```console
$ strings target/release/hello | wc -c
   94339
```

Huh, one sixth of our executable is for strings we don’t really use! At the closer inspection, this observation is not correct as `strings` also give many false positives, but there are some significant strings as well:

* Those starting with `jemalloc_` and `je_`. These are names from [jemalloc](http://www.canonware.com/jemalloc/), a high-performance memory allocator. So that’s what Rust uses for the memory management, in place of classic `malloc`/`free`. It is not a small library however, and we don’t do the dynamic allocation by ourselves anyway.

* Those starting with `backtrace_` and `DW_`. These are yet another names from libbacktrace, a library to produce stack trace. Rust uses it to print a helpful backtrace on panic (available with `RUST_BACKTRACE=1` environment). We don’t panic ourselves however.

* Those starting with `_ZN`. They are “mangled” names from Rust standard libraries.

Why do we have those strings at first place? They are debug symbols, which give an appropriate (possibly human-readable) name for otherwise machine-processed binary. Do you remember libbacktrace above? It has to use those debug symbols to print any useful information. Yet, since we are really making a release build we may choose not to include them. (Rust does not have this option by itself, since it is typically stripped by an external utility called `strip`.) So let’s look at what can be done about them.

## Debug symbols, get off my lawn!

So we have three goals: no jemalloc, no libbacktrace, and no debug symbols. I’ve mentioned that `strip` strips debug symbols, so let’s do that first. Note that `strip` also comes with binutils, so you can just run that.

```console
$ strip target/release/hello
$ target/release/hello
Hello, world!
$ ls -al target/release/hello
-rwxrwxr-x 1 lifthrasiir 347648 May 31 20:23 target/release/hello*
```

Now that IS smaller! About a half of the entire executable was for debugging symbols. Now that, having stripped our symbols, we cannot have a nice backtrace nor panic recovery:

```console
$ sed -i.bak s/println/panic/ src/main.rs
$ cat src/main.rs
fn main() {
    panic!("Hello, world!");
}

$ cargo build --release && strip target/release/hello
$ RUST_BACKTRACE=1 target/release/hello
thread '<main>' panicked at 'Hello, world!', src/main.rs:2
stack backtrace:
   1:     0x7fde451c1e41 - <unknown>
Illegal instruction
$ mv src/main.rs.bak src/main.rs     # tidy it up
```

…and it somehow aborted. Probably a libbacktrace issue, I don’t know, but that doesn’t harm much anyway.

## Knocking jemalloc down

We have knocked debug symbols down, now let’s get rid of remaining libraries. A bad news: From this point you are entering the realm of nightly Rust. The realm is not as scary as you think, as it does not break at your face, but it may break in smaller ways (that’s why we have nightlies!). That’s the major reason that we don’t yet have nightly features in stable, they may change. Fortunately features we are going to use have been quite stable and you can probably follow the remainder of this post with more recent nightlies. But for the posteriority, I will stick to a particular nightly version.

A good news: Installing nightlies (either the latest or any specific) is very simple with rustup.

```console
$ rustup override set nightly-2016-05-31
$ cargo build --release
$ ls -al target/release/hello
-rwxrwxr-x 1 lifthrasiir 620490 May 31 20:35 target/release/hello*
$ strip target/release/hello
$ ls -al target/release/hello
-rwxrwxr-x 1 lifthrasiir 351520 May 31 20:35 target/release/hello*
```

Okay, the size hadn’t changed much. Let’s knock jemalloc down first—it is well documented in [the Book](https://doc.rust-lang.org/book/custom-allocators.html), but the gist is that it just takes two additional lines to change an allocator:

```rust
#![feature(alloc_system)]
extern crate alloc_system;

fn main() {
    println!("Hello, world!");
}
```

And that again does make a difference:

```console
$ cargo build --release
$ ls -al target/release/hello
-rwxrwxr-x 1 lifthrasiir 210364 May 31 20:39 target/release/hello*
$ strip target/release/hello
$ ls -al target/release/hello
-rwxrwxr-x 1 lifthrasiir 121792 May 31 20:39 target/release/hello*
```

Okay! We have down from 600 whooping kilobytes to about 120 KB. Jemalloc indeed is a big library; it really has [tons of configuration](http://www.canonware.com/download/jemalloc/jemalloc-latest/doc/jemalloc.html) so that you can fine-tune its performance, and that has to go somewhere.

## No panic, no gain

We are now left with libbacktrace. We don’t panic ourselves so we don’t need to print a backtrace, right? Well, we have reached a limit: libbacktrace is deeply integrated to the standard library, and the only way to avoid it is not to use libstd. Quite a dead end.

But that is not the end of story. Panicking gives us a backtrace, but also an ability to unwind anything. And unwinding is supported by yet another bit of code called libunwind. It turns out that we *can* get rid of this by disabling unwinding. Put this to Cargo.toml:

```toml
[profile.release]
lto = true
panic = 'abort'
```

And we can see some effect:

```console
$ cargo build --release
$ ls -al target/release/hello
-rwxrwxr-x 1 lifthrasiir 200131 May 31 20:44 target/release/hello*
$ strip target/release/hello
$ ls -al target/release/hello
-rwxrwxr-x 1 lifthrasiir 113472 May 31 20:44 target/release/hello*
```

**And that’s it!** It is about the end of story if you don’t want to change the code.

## Intermission: Linkage

Before looking at more obscure area, it is perfect time to admit that I was cheating with the size of C and C++ binaries. The *fair* (well, fair*er*) comparison would be as follows:

```console
$ touch *.c *.cpp
$ make hello-c CFLAGS='-Os -flto -Wl,--gc-sections -static -s'
cc -Os -flto -Wl,--gc-sections -static    hello-c.c   -o hello-c
$ make hello-cpp CXXFLAGS='-Os -flto -Wl,--gc-sections -static -static-libstdc++ -s'
g++ -Os -flto -Wl,--gc-sections -static -static-libstdc++    hello-cpp.cpp   -o hello-cpp
$ ls -al hello-c hello-cpp
-rwxrwxr-x 1 lifthrasiir  758704 May 31 20:50 hello-c*
-rwxrwxr-x 1 lifthrasiir 1127784 May 31 20:50 hello-cpp*
```

(Note all the options required to be in line with Rust equivalents. `-Wl,--gc-sections` is probably the only option missing; it is a simple-minded cousin of LTO which does not optimize but just remove unused code sections—huge thanks to [Alexis Beingessner and Corey Richardson](https://www.reddit.com/r/rust/comments/4m7kha/rustlog_why_is_a_rust_executable_large/d3tb8v5) for pointing out it was missing. Rust has that implied by default, and it can be applied independently from LTO, so a fair comparison also needs that.)

Also the Rust binary needs to be rebuilt:

```console
$ cargo rustc --release -- -C link-args=-static
$ strip target/release/hello
$ ls -al target/release/hello
-rwxrwxr-x 1 lifthrasiir 105216 May 31 20:51 target/release/hello*
```

Okay, so it seems that Rust was actually far, *far* better than C and C++. But… why is it “fair”? Isn’t an 1 MB executable too much for such a simple program regardless of the language?

A binary executable is not a simple data format. It is normally processed and often altered by an OS routine called a “dynamic linker” (not to be confused an aforementioned “linker”). The use of dynamic linker allows programs to *dynamically* link to other (often common) libraries including the system ones, and until now we was implicitly linking to the C and C++’s standard libraries—glibc and libstdc++ in this case! Rust does not (well, almost) make use of them however, so the entire comparison was unfair to Rust.

This kind of **dynamic linkage** is a double-edged sword. It makes trivial to update libraries used by multiple programs and in theory the total binary size should be reduced. There is a strong minority against dynamic linkage though, as it also makes trivial to *break* libraries (dynamic linker is not like Cargo, how pity) and its effect on the total binary size has been exaggerated. To elaborate on the latter, earlier in this post I’ve mentioned a problem that the LTO eventually solves, and the dynamic linkage suffers from the same problem but without no solution—LTO on dynamic library would ruin its advantage.

But in spite of those problems, dynamic linkage remains a popular choice for many platforms and especially C and C++ standard libraries are often only available as a dynamic library. Yeah, we instructed the linker to statically link everything, but [glibc lies](https://stackoverflow.com/questions/8140439/why-would-it-be-impossible-to-fully-statically-link-an-application) and some functions still require the dynamic library:

```console
$ # restore debug symbols
$ touch *.c
$ make hello-c CFLAGS='-Os -flto -static'
cc -Os -flto -static    hello-c.c   -o hello-c
$ strings hello-c | grep dlopen
shared object cannot be dlopen()ed
dlopen
invalid mode for dlopen()
do_dlopen
sdlopen.o
dlopen_doit
__dlopen
__libc_dlopen_mode
```

It is actually a clever way to make a near-static library with some dynamic-only features (iconv for example). Of course, if you depend on those features you are doomed. That aside, however, you need to enable static linkage in the linker side to avoid biases; `-static` linker option is therefore necessary. It is very interesting to see that the Rust binary is actually *smaller* after the static linkage. It does depend on the C standard library but only a little bit (standard I/O completely bypasses `stdio`, for example), so it did manage to escape the weight of glibc somehow.

The picture has changed so much that the comparison still looks biased. It is all a fault of glibc after all! There are several libc alternatives, and [musl](https://www.musl-libc.org/) is a promising one. It results in very compact binary even after static linkage:

```console
$ touch *.c
$ make hello-c CFLAGS='-Os -flto -static -s' CC=musl-gcc
musl-gcc -Os -flto -static -s    hello-c.c   -o hello-c
$ ls -al hello-c
-rwxrwxr-x 1 lifthrasiir 5328 May 31 20:59 hello-c*
```

Can we use musl in Rust? Of course. You can install standard libraries for other targets with `rustup`. This time `-C link-args=-static` is not necessary as musl links statically by default.

```console
$ rustup target install x86_64-unknown-linux-musl
$ cargo build --release --target=x86_64-unknown-linux-musl
$ ls -al target/x86_64-unknown-linux-musl/release/hello
-rwxrwxr-x 1 lifthrasiir 263743 May 31 21:07 target/x86_64-unknown-linux-musl/release/hello*
$ strip target/x86_64-unknown-linux-musl/release/hello
$ ls -al target/x86_64-unknown-linux-musl/release/hello
-rwxrwxr-x 1 lifthrasiir 165448 May 31 21:07 target/x86_64-unknown-linux-musl/release/hello*
```

Okay, so this finally looks fair. The entirety of this 160 KB executable can be properly attributed to Rust’s “bloat”; it contains libbacktrace and libunwind (weighing about 50 KB combined), and libstd is still hard to completely optimize out (having 40 KB of pure Rust code, referencing various bits of libc). This is the status quo of the Rust standard library, and has to be worked out. Well, at least this is one time cost per executable, so practically they wouldn’t matter much anyway.

There is yet another approach for the fairness. What if we can use dynamic linkage for Rust? This completely ruins the distribution until every OS ships with Rust standard library, but well, worth trying. Note that LTO, custom allocator and different panic strategy is not compatible to dynamic linkage, so you have to revert them first.

```console
$ cargo rustc --release -- -C prefer-dynamic
$ ls -al target/release/hello
-rwxrwxr-x 1 lifthrasiir 8831 May 31 21:10 target/release/hello*
```

So this is comparable to ordinary C/C++ programs with dynamic linkage. Mostly a curiosity in this case though.

## Say goodbye to libstd

We have so far looked at reducing the executable size without changing your code a lot. But if you are willing to pay some cost for that, you may result in much smaller executable. Note: This is *not* recommended in general, this section is going to be a strict cost-effect analysis.

We all know that libstd is friendly and convenient, but sometimes it is too much (and it is a source of libbacktrace and libunwind that we don’t really use). Start by avoiding libstd first. The Book has a whole section about [no stdlib mode](https://doc.rust-lang.org/book/no-stdlib.html), so let’s follow that. A new source code is as follows:

```rust
#![feature(lang_items, start)]
#![no_std]

extern crate libc;

#[start]
fn start(_argc: isize, _argv: *const *const u8) -> isize {
    // as we are passing a C string,
    // the final null character is mandatory
    const HELLO: &'static str = "Hello, world!\n\0";
    unsafe { libc::printf(HELLO.as_ptr() as *const _); }
    0
}

#[lang = "eh_personality"] extern fn eh_personality() {}
#[lang = "panic_fmt"] extern fn panic_fmt() -> ! { loop {} }
```

And you need to add a dependency to the `libc` crate in Cargo.toml:

```toml
[dependencies]
libc = { version = "0.2", default-features = false }
```

That’s it! It is very close to what you would get with C program: 8503 bytes before `strip`, 6288 bytes after `strip`. (I’m tried of faking the console up (note the modified time), so I’ll omit them from now on.) musl goes further, but you need to give an additional option to properly link the `libc` crate to musl:

```console
$ export RUSTFLAGS='-L native=/usr/lib/x86_64-linux-musl'
$ cargo build --release --target=x86_64-unknown-linux-musl
$ strip target/x86_64-unknown-linux-musl/release/hello
```

Now we are down to 5360 bytes. Only 32 bytes behind the best C program! How can we do better? Instead of using stdio, you can use the direct system call (in Unix, of course):

```rust
#[start]
fn start(_argc: isize, _argv: *const *const u8) -> isize {
    const HELLO: &'static str = "Hello, world!\n";
    unsafe {
        libc::write(libc::STDOUT_FILENO,
                    HELLO.as_ptr() as *const _, HELLO.len());
    }
    0
}
```

This strips yet another bits of binary and weighs 5072 bytes after `strip`.

One may argue that the above is not a real Rust code, but a Rust-esque code depending on Unix system calls. Fine. No real Rust code would look like that. Instead we can make our own `Stdout` type, wired directly to system calls, and use ordinary formatting macros:

```rust
use core::fmt::{self, Write};

struct Stdout;
impl Write for Stdout {
    fn write_str(&mut self, s: &str) -> fmt::Result {
        let ret = unsafe {
            libc::write(libc::STDOUT_FILENO,
                        s.as_ptr() as *const _, s.len())
        };
        if ret == s.len() as isize {
            Ok(())
        } else {
            Err(fmt::Error)
        }
    }
}

#[start]
fn start(_argc: isize, _argv: *const *const u8) -> isize {
    let _ = writeln!(&mut Stdout, "Hello, world!");
    0
}
```

This is indeed much more natural Rust code. It weighs 9248 bytes after `strip`, suggesting that the minimal formatting code costs about 4 KB. Other types impose more overhead though; printing `3.14f64` for example costs about 25 KB of additional binary, as [floating-to-decimal conversion is hard](https://github.com/rust-lang/rust/pull/24612). Also mind that this never buffers (except for the internal kernel buffer for pipes).

## And another thing...

We can go much further even from this point, our executable still contains seemingly unused chunk of bytes. Welcome to the realm of actual low-level programming (in contrast to *lower-level* programming we normally do with C/C++). This area has been very thoroughly explored by pioneers however, so I will end the trip by linking to them:

* Keegan McAllister made a [151-byte x86-64 Linux program](http://mainisusuallyafunction.blogspot.kr/2015/01/151-byte-static-linux-binary-in-rust.html) printing “Hello!”. Well, that’s not we wanted though, and the proper “Hello, world!” program will probably cost 165 bytes; the original program (ab)used the ELF header to put the string constant but there is not much space to put a longer “Hello, world!”.

* Peter Atashian made a [1536-byte Windows program](https://github.com/retep998/hello-rs/blob/master/windows/src/main.rs) printing “Hello world!” (note no comma). I’m less sure about the size of the proper program, but yeah, you have an idea. This is notable because Windows essentially forces you to use dynamic linkage (Windows system call is not stable across versions), and the dynamic symbol table costs bytes.

* For the comparison and your geeky pleasure, the shortest known version of x86-64 Linux program printing “Hello, world” costs (note no exclamation mark) only [62 bytes](http://www.muppetlabs.com/~breadbox/software/tiny/hello.asm.txt). The general technique is better explained in [this classical article](http://www.muppetlabs.com/~breadbox/software/tiny/teensy.html).

* Hey, I’ve learned the hard way that we have tons of variations over the “Hello, world!” programs, none of those programs print the proper and longest version of greeting!

<a name="takeaway"></a> This post is intended as a tour to various techniques (not necessarily practical) reducing the program size, but if you demand some conclusion, the pragmatic takeaway is that:

* Compile with `--release`.
* Before the distribution, enable LTO and strip the binary.
* If your program is not memory-intensive, use the system allocator (assuming nightly).
* You may be able to use the optimization level `s`/`z` in the future as well.
* I didn’t mention this because it doesn’t improve such a small program, but you can also try [UPX](http://upx.sourceforge.net/) and other executable compressors if you are working with much larger application.

That’s all, folks!
