---
layout: post
title: "Cross-compiling for DragonFly BSD"
date: 2014-02-23 19:03:05 +0100
comments: true
categories: 
---

In this short article I want to show the basics of how to generate an
executable for the [DragonFly BSD][dragonfly] operating system from a Linux
system.  This process is called _cross compiling_. The reason why I
investigated into this topic was that I wanted to cross-compile the [Rust]
compiler.

All you need is <code>clang</code> installed and a [DragonFly BSD][dragonfly]
installer ISO image which you can obtain from it's homepage. The application we
want to cross-compile is the one listed below:

```c
#include <stdio.h>

int main(int argc, char **argv) {
  printf("Hello World\n");
  return 0;
}
```

The first step is to <code>mount</code> the ISO image into the local directory
<code>./iso</code> because we need the header files for compilation as well as
the files <code>crt{1,i,begin}.o</code> and <code>libgcc.a</code> for linking.

```sh
mkdir iso
sudo mount -o loop,ro DragonFly-x86_64-LATEST-ISO.iso ./iso
```

After that, we are ready to compile our simple Hello World example application.
To compile we need to specify the corresponding include and link paths and tell
the compiler the target we want to create code for, in our case this is
<code>x86\_64-pc-dragonfly-elf</code>.

```sh
clang -I./iso/usr/include \
      -L./iso/usr/lib -L./iso/usr/lib/gcc47 \
      -B./iso/usr/lib -B./iso/usr/lib/gcc47 \
      -target x86_64-pc-dragonfly-elf \
      -o hw hw.c 
```

Running <code>file hw</code> should now display something like this:

    hw: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked
        (uses shared libs), for DragonFly 3.0.702, not stripped

Voilà, there it is, our binary for [DragonFly BSD][dragonfly].

[dragonfly]: http://www.dragonflybsd.org/
[Rust]: http://www.rust-lang.org/
