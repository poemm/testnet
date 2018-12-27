# Compiling C/C++ to Ewasm


## Introduction

The goal of this document is to provide a guide to compile C/C++ to Ewasm. This document assumes knowledge of the [Ewasm design repo](https://github.com/ewasm/design). Especially relevant is that an Ewasm contract is a WebAssembly module with [restrictions](https://github.com/ewasm/design/blob/master/contract_interface.md), and that an Ewasm contract can call [Ewasm helper functions](https://github.com/ewasm/design/blob/master/eth_interface.md), which resemble EVM opcodes.

## Caveats

When writing Ewasm contracts in C/C++, one should bear in mind the following caveats.

1. WebAssembly is still primitive and [lacks features](https://github.com/WebAssembly/design/blob/master/FutureFeatures.md). For example, WebAssembly lacks support for exceptions. Also, compilers and standard libraries support is still primitive, often existing as experimental features or third party patches.

1. Ewasm does not support operating system calls, for example `printf()` and `clock()` are unsupported. This is especially relevant for contracts which require memory management -- we patch libc with a WebAssembly-compatible version of `malloc` which we compile into the module. But the patches are not yet enough for `std::vector` because it requries other memory managment calls which are not yet patched.

1. [Ewasm helper functions](https://github.com/ewasm/design/blob/master/eth_interface.md) communicate with the contract by reading/writing to/from the WebAssembly module's memory at locations provided by a C/C++ pointer. For example, this C/C++ pointer can point to a statically allocated array in global scope (since our LLVM compiler maps C/C++ global arrays to WebAssembly memory), or to dynamically allocated memory using `malloc`. To conserve gas, it may be wise to avoid `malloc` whenever possible.

1. Ethereum clients read/write data into WebAssembly memory as big-endian, but WebAssembly memory is little-endian, so bytes are reversed when brought to/from the WebAssembly operand stack. For example, when the call data is brought into memory using Ewasm helper function `callDataCopy`, and those bytes are loaded to the WebAssembly stack using `i64.load`, all of the bytes are reversed. So extra C/C++ code may be needed to to reverse the loaded bytes.

1. The output of compilers is a `.wasm` binary which may not meet [Ewasm requirements](https://github.com/ewasm/design/blob/master/contract_interface.md). In the examples below, we use tools which automatically apply changes to meet these requirmentes. When writing more complicated contracts, manual inspection and troubleshooting may be required.

1. Tools are needed for developers to debug/test their Ewasm contracts. Early adopters may debug/test on the testnet by printing things to storage.

## Basic Step-by-Step Guide

First build the latest version of LLVM, clang, and lld. Below we devaite from the [official instructions](https://clang.llvm.org/get_started.html) by using git and adding some flags to `cmake`.

Aside: If you wish to use C/C++ standard libraries, then build the version of LLVM in the Advanced section below. That version can also be used here.

Aside: The following dowloads hundreds of megabytes. The `make` step can take hours, require a lot of disk space and memory, and may even cause your computer to freeze. If there is an out-of-memory error, try again without the `-j 8` argument (which attempts to run eight parallel build processes).

```sh 
# checkout LLVM, clang, and lld
git clone http://llvm.org/git/llvm.git
cd llvm/tools
git clone http://llvm.org/git/clang.git
git clone http://llvm.org/git/lld.git
cd ../..

# build LLVM, clang, and lld
mkdir llvm-build
cd llvm-build
cmake -G "Unix Makefiles" -DLLVM_TARGETS_TO_BUILD= -DLLVM_EXPERIMENTAL_TARGETS_TO_BUILD=WebAssembly ../llvm                 
make -j 8
``` 


Next download and compile an example ewasm contract written in C. Note that in `main.c`, there are many arrays in global scope: LLVM puts global arrays in WebAssembly memory, which allows them to be used as pointer arguments to Ethereum helper functions. Before compiling, make sure that the `Makefile` has a path to `llvm-build` above, and that `main.syms` has a list of Ewasm helper functions you are using.

Aside: If you are using C++, make sure to modify the Makefile to `clang++`, use `extern "C"` around the helper function declarations.

```sh
git clone https://gist.github.com/poemm/68a7b70ec353abaeae64bf6fe95d2d52.git cwrc20
cd cwrc20
# edit the Makefile and main.syms as described above
make
```

The output is `main.wasm` which needs a cleanup of imports and exports to meet [Ewasm requirements](https://github.com/ewasm/design/blob/master/contract_interface.md). For this, we can use [PyWebAssembly](https://github.com/poemm/pywebassembly), perform the cleanup manually, or use [wasm-chisel](https://github.com/wasmx/wasm-chisel), a program in Rust which can be installed with `cargo install chisel`. `wasm-chisel` is stricter and has more features, whereas `PyWebAssembly` is just enough for our use case, and Python is available on most machines. We therefore recommend using PyWebAssembly as follows.

```
cd ..
git clone https://github.com/poemm/pywebassembly.git
cd pywebassembly/examples/
python3 ewasmify.py ../../cwrc20/main.wasm
cd ../../cwrc20
```

The output `main_ewasmified.wasm` should be a valid Ewasm contract. But it should be inspected to make sure that the imports and exports meet [Ewasm requirements](https://github.com/ewasm/design/blob/master/contract_interface.md). We do this inspection by first converting from the `.wasm` binary format to the `.wat` text format (these are equivalent formats and can be converted back-and-forth). This conversion can be done with Binaryen's `wasm-dis`.

Aside: Alternatively one can use Wabt's `wasm2wat`. But Binaryen's `wasm-dis` is recommended because Ewasm studio uses Binaryen internally, and Binaryen can be quirky and fail to read a `.wat` generated by another program. Another tip: if Binaryen's `wasm-dis` can't read the `.wasm`, try using Wabt's `wasm2wat` then `wat2wasm` before trying again with Binaryen.

```sh
cd ..
git clone https://github.com/WebAssembly/binaryen.git	# warning 90 MB, can also download precompiled binaries which are 15 MB
cd binaryen
mkdir build && cd build
cmake ..
make -j4
cd ../../cwrc20
../binaryen/build/bin/wasm-dis main_ewasmified.wasm > main_ewasmified.wat
```

`main_ewasmified.wat` is an ewasm contract in text format. See other notes for how to deploy it. Happy hacking!


## Advanced

The above guide is for compiling a C file with no libc. Next we use a package which provides a minimal toolchain which includes libc and libc++, as well as patches allowing things like `malloc`.

```
git clone https://github.com/yurydelendik/wasmception.git
cd wasmception
make	# Warning: this required lots of internet bandwidth, RAM, disk space, and one hour compiling on a mid-level laptop.
cd ..
```
Write down the end of the output of the above `make` command, it should include something like: `--sysroot=/home/user/repos/wasmception/sysroot`.

Next we will download and build a version of wrc20 which uses `malloc`. Make sure to edit the `Makefile` with the sysroot data above, and change the path of `clang` to our newly compiled version which may look something like `/home/user/repos/wasmception/dist/bin/clang`. Make sure that `main.syms` has a list of Ewasm helper functions you are using.

Aside: If you are using C++, make sure to modify the Makefile to `clang++`, use `extern "C"` around the helper function declarations, and follow other tips from wasmception.

```sh
git clone https://gist.github.com/poemm/91b64ecd2ca2f1cb4a88d31315313b9b.git cwrc20_with_malloc
cd cwrc20_with_malloc
# edit the Makefile and main.syms as described above
make
```

Now follow the same steps above to transform the output `main.wasm` into a valid Ewasm contract.

Tutorials are needed for more advanced things. For example, to statically link against other C files, one can link the LLVM IR as described here https://aransentin.github.io/cwasm/.
