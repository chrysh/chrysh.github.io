---
title: Out of tree Kernel module
date: '2024-07-06'
categories:
  - Rust
  - Kernel Driver
  - Linux Kernel
tags:
  - Rust
  - Kernel module
  - Linux Kernel
  - Out-of-tree module
---

[<img class="penguin" src="/static/img/rusty_penguin_10.jpeg" alt="Rusty penguin. Created by DALL·E 3." />](https://github.com/Rust-for-Linux/)

# Compiling an out of tree rust module
This chapter will be less Rust focused, and more an introduction on how to
compile Out-of-tree kernel modules in general. The examples were compiled with a
[net-next](https://git.kernel.org/pub/scm/linux/kernel/git/netdev/net-next.git)
kernel based on `v6.8-rc1`.

## Module code

The easiest way to get started is to copy over a module to your directory, e.g.
copy over `samples/rust/rust_minimal.rs` and try compiling it. You can read up
more information about modules in Rust in the blog entry [Porting my first phy driver](../../../02/06/porting-my-first-phy-driver/) or look at the [commits on github](https://github.com/Rust-for-Linux/linux/compare/rust-next...chrysh:Rust-for-Linux:rockchip_driver_rust).


## Rust available?

The second test is to make sure that Rust is available for your compilation:

```
export PATH="$HOME/.cargo/bin:$PATH"
export LIBCLANG_PATH=/usr/lib/llvm-16/lib
export LLVM=1

$ KSRC=~/code/kernel/net-next
$ make -C $KSRC rustavailable

make: Entering directory '~/code/kernel/net-next'
Rust is available!
make: Leaving directory '~/code/kernel/net-next'
```

If it does not show up, make sure to check out the blog entry on [Getting
started](../../../01/31/getting-started/).

Now, you can compile your external Out-of-tree module with the following
command:

```
$ make -C $KSRC M=$(pwd) rustavailable
```

## Makefile

Now, to make you life easier, the next step is creating a Makefile.

```
chrysh@bentobox ~/code/rust/quarto_rs/src (git)-[main] % cat Makefile
# SPDX-License-Identifier: GPL-2.0

# obj-$(CONFIG_SAMPLE_RUST_QUARTO)		+= rust_quarto.o
obj-m += rust_quarto.o
KERNEL_SRC := ~/code/kernel/net-next
SRC := $(shell pwd)

all:
	$(MAKE) -C $(KERNEL_SRC) M=$(SRC) modules

modules_install:
	$(MAKE) -C $(KERNEL_SRC) M=$(SRC) modules_install
```

Don't forget to check out a few environment variables before you compile:
```
$ export PATH="$(HOME)/.cargo/bin:$(PATH)"
$ export LIBCLANG_PATH=/usr/lib/llvm-16/lib
$ export LLVM=1
```

Now, when you enter `make`, you module gets compiled

```
% make
make -C ~/code/kernel/net-next M=/home/chrysh/code/rust/quarto_rs/src modules
make[1]: Entering directory '/home/chrysh/code/kernel/net-next'
  RUSTC [M] /home/chrysh/code/rust/quarto_rs/src/rust_quarto.o
  MODPOST /home/chrysh/code/rust/quarto_rs/src/Module.symvers
  LD [M]  /home/chrysh/code/rust/quarto_rs/src/rust_quarto.ko
make[1]: Leaving directory '/home/chrysh/code/kernel/net-next'
```

## References

You can find this code in the branch [out_of_tree](https://github.com/chrysh/quarto_rs/tree/out_of_tree) on github.
