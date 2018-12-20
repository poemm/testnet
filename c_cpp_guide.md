# Compiling C/C++ to Ewasm

First an introduction, then a step-by-step guide. Warning: the Ewasm spec and tools below are subject to change.

## Introduction

An Ewasm contract is a WebAssembly module with the following restrictions. The module's imports must be among the [Ewasm helper functions](https://github.com/ewasm/design/blob/master/eth_interface.md) which resemble EVM opcodes to interact with the client. The module's exports must be a `main` function which takes no arguments and returns nothing, and the `memory` of the module. The module can't using floats or other sources of non-determinism.

Below are quirks to be aware of when writing Ewasm contracts in C/C++. The last three quirks are not limited to C/C++. These quirks may improve as tools and Ewasm improve.

Quirk 1) WebAssembly is still primitive and lacks features. For example, WebAssembly lacks support for exceptions and we have no way to do system calls in Ewasm. Compilers and libraries are still primitive. For example, we compile against patched versions of libc and libc++ to allow us to use `malloc`, but the patches are not enough for `std::vector` which uses more than just `malloc` to manage memory. But anything beyond memory allocation may be unwanted for Ewasm contracts since it costs gas. This situation will improve as WebAssembly, compilers, and libraries mature.

Quirk 2) In the current Ewasm design, all communication between the contract and the client is done through the module's memory. For example, the message data ("call data") sent to the contract is accessed by calling `callDataCopy()`, which puts this data to WebAssembly memory at a location given by a pointer. We must allocate this pointer's memory in C/C++ using `malloc`. For example, before calling `callDataCopy()`, one may use `getCallDataSize()` to see how many bytes of memory to `malloc`.

Quirk 3) In the current Ewasm design, the Ethereum client writes data into WebAssembly as big-endian, and WebAssembly memory is little-endian, so has reversed bytes when the data is brought to/from the WebAssembly operand stack. For example, when the call data is brought into memory using `callDataCopy`, and those bytes are loaded to the WebAssembly stack using `i64.load`, all of the bytes are reversed. So extra C/C++ code may be needed load bytes from the correct location and to reverse the loaded bytes.

Quirk 4) The output of compilers is a `.wasm` binary which may have imports and exports which do not meet Ewasm requirements. These must be manually fixed to be a valid Ewasm contract.

Quirk 5) There are no tutorials for debugging/testing a contract. Hera supports extra Ewasm helper functions to print things, to help in writing test cases. A tutorial is needed to allow early adopters to debug/test their contracts.

## Basic Step-by-Step Guide

First build the latest version of LLVM. Note that if you wish to use standard libraries, use the version of LLVM in the Advanced section below. That version can also be used here.

```sh 
# checkout LLVM, clang, and lld
svn co http://llvm.org/svn/llvm-project/llvm/trunk llvm
cd llvm/tools
svn co http://llvm.org/svn/llvm-project/cfe/trunk clang
svn co http://llvm.org/svn/llvm-project/lld/trunk lld
cd ../..

# build LLVM
mkdir llvm-build
cd llvm-build
# note: if you want other targets than WebAssembly, then delete -DLLVM_TARGETS_TO_BUILD 
cmake -G "Unix Makefiles" -DLLVM_TARGETS_TO_BUILD= -DLLVM_EXPERIMENTAL_TARGETS_TO_BUILD=WebAssembly $WORKDIR/llvm                 
make -j 8       # WARNING, THIS CAN TAKE HOURS, NEEDS LOTS OF DISK AND RAM, IF IT FAILS, TRY AGAIN WITHOUT -j 8
``` 

Next download and compile a wrc20 ewasm contract written in C. In `main.c` there are many arrays in global scope -- LLVM puts global arrays in WebAssembly memory, which is needed so that they can be used to communicate with Ethereum helper functions. Before compiling, make sure that the `Makefile` has a path to `llvm-build` above, and that `main.syms` has a list of Ewasm helper functions you are using.

Aside: If you are using C++, make sure to modify the Makefile to `clang++`, use `extern "C"` around the helper function declarations.

```sh
git clone https://gist.github.com/poemm/68a7b70ec353abaeae64bf6fe95d2d52.git cwrc20
cd cwrc20
# edit the Makefile and main.syms as described above
make
```

The output is `main.wasm` which needs a cleanup of imports and exports to meet Ewasm requirements. For this, we use PyWebAssembly.

Aside: Alternatively, one can manually cleanup. Alternatively, one can use [wasm-chisel](https://github.com/wasmx/wasm-chisel) which is a program in Rust which can be installed with `cargo install chisel`. The Rust version is stricter and has more features, the Python version is just enough for our use-case, and Python is available on most machines.

```
cd ..
git clone https://github.com/poemm/pywebassembly.git
cd pywebassembly/examples/
python3 ewasmify.py ../../cwrc20/main.wasm
cd ../../cwrc20
```

Check whether the command line output of `ewasmify.py` above lists only valid Ewasm imports and exports. If not, we may need libc to fill these in, see the Advanced section.

To deploy the Ewasm contract from http://ewasm.ethereum.org/studio/, we need to convert it from the `.wasm` binary format to the `.wat` (or `.wast`) text format (these are equivalent formats and can be converted back-and-forth). This conversion can be done with Binaryen's `wasm-dis`.

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

Now `main_ewasmified.wat` can be pasted into http://ewasm.ethereum.org/studio/ and deployed. Happy hacking!


## Advanced

The above guide is for compiling a single C file with no libc or system calls. The wasmception package offers patches to libc and libc++ which allow using libc, including things like `malloc`, when targeting WebAssembly. It also allows using libc++. Wasmception is not specific to Ewasm, it is maintained by a third party.

```
git clone https://github.com/yurydelendik/wasmception.git
cd wasmception
make -j4	# Warning: this required lots of internet bandwidth, RAM, disk space, and one hour compiling on a mid-level laptop.
cd ..
```
Write down the end of the output of the above `make` command, it should include something like: `--sysroot=/home/user/repos/wasmception/sysroot`.

We modified the wasmception example to create a version of wrc20 which uses `malloc`, which we will now download and compile. Make sure to edit the `Makefile` with the sysroot data above, and change the path to `clang` to our newly compiled version which may look something like `/home/user/repos/wasmception/dist/bin/clang`. Make sure that `main.syms` has a list of Ewasm helper functions you are using.

Aside: If you are using C++, make sure to modify the Makefile to `clang++`, use `extern "C"` around the helper function declarations, and follow other tips from wasmception.

```sh
git clone https://gist.github.com/poemm/91b64ecd2ca2f1cb4a88d31315313b9b.git cwrc20_with_malloc
cd cwrc20_with_malloc
# edit the Makefile and main.syms as described above
make
```

The user is encouraged to expore more advanced things. To statically link against other C files, we can link the LLVM IR as described here https://aransentin.github.io/cwasm/. The user is encouraged to explore ways to do advanced things. 
