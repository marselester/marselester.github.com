title: Zig SIMD in assembly
date: 2026-07-14
tags: assembler, zig, simd
category: Zig
slug: zig-simd

In [the C ABI](https://marselester.com/zig-c-abi.html) post we saw how
convenient it's to write Intel-style assembly code in Zig.
This time let's see what it takes to use SIMD in assembly.

## Assembly side

[The first SIMD example](https://github.com/Apress/modern-x86-assembly-language-programming-3e/blob/main/Chapter08/Ch08_01/Ch08_01_fasm.s)
in Modern x86 Assembly book demonstrates 16-bit integers addition
using both wraparound and saturated arithmetic.

```console
[10,   200,     30, -32766,  50,   60, 32000,  -32000] xmm0
                                                    +
[100, -200,  32760,   -400, 500, -600,  1200,    -950] xmm1
                                                    =
[110,    0, -32746,  32370, 550, -540, -32336,  32586] xmm2 (wraparound)
[110,    0,  32767, -32768, 550, -540,  32767, -32768] xmm3 (saturated)
   0     1       2       3    4     5       6       7  lane
```

Wraparound is what we normally expect to happen (arithmetic overflow/underflow):

- lanes 0, 1, 4, and 5 stayed within boundaries
- lanes 2 and 6 overflowed the maximum capacity of an `i16` (32,767)
- lanes 3 and 7 underflowed past the minimum capacity (-32,768)

With saturated arithmetic, the results are clipped to the maximum capacity:

- lanes 2 and 6 are capped at 32,767
- lanes 3 and 7 are capped at -32,768

Here is an assembly code that LLVM understands
[llvm/ch08_01.s](https://github.com/marselester/misc/blob/main/zig-c-abi/llvm/ch08_01.s).
As we can see, a C++ call [AddI16_avx(&c1, &c2, &a, &b)](https://github.com/Apress/modern-x86-assembly-language-programming-3e/blob/main/Chapter08/Ch08_01/Ch08_01.cpp)
of the assembly function supplies 4 pointers:

- `&a` and `&b` are the inputs
- `&c1` and `&c2` are the outputs

In essence, the function sums the input vectors with wraparound/saturated arithmetic
and stores the resulting vectors in `&c1` and `&c2` respectively.

```asm
.intel_syntax noprefix
.text
.global _AddI16_avx

# Calculates *c1 = *a + *b (wraparound), *c2 = *a + *b (saturated).
# AddI16_avx(c1: *XmmVal, c2: *XmmVal, a: *const XmmVal, b: *const XmmVal) void;
_AddI16_avx:
    vmovdqa xmm0, [rdx]         # xmm0 = *a
    vmovdqa xmm1, [rcx]         # xmm1 = *b

    vpaddw xmm2, xmm0, xmm1     # xmm2 = xmm0 + xmm1 (wraparound)
    vpaddsw xmm3, xmm0, xmm1    # xmm3 = xmm0 + xmm1 (saturated)

    vmovdqa [rdi], xmm2         # *c1 = xmm2
    vmovdqa [rsi], xmm3         # *c2 = xmm3
    ret
```

Since those pointers are basically 64-bit unsigned integers holding memory addresses,
they are passed in `rdi` (c1), `rsi` (c2), `rdx` (a), `rcx` (b) registers in accordance with the C ABI.

The first instruction `vmovdqa xmm0, [rdx]` copies data from a memory address stored
in a general purpose register `rdx` (a) to a vector register `xmm0`.
The instruction name stands for **V**ector Extension (VEX) **MOV**e **D**ouble **Q**uadword **A**ligned.
The eight `i16` items (128 bits total or a double quadword)
`[10, 200, 30, -32766, 50, 60, 32000, -32000]` are copied from an aligned memory address stored in `rdx` to `xmm0`.

Aligned means starting at a memory address that is a multiple of the vector's size in bytes.
That means the variable `&a` (memory address) must be aligned to 16 bytes for XMM register
(128 bits / 8 bits = 16 bytes).
Note, there is `vmovdqu` instruction that works with unaligned addresses,
see [Intro to SIMD in avo](https://marselester.com/intro-to-simd-in-avo.html).

The second instruction `vmovdqa xmm1, [rcx]` is similar, i.e.,
it copies from a memory address `&b` held in `rcx` register to `xmm1`
like this `xmm1 = *rcx`, i.e., `xmm1 = [100, -200, 32760, -400, 500, -600, 1200, -950]`.

The third and fourth instructions `vpaddw xmm2, xmm0, xmm1` and `vpaddsw xmm3, xmm0, xmm1`
represent wraparound and saturated additions:

- **V**ector (VEX) **P**acked **ADD** **W**ord:
  16-bit elements of vectors `xmm0` and `xmm1` are added and the result is stored in
  `xmm2 = [110, 0, -32746, 32370, 550, -540, -32336, 32586]`
- **V**ector (VEX) **P**acked **ADD** **S**aturated **W**ord:
  the same 16-bit elements are added and the result is stored in
  `xmm3 = [110, 0, 32767, -32768, 550, -540, 32767, -32768]`

The last instructions `vmovdqa [rdi], xmm2` and `vmovdqa [rsi], xmm3`
copy the computed results from `xmm2`/`xmm3` vector registers
to memory addresses held in `rdi`/`rsi` registers (`&c1` and `&c2` variables on the caller side).

## C++ side

In the C++ example, [XmmVal](https://github.com/Apress/modern-x86-assembly-language-programming-3e/blob/main/Include/XmmVal.h)
type represents all sorts of 128-bit values we could place into XMM registers, e.g., four floats, eight 16-bit integers, etc.
It is an anonymous union defined within a struct,
allowing its members to be accessed directly as if they were part of the struct.
The `alignas(16)` specifier tells the compiler to place `XmmVal` variables/constants
at a memory address that is a multiple of 16 bytes (`vmovdqa` instruction requires this, we talked about it above).

```cpp
struct alignas(16) XmmVal {
    union {
        int8_t   m_I8[16];
        int16_t  m_I16[8];
        int32_t  m_I32[4];
        int64_t  m_I64[2];
        uint8_t  m_U8[16];
        uint16_t m_U16[8];
        uint32_t m_U32[4];
        uint64_t m_U64[2];
        float    m_F32[4];
        double   m_F64[2];
    };
};
```

Since we're dealing with 16-bit integers, we must use `int16_t m_I16[8]` field in our
[cpp/ch08_01.cpp](https://github.com/marselester/misc/blob/main/zig-c-abi/cpp/ch08_01.cpp) program.

```cpp
extern "C" void AddI16_avx(XmmVal* c1, XmmVal* c2, const XmmVal* a, const XmmVal* b);

int main() {
    XmmVal c1, c2;
    const XmmVal a = { .m_I16 = { 10, 200, 30, -32766, 50, 60, 32000, -32000 } };
    const XmmVal b = { .m_I16 = { 100, -200, 32760, -400, 500, -600, 1200, -950 } };

    AddI16_avx(&c1, &c2, &a, &b);
    // Print the results...

    return 0;
}
```

The results of wraparound `c1` and saturated `c2` addition match the expectations.

```console
﹩ zig c++ ./cpp/ch08_01.cpp ./llvm/ch08_01.s -o main && ./main
C++ says
c1={ 110, 0, -32746, 32370, 550, -540, -32336, 32586 }
c2={ 110, 0, 32767, -32768, 550, -540, 32767, -32768 }
```

🦉 Note, C++ has C-compatible struct layout as long as there are no virtual functions.
If `XmmVal` had one, C++ compiler would have injected a hidden pointer to a vtable at the beginning of the struct 😰:

- `vmovdqa xmm0, [rdx]` would read `_vptr` (8 bytes) and half of `m_I16` (8 bytes)
- `XmmVal` size would increase from 16 bytes to 32 bytes: 8 bytes pointer, 16 bytes of `m_I16`,
  and 8 bytes of trailing padding.
  The compiler adds 8 bytes to make sure the total struct size is a multiple of 16
  (24 bytes of `_vptr` and `m_I16` are not divisible by 16, but 32 is)
  in case `XmmVal` is used in an array, e.g., `XmmVal x[2];`.
- `vpaddw xmm2, xmm0, xmm1` would sum a garbage data
- `vmovdqa [rdi], xmm2` would produce a corrupted `c1`, e.g., its `_vptr` would be broken
  because we would copy `xmm2` there

```cpp
struct alignas(16) XmmVal {
    void*   _vptr;        // 8 bytes
    int16_t m_I16[8];     // 16 bytes
    char    __padding[8]; // 8 bytes
};
```

## Zig side

Zig code [zig/ch08_01.zig](https://github.com/marselester/misc/blob/main/zig-c-abi/zig/ch08_01.zig)
isn't very different from C++ one: we'got `XmmVal` struct wrapping a union,
it also has a 16-byte alignment requirement.

Note, `const XmmVal = extern struct { ... }` has `extern` keyword
that instructs Zig compiler to keep the stable memory layout.

```zig
const XmmVal = extern struct {
    data: extern union {
        m_I8: [16]i8,
        m_I16: [8]i16,
        m_I32: [4]i32,
        m_I64: [2]i64,
        m_U8: [16]u8,
        m_U16: [8]u16,
        m_U32: [4]u32,
        m_U64: [2]u64,
        m_F32: [4]f32,
        m_F64: [2]f64,
    } align(16),
};
```

We use `m_I16: [8]i16` field to set an array of 16-bit integers just like in the C++ example.

```zig
extern fn AddI16_avx(c1: *XmmVal, c2: *XmmVal, a: *const XmmVal, b: *const XmmVal) void;

pub fn main() void {
    const a = XmmVal{
        .data = .{
            .m_I16 = [8]i16{ 10, 200, 30, -32766, 50, 60, 32000, -32000 },
        },
    };
    const b = XmmVal{
        .data = .{
            .m_I16 = [8]i16{ 100, -200, 32760, -400, 500, -600, 1200, -950 },
        },
    };
    var c1 = XmmVal{ .data = .{ .m_I16 = undefined } };
    var c2 = XmmVal{ .data = .{ .m_I16 = undefined } };

    AddI16_avx(&c1, &c2, &a, &b);
    // Print the results...
}
```

The results match C++ output.

```console
﹩ zig run ./zig/ch08_01.zig ./llvm/ch08_01.s
Zig says
c1={ 110, 0, -32746, 32370, 550, -540, -32336, 32586 }
c2={ 110, 0, 32767, -32768, 550, -540, 32767, -32768 }
```

---

Even though the compilers can auto-vectorize code, it's good to know how to write SIMD in assembly.
Calling such functions from C++/Zig isn't always straightforward due to alignment requirements.
