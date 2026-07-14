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
	// Otherwise, keep adding YMM vectors in the vector loop.
	if inputLen >= 8 {
		y0 := archsimd.LoadInt64x4Slice(input)
		loopEnd := inputLen - inputLen%4

		for i += 4; i < loopEnd; i += 4 {
			y1 := archsimd.LoadInt64x4Slice(input[i : i+4])
			y0 = y0.Add(y1)
		}

		// Horizontal reduction.
		x0, x1 := y0.GetLo(), y0.GetHi()
		x0 = x0.Add(x1)
		sum = x0.GetElem(0) + x0.GetElem(1)
	}

	// Scalar loop summarizes what we couldn't with SIMD.
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
- `loopEnd := inputLen - inputLen%4` calculates the slice index beyond which we mustn't iterate, e.g.,
  if a slice length is `9`, the `loopEnd` will be `8 = 9 - (9 % 4)`,
  so `y1` vector register is always fully filled with 4 integers on each iteration.
- `y1 := archsimd.LoadInt64x4Slice(input[i : i+4])` loads a next batch of 4 integers
  into `y1` 256-bit SIMD register, e.g., `y1 = [5, 6, 7, 8]`.
- `y0 = y0.Add(y1)` adds corresponding elements of two vectors, e.g.,
  `y0 = y0 + y1 = [1, 2, 3, 4] + [5, 6, 7, 8] = [6, 8, 10, 12]`.
- `x0 := y0.GetLo()` returns the lower half of register `y0 = [6, 8, 10, 12]`, e.g., `[6, 8]`.
  It sounds a bit confusing that the lower half (right side) isn't `[10, 12]`.
  The thing is that those 4 numbers are stored in the register in "reverse order", e.g., `[12, 10, 8, 6]`,
  but if we print it `fmt.Println(y0)`, Go would display `[6, 8, 10, 12]`.
- `x1 := y0.GetHi()` returns the upper half of `y0` register, e.g., `[10, 12]`.
  It's represented by 128-bit SIMD vector `x1`, see [Int64x2](https://pkg.go.dev/simd/archsimd#Int64x2).
- `x0 = x0.Add(x1)` adds corresponding elements of two XMM registers, e.g.,
  `x0 = x0 + x1 = [6, 8] + [10, 12] = [16, 20]`
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
00009   CMPQ    BX, $8		; if inputLen >= 8
00013   JLT     39			; Jump to the scalar loop, i.e., "00039 XORL" line
```

Otherwise, prepare the vector loop as follows:

- calculate the `loopEnd`
- load the first 4 elements from the `input` slice
- set the slice index `i = 4` and jump to the vector loop,
  i.e., an instruction responsible for `i < loopEnd` comparison

```asm
00015	MOVQ	BX, DX		; DX = inputLen
00018	ANDL	$3, BX		; BX = inputLen % 4
00021	XCHGL	AX, AX
00022	MOVQ	DX, SI		; SI = DX = inputLen
00025	SUBQ	BX, DX		; loopEnd = DX - BX = inputLen - inputLen%4
00028	VMOVDQU	(AX), Y0	; y0 = archsimd.LoadInt64x4Slice(input) = [1, 2, 3, 4]
00032	MOVL	$4, BX		; i = 4
00037	JMP		87			; Jump to the vector loop, i.e., "00087 CMPQ" line
```

Here is the vector loop.
In a hand-written assembly we used `VMOVDQU (AX)(BX*8), Y1` instruction whereas here we see two:

- `LEAQ (AX)(BX*8), R8` loads effective address of the next chunk of four `int64`s into register `R8`.
  It performs a memory address calculation as follows `AX + BX * 8` where `AX` holds a memory address
  of the underlying array, `BX` is an array index, and `8` represents `int64` type size in bytes.
- `VMOVDQU (R8), Y1` copies 256 bits from a memory address stored in `R8` to `Y1` YMM register.

The vectors addition uses the same `VPADDQ Y1, Y0, Y0`,
but there are way more instructions than we had in `sumv3`.
From what I understand, most of them are related to the slice bounds checks that
the compiler inserted for our own safety:

- `LEAQ 4(BX), DI` calculates a look-ahead index `i+4` and stores it in register `DI`.
  As we can see, `LEAQ` can be used in lieu of `ADDQ` (we used `ADDQ $0x00000004, BX` in `sumv3`),
  not only for memory address calculations.
- that look-ahead index `i+4` is checked against the slice boundaries.
  If it's ok, the index is updated `MOVQ DI, BX` or else we'll get "index out of range" panic,
  see `runtime.panicBounds(SB)`.

```asm
00071	LEAQ	(AX)(BX*8), R8	; R8 = AX + BX*8 = inputData + i*8
00075	VMOVDQU	(R8), Y1		; y1 := archsimd.LoadInt64x4Slice(input[i : i+4])
00080	NOP
00080	VPADDQ	Y1, Y0, Y0		; y0 = y0.Add(y1)
00084	MOVQ	DI, BX			; i += 4
00087	CMPQ	BX, DX			; i < loopEnd
00090	JGE		108				; Jump to the horizontal reduction, i.e., "00108 VEXTRACTI128" line
00092	LEAQ	4(BX), DI		; DI = 4 + BX = 4 + i
00096	CMPQ	CX, DI			; inputCap < DI
00099	JCS		153				; Go to line 153 (index out of range)
00101	CMPQ	BX, DI			; i < DI
00104	JLS		71				; Jump to the next iteration.
00106	JMP		148				; Otherwise, go to line 148 ("index out of range" error)
 ...
00148	CALL	runtime.panicBounds(SB)
 ...
00153	CALL	runtime.panicBounds(SB)
```

The horizontal reduction has a different code as well:

- we used register `X0` to refer to `Y0`'s lower part in `sumv3` assembly,
  but here we used `VEXTRACTI128 $0, Y0, X0` because
  I couldn't find a better way in `archsimd` docs
- `VPSRLDQ $0x08, X0, X1` was used to shift `X0`'s bits to the right by 8 bytes
  to line up `16` with `20` in XMM registers,
  so we could add them up `VPADDQ X0, X1, X0` and get the final result `36`.
  I couldn't figure out how to shift the bytes using `archsimd` package,
  so I ended up with `VPEXTRQ` (see `GetElem`) to get both scalars from `X0` register,
  and then added them, see `LEAQ (R8)(DI*1), DX` below.

```console
    sumv4                sumv3
X0 = [8,   6]        X0 = [8,   6]
           +
X1 = [12, 10]        X1 = [12, 10]
           =                    =
X0 = [20, 16]        X0 = [20, 16]
                                +
DI = 16              X1 = [0,  20]
      +                         =
R8 = 20              X0 = [20, 36]
      =
DX = 36              DX = 36
```

```asm
00108	VEXTRACTI128	$1, Y0, X1	; x1 := y0.GetHi()
00114	VEXTRACTI128	$0, Y0, X0	; x0 := y0.GetLo()
00120	VPADDQ	X0, X1, X0			; x0 = x0.Add(x1)
00124	VPEXTRQ	$0, X0, DI			; DI = x0.GetElem(0)
00130	VPEXTRQ	$1, X0, R8			; R8 = x0.GetElem(1)
00136	LEAQ	(R8)(DI*1), DX		; sum = R8 + DI*1 = 20 + 16*1 = 36
00140	MOVQ	BX, CX				; i = loopEnd
00143	MOVQ	SI, BX				; Restore inputLen from SI
00146	JMP		52					; Jump to the scalar loop, i.e., "00052 CMPQ" line
```

The final section of the `SumVec` function is the scalar loop.
As expected, it also contains the boundaries checks.

```asm
00039	XORL	CX, CX			; i = 0
00041	XORL	DX, DX			; sum = 0
00043	JMP		52				; Jump to line 52 to check the loop condition
00045	ADDQ	(AX)(CX*8), DX	; sum += input[i]
00049	INCQ	CX				; i++
00052	CMPQ	BX, CX			; i < inputLen
00055	JLE		61				; Exit the loop
00057	JHI		45				; Go to the next iteration
00059	JMP		66				; Go to line 66 (index out of range)
00061	MOVQ	DX, AX			; Moves the sum into the return register AX
 ...
00066	CALL	runtime.panicBounds(SB)
```

## Bounds-checking elimination

It would be great to convince the Go compiler that our code safely accesses
the `input` slice and index `i` can't be out of range.
This will improve the performance since the CPU won't need to execute extra
compare-and-branch instructions in the hot loop introduced by the boundaries checks.

The bounds check was eliminated from the scalar loop as follows.
Effectively it was moved outside the loop, i.e., at `tail := input[i:]` line.

```go
tail := input[i:]
for _, v := range tail {
	sum += v
}
```

I've tried to apply a similar approach to the vector loop, but the checks stayed in place,
so I resorted to `unsafe` package to get rid of them:

- `unsafe.Pointer` allows us to read arbitrary memory.
  In this case `unsafe.Pointer(&input[0])` points to first element of the underlying array.
- `unsafe.Add(inputData, i*8)` calculates the memory address of the `i`-th array element
  (beginning of the next chunk of numbers), i.e., `inputData + i*8`
  where `8` is the size of `int64` in bytes.
  Basically the `Add` function returns a new `unsafe.Pointer` that is `i*8` bytes higher in memory
  than `inputData` pointer.
- `(*[4]int64)` converts our `unsafe.Pointer` to `*[4]int64` type, i.e.,
  a pointer to the next chunk of numbers `[4]int64`
- `archsimd` package provides `LoadInt64x4(y *[4]int64) Int64x4` function that
  loads our chunk from the array

```go
y0 := archsimd.LoadInt64x4Slice(input)
loopEnd := inputLen - inputLen%4
inputData := unsafe.Pointer(&input[0])

for i += 4; i < loopEnd; i += 4 {
	chunk := (*[4]int64)(unsafe.Add(inputData, i*8))
	y1 := archsimd.LoadInt64x4(chunk)
	y0 = y0.Add(y1)
}
```

If you're wondering why not take a pointer `&input[i]` directly in the loop,
you'll find that the bounds check is back due to a slice access.

```go
y0 := archsimd.LoadInt64x4Slice(input)
loopEnd := inputLen - inputLen%4

for i += 4; i < loopEnd; i += 4 {
	// Bounds check is back 😭!
	chunk := (*[4]int64)(unsafe.Pointer(&input[i]))
	y1 := archsimd.LoadInt64x4(chunk)
	y0 = y0.Add(y1)
}
```

Here is the updated `sumv5.SumVec`.

<details>

<summary>🔻 sumv5.SumVec</summary>

```go
//go:noinline
func SumVec(input []int64) (sum int64) {
	i := 0
	inputLen := len(input)

	// If we can't use two YMM vectors, fallback to a scalar sum.
	// Otherwise, keep adding YMM vectors in the vector loop.
	if inputLen >= 8 {
		y0 := archsimd.LoadInt64x4Slice(input)
		loopEnd := inputLen - inputLen%4
		inputData := unsafe.Pointer(&input[0])

		for i += 4; i < loopEnd; i += 4 {
			chunk := (*[4]int64)(unsafe.Add(inputData, i*8))
			y1 := archsimd.LoadInt64x4(chunk)
			y0 = y0.Add(y1)
		}

		// Horizontal reduction.
		x0, x1 := y0.GetLo(), y0.GetHi()
		x0 = x0.Add(x1)
		sum = x0.GetElem(0) + x0.GetElem(1)
	}

	// Scalar loop summarizes what we couldn't with SIMD.
	tail := input[i:]
	for _, v := range tail {
		sum += v
	}

	return sum
}
```

</details>

---

Looking at the `sumv5` assembly code, there is only one `runtime.panicBounds(SB)` call
that is reachable from a single jump `JCS 97` line.
The instructions above `JCS 97` set up the scalar loop.
This means the bounds checks were eliminated! 🎉

```console
﹩ GOEXPERIMENT=simd go1.26rc2 build -gcflags -S ./sumv5/
```

<details>

<summary>🔻 TEXT intro/sumv5.SumVec</summary>

```asm
00000	TEXT	intro/sumv5.SumVec(SB), NOSPLIT|ABIInternal, $8-24
00000	PUSHQ	BP
00001	MOVQ	SP, BP
00004	MOVQ	AX, intro/sumv5.input+16(FP)
00009	FUNCDATA	$0, gclocals·wvjpxkknJ4nY1JtrArJJaw==(SB)
00009	FUNCDATA	$1, gclocals·J26BEvPExEQhJvjp9E8Whg==(SB)
00009	FUNCDATA	$5, intro/sumv5.SumVec.arginfo1(SB)
00009	FUNCDATA	$6, intro/sumv5.SumVec.argliveinfo(SB)
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
00037	JMP		118
00039	XORL	DX, DX
00041	XORL	SI, SI
00043	CMPQ	BX, DX
00046	JCS		97
00048	MOVQ	DX, DI
00051	SUBQ	CX, DX
00054	MOVQ	DI, CX
00057	SHLQ	$3, DI
00061	SARQ	$63, DX
00065	ANDQ	DI, DX
00068	SUBQ	CX, BX
00071	LEAQ	(AX)(DX*1), CX
00075	XORL	AX, AX
00077	JMP		86
00079	ADDQ	(CX)(AX*8), SI
00083	INCQ	AX
00086	CMPQ	AX, BX
00089	JLT		79
00091	MOVQ	SI, AX
00094	POPQ	BP
00095	NOP
00096	RET
00097	PCDATA	$1, $1
00097	PCDATA	$4, $3666
00097	CALL	runtime.panicBounds(SB)
00102	LEAQ	(AX)(BX*8), DI
00106	VMOVDQU	(DI), Y1
00110	VPADDQ	Y1, Y0, Y0
00114	ADDQ	$4, BX
00118	CMPQ	BX, DX
00121	JLT		102
00123	VEXTRACTI128	$0, Y0, X1
00129	VEXTRACTI128	$1, Y0, X0
00135	VPADDQ	X0, X1, X0
00139	VPEXTRQ	$0, X0, DI
00145	VPEXTRQ	$1, X0, R8
00151	ADDQ	R8, DI
00154	MOVQ	BX, DX
00157	MOVQ	SI, BX
00160	MOVQ	DI, SI
00163	JMP		43
```

</details>

---

The SIMD sum with bounds checks is ~47.57% faster than the scalar sum.

```console
﹩ GOEXPERIMENT=simd go1.26rc2 test -bench=^ -count=10 ./sumv4 | tee bench.txt
﹩ grep Benchmark bench.txt | sed 's/Benchmark[A-z]*/BenchmarkSum/g' | split -l 10 -a 1 - bench_
﹩ benchstat bench_a bench_b
name    old time/op  new time/op  delta
Sum-12  38.4µs ± 1%  20.1µs ± 3%  -47.57%  (p=0.000 n=10+9)
```

<details>

<summary>🔻 sumv4 benchmarks</summary>

```
goos: darwin
goarch: amd64
pkg: intro/sumv4
cpu: Intel(R) Core(TM) i5-10600 CPU @ 3.30GHz
BenchmarkSum-12       	   30892	     38535 ns/op
BenchmarkSum-12       	   30442	     38727 ns/op
BenchmarkSum-12       	   31269	     38349 ns/op
BenchmarkSum-12       	   31398	     38198 ns/op
BenchmarkSum-12       	   31405	     38337 ns/op
BenchmarkSum-12       	   30933	     38360 ns/op
BenchmarkSum-12       	   31378	     38753 ns/op
BenchmarkSum-12       	   31352	     38087 ns/op
BenchmarkSum-12       	   31072	     38390 ns/op
BenchmarkSum-12       	   31502	     38200 ns/op
BenchmarkSumVec-12    	   61132	     19711 ns/op
BenchmarkSumVec-12    	   61512	     19587 ns/op
BenchmarkSumVec-12    	   64653	     20022 ns/op
BenchmarkSumVec-12    	   58118	     20151 ns/op
BenchmarkSumVec-12    	   57780	     21672 ns/op
BenchmarkSumVec-12    	   53917	     20405 ns/op
BenchmarkSumVec-12    	   59610	     20685 ns/op
BenchmarkSumVec-12    	   55443	     20430 ns/op
BenchmarkSumVec-12    	   54528	     20238 ns/op
BenchmarkSumVec-12    	   55696	     19937 ns/op
PASS
ok  	intro/sumv4	24.203s
```

</details>

---

Removing those branches makes it ~54.69% faster than the scalar sum
which is similar to `sumv3`'s 54.23% (manually written assembly).
It looks like the bounds checks elimination accounted for ~14% speed-up (from 20.1µs to 17.3µs).

```console
﹩ GOEXPERIMENT=simd go1.26rc2 test -bench=^ -count=10 ./sumv5 | tee bench.txt
﹩ grep Benchmark bench.txt | sed 's/Benchmark[A-z]*/BenchmarkSum/g' | split -l 10 -a 1 - bench_
﹩ benchstat bench_a bench_b
name    old time/op  new time/op  delta
Sum-12  38.2µs ± 1%  17.3µs ± 4%  -54.69%  (p=0.000 n=9+10)
```

<details>

<summary>🔻 sumv5 benchmarks</summary>

```
goos: darwin
goarch: amd64
pkg: intro/sumv5
cpu: Intel(R) Core(TM) i5-10600 CPU @ 3.30GHz
BenchmarkSum-12       	   31548	     38370 ns/op
BenchmarkSum-12       	   31383	     38033 ns/op
BenchmarkSum-12       	   31339	     38117 ns/op
BenchmarkSum-12       	   31322	     37882 ns/op
BenchmarkSum-12       	   30944	     38196 ns/op
BenchmarkSum-12       	   31176	     38420 ns/op
BenchmarkSum-12       	   31063	     38860 ns/op
BenchmarkSum-12       	   31437	     38087 ns/op
BenchmarkSum-12       	   31302	     38235 ns/op
BenchmarkSum-12       	   31333	     38323 ns/op
BenchmarkSumVec-12    	   66858	     17200 ns/op
BenchmarkSumVec-12    	   76434	     17156 ns/op
BenchmarkSumVec-12    	   71926	     17628 ns/op
BenchmarkSumVec-12    	   71235	     17908 ns/op
BenchmarkSumVec-12    	   70063	     17424 ns/op
BenchmarkSumVec-12    	   71061	     16646 ns/op
BenchmarkSumVec-12    	   74750	     16936 ns/op
BenchmarkSumVec-12    	   68294	     17638 ns/op
BenchmarkSumVec-12    	   70453	     17550 ns/op
BenchmarkSumVec-12    	   68662	     16926 ns/op
PASS
ok  	intro/sumv5	24.647s
```

</details>

---

I hope you liked this post, I certainly learned a lot writing it 🙂.
Note, the provided examples are for sure not the pinacle of performance,
one could achieve better results with loop unrolling.

References:

- [https://pkg.go.dev/simd/archsimd](https://pkg.go.dev/simd/archsimd)
- [https://github.com/golang/go/issues/73787](https://github.com/golang/go/issues/73787) by @cherrymui
- [DotAVX256a](https://go.dev/play/p/ZaruM_PzP1X) by @cherrymui
- [dotGoSIMD](https://go.dev/play/p/NY5rJYPoJcl) by @cherrymui
