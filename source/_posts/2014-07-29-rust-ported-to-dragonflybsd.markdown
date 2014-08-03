---
layout: post
title: "Rust ported to DragonFly BSD"
date: 2014-07-29 11:27:14 +0200
comments: true
categories: 
---

During the last week I spent numerous hours on porting my favorite programming
language [Rust][rust] to operate on my favorite operating system
[DragonFly][dragonfly]. While my [first attempt][first] months earlier failed,
this time I was successful! In the following article I go through all the
problems I had to deal with and how you can build Rust for DragonFly yourself.
This might also be of interest to people who want to port Rust to further
platforms like NetBSD or OpenBSD.

You find the scripts needed to compile Rust on DragonFly
[here][rust-cross-dragonfly] and my DragonFly branch of Rust
[here][rust-dragonfly] (hopefully to be merged soon).

You can download the binary distribution of the Rust compiler including the
Cargo package manager [here][download-binary] (size: 93 MB, [checksum]). The
script to bootstrap [Cargo][cargo] can be found [here][cargo-script].

## Quick overview

In short what needs to be done to port Rust to a new platform is the following:

1. Extend the ```rustc``` compiler to generate code for the platform you are porting to. 

2. As ```rustc``` depends on [LLVM][llvm] this might require some patching here too, to
   be able to generate code for the target operating system (in my case segmented stacks
   were unsupported for DragonFly).

3. As Rust (still) requires segmented stacks there needs to be support for it in some way
   in the operating system. All Rust and LLVM need is a thread local field where they can
   store the size of the stack.

4. The Rust libraries (e.g. liballoc, liblibc, libstd, libnative) need to be ported. This basically
   involves adding some ```#[cfg(target_os = "dragonfly")]``` somewhere, but also some system 
   structures need to be adapted.

Once this is done we are ready to cross-compile Rust to DragonFly.

## Cross compiling

After trying to cross-compile Rust by specifying ```--target x86_64-pc-dragonfly-elf``` to Rust's 
own ```configure``` script and spending numerous hours just to note that the build fails, I gave 
up on this idea. Instead I took a different approach as shown below. Note that this all requires 
my [dragonfly branch of rust][rust-dragonfly]. All stages depend sequentially on another. Below
we are using the scripts from my [rust-cross-dragonfly][rust-cross-dragonfly] project.

### Stage1 (Linux, DragonFly)

* On Linux: Compile a modified ```rustc``` on Linux that supports DragonFly as target (rustc natively is
  able to generate code for different targets!).

* On DragonFly: At the same time we can build the C libraries Rust depends on.

Use scripts ```stage1-linux.sh``` and ```stage1-dragonfly.sh``` respectively.

Actually I tried to cross-compile everything on Linux, but I failed for LLVM.
So I decided to build everything that involved compiling C or C++ on the target
(DragonFly) itself.

### Stage2 (Linux)

We have to copy the outcome of stage1-dragonfly (which is stage1-dragonfly.tgz)
to the Linux box and extract it to create ```stage1-dragonfly/```. This
includes the C libraries required for Rust as well as some system libraries.

Stage2 (```sh stage2-linux.sh```) cross-compiles all of Rust's libraries (e.g.
```libstd```, ```libsyntax```, ```librustc```) and generates the object file
for the ```rustc``` binary (```driver.o```). I don't generate the executable
here itself, which I could, but when I did, static initializations were omitted
and the generated ```rustc``` was unable to find statically registered command
line options of LLVM. I spent hours to figure this out. The solution was to
just generate the object file for ```rustc``` and together with the Rust
libraries pass it on to Stage3, which links the ```rustc``` binary on
DragonFly.

Copy the outcome of Stage2 (```stage2-linux.tgz```) back to the DragonFly system.

### Stage3 (DragonFly)

Finally back again on DragonFly. Extract ```stage2-linux.tgz``` to become ```stage2-linux```.
In this stage we will construct a working ```rustc``` binary (the Rust compiler) and test to
compile a simple Hello World application ```hw.rs``` natively on DragonFly.

All you have to do is to execute ```sh stage3-dragonfly.sh```.

### Stage4 (DragonFly)

Stage4 is the last stage and it builds my [dragonfly branch][rust-dragonfly]
with the compiler from Stage3. It uses Rust's own build infrastructure to do so
and not my "hacked" build infrastructure.

All you have to do is to execute ```sh stage4-dragonfly.sh``` and wait. It does
not automatically install Rust for you, so you have to ```gmake install```
inside directory ```stage4-dragonfly/rust``` yourself (needing admin
privileges).

## Problems faced

* LLVM does not support segmented stacks on DragonFly. So I had to [patch LLVM][patch-llvm] and
  will submit the patch upstream soon.

* The LLVM patch implies having a special field in the thread control block to record the stack size.
  Thanks to Matthew Dillon I was "allowed" to use a yet unused field for it and [the patch][patch-dragonfly]
  got accepted.

* Rust fails with a memory corruption when jemalloc is not used, see my [bug report #16071][jemalloc-bug-report].
  I noticed that earlier on Linux but ignored it. Later when I had a ```rustc``` compiler working on DragonFly
  I got hit by this again as I built it without jemalloc. It cost me a whole night ktracing it and several reboots.
  The solution was to compile with jemalloc. 

* Intially the port of ```libstd``` and ```liblibc``` were basically copies of the related FreeBSD code. Several times I got
  hit by differences in system structures and return codes. The right solution would be to generate code like this
  in an automated fashion.

* Cross-linking, that is linking an executable targeted for DragonFly on Linux, has some limitations, or more
  probable is that I did something wrong. Static initializations are handled incorrectly (they are not executed). 
  It took me a while to figure this out after searching through the LLVM code.

* Last but not least, compiling Rust takes an enormous amount of time. That is, the turn-around times are
  quite high. Compiling, fixing an error, compiling again. And so on and on.

## Conclusions

Rust works on DragonFly, yippey! I work on uploading a snapshot and maybe
setting up a regular build and add it to dports. Hopefully this document also
helps others in porting Rust to further platforms, mainly OpenBSD and NetBSD.

[rust]: http://www.rust-lang.org/
[dragonfly]: http://www.dragonflybsd.org/
[first]: https://mail.mozilla.org/pipermail/rust-dev/2014-January/007730.html
[llvm]: http://www.llvm.org/
[rust-cross-dragonfly]: https://github.com/mneumann/rust-cross-dragonfly
[rust-dragonfly]: https://github.com/mneumann/rust/tree/dragonfly
[patch-llvm]: https://github.com/mneumann/rust/blob/dragonfly/patch-llvm
[patch-dragonfly]: https://github.com/DragonFlyBSD/DragonFlyBSD/commit/166bae116c2735bbfd353b135de67878e1e5744b
[jemalloc-bug-report]: https://github.com/rust-lang/rust/issues/16071
[download-binary]: /downloads/rust/rust-0.12.0-pre-dragonfly.tar.bz2
[checksum]: /downloads/rust/rust-0.12.0-pre-dragonfly.tar.bz2.sha1.txt
[cargo]: http://crates.io/
[cargo-script]: /downloads/rust/cargo.sh
