---
title: Getting Started
date: '2024-01-31'
categories:
  - Rust
  - Kernel Driver
  - Linux Kernel
tags:
  - Rust
  - Kernel Driver
  - Linux Kernel
  - Compiling
  - make
  - Kernel Module
  - Rust for Linux
---

# Check out code

I recommend checking out the [Rust-for-Linux kernel](https://github.com/Rust-for-Linux) to play around with, because it has many more sample modules than the mainline kernel. Even though the interfaces are probably not stable, you can get an idea where development is headed.

[<img src="/static/img/rusty_penguin_1.jpeg" style="max-width:30%;min-width:40px;float:right;padding:50px" alt="Rusty penguin. Created by DALL·E 3." />](https://github.com/Rust-for-Linux/)

# Get your tools ready
After checking out, you should follow the instructions in
`Documentation/rust/quick-start.rst` for setting up your tools. I had more luck
using rustup to get the versions of bindgen and rustc compatible with the kernel
than using my distributions aptitude command. There is a nifty little script
called `rust_is_available.sh` which will tell you whether your setup works for
compiling a Rust kernel.

In the end, after following the instructions from the quick start guide, I had
to set a few environment variables myself to make it work:




```
$ export BINDGEN=`which bindgen`
$ export RUSTC=`which rustc`
$ export CC=clang
$ export PATH="$HOME/.cargo/bin:$PATH"
$ export LIBCLANG_PATH=/usr/lib/llvm-16/lib
$ export LLVM=1
$ ./scripts/rust_is_available.sh
```

Or here as a oneliner:
```
# oneline:
BINDGEN=`which bindgen` RUSTC=`which rustc` CC=clang PATH="$HOME/.cargo/bin:$PATH" LIBCLANG_PATH=/usr/lib/llvm-16/lib LLVM=1 ./scripts/rust_is_available.sh
```

# Enable Rust in config

Every kernel you want to compile needs a .config file where the options you want
to enable for your kernel are specified. After creating an initial .config from
an existing defconfig, you can use `make menuconfig` to enable Rust. Menuconfig
allows you to search for options using "/", see the dependencies of those
options and directly go to the place where you can activate them using the
numbers which are written left of the option description.

```
# somehow/somewhere get a minimal config where serial, ext4, etc are activated
# or just use a defconfig with make x86_64_defconfig

$ make  LLVM=1 menuconfig
```

# Compile examples

After activating for example `SAMPLE_RUST_PRINT`, you can compile the kernel
using `make` and the modules using `make modules`. At this point, you can play
around with the file `samples/rust/rust_print.rs` and see what other Rust
functions you can add.

To compile out of tree modules, you need to pass the kernel directory to the make command.

```
KDIR=~/code/kernel/Rust-for-Linux make
```

# Load examples
Now you can boot your kernel in qemu and load and unload your modules using
`insmod/rmmod`.

```
devbox# insmod ./samples/rust/rust_print.ko
rust_print: Rust printing macros sample (init)
rust_print: Emergency message (level 0) without args
rust_print: Alert message (level 1) without args
rust_print: Critical message (level 2) without args
rust_print: Error message (level 3) without args
rust_print: Warning message (level 4) without args
rust_print: Notice message (level 5) without args
rust_print: Info message (level 6) without args
rust_print: A line that is continued without args
rust_print: Emergency message (level 0) with args
rust_print: Alert message (level 1) with args
rust_print: Critical message (level 2) with args
rust_print: Error message (level 3) with args
rust_print: Warning message (level 4) with args
rust_print: Notice message (level 5) with args
rust_print: Info message (level 6) with args
rust_print: A line that is continued with args
```

```
devbox# rmmod rust_print.ko
rust_print: Rust printing macros sample (exit)
```
