title: The C ABI: Using Assembly with Zig
date: 2026-05-14
tags: assembler, zig
category: Zig
slug: zig-c-abi

I am currently reading [Modern x86 Assembly Language Programming](https://www.goodreads.com/book/show/156500234-modern-x86-assembly-language-programming) by Daniel Kusswurm.
The book provides examples in C++, NASM, and MASM,
but since I'm more interested in Zig, I’ve decided to follow along using it as my primary language.

As we'll see in this post, the C ABI (Application Binary Interface)
is the cornerstone of language interoperability.
You can find the code on [GitHub](https://github.com/marselester/misc/tree/main/zig-c-abi).

## C++ example

The first example in the book shows us how to write an assembly function `AddSubI32_a`
and call it on C++ side.
Here is a NASM implementation `nasm/ch02_01.asm`.

```asm
section .text
global AddSubI32_a

; Calculates (a + b) - (c + d) + 7.
AddSubI32_a:
    add edi, esi    ; edi = edi + esi = a + b
    add edx, ecx    ; edx = edx + ecx = c + d
    sub edi, edx    ; edi = edi - edx
    add edi, 7      ; edi = edi + 7
    mov eax, edi    ; eax = edi
    ret             ; return to caller
```

The `global` assembly directive makes the `AddSubI32_a` symbol visible to the C++ linker.
This is how the function is declared and called in the C++ program `cpp/main.cpp`.

```cpp
#include <iostream>

// Calculates (a + b) - (c + d) + 7.
extern "C" int AddSubI32_a(int a, int b, int c, int d);

int main() {
    int r = AddSubI32_a(1, 2, 3, 4);
    std::cout << "C++ says " << r << std::endl;
    return 0;
}
```

Let's assemble the NASM file to create the object file `ch02_01.o`:

- `nasm -f macho64` produces Mach-O x86-64 file.
  If you're on Linux, you would need `elf64` format, see `nasm -hf`.
- `nm ch02_01.o` shows a symbol table of the object file `ch02_01.o`.
  The capital `T` (text section symbol) in `T AddSubI32_a` tells us that the function is global
  (local symbols are lower case `t`).

```console
﹩ nasm -f macho64 ./nasm/ch02_01.asm -o ch02_01.o
﹩ nm ch02_01.o
0000000000000000 T AddSubI32_a
```

The `extern "C"` keyword instructs the C++ compiler to adhere to the C ABI
when dealing with this function, e.g.,
use C-style naming (no [name mangling](https://en.wikipedia.org/wiki/Name_mangling))
so the same `AddSubI32_a` symbol is used consistently in C++ and assembly.
Let's compile the C++ file into an object file `main.o` without linking it yet:

- the capital `U` in `U _AddSubI32_a` stands for **U**ndefined global symbol, i.e.,
  the `main.o` knows that the function exists but doesn't know where its machine code is
- `T _main` says that the `_main` symbol is global and present in the text section of this object file

```console
﹩ g++ -c ./cpp/main.cpp -o main.o
﹩ nm main.o
000000000000019c s GCC_except_table3
                 U _AddSubI32_a
...
0000000000000000 T _main
```

The linker should produce the executable file `main` since it can resolve
the undefined symbol from `main.o` to the one defined in `ch02_01.o`.

```console
﹩ g++ main.o ch02_01.o -o main
Undefined symbols for architecture x86_64:
  "_AddSubI32_a", referenced from:
      _main in main.o
ld: symbol(s) not found for architecture x86_64
```

Unfortunately, it failed to link because the symbols are different: `U _AddSubI32_a` vs `T AddSubI32_a`.
The `main.o` file expects the symbol to have an underscore on macOS, i.e., `_AddSubI32_a`.
Once the function is renamed from `AddSubI32_a` to `_AddSubI32_a` in `nasm/ch02_01.asm`
we finally get our program working:

```console
﹩ nasm -f macho64 ./nasm/ch02_01.asm -o ch02_01.o
﹩ nm ch02_01.o
0000000000000000 T _AddSubI32_a
﹩ g++ main.o ch02_01.o -o main
﹩ ./main
C++ says 3
```

Note, you don't need to add an underscore if you're on Linux.

## Zig example

Zig can compile C and C++ code for different architectures and operating systems
because it comes with Clang/LLVM embedded inside it.
Here is how we can link `main.o` and `ch02_01.o` to get the executable file `main`.

```console
﹩ zig c++ main.o ch02_01.o -o main
```

These examples compile C, C++ code and link with `ch02_01.o` object file that we built with NASM.

```console
﹩ zig cc ./c/main.c ch02_01.o -o main
﹩ zig c++ ./cpp/main.cpp ch02_01.o -o main
```

Of course we can write a Zig equivalent of C/C++ program, see `zig/main.zig`.
Just like C/C++, Zig supports `extern` and `callconv(.c)` keywords
to use the C ABI when calling `AddSubI32_a()`.
The `extern` defaults to the C calling convention, so we can actually omit `callconv(.c)` keyword.

```zig
const std = @import("std");

// Calculates (a + b) - (c + d) + 7.
extern fn AddSubI32_a(a: i32, b: i32, c: i32, d: i32) callconv(.c) i32;

pub fn main() void {
    const r = AddSubI32_a(1, 2, 3, 4);
    std.debug.print("Zig says {d}\n", .{r});
}
```

As we can see, it's compiled and linked with `ch02_01.o` no problem.

```console
﹩ zig build-exe ./zig/main.zig ch02_01.o --name main
﹩ ./main
Zig says 3
```

Since Zig includes Clang/LLVM, we can use it to build everything (including our assembly code).

```console
﹩ zig build-exe ./zig/main.zig ./llvm/ch02_01_att.s --name main
```

The caveat is that we have to use LLVM Assembler which mimics GAS (GNU Assembler),
check out `llvm/ch02_01_att.s` example.

```asm
.text
.global _AddSubI32_a

# Calculates (a + b) - (c + d) + 7.
_AddSubI32_a:
    addl %esi, %edi    # edi = edi + esi
    addl %ecx, %edx    # edx = edx + ecx
    subl %edx, %edi    # edi = edi - edx
    addl $7, %edi      # edi = edi + 7
    movl %edi, %eax    # eax = edi
    ret                # return to caller
```

GAS prefers `.s` file extension instead of `.asm`, and you've probably noticed that
the assembly code looks quite different from NASM:

- directives start with a dot (`.global` vs `global`)
- comments start with `#` instead of `;`
- GAS uses AT&T-style syntax, so operand order is reversed,
  registers have `%` prefix, instructions have size suffixes (`l` in `addl` means long),
  and scalars have `$` prefixes, e.g., `addl $7, %edi` (AT&T) vs `add edi, 7` (Intel)

The `.intel_syntax noprefix` directive helps us keep the code changes minimal.
It tells the LLVM Assembler to use Intel-style syntax like in NASM, for example `llvm/ch02_01_intel.s`:

```asm
.intel_syntax noprefix
.text
.global _AddSubI32_a

# Calculates (a + b) - (c + d) + 7.
_AddSubI32_a:
    add edi, esi    # edi = edi + esi = a + b
    add edx, ecx    # edx = edx + ecx = c + d
    sub edi, edx    # edi = edi - edx
    add edi, 7      # edi = edi + 7
    mov eax, edi    # eax = edi
    ret             # return to caller
```

Let's build our Zig program with that Intel-style assembly code,
but this time we'll leverage `build.zig`.

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    const exe = b.addExecutable(.{
        .name = "main",
        .root_module = b.createModule(.{
            .root_source_file = b.path("zig/main.zig"),
            .target = target,
            .optimize = optimize,
        }),
    });

    exe.root_module.addAssemblyFile(b.path("llvm/ch02_01_intel.s"));

    b.installArtifact(exe);
}
```

The `zig build` will make the `main` program just like `zig build-exe` did.

```console
﹩ zig build
﹩ ./zig-out/bin/main
Zig says 3
```

## C ABI

We instructed C++ and Zig to adhere to the C ABI when calling `AddSubI32_a()` function.
But what does it mean for the assembly side that implements it?

The C ABI defines a set of calling conventions and data layouts specific
to a given CPU architecture and operating system.
This implies that the assembly code for `AddSubI32_a(a: i32, b: i32, c: i32, d: i32) i32`
will look different on Linux x86-64 vs Windows x86-64.

Here is NASM and MASM code to quickly demonstrate **System V AMD64 ABI** (Linux/macOS)
and **Microsoft x64 ABI** (Windows) basics:

- on Linux/macOS we pass the `a`, `b`, `c`, `d` 32-bit arguments in `edi`, `esi`, `edx`, `ecx` registers,
  and return the 32-bit integer value from `eax` register
- on Windows we use `ecx`, `edx`, `r8d`, `r9d` instead

```asm
; NASM
AddSubI32_a:
    add edi, esi    ; edi = edi + esi = a + b
    add edx, ecx    ; edx = edx + ecx = c + d
    sub edi, edx    ; edi = edi - edx
    add edi, 7      ; edi = edi + 7
    mov eax, edi    ; eax = edi
    ret             ; return to caller

; MASM
AddSubI32_a proc
    add ecx, edx    ; ecx = ecx + edx = a + b
    add r8d, r9d    ; r8d = r8d + r9d = c + d
    sub ecx, r8d    ; ecx = ecx - r8d
    add ecx, 7      ; ecx = ecx + 7
    mov eax, ecx    ; eax = ecx
    ret             ; return to caller
AddSubI32_a endp
```

Since I am writing this post on macOS (Intel i5-10600), I'll stick to System V AMD64 ABI.
[Wikipedia](https://en.wikipedia.org/wiki/X86_calling_conventions) outlines it as follows:

- integer arguments are passed in `rdi`, `rsi`, `rdx`, `rcx`, `r8`, and `r9`.
  The 8, 16, and 32-bit integers are passed in the low-order bits (the high-order bits are undefined).
- floating point arguments go in `xmm0`-`xmm7` (bits `[31..0]` for float, `[63..0]` for double,
  all other bits are undefined)
- the return value go in `rax`/`rdx` for integer, and `xmm0`/`xmm1` for floating point.
  If a return value is larger than 16 bytes, the caller allocates a return struct on its stack.
  That struct address is passed in `rdi` register, so the first argument has to go in `rsi`.
  Upon returning the `rax` must contain the memory address it got from `rdi` register.
  For example, `rdi=&r`, `rsi=a`, `rdx=b` when a function is called as follows `r = f(a, b)`.
- additional arguments are pushed onto the stack in reverse order
- for variadic calls such as `printf`, the `al` register must be set
  to the number of used `xmm` registers, set it zero if none are used
- caller-saved (volatile, can be used freely): `rax`, `rcx`, `rdx`, `rdi`, `rsi`, `r8`-`r11`
- callee-saved (non-volatile, the function must restore them): `rbx`, `rsp`, `rbp`, `r12`-`r15`
- bits `[255..128]` of registers `ymm0`-`ymm15` are volatile,
- registers `zmm16`-`zmm31` are volatile
- `RFLAGS.DF` (direction flag for string instructions)
  and `MXCSR.RC` (rounding mode control flag) are non-volatile.
  If `RFLAGS.DF` is set, it must be zeroed before a function call or return.
  If `MXCSR.RC` is modified, the original rounding mode must be restored.
- the stack must be aligned to a 16-byte boundary before calling a function,
  i.e., `rsp` must be a multiple of 16
- leaf functions can use 128-byte area (the red zone) below `rsp` for temporary data
  without moving the stack pointer (this works only in user-space)
- struct members must be aligned to addresses that are multiples of their size,
  and a struct's total size must be a multiple of its largest member's alignment

| register | volatile? | usage
| ---      | ---       | ---
| rax      | yes       | 1st int return value
| rbx      | **no**    | long-lived local variables
| rcx      | yes       | 4th int argument
| rdx      | yes       | 3rd int argument, 2nd int return value
| rsi      | yes       | 2nd int argument
| rdi      | yes       | 1st int argument
| rbp      | **no**    | stack frame pointer or scratch
| rsp      | **no**    | stack pointer
| r8       | yes       | 5th int argument
| r9       | yes       | 6th int argument
| r10      | yes       | scratch (temporary)
| r11      | yes       | scratch
| r12-15   | **no**    | long-lived local variables
| xmm0     | yes       | 1st float argument, return value
| xmm1     | yes       | 2nd float argument, return value
| xmm2     | yes       | 3rd float argument
| xmm3     | yes       | 4th float argument
| xmm4     | yes       | 5th float argument
| xmm5     | yes       | 6th float argument
| xmm6     | yes       | 7th float argument
| xmm7     | yes       | 8th float argument
| xmm8-15  | yes       | scratch
| zmm16-31 | yes       | scratch

If we follow the aforementioned rules when writing an assembly code (or any code really),
we can use our function with any language as long as it supports this calling convention.
For instance, `AddSubI32_a` can be implemented in Zig and called from C++ using the C ABI.
We just need to use the `export` keyword instead of `extern`, see `zig/ch02_01.zig`.

```zig
export fn AddSubI32_a(a: i32, b: i32, c: i32, d: i32) callconv(.c) i32 {
    return (a + b) - (c + d) + 7;
}
```

Let's build it and use in the C++ program.

```console
﹩ zig build-obj zig/ch02_01.zig -O ReleaseFast -femit-bin=ch02_01.o
﹩ nm ch02_01.o
0000000000000000 T _AddSubI32_a
0000000000000000 t _ch02_01.AddSubI32_a
﹩ g++ ./cpp/main.cpp ch02_01.o -o main
﹩ ./main
C++ says 3
```

The static library can also be created with the following `build.zig`.

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    const lib = b.addLibrary(.{
        .linkage = .static,
        .name = "ch02_01",
        .root_module = b.createModule(.{
            .root_source_file = b.path("zig/ch02_01.zig"),
            .target = target,
            .optimize = optimize,
        }),
    });

    b.installArtifact(lib);
}
```

It produces `zig-out/lib/libch02_01.a` archive that contains an object file `libch02_01_zcu.o`.

```console
﹩ zig build -Doptimize=ReleaseFast
﹩ nm zig-out/lib/libch02_01.a
libch02_01_zcu.o:
0000000000000000 T _AddSubI32_a
0000000000000000 t _ch02_01.AddSubI32_a
﹩ g++ ./cpp/main.cpp ./zig-out/lib/libch02_01.a -o main
﹩ ./main
C++ says 3
```

---

To sum up, we have successfully compiled the Zig and assembly code into a static library
and linked it across C, C++, and Zig.
It all worked seamlessly thanks to the C ABI!

References:

- Modern x86 Assembly Language Programming by Daniel Kusswurm
- [wiki/Name_mangling](https://en.wikipedia.org/wiki/Name_mangling)
- [wiki/X86_calling_conventions](https://en.wikipedia.org/wiki/X86_calling_conventions)
