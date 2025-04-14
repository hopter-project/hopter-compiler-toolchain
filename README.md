<img src="./.github/assets/hopter-logo.png" alt="Hopter's Logo" width="400"/>

This repo contains patches for LLVM, `rustc`, and the `core` library of Rust to include additional features to support [Hopter OS](https://github.com/hopter-project/hopter). The latest patched version is 1.81.0 released on Sep.5th, 2024. The easiest way to get started is downloading a prebuilt patched version from the release page. The rest describes the customization made to the compiler toolchain and the building procedure.

# Quick Start

Download the prebuilt Rust compiler toolchain from the [release](https://github.com/hopter-project/hopter-compiler-toolchain/releases/) page. Currently Linux with x86_64 and MacOS with Apple silicon are supported. Windows users please consider using WSL.

**MacOS users must** download and decompress it through command line tools to avoid putting the binaries into quarantine state, for example using `wget` and `tar`.

```
wget https://github.com/hopter-project/hopter-compiler-toolchain/releases/download/1.81.0-Patched-2024-09-12/240912-rust-aarch64-macos.tar.xz
```

Decompress the downloaded compressed file with `tar`. Double check the chosen OS and architecture.
```
tar -xf xxxx.tar.xz
```

Let us call the decompressed directory `$RUST_INSTALL_DIR`.

Register the downloaded toolchain with `rustup` by running
```
rustup toolchain link segstk-rust $RUST_INSTALL_DIR
```

In case `rustup` is not installed, follow the [Rust official website](https://www.rust-lang.org/tools/install) to install it.

That's it.

# Compiler Modification

Here we summarize the modification for each part of the compiler toolchain.

**LLVM**
- Support segmented stack function prologue generation for `thumbv7em` and `thumbv6m` targets. The prologue examines the free stack space before executing the function body. If the free space is insufficient, the prologue invokes the kernel through SVC. The kernel may decide to extend the stack by making a new memory allocation.
- Generate a slightly different segmented stack function prologue for `core::ptr::drop` functions. If a task exceeds its stack usage allowance, the kernel will kill it by forcing an unwinding. However, the unwinding must not be initiated from a drop function. The kernel can recognize that the function causing an overflow is a drop function, thus defer the forced unwinding.
- Disable `nounwind` attribute deduction. Since any function can potentially cause the stack allowance to be exceeded, any function may initiate a forced unwinding. This breaks the assumption of the compiler when automatically deducing `nounwind` attributes, so we disable it.

**`rustc`**
- Force every Rust function to have the segmented stack function prologue by tagging the `split-stack` attribute to every function during LLVM IR generation. `[naked]` functions are not affected as expected. Also provide the `[no_split_stack]` attribute to supress segmented stack prologue generation for a function.
- Support deferred forced stack unwinding. Increment a global counter before executing the body of a drop function and decrement the counter after executing the body. The counter is located at a fixed address 0x2000_0004. Invoke the stack unwinder at the end of a drop function if the counter is zero and the unwind pending flag located at 0x2000_0008 is set to true.
- Never tag `nounwind` attribute to any function during LLVM IR generation and prevent performing optimization when calling `nounwind` functions.
- Change the Rust target description for the LLVM target `thumbv6m-none-eabi`. It enables the `target_has_atomic` family feature gates. The `target_os` is hard coded to `"hopter"`. NOTE: the Rust `target_os` remains to be `"none"` for the LLVM target `thumbv7em-none-eabi`. In the modification to the `core` library, we need to distinguish between compiling for `thumbv6m` and `thumbv7em`. However, the Rust compiler currently has no feature gate that differentiates between these two architectures. Our patched code currently looks at the `target_os` value to tell the difference.

**Rust `core` Library**
- Strip away unnecessary fields in `PanicInfo` so that the panic handling infrastructure can have minimal code storage overhead.
- Provide atomic methods (e.g., `compare_exchange`, `fetch_add`) for atomic types on ARMv6M architectures.

# Build the Modified Compiler Toolchain

## Prerequisite

Make sure `gcc` and `g++` are available, and then install `cmake`, `pkg-config`. On ubuntu, also install `libssl`. On MacOS, also install `coreutils`.

On Ubuntu, run
```
sudo apt install cmake pkg-config libssl-dev
```

On MacOS, run
```
brew install cmake pkg-config coreutils
```

Finally, make sure `rustup` is available. If not yet, run the following command.
```
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

## Patches

Clone the patches in this repo.
```
https://github.com/hopter-project/hopter-compiler-toolchain.git
```

In the following, we call the cloned patches directory `$PATCHES_DIR`.


## LLVM

Clone Rust's fork of LLVM.
```
git clone git@github.com:rust-lang/llvm-project.git
```

In the following, we call the cloned LLVM source code directory `$LLVM_SRC_DIR`.

Enter the cloned directory.
```
cd llvm-project
```

Checkout the specific branch.
```
git checkout rustc/18.1-2024-05-19
```

Apply the patches from `$PATCHES_DIR/llvm`. If there are multiple patches, apply them *in number sequence* by running the command once for each patch.
```
git am < $PATCHES_DIR/llvm/XXXXX.patch
```

Configure compilation. Substitute `$LLVM_INSTALL_DIR` with your preferred LLVM installation path.
```
cmake -S llvm -B build -G "Unix Makefiles" \
    -DCMAKE_INSTALL_PREFIX=$LLVM_INSTALL_DIR \
    -DCMAKE_BUILD_TYPE=Release \
    -DLLVM_ENABLE_PROJECTS="clang;clang-tools-extra;compiler-rt;lld" \
    -DCOMPILER_RT_BUILD_BUILTINS=ON \
    -DCOMPILER_RT_DEFAULT_TARGET_ONLY=OFF \
    -DLLVM_TARGETS_TO_BUILD="AArch64;ARM;X86" \
    -DLLVM_ENABLE_ZSTD=OFF
```

Compile LLVM.
```
cd build

# If running out of memory, set a smaller number after -j.
make -j$(nproc)
```

Install LLVM.
```
make install
```

Get out of LLVM directory.
```
cd ../..
```

## Rust

Clone rustc.
```
git clone git@github.com:rust-lang/rust.git
```

Enter the cloned directory.
```
cd rust
```

Clone submodules.
```
git submodule update --init --recursive
```

Checkout the specific branch.
```
git checkout 1.81.0
```

Apply the patches from `$PATCHES_DIR/rust`. If there are multiple patches, apply them *in number sequence* by running the command once for each patch.
```
git am < $PATCHES_DIR/rust/XXXXX.patch
```

Create or edit a file with name `config.toml` to include the following. Substitute `$LLVM_INSTALL_DIR` with your LLVM installation path. Substitute `$LLVM_SRC_DIR` with the directory where the the LLVM repo is cloned into. Substitute `$RUST_INSTALL_DIR` with the directory you want rustc to be installed into.

```
profile = "user"

[build]
extended = true

[install]
prefix = "$RUST_INSTALL_DIR"
sysconfdir = "etc"

[target.aarch64-apple-darwin]
llvm-config = "$LLVM_INSTALL_DIR/bin/llvm-config"
llvm-filecheck = "$LLVM_SRC_DIR/build/utils/FileCheck"

[target.x86_64-unknown-linux-gnu]
llvm-config = "$LLVM_INSTALL_DIR/bin/llvm-config"
llvm-filecheck = "$LLVM_SRC_DIR/build/utils/FileCheck"

[target.thumbv6m-none-eabi]
llvm-config = "$LLVM_INSTALL_DIR/bin/llvm-config"
llvm-filecheck = "$LLVM_SRC_DIR/build/utils/FileCheck"

[target.thumbv7em-none-eabihf]
llvm-config = "$LLVM_INSTALL_DIR/bin/llvm-config"
llvm-filecheck = "$LLVM_SRC_DIR/build/utils/FileCheck"
```

Compile Rust.
```
./x.py build
```

Install Rust.
```
./x.py install
```

## Rust `core` Library

Enter the directory containing the Rust core libray within the installed Rust compiler toolchain.
```
cd $RUST_INSTALL_DIR/lib/rustlib/src/rust/library/
```

Backup the original `core` library source code.
```
cp -r core core_backup
```

Enter the `core` library.
```
cd core
```

Apply the patch.
```
patch -p1 < $PATCHES_DIR/rust-core/core_diff.patch
```

## Register the Toolchain

Register the compiled Rust compiler toolchain with `rustup`. Substitute `$RUST_INSTALL_DIR` with the directory rustc is installed in.
```
rustup toolchain link segstk-rust $RUST_INSTALL_DIR
```

If later one wants to remove the registration, run the following command. Do *not* run it for now.
```
rustup toolchain uninstall segstk-rust
```

Now we have finished all patching and installation. Below are troubleshooting sections.

# Troubleshooting

## MacOS Execution Error with Prebuilt Toolchain

Go to "Settings", find the "Privacy & Security" tab, scroll down to the bottom, and give the permission in the "Security" section. Usually this need to be done a few times, one for each executable. The problem occurs only when the toolchain is prebuilt and downloaded from the internet, triggering MacOS unsigned executable protection.

## MacOS Rust Installation Error

```
# Run the following commands and try again with ./x.py install.

cd /etc
sudo mkdir bash_completion.d
sudo chmod o=rwx bash_completion.d
```
