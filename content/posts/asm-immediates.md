title: Writing immediates to memory
date: 2026-09-02
tags: assembler, zig
category: Zig
slug: asm-immediates

An immediate in assembly is a scalar value such as `123`.
While the concept is familiar from high-level languages, immediates require more attention in x86-64 assembly.
There are no floating-point immediates, so everything below is about integers.

## Memory access

Writing `123` immediate value to a memory address like so `mov [rdi], 123` can catch us off guard.

```console
﹩ cat > demo.s <<ASM
.intel_syntax noprefix
.text
.global _demo
_demo:
    mov [rdi], 123
    ret
ASM

﹩ zig build-obj demo.s
error(compilation): clang failed with stderr: demo.s:5:9: error: ambiguous operand size for instruction 'mov'
    mov [rdi], 123
        ^~~~
```

The LLVM assembler behind `zig build-obj` tells us that our `mov` instruction has an ambiguous operand size
(if the example seems confusing, try my earlier post on [using Assembly with Zig](https://marselester.com/zig-c-abi.html)).
Indeed, the assembler is not sure how many bytes to use to represent `123` in memory.
It can use a byte, a word, a double word, and so on.
Interestingly, NASM v3.02 defaults to using a byte size and truncates larger values with a warning.

```console
﹩ cat > demo.asm <<ASM
section .text
global _demo
_demo:
    mov [rdi], 123
    ret
ASM

﹩ nasm -f macho64 demo.asm
﹩ objdump -d --x86-asm-syntax=intel demo.o
       0: c6 07 7b                     	mov	byte ptr [rdi], 0x7b
       3: c3                           	ret
```

Let's see how our `mov [rdi], 123` is different from disassembled `mov byte ptr [rdi], 0x7b`:

- `byte ptr` in front of `[rdi]` means to treat the memory address in register `rdi`
  as a pointer to a byte size integer, e.g., `i8` or `u8`
- `0x7b` is `123` in hex

The LLVM assembler has no problems building the object file `demo.o` once we add `byte ptr`.
The `objdump` confirms that `zig build-obj` emits the same machine code for the `_demo()` function as NASM.

```console
﹩ cat > demo.s <<ASM
.intel_syntax noprefix
.text
.global _demo
_demo:
    mov byte ptr [rdi], 123
    ret
ASM

﹩ zig build-obj demo.s
﹩ objdump -d --x86-asm-syntax=intel demo.o
       0: c6 07 7b                     	mov	byte ptr [rdi], 0x7b
       3: c3                           	ret
```

We've got the function in `demo.o` file, let's make sure it actually works.
Here is a Zig program that declares a byte size variable `var v: i8 = 0`
and passes its pointer when calling `demo(&v)`.
The assembly function `_demo()` writes `123` to that dereferenced pointer and returns.
Finally, we print the variable `v` and its raw memory with `std.mem.asBytes(&v)`.
If you're unsure how it all works, see [The C ABI: Using Assembly with Zig](https://marselester.com/zig-c-abi.html) post.

```console
﹩ cat > main.zig <<ZIG
const std = @import("std");

extern fn demo(v: *i8) void;

pub fn main() void {
    var v: i8 = 0;
    demo(&v);
    std.debug.print("{d} is stored as 0x{x} at {*}\n", .{
        v,
        std.mem.asBytes(&v),
        &v,
    });
}
ZIG

﹩ zig build-exe main.zig demo.s
﹩ ./main
123 is stored as 0x7b at i8@7ff7b00a2000
```

As expected, the program's output shows that `123` was stored as `0x7b`.
It also printed the memory address `0x7ff7b00a2000` where the value was placed.
What if we wanted to use 4 bytes of memory instead of just one byte?
We would need to replace `byte` with `dword` in the `mov` instruction like this `mov dword ptr [rdi], 123`,
and update `main.zig` to use `i32` instead of `i8`.

<details style="margin-bottom: 1rem">

<summary>🔻 <code>mov dword ptr [rdi], 123</code></summary>

```console
﹩ cat > demo.s <<ASM
.intel_syntax noprefix
.text
.global _demo
_demo:
    mov dword ptr [rdi], 123
    ret
ASM

﹩ cat > main.zig <<ZIG
const std = @import("std");

extern fn demo(v: *i32) void;

pub fn main() void {
    var v: i32 = 0;
    demo(&v);
    std.debug.print("{d} is stored as 0x{x} at {*}\n", .{
        v,
        std.mem.asBytes(&v),
        &v,
    });
}
ZIG

﹩ zig build-exe main.zig demo.s
﹩ ./main
123 is stored as 0x7b000000 at i32@7ff7b4fe9000
```

</details>

The updated program reported that it stored `0x7b` as `0x7b000000` (not `0x0000007b`) at `0x7ff7b4fe9000` address.
This is because `123` immediate's bytes `00 00 00 7b` are arranged in little-endian order `7b 00 00 00`:
the least significant byte `7b` is placed at the smallest memory address `7ff7b4fe9000`.

```console
   Address     Byte
7ff7b4fe9003    00  ← most significant byte
7ff7b4fe9002    00
7ff7b4fe9001    00
7ff7b4fe9000    7b  ← least significant byte
```

## Two's complement

What if we store a negative immediate value `mov byte ptr [rdi], -123`?

<details style="margin-bottom: 1rem">

<summary>🔻 <code>mov byte ptr [rdi], -123</code></summary>

```console
﹩ cat > demo.s <<ASM
.intel_syntax noprefix
.text
.global _demo
_demo:
    mov byte ptr [rdi], -123
    ret
ASM

﹩ cat > main.zig <<ZIG
const std = @import("std");

extern fn demo(v: *i8) void;

pub fn main() void {
    var v: i8 = 0;
    demo(&v);
    std.debug.print("{d} is stored as 0x{x} at {*}\n", .{
        v,
        std.mem.asBytes(&v),
        &v,
    });
}
ZIG

﹩ zig build-exe main.zig demo.s
﹩ ./main
-123 is stored as 0x85 at i8@7ff7b8653000
```

</details>

The `-123` value got stored as `0x85` in memory.
This is due to [two's complement](https://en.wikipedia.org/wiki/Two%27s_complement)
method of representing a signed integer (invert all bits and add one):

- `0111 1011 = 0x7b` -- `123` in binary format
- `1000 0100 = 0x84` -- invert all bits of `123`
- `1000 0101 = 0x85` -- add one

The Python one liner below confirms it.

```console
﹩ python3 -c 'print("0x7b = {0:0>8b}\n0x85 = {1:0>8b}".format(0x7b, 0x85))'
0x7b = 01111011
0x85 = 10000101
```

If we store `-123` as four bytes `mov dword ptr [rdi], -123`, we'll get `0x85ffffff` memory representation.

<details style="margin-bottom: 1rem">

<summary>🔻 <code>mov dword ptr [rdi], -123</code></summary>

```console
﹩ cat > demo.s <<ASM
.intel_syntax noprefix
.text
.global _demo
_demo:
    mov dword ptr [rdi], -123
    ret
ASM

﹩ cat > main.zig <<ZIG
const std = @import("std");

extern fn demo(v: *i32) void;

pub fn main() void {
    var v: i32 = 0;
    demo(&v);
    std.debug.print("{d} is stored as 0x{x} at {*}\n", .{
        v,
        std.mem.asBytes(&v),
        &v,
    });
}
ZIG

﹩ zig build-exe main.zig demo.s
﹩ ./main
-123 is stored as 0x85ffffff at i32@7ff7b2e8a00c
```

</details>

Converting `0x85ffffff` to a big-endian order gives us `0xffffff85`:

- `0000 0000 0000 0000 0000 0000 0111 1011 = 0x7b` -- `123` in binary format
- `1111 1111 1111 1111 1111 1111 1000 0100 = 0xffffff84` -- invert all bits of `123`
- `1111 1111 1111 1111 1111 1111 1000 0101 = 0xffffff85` -- add one

```console
﹩ python3 -c 'print("0xffffff85 = {0:0>32b}".format(0xffffff85))'
0xffffff85 = 11111111111111111111111110000101
```

## Sign extension

How about storing a positive number `2,147,483,648` (2^31) as eight bytes `mov qword ptr [rdi], 2147483648`?

```console
﹩ cat > demo.s <<ASM
.intel_syntax noprefix
.text
.global _demo
_demo:
    mov qword ptr [rdi], 2147483648
    ret
ASM

﹩ cat > main.zig <<ZIG
const std = @import("std");

extern fn demo(v: *i64) void;

pub fn main() void {
    var v: i64 = 0;
    demo(&v);
    std.debug.print("{d} is stored as 0x{x} at {*}\n", .{
        v,
        std.mem.asBytes(&v),
        &v,
    });
}
ZIG

﹩ zig build-exe main.zig demo.s
error(compilation): clang failed with stderr: demo.s:5:5: error: invalid operand for instruction
    mov qword ptr [rdi], 2147483648
    ^
```

Zig tooling straight up refuses to build with `invalid operand for instruction` error,
and NASM only warns `signed dword exceeds bounds`.

```console
﹩ cat > demo.asm <<ASM
section .text
global _demo
_demo:
    mov qword [rdi], 2147483648
    ret
ASM

﹩ nasm -f macho64 demo.asm
demo.asm:4: warning: signed dword exceeds bounds [-w+number-overflow]

﹩ zig build-exe main.zig demo.o
﹩ ./main
-2147483648 is stored as 0x00000080ffffffff at i64@7ff7bf1ff008
```

The program's output is surprising `-2147483648 is stored as 0x00000080ffffffff`:

- `-2147483648` stored value is negative
- `0x00000080ffffffff` memory representation has four `0xff` bytes
  whereas we expect four zeros `0x0000008000000000`

Let's transform `0x00000080ffffffff` to a big-endian order `0xffffffff80000000`:

- `0000 0000 0000 0000 0000 0000 0000 0000 1000 0000 0000 0000 0000 0000 0000 0000 = 00 00 00 00 80 00 00 00` -- `2147483648` in binary format
- `1111 1111 1111 1111 1111 1111 1111 1111 0111 1111 1111 1111 1111 1111 1111 1111 = ff ff ff ff 7f ff ff ff` -- invert all bits of `2147483648`
- `1111 1111 1111 1111 1111 1111 1111 1111 1000 0000 0000 0000 0000 0000 0000 0000 = ff ff ff ff 80 00 00 00` -- add one

```console
﹩ python3 -c 'print("0xffffffff80000000 = {0:0>64b}".format(0xffffffff80000000))'
0xffffffff80000000 = 1111111111111111111111111111111110000000000000000000000000000000
```

Let's look at the machine code NASM produced.

```console
﹩ objdump -d --x86-asm-syntax=intel demo.o
       0: 48 c7 07 00 00 00 80         	mov	qword ptr [rdi], -0x80000000
       7: c3                           	ret
```

The machine code shows that despite our 8-byte memory destination,
the immediate `00 00 00 80` (in little-endian form) is only 4 bytes.
The `mov` instruction in x86-64 can't write a 64-bit immediate to memory,
so the CPU sign-extends those 4 bytes to fill the 8.
Our `2147483648` is `0x80000000` and its bit 31 is set, so the CPU fills the upper 32 bits with ones
and writes `0xffffffff80000000` which is `-2147483648`.

The solution is to write to a register first, and then to memory.

```console
﹩ cat > demo.asm <<ASM
section .text
global _demo
_demo:
    mov rax, 2147483648
    mov [rdi], rax
    ret
ASM

﹩ nasm -f macho64 demo.asm

﹩ zig build-exe main.zig demo.o
﹩ ./main
2147483648 is stored as 0x0000008000000000 at i64@7ff7b8227008
```

Note that no size specifier is needed now because the register `rax` already tells the assembler
that the store is eight bytes wide.

Only the register form of `mov` accepts a 64-bit immediate, though NASM doesn't even need it here.

```console
﹩ objdump -d --x86-asm-syntax=intel demo.o
       0: b8 00 00 00 80               	mov	eax, 0x80000000
       5: 48 89 07                     	mov	qword ptr [rdi], rax
       8: c3                           	ret
```

NASM replaced our `mov rax, 2147483648` with `mov eax, 0x80000000`
which puts `0x0000008000000000` into `rax` because writing to a 32-bit register `eax`
zero-extends (not sign-extends!) into the full 64-bit register `rax`.
Note that this is specific to 32-bit writes: `ax` and `al` leave the upper bits of `rax` untouched.

The same 4-byte immediate limit also applies to `add`, `cmp`, and the other instructions.
Anything larger has to go through a register.

---

References:

- Modern x86 Assembly Language Programming by Daniel Kusswurm
- [Two's complement](https://en.wikipedia.org/wiki/Two%27s_complement)
