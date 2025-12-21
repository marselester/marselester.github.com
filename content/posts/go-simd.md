title: Intro to SIMD in avo
date: 2025-12-21
tags: assembler, golang, simd
category: Go
slug: intro-to-simd-in-avo

In the previous post we wrote a [Hello World in avo](https://marselester.com/hello-world-in-avo.html).
Let's do something practical this time, e.g., related to performance
since we go into all this trouble of writing Go assembly.
You can find the code examples in
[github.com/marselester/misc](https://github.com/marselester/misc/tree/main/simd/go/intro).

Processing more data in a single CPU instruction makes our programs faster.
That's what SIMD (Single Instruction Multiple Data) technique is for.
The caveat is that we need to think in terms of vectors, not scalars.
For example, let's say we want to find a sum of eight 64-bit integers.
Our options look as follows:

1. sum of scalars: `1 + 2 + 3 + 4 + 5 + 6 + 7 + 8`
2. sum of vectors: `[1, 2, 3, 4] + [5, 6, 7, 8]` or `[1, 2] + [3, 4] + [5, 6] + [7, 8]`

The first option is straightforward.

```go
func Sum(input []int64) int64 {
	var sum int64
	for _, v := range input {
		sum += v
	}

	return sum
}
```

The second one — not so much 😬.
At least my CPU (Intel i5-10600) supports [AVX2](https://en.wikipedia.org/wiki/Advanced_Vector_Extensions#Advanced_Vector_Extensions_2),
meaning it can execute 256-bit SIMD instructions.
That's exactly enough to add our vectors `[1, 2, 3, 4]` and `[5, 6, 7, 8]` with just a single CPU instruction.

The plan is to add the 4-element vectors, then keep folding the resulting vectors adding their halfes,
see the calculations below.

```console
[1,  2,  3,  4]       [6,   8]       [16, 20]
             +              +              +
[5,  6,  7,  8]   ➡   [10, 12]   ➡   [0,  16]
             =              =              =
[6,  8, 10, 12]       [16, 20]       [16, 36]
                                          🏁
```

With this in mind, let's implement it in Go assembly!

## Adding vectors

We can start small and just focus on adding 8 numbers.
The first step is to create a dummy function `SumVec` and a corresponding test.
It always returns zero no matter the input it gets.
Note, we used `asm.XORQ(sum, sum)` to set the register associated with `sum` variable to zero.
We'll see `Q` postfix quite often later on, it stands for quad word (8 bytes) on amd64.

```go
//go:build ignore

package main

import asm "github.com/mmcloughlin/avo/build"

//go:generate go run asm.go -out sum.s -stubs sum.go

func main() {
	asm.TEXT("SumVec", asm.NOSPLIT, "func(input []int64) int64")
	sum := asm.GP64()
	asm.XORQ(sum, sum)
	asm.Store(sum, asm.ReturnIndex(0))
	asm.RET()

	asm.Generate()
}
```

<details>

<summary>sum_test.go</summary>

```go
package sum

import "testing"

func TestSumVec(t *testing.T) {
	input := []int64{1, 2, 3, 4, 5, 6, 7, 8}

	var want int64 = 36
	if got := SumVec(input); got != want {
		t.Fatalf("expected %d got %d", want, got)
	}
}
```

</details>

Not surprisingly, the test fails as it expects the sum to be `36`.

```console
﹩ go generate ./sum/asm.go && go test ./sum
--- FAIL: TestSumVec (0.00s)
    sum_test.go:10: expected 36 got 0
```

The second step is to learn the `input []int64` slice length and
where its backing array is located in memory,
so we could load its elements into a vector register.
When the function is called, a three-field `slice` structure is passed on the stack.

```go
type slice struct {
	array unsafe.Pointer
	len   int
	cap   int
}
```

Its fields can be accessed in assembler as follows:

- `input_base+0(FP)` pointer to the underlying array (the base memory address)
- `input_len+8(FP)` length of the slice
- `input_cap+16(FP)` capacity of the slice

[avo API](https://github.com/mmcloughlin/avo/blob/master/examples/args/README.md) is very similar,
here is how we can load the array pointer and the length
into general-purpose registers `AX` and `CX` assigned by `avo`:

```go
inputData := asm.GP64() // Base pointer of the slice is in AX.
inputLen := asm.GP64()  // Number of elements in the slice is in CX.
// MOVQ input_base+0(FP), AX
asm.Load(asm.Param("input").Base(), inputData)
// MOVQ input_len+8(FP), CX
asm.Load(asm.Param("input").Len(), inputLen)
```

The third step is to load the left half of the array into a vector register.

```go
vecLeft := asm.YMM() // 256-bit vector register Y0.
// VMOVDQU (AX), Y0
asm.VMOVDQU(operand.Mem{Base: inputData}, vecLeft)
```

Examining a generated Go assembly, we'll see `VMOVDQU (AX), Y0` instruction:

- `VMOVDQU` stands for **V**ector **MOV**e **D**ouble **Q**uadword **U**naligned.
  It copies the `[1, 2, 3, 4]` elements from a possibly unaligned
  memory address stored in `AX` to vector register `Y0`.
  Unaligned means not starting at a memory address that is a multiple of the vector's size.
  We don't use `VMOVDQA` (the aligned version) since we don't know
  if the array's address is aligned to 256.

  Despite its "double quadword" (128-bit vector) naming, the instruction is capable of moving 256 bits.
- `(AX)` operand means use address from register `AX`.
  Its `avo` equivalent is `operand.Mem{Base: inputData}`.
- `Y0` operand is a 256-bit vector register allocated by `vecLeft := asm.YMM()`

🦉 Since we mentioned vectors of different sizes, let's name them for reference:

- 512-bit ZMM registers: `Z0` ... `Z31` for AVX-512 (not our case)
- 256-bit YMM registers: `Y0` ... `Y15` for AVX and `Y31` for AVX-512
- 128-bit XMM registers: `X0` ... `X15` for AVX and `X31` for AVX-512

Moving on to the fourth step — loading the right half of the array into another vector register.
The important part is to determine the memory address from which to copy four 64-bit integers.
As we can see from the diagram below, we need to start at the array index `4`.
We can deduce the address of element `5` like this
`inputData + index * int64InBytes = 0xc000054760 + 4 * 8` assuming the array is stored at `0xc000054760`.

```console
        0xc000054760
		⬇️
array: [1, 2, 3, 4, 5, 6, 7, 8]
index:  0  1  2  3  4  5  6  7
                    ⬆️
                    0xc000054760 + 4 * 8
```

The assembly code looks similar to what we saw in the previous step:

- `MOVQ` copies 64 bits of a literal value `0x00000004` (our index `4` represented as 32-bit unsigned integer)
   to `CX` register.
- `VMOVDQU` copies 256 bits starting from memory address defined by operand `(AX)(CX*8)` to vector register `Y1`.
  The operand `(AX)(CX*8)` reads as `AX + CX * 8`, i.e., take memory address stored in `AX` register
  (`0xc000054760` in our example), then add it to a product of value stored in `CX` register (`$0x00000004`)
  and a scaling factor `8` since the array contains 64-bit integers.

```asm
MOVQ    $0x00000004, CX
VMOVDQU (AX)(CX*8), Y1
```

The assembler DSL is a little bit verbose, but it provides type safety.
For instance, it makes sure we pass a valid immediate value when setting up the index to `4`
(`asm.MOVQ()` docs indicate `imm32` and `imm64`) as the first operand in `asm.MOVQ(operand.U32(4), index)`.
Note, `operand.U64(4)` would also work.

```go
index := asm.GP64() // The array index is stored in register CX.
// MOVQ $0x00000004, CX
asm.MOVQ(operand.U32(4), index)

vecRight := asm.YMM() // 256-bit vector register Y1.
// VMOVDQU (AX)(CX*8), Y1
asm.VMOVDQU(
	operand.Mem{
		Base:  inputData, // Array starts at 0xc000054760 address.
		Index: index,     // Array index is 4.
		Scale: 8,         // The multiplier of the index is 8 bytes (int64).
	},
	vecRight,
)
```

Now we've got both vectors filled, we can finally add them up!
It's done with `VPADDQ Y0, Y1, Y0` instruction which reads
as **V**ector **P**acked **ADD** **Q**uadword, i.e., 64-bit elements of vectors `Y0` and `Y1` are added
and the result is stored in `Y0`.
"Packed" signifies that the instruction operates on all the elements packed within the register,
i.e., it is not a scalar operation.

```go
// VPADDQ Y0, Y1, Y0
asm.VPADDQ(vecLeft, vecRight, vecLeft)
```

Now `Y0` contains `[6, 8, 10, 12]`.

## Adding half-vectors

We summarize the `Y0 = [6, 8, 10, 12]` vector by adding its halfes `[6, 8]` and `[10, 12]`.
That's called a horizontal reduction summation.

```console
[6,   8]
      +
[10, 12]
      =
[16, 20]
```

To do that, we can copy its left half (bits 128-255) to a 128-bit XMM vector register `X1`
using `VEXTRACTI128` (**V**ector **E**xtract **I**nteger **128**-bit) instruction.

```console
Y0 = [6, 8, 10, 12]
      ⬇️ ⬇️
X1 = [6, 8]
```

The first operand `$0x01` in `VEXTRACTI128 $0x01, Y0, X1` is a control byte
that refers to extracting the upper 128-bit lane.
The second operand is the source YMM register (`vecLeft` in our `avo` program),
and the third one is an XMM register (we use `vecRight.AsX()`
which is the lower portion of `vecRight` register).

```go
vecRightLow := vecRight.AsX()
// VEXTRACTI128 $0x01, Y0, X1
asm.VEXTRACTI128(operand.U8(1), vecLeft, vecRightLow)
```

Since `X0` represents the right half of `Y0`, we can add `X0` and `X1`
which by now contains the left half of `Y0`.

```console
Y0 = [6, 8, 10, 12]
      ⬇️ ⬇️ [10, 12] = X0
X1 = [6, 8]
```

Go code looks familiar.

```go
vecLeftLow := vecLeft.AsX()
// VPADDQ X0, X1, X0
asm.VPADDQ(vecLeftLow, vecRightLow, vecLeftLow)
```

At this point `X0` contains `[16, 20]`.
Our goal is to line up `16` with `20` to get our scalar result `36`.
We can shift `16` right by 8 bytes since we're dealing with 64-bit integers.

```console
Before: [16, 20]
         ➡️
After:  [    16] 20
```

The `VPSRLDQ $0x08, X0, X1` instruction does that, i.e., it shifts `X0` bits right,
fills the empty space with zeros, and stores the result in `X1`.
The addition instruction is the same `VPADDQ X0, X1, X0`.

```console
[16, 20]    X0
      +
[0,  16]    X1
      =
[16, 36]    X0
     🏁
```

Here is an `avo` code.

```go
// VPSRLDQ $0x08, X0, X1
asm.VPSRLDQ(operand.U8(8), vecLeftLow, vecRightLow)
// VPADDQ X0, X1, X0
asm.VPADDQ(vecLeftLow, vecRightLow, vecLeftLow)
```

That's it, we got out final result `36` in the `X0 = [16, 36]` vector.
We just need to somehow return it from the `SumVec` function 🤔.

The cool thing about `VMOVQ` instruction is that it can copy the lower quad word
(our `36` value) from a vector to a scalar register like this `VMOVQ X0, AX`.
Note, `VMOVQ Y0, AX` wouldn't work since a YMM operand isn't supported.

These are the final lines of Go code that generate Go assembly.
It's pretty neat that `AX` was reused by `avo` to store the `sum`.

```go
sum := asm.GP64()
// VMOVQ X0, AX
asm.VMOVQ(vecLeftLow, sum)

// MOVQ AX, ret+24(FP)
asm.Store(sum, asm.ReturnIndex(0))
// RET
asm.RET()
```

This time the tests should pass.

```console
﹩ go generate ./sum/asm.go && go test ./sum
ok  	myprog/sum	0.289s
```

## Working with larger arrays

Coming soon...

References:

- [avo docs and examples](https://github.com/mmcloughlin/avo) by Michael McLoughlin
- [From slow to SIMD: A Go optimization story](https://sourcegraph.com/blog/slow-to-simd) by Camden Cheek
- [Advanced Vector Extensions](https://en.wikipedia.org/wiki/Advanced_Vector_Extensions)
