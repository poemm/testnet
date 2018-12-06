# Compiling C/C++ to Ewasm

First an introduction to discuss quirks, then a step-by-step guide for how to compile Ewasm contracts from C and C++. Warning: the Ewasm spec and tools below are subject to change.

## Introduction

An Ewasm contract is a WebAssembly module with the following restrictions. The module's imports must be among the [Ewasm helper functions](https://github.com/ewasm/design/blob/master/eth_interface.md) which resemble EVM opcodes to interact with the client. The module's exports must be a `main` function which takes no arguments and returns nothing, and the `memory` of the module. The module can't using floats and other sources of non-determinism.

Below are four quirks of using C/C++ to write Ewasm smart contracts. These quirks may improve as tools and Ewasm improve.

Quirk 1) WebAssembly is still primitive and lacks features. For example WebAssembly supports function pointers, but lacks support for exceptions. We have no way to do systeam calls. We compile against patched versions of libc and libc++ to allow us to use `malloc`, but the patches are not enough for `std::vector` which uses more than just `malloc` to manage memory. Anyway, smart contracts may not want to manage and `free` memory since this costs gas. This situation will improve as WebAssembly, compilers, and libraries mature.

Quirk 2) In the current Ewasm design, all communication between the contract and the client is done through the module's memory. For example, the message data ("call data") to the contract is accessed by calling `callDataCopy` to put this data to a memory location. Accessing a contract's storage requires calls to `storageStore` or `storageLoad`, which require memory offsets of the storage key and value. We will allocate memory in C/C++ using `malloc`. For example, before calling `callDataCopy`, one may use `getCallDataSize` to see how many bytes of memory to `malloc`.

Quirk 3) In the current Ewasm design, the Ethereum client writes data into WebAssembly as big-endian, and WebAssembly memory is little-endian, so all data has reversed bytes when they are in the WebAssembly operand stack. For example, when the call data is brought into memory using `callDataCopy`, and those bytes are loaded to the WebAssembly stack using `i64.load`, all of the bytes are reversed. So a function may be needed in C/C++ to reverse the bytes.

Quirk 4) The output of compilers is a `.wasm` binary which may have some extra imports and exports with strange names. These must be manually fixed to be a valid Ewasm contract.


## Step-by-Step Guide

The wasmception package offers patches to libc and libc++ which allow using `malloc` when targeting WebAssembly. This tool is not specific to Ewasm, it is maintained by a third party. Warning: compiling wasmception will download llvm, llvm tools, musl C library, and libc++, which will require internet bandwidth, lots of RAM, and time.

```sh 
git clone https://github.com/yurydelendik/wasmception.git
cd wasmception
make -j4	# Warning: this required lots of internet bandwidth, RAM, and one hour on a mid-level laptop.
cd ..
``` 

Write down the end of the output of the above `make` command, it should include something like: `--sysroot=/home/user/repos/wasmception/sysroot`.

We modified the wasmception example into an Ewasm contract, which we will now download. Make sure to edit the `Makefile` with the sysroot data above, and change the path to clang to our newly compiled version which may look something like `/home/user/repos/wasmception/dist/bin/clang`. Make sure that `main.syms` has a list of Ewasm helper functions you are using. If you are using C++, make sure to modify the Makefile to clang++, use `extern "C"` around the helper function declarations, and follow other tips from wasmception.

```sh
git clone https://gist.github.com/poemm/91b64ecd2ca2f1cb4a88d31315313b9b.git cwrc20
cd cwrc20
# edit the Makefile and main.syms as described above
make
```

The output is `main.wasm`, but it needs some cleanup to remove some of it's extra import and export junk. For this, we use PyWebAssembly. 

Aside: Alternatively, one can manually clean this junk. Alternatively, one can use a [rust version of wasm-chisel](https://github.com/wasmx/wasm-chisel) which can be installed with `cargo install chisel`. The Rust version has more features, the Python version is just enough for our use. In either case, before deploying, contracts should be visually inspected to make sure that imports and exports meet Ewasm requirements.

```
cd ..
git clone https://github.com/poemm/pywebassembly.git
cd pywebassembly/examples/
python3 ewasm_chisel.py ../../cwrc20/main.wasm
cd ../../cwrc20
```

The output `main_chiseled.wasm` is an Ewasm contract. To deploy it from http://ewasm.ethereum.org/studio/, we need to convert it form the `.wasm` binary format to the `.wat` (or `.wast`) text format. This conversion can be done with binaryen's `wasm-dis`.

Aside: Alternatively one can download the precompiled release of Binaryen, or try using Wabt's `wasm2wat` instead. But Binaryen's `wasm-dis` is recommended because Ewasm studio uses Binaryen internally, and Binaryen can be quirky and fail to read the `.wat` generated somewhere else. Also, if Binaryen's `wasm-dis` can't read the `.wasm`, try using Wabt's `wasm2wat` then `wat2wasm` before trying again with Binaryen.

```sh
cd ..
git clone https://github.com/WebAssembly/binaryen.git	# warning 90 MB, can also download precompiled binaries which are 15 MB
cd binaryen
mkdir build && cd build
cmake ..
make -j4
cd ../../cwrc20
../binaryen/build/bin/wasm-dis main_chiseled.wasm > main_chiseled.wat
```

Now `main_chiseled.wat` can be pasted into http://ewasm.ethereum.org/studio/ and deployed. Happy hacking!
