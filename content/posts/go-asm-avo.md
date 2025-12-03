title: Hello World in avo 🥑
date: 2025-12-02
tags: assembler, golang
category: Go
slug: hello-world-in-avo

Let's learn together how to write some Go assembly using [avo](https://github.com/mmcloughlin/avo)
aka writing assembly-like Go code to generate assembly.
To make it more clear, here is an [avo program](https://github.com/mmcloughlin/avo/tree/master/examples/add) `add/asm.go`.

```go
package main

import asm "github.com/mmcloughlin/avo/build"

func main() {
	asm.TEXT("Add", asm.NOSPLIT, "func(x, y uint64) uint64")
	x, y := asm.GP64(), asm.GP64()
	asm.Load(asm.Param("x"), x)
	asm.Load(asm.Param("y"), y)
	asm.ADDQ(x, y)
	asm.Store(y, asm.ReturnIndex(0))
	asm.RET()
	asm.Generate()
}
```

And this is its output `add/add.s`.

```asm
// func Add(x uint64, y uint64) uint64
TEXT ·Add(SB), NOSPLIT, $0-24
	MOVQ x+0(FP), AX
	MOVQ y+8(FP), CX
	ADDQ AX, CX
	MOVQ CX, ret+16(FP)
	RET
```

As we can see, the program generates Go assembly for `Add` function along with `add/stub.go` file
to access our function from Go.

```console
﹩ go run asm.go -out add.s -stubs stub.go
```

Here is a usage example `main.go`.

```go
package main

import "myprog/add" // Import the stub.

func main() {
	println(add.Add(2, 3))
}
```

If we build this program `myprog` for amd64 architecture and inspect its binary contents,
we'll see that `Add` function looks slightly different:

- `TEXT ·Add` became `TEXT myprog/add.Add.abi0`
- `x` and `y` are gone
- `FP` (frame pointer) usage is replaced with `SP` (stack pointer)

```console
﹩ go mod init myprog
﹩ GOOS=linux GOARCH=amd64 go build -o myprog main.go
﹩ go tool objdump -s add.Add myprog
TEXT myprog/add.Add.abi0(SB) /Users/u/code/myprog/add/add.s
  add.s:7		0x46fac0		488b442408		MOVQ 0x8(SP), AX
  add.s:8		0x46fac5		488b4c2410		MOVQ 0x10(SP), CX
  add.s:9		0x46faca		4801c1			ADDQ AX, CX
  add.s:10		0x46facd		48894c2418		MOVQ CX, 0x18(SP)
  add.s:11		0x46fad2		c3			    RET
```

Why is that so?
[Go's assembler docs](https://go.dev/doc/asm) state that their assembler is not a direct representation
of the underlying machine (amd64 in our case).
That sort of explains the difference 🤔.

> The assembler works on the semi-abstract form...
> In general, machine-specific operations tend to appear as themselves,
> while more general concepts like memory move and subroutine call and return are more abstract.

To sum up, we would write an assembly-like Go code which generates a Go assembly
which ends up an architecture specific assembly.

## Go assembly

Now, let's have a closer look at Go assembly.

```asm
// func Add(x uint64, y uint64) uint64
TEXT ·Add(SB), NOSPLIT, $0-24
	MOVQ x+0(FP), AX
	MOVQ y+8(FP), CX
	ADDQ AX, CX
	MOVQ CX, ret+16(FP)
	RET
```

The `TEXT` directive declares the symbol `·Add` (our function name with a leading dot `U+00B7` character).
The full name of the symbol is `myprog∕add·Add` — the package path followed by a dot and the function name
(note the division slash `U+2215` character).

`avo` didn't need to hard-code the package's import path `myprog∕add` in `add.s` because
the linker inserts the package path at the beginning of any name starting with a dot `·` character,
If we had a global variable `mySum` in the `add` package, we could access it with a dot as well `·mySum`.

```go
package add

var mySum int64
```

The function name `Add` is followed by `(SB)`:

- `SB` stands for static base pointer. It's a pseudo-register maintained by the Go toolchain.
- all global symbols such as `·Add` and `·mySum` are written as offsets from the pseudo-register `SB`,
  for example, `TEXT ·Add(SB)` or `MOV ·mySum(SB), R1`, so we can think of the symbols as named offsets
- parenthesis around `SB` pseudo-register mean register indirect, i.e., we're dereferencing `SB` like this `*SB`
  (that's merely an analogy, not an actual code)

After the symbol, we have `NOSPLIT` flag which is an argument to the `TEXT` directive.
It tells the linker not to insert the preamble that checks if the goroutine stack must be split.
Normally, Go inserts code to check if the stack needs to grow, but `NOSPLIT` disables this.
This reduces the `Add` function call overhead, but limits the size of the stack.
The stack frame for a given function, plus anything it calls, must fit in the spare space
remaining in the current stack segment whose minimum size is 2 KB.
That's not a problem for a leaf function like ours.

After the flag, there is a `TEXT` argument `$0-24` stating:

- `$0` — the stack frame size,
- `-24` — the `Add` function's arguments size in bytes (a minus sign is just a separator).

In our case, the `Add` function has no local stack frame (its size is zero bytes),
meaning there are no local variables, but the frame itself still gets allocated
since we didn't use `NOFRAME` flag.

```go
func Add(x uint64, y uint64) uint64 {
	return x + y
}
```

The function has two 8-bytes arguments and one 8-bytes return value that add up to a total size of 24 bytes.
These 24 bytes live on the caller's stack frame, located at positive offsets from the `FP` pseudo-register.
`FP` stands for frame pointer which is used to refer to function arguments.
Thus `0(FP)` is the argument `x`, `8(FP)` is the second argument `y`,
and `16(FP)` is the return argument named by default as `ret`.

| x     | y     | ret
| ---   | ---   | ---
| 0(FP) | 8(FP) | 16(FP)

Note, the assembler enforces `x+0(FP)`, `y+8(FP)`, and `ret+16(FP)` convention for readability,
rejecting plain `0(FP)` syntax.
Therefore we must place an argument name at the beginning.

```asm
TEXT ·Add(SB), NOSPLIT, $0-24
	MOVQ x+0(FP), AX
	MOVQ y+8(FP), CX
	ADDQ AX, CX
	MOVQ CX, ret+16(FP)
	RET
```

The instructions after the `TEXT` directive form the body of the `Add` function:

- `MOVQ x+0(FP), AX` copies the argument `x` to the `AX` general-purpose register, i.e.,
  it performs a 64-bit `MOV` (`Q` stands for quad on amd64)
  from the caller's stack frame at `0(FP)` offset to the register
- `MOVQ y+8(FP), CX` copies the argument `y` to the `CX` general-purpose register
- `ADDQ AX, CX` adds 64-bit numbers stored in `AX` and `CX` registers, and places the result in the `CX`
- `MOVQ CX, ret+16(FP)` copies the 64 bits from the `CX` register to the return argument `ret`
- `RET` is a pseudo-instruction to return from a function

`avo` package took care of:

- allocating the `AX` and `CX` registers (we used `asm.GP64()` virtual registers in an avo program)
- declaring the function using its signature (the stack frame size and arguments size were calculated for us)
- loading the function arguments `x` and `y` into those registers,
  ensuring memory offsets are correct
- appending `ADDQ` instruction with allocated registers `AX` and `CX`
- storing function return value (again, with correct offset).
  Note, `asm.ReturnIndex(0)` returns the first return argument of the active function.

```go
x, y := asm.GP64(), asm.GP64()
asm.TEXT("Add", asm.NOSPLIT, "func(x, y uint64) uint64") // TEXT ·Add(SB), NOSPLIT, $0-24
asm.Load(asm.Param("x"), x)                              // MOVQ x+0(FP), AX
asm.Load(asm.Param("y"), y)                              // MOVQ y+8(FP), CX
asm.ADDQ(x, y).                                          // ADDQ AX, CX
asm.Store(y, asm.ReturnIndex(0))                         // MOVQ CX, ret+16(FP)
asm.RET()                                                // RET
```

That's neat.

## Go stack

Previously we mentioned pseudo-registers such as `FP` and positive offsets from it
like `y+8(FP)` to access function arguments.
If our function had local variables `var fizz, bazz int64`, we would have spotted
negative offsets from `SP` like `fizz-8(SP)` and `bazz-16(SP)` in the code.
`SP` is yet another pseudo-register, and actually there are four of them that exist in all architectures:

- `SP` stack pointer points to the top of the space allocated for local variables
- `FP` frame pointer points to the bottom of the space allocated for the arguments
- `SB` static base pointer is a global base for global symbols
- `PC` program counter counts pseudo-instructions (we can use the true R name, e.g.,
  `R15` on ARM to access the hardware program counter register)

Note, if we omit the local variable name `fizz` from `fizz-8(SP)` like this `-8(SP)`,
we would reference the hardware register `SP`.
Therefore we can use positive offsets from hardware register `SP` on amd64 architecture
to refer to `fizz` as follows `8(SP)`.

With a diagram of the Go stack everything should be a little more clear.
Here we've got the top stack frame depicting the `Add` function call:

- the stack grows from high to low memory addresses
- arguments are located above `FP`
- local variables (if `Add` had them) would have been below `SP` pseudo-register
  or above `SP` hardware register
- return address is pushed on the stack by the caller, e.g., on architecture independent
  pseudo-instruction `CALL myprog∕add·Add(SB)`
- caller's `RBP` register is saved as well as the frame pointer to link the stack frames

```console
|          ...            | high address
|      caller frame       |
|          ...            |
+-------------------------+
| arguments, e.g.,        |
| ret+16(FP)              |
| y+8(FP)                 |
| x+0(FP)                 |  ⬆️
|-------------------------|← FP pseudo-register
| return address (PC)     |
|-------------------------|
| frame pointer (RBP)     |
|-------------------------|← SP pseudo-register
| local variables, e.g.,  |  ⬇️
| fizz-8(SP)              |
| bazz-16(SP)             |  ⬆️
+-------------------------+← SP hardware register (the top of the stack)
|          ...            |
|       free space        |
|          ...            | low address
```

Zooming out we see the whole stack (just two stack frames in our case).
By the way, we can [get a stack trace](https://blog.felixge.de/reducing-gos-execution-tracer-overhead-with-frame-pointer-unwinding/)
if we follow the `RBP` hardware register's value:

0. grab the current value of `PC` register
1. get to the first frame pointer stored in the frame #1
2. grab the return address of the caller that sits above the frame pointer
3. proceed to the next frame pointer by following the value (caller's `RBP`)
   of the current frame pointer
4. grab the return address above it
5. end the stack walk since the current frame pointer's value is `0`
6. [symbolize the caller addresses](https://marselester.com/diy-cpu-profiler-the-simplest-case-of-symbolization.html)
   we've collected, i.e., resolve those memory addresses to function names

```console
    |          ...            |
    +-------------------------+
    | arguments               | stack frame #0 (caller) is at the bottom of the stack
    |-------------------------|
    | return address (PC)     |
    |-------------------------|
 ↗- | frame pointer (0)       |
|   |-------------------------|
↑   | local variables         |
|   +-------------------------+
↑   | arguments               | stack frame #1 (callee) is at the top of the stack
|   |-------------------------|
↑   | return address (PC)     |
|   |-------------------------|
 ↖_ | frame pointer (RBP)     |
    |-------------------------|← RBP hardware register (starting point for unwinding frame pointers)
    | local variables         |
    +-------------------------+
    |          ...            |
    |       free space        |
    |          ...            |
```

That should cover the basics to get started writing Go assembly,
though I would like to finish this post with a cheat sheet taken from
[Michael Munday's slides](https://youtu.be/9jpnFmJr2PE?t=712).

```asm
                      ; Data moves from left to right
ADD R1, R2            ; R2 += R1
SUB R3, R4, R5        ; R5 = R4 - R3
MUL $7, R6            ; R6 *= 7                 $7 is a literal value 7

                      ; Memory operands
MOV (R1), R2          ; R2 = *R1                register indirect
MOV 8(R3), R4         ; R4 = *(8 + R3)          register indirect with offset
MOV 16(R5)(R6*1), R7  ; R7 = *(16 + R5 + R6*1)  offset + reg1 + reg2*scale
MOV ·mySum(SB), R8    ; R8 = *mySum             access mySum global variable

                      ; Addresses
MOV $8(R1)(R2*1), R3  ; R3 = 8 + R1 + R2
MOV $·mySum(SB), R4   ; R4 = &mySum             dollar sign takes the absolute address
```

References:

- [Dropping Down Go Functions in Assembly](https://www.youtube.com/watch?v=9jpnFmJr2PE) by Michael Munday
- [A Quick Guide to Go's Assembler](https://go.dev/doc/asm)
- [Stack Traces in Go](https://github.com/DataDog/go-profiler-notes/blob/main/stack-traces.md) by Felix Geisendörfer
- [Reducing Go Execution Tracer Overhead With Frame Pointer Unwinding](https://blog.felixge.de/reducing-gos-execution-tracer-overhead-with-frame-pointer-unwinding/) by Felix Geisendörfer
