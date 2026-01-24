title: Go archsimd preview
date: 2026-01-23
tags: assembler, golang, simd
category: Go
slug: go-archsimd-preview

In [the previous post](https://marselester.com/intro-to-simd-in-avo.html)
we've implemented SIMD sum in Go assembly `[1, 2, 3, 4] + [5, 6, 7, 8]`.
This is going to be much easier to do in Go 1.26 because of
[simd/archsimd](https://pkg.go.dev/simd/archsimd) package,
see [#73787](https://github.com/golang/go/issues/73787) proposal.
So far the package provides access to amd64-specific SIMD operations.

Let's give it a go and implement the same `func SumVec(input []int64) int64`
keeping it close to an already familiar assembly code
[sumv3/sum.s](https://github.com/marselester/misc/blob/main/simd/go/intro/sumv3/sum.s).

```go
//go:noinline
func SumVec(input []int64) (sum int64) {
	i := 0
	inputLen := len(input)

	// If we can't use two YMM vectors, fallback to a scalar sum.
	// Otherwise keep adding YMM vectors in the vector loop.
	if inputLen >= 8 {
		y0 := archsimd.LoadInt64x4Slice(input)
		loopEnd := inputLen - inputLen%4
		for i += 4; i < loopEnd; i += 4 {
			y1 := archsimd.LoadInt64x4Slice(input[i : i+4])
			y0 = y0.Add(y1)
		}

		// Horizontal reduction.
		x0 := y0.GetHi()
		x1 := y0.GetLo()
		x0 = x0.Add(x1)
		sum = x0.GetElem(0) + x0.GetElem(1)
	}

	// Summarize what we couldn't with SIMD.
	for ; i < inputLen; i++ {
		sum += input[i]
	}

	return sum
}
```

The most interesting part is of course the SIMD operations, i.e.,
everything in `if inputLen >= 8 { ... }` branch:

- `y0 := archsimd.LoadInt64x4Slice(input)` loads the first four `int64`s from the `input []int64` slice,
  e.g., `1, 2, 3, 4`.
  Our 256-bit SIMD vector `y0` is represented by [Int64x4](https://pkg.go.dev/simd/archsimd#Int64x4) type.
- `loopEnd := inputLen - inputLen%4` calculates the slice index beyoud which we mustn't iterate, e.g.,
  if a slice length is `9`, the `loopEnd` will be `8 = 9 - (9 % 4)`,
  so `y1` vector register is always fully filled with 4 integers on each iteration.
- `y1 := archsimd.LoadInt64x4Slice(input[i : i+4])` loads a next batch of 4 integers
  into `y1` 256-bit SIMD register, e.g., `y1 = [5, 6, 7, 8]`.
- `y0 = y0.Add(y1)` adds corresponding elements of two vectors, e.g.,
  `y0 = y0 + y1 = [1, 2, 3, 4] + [5, 6, 7, 8] = [6, 8, 10, 12]`.
- `x0 := y0.GetHi()` returns the upper half of `y0` register, e.g., `[10, 12]`,
  represented by 128-bit SIMD register `x0`, see [Int64x2](https://pkg.go.dev/simd/archsimd#Int64x2).
- `x1 := y0.GetLo()` returns the lower half of register `y0 = [6, 8, 10, 12]`, e.g., `[6,  8]`.
  It sounds a bit confusing that the lower half (right side) isn't `[10, 12]`.
  The thing is that those 4 numbers are stored in the register in "reverse order", e.g., `[12, 10, 8, 6]`.
- `x0 = x0.Add(x1)` adds corresponding elements of two XMM registers, e.g.,
  `x0 = x0 + x1 = [10, 12] + [6, 8] = [16, 20]`
- `sum = x0.GetElem(0) + x0.GetElem(1)` wraps up the horizontal reduction summation
  by adding two scalars `16 + 20`. They were retrieved using `VPEXTRQ` instructions.

I am curious to see what assembly instructions were used by the compiler.
Go 1.26 RC2 can give us a sneak peek.

```console
﹩ go install golang.org/dl/go1.26rc2@latest
﹩ go1.26rc2 download
﹩ GOEXPERIMENT=simd go1.26rc2 build -gcflags -S ./sumv4/
```

<details>

<summary>🔻 TEXT intro/sumv4.SumVec</summary>

```asm
00000	TEXT intro/sumv4.SumVec(SB), NOSPLIT|ABIInternal, $8-24
00000	PUSHQ	BP
00001	MOVQ	SP, BP
00004	MOVQ	AX, intro/sumv4.input+16(FP)
00009	FUNCDATA	$0, gclocals·wvjpxkknJ4nY1JtrArJJaw==(SB)
00009	FUNCDATA	$1, gclocals·J26BEvPExEQhJvjp9E8Whg==(SB)
00009	FUNCDATA	$5, intro/sumv4.SumVec.arginfo1(SB)
00009	FUNCDATA	$6, intro/sumv4.SumVec.argliveinfo(SB)
00009	PCDATA	$3, $1
00009	CMPQ	BX, $8
00013	JLT		39
00015	MOVQ	BX, DX
00018	ANDL	$3, BX
00021	XCHGL	AX, AX
00022	MOVQ	DX, SI
00025	SUBQ	BX, DX
00028	VMOVDQU	(AX), Y0
00032	MOVL	$4, BX
00037	JMP		87
00039	XORL	CX, CX
00041	XORL	DX, DX
00043	JMP		52
00045	ADDQ	(AX)(CX*8), DX
00049	INCQ	CX
00052	CMPQ	BX, CX
00055	JLE		61
00057	JHI		45
00059	JMP		66
00061	MOVQ	DX, AX
00064	POPQ	BP
00065	RET
00066	PCDATA	$1, $1
00066	PCDATA	$4, $3591
00066	CALL	runtime.panicBounds(SB)
00071	LEAQ	(AX)(BX*8), R8
00075	VMOVDQU	(R8), Y1
00080	NOP
00080	VPADDQ	Y1, Y0, Y0
00084	MOVQ	DI, BX
00087	CMPQ	BX, DX
00090	JGE		108
00092	LEAQ	4(BX), DI
00096	CMPQ	CX, DI
00099	JCS		153
00101	CMPQ	BX, DI
00104	JLS		71
00106	JMP		148
00108	VEXTRACTI128	$1, Y0, X1
00114	VEXTRACTI128	$0, Y0, X0
00120	VPADDQ	X0, X1, X0
00124	VPEXTRQ	$0, X0, DI
00130	VPEXTRQ	$1, X0, R8
00136	LEAQ	(R8)(DI*1), DX
00140	MOVQ	BX, CX
00143	MOVQ	SI, BX
00146	JMP		52
00148	PCDATA	$4, $8346
00148	CALL	runtime.panicBounds(SB)
00153	PCDATA	$4, $1721
00153	CALL	runtime.panicBounds(SB)
00158	XCHGL	AX, AX
```

</details>

---

Looking at the beginning of the function, we don't see instructions that
determine the memory address of the `input`'s underlying array `input_base+0(FP)`
and the slice length `input_len+8(FP)`.

```asm
MOVQ input_base+0(FP), AX
MOVQ input_len+8(FP), CX
```

It's because Go ensures that all three parts of the slice header
(array pointer, length, and capacity) are in the registers `AX`, `BX`, `CX` respectively
before calling the `sumv4.SumVec` function.
Since we've implemented [sumv3.SumVec](https://github.com/marselester/misc/blob/main/simd/go/intro/sumv3/sum.s)
in assembly, the compiler defaulted to using the stack to pass the arguments.

That explains why `CMPQ BX, $8` compares the slice length `BX` to `8`, see `if inputLen >= 8`.
If the length is less than `8`, we jump directly to the scalar loop using `JLT 39` instruction, see `00039` address.

```asm
00009   CMPQ    BX, $8			; if inputLen >= 8
00013   JLT     39
 ...
00039	XORL	CX, CX			; i = 0
00041	XORL	DX, DX			; sum = 0
00043	JMP		52
00045	ADDQ	(AX)(CX*8), DX	; sum += input[i]
00049	INCQ	CX				; i++
00052	CMPQ	BX, CX			; i < inputLen
00055	JLE		61
00057	JHI		45
00059	JMP		66
00061	MOVQ	DX, AX			; Moves the sum into the return register AX.
```

Otherwise, prepare the vector loop as follows.

```asm
00015	MOVQ	BX, DX		; DX = inputLen
00018	ANDL	$3, BX		; BX = inputLen % 4
00021	XCHGL	AX, AX
00022	MOVQ	DX, SI		; SI = DX = inputLen
00025	SUBQ	BX, DX		; loopEnd = DX - BX = inputLen - inputLen%4
00028	VMOVDQU	(AX), Y0	; y0 = archsimd.LoadInt64x4Slice(input) = [1, 2, 3, 4]
00032	MOVL	$4, BX		; i = 4
00037	JMP		87
```

To be continued...
