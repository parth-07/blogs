---
layout: post
title: "Build and run RISCV 'Hello, World!' program on x86_64 Linux"
---

RISCV is the new shiny promising architecture that many major companies have taken a keen interest
in. But how do we play around with RISCV without having a RISCV hardware? Let's see.

This is a short and simple guide to quickly build RISCV programs on x86_64 Linux.
To build and run RISCV programs on x86_64 Linux, we need two tools:

  - RISCV cross-compilation toolchain. That is a toolchain that runs on x86_64 Linux machines but 
    generates code for RISCV Linux machines. We will use the GNU RISCV cross-compilation toolchain.

  - An emulator to emulate RISCV programs on x86_64 Linux. We will use QEMU for emulating 
    RISCV programs.

# Building RISCV GNU cross-compilation toolchain

See [riscv-gnu-toolchain](https://github.com/riscv-collab/riscv-gnu-toolchain) for detailed
information and build instructions

```bash
$ git clone https://github.com/riscv-collab/riscv-gnu-toolchain.git
$ ./configure --prefix=$(pwd)/../inst --enable-multilib
$ make linux
$ make install
```

# Building RISCV QEMU

See [qemu](https://github.com/qemu/qemu) for detailed information and build instructions.

```bash
$ git clone https://github.com/qemu/qemu.git
$ ./configure --target-list=riscv64-linux-user  --prefix=$(pwd)/../inst
$ make install
```

Now, all the prerequisites are complete. Let's build and run the "Hello, World" program.

# Building and running 'Hello, World' program

```bash
$ export PATH="${riscv-gnu-toolchain_install-root}/inst/bin:$PATH"
$ export PATH="${qemu_install-root}/bin:$PATH"
```

```c
#include <stdio.h>

int main() {
  printf("Hello, World!\n");
  return 0;
}
```
*hello_world.c*

```bash
$ riscv64-unknown-linux-gnu-gcc hello_world.c
$ qemu-riscv64 -L ${riscv-gnu-toolchain_install-root}/sysroot a.out
Hello, World!
```

We have passed the riscv gnu toolchain sysroot to QEMU so that QEMU knows where to find
the standard libraries.
