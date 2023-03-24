title: DIY CPU profiler: the simplest case of symbolization
date: 2023-03-23
tags: bpf, profiling
category: Profiler
slug: diy-cpu-profiler-the-simplest-case-of-symbolization

In the [BPF maps to pprof](https://marselester.com/diy-cpu-profiler-from-bpf-maps-to-pprof.html) post
we managed to collect CPU samples and store them in pprof format.
Here I would like to explore the simplest case of symbolization
and put shared libraries aside.

Symbolization is resolving sampled memory addresses to function names (symbols).
In our case, simply searching for an address in `.symtab` ELF section.

## What's in the profile?

Let's profile a C program that ensures we'll get lots of CPU samples.
It computes Fibonacci numbers naively (without caching),
so `fibNaive` function should dominate the profile.

```c
long fibNaive(long n) {
    if (n <= 2) {
        return 1;
    }
    return fibNaive(n-2) + fibNaive(n-1);
}
```

<details>

```c
#include <stdio.h>

// fibNaive calculates a Fibonacci number n,
// e.g., when n=7, the result is 13: 1 1 2 3 5 8 13.
//
// The running time complexity is exponential 2**n,
// the space complexity is n because
// when we descend the binary tree recursively,
// we use number of stacks equal to its height which is n
// (the stack frames n=7->5->3->1).
long fibNaive(long n) {
    if (n <= 2) {
        return 1;
    }
    return fibNaive(n-2) + fibNaive(n-1);
}

int main() {
    // Calculating 50th Fibonacci number will keep CPU busy
    // and give us time to run a CPU profiler.
    long n = 50;
    long res = fibNaive(n);
    printf("Fibonacci number %li: %li\n", n, res);
    return 0;
}
```

</details>

```console
﹩ # For reference, I used gcc 11.3.0 and Ubuntu 22.04.
﹩ gcc -Og -fno-pie -no-pie -fcf-protection=none -o fib main.c
﹩ ./fib &
﹩ sudo go run ./cmd/profiler3 -pid=$(pidof fib)
/proc/14548/maps
start=0x401000 limit=0x402000 offset=0x1000 /vagrant/diy-parca-agent/fib
start=0x7f3cd28ac000 limit=0x7f3cd2a41000 offset=0x28000 /usr/lib/x86_64-linux-gnu/libc.so.6
start=0x7f3cd2ab6000 limit=0x7f3cd2ae0000 offset=0x2000 /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
start=0x7ffc69d62000 limit=0x7ffc69d64000 offset=0x0 [vdso]
start=0xffffffffff600000 limit=0xffffffffff601000 offset=0x0 [vsyscall]

Waiting for stack traces for 10s...
```

After ten seconds the profiler will terminate leaving us with `cpu.pprof` file.
Let's inspect it with
[protoc](https://github.com/DataDog/go-profiler-notes/blob/main/pprof.md#using-protoc).

```console
﹩ git clone https://github.com/google/pprof.git
﹩ sudo snap install protobuf --classic
﹩ zcat cpu.pprof | \
    protoc --decode perftools.profiles.Profile ./pprof/proto/profile.proto

...
sample {
  location_id: 10
  value: 138
}
...
mapping {
  id: 1
  memory_start: 4198400
  memory_limit: 4202496
  file_offset: 4096
  filename: 3
}
...
location {
  id: 10
  mapping_id: 1
  address: 4198694
}
...
string_table: ""
string_table: "samples"
string_table: "count"
string_table: "/vagrant/diy-parca-agent/fib"
string_table: "/usr/lib/x86_64-linux-gnu/libc.so.6"
string_table: "/usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2"
string_table: "[vdso]"
string_table: "[vsyscall]"
```

<details>

```toml
sample_type {
  type: 1
  unit: 2
}
sample {
  location_id: 1
  value: 18
}
sample {
  location_id: 2
  value: 27
}
sample {
  location_id: 3
  value: 7
}
sample {
  location_id: 4
  value: 286
}
sample {
  location_id: 5
  value: 1
}
sample {
  location_id: 6
  value: 111
}
sample {
  location_id: 7
  value: 7
}
sample {
  location_id: 8
  value: 79
}
sample {
  location_id: 5
  value: 2
}
sample {
  location_id: 9
  value: 29
}
sample {
  location_id: 10
  value: 138
}
sample {
  location_id: 11
  value: 1
}
sample {
  location_id: 11
  value: 36
}
sample {
  location_id: 12
  value: 34
}
sample {
  location_id: 10
  value: 2
}
sample {
  location_id: 13
  value: 1
}
sample {
  location_id: 14
  value: 22
}
sample {
  location_id: 12
  value: 1
}
sample {
  location_id: 15
  value: 37
}
sample {
  location_id: 10
  value: 1
}
sample {
  location_id: 5
  value: 105
}
sample {
  location_id: 16
  value: 54
}
mapping {
  id: 1
  memory_start: 4198400
  memory_limit: 4202496
  file_offset: 4096
  filename: 3
}
mapping {
  id: 2
  memory_start: 139899207073792
  memory_limit: 139899208732672
  file_offset: 163840
  filename: 4
}
mapping {
  id: 3
  memory_start: 139899209211904
  memory_limit: 139899209383936
  file_offset: 8192
  filename: 5
}
mapping {
  id: 4
  memory_start: 140722084126720
  memory_limit: 140722084134912
  filename: 6
}
mapping {
  id: 5
  memory_start: 18446744073699065856
  memory_limit: 18446744073699069952
  filename: 7
}
location {
  id: 1
  mapping_id: 1
  address: 4198715
}
location {
  id: 2
  mapping_id: 1
  address: 4198705
}
location {
  id: 3
  mapping_id: 1
  address: 4198700
}
location {
  id: 4
  mapping_id: 1
  address: 4198724
}
location {
  id: 5
  mapping_id: 1
  address: 4198706
}
location {
  id: 6
  mapping_id: 1
  address: 4198708
}
location {
  id: 7
  mapping_id: 1
  address: 4198727
}
location {
  id: 8
  mapping_id: 1
  address: 4198736
}
location {
  id: 9
  mapping_id: 1
  address: 4198745
}
location {
  id: 10
  mapping_id: 1
  address: 4198694
}
location {
  id: 11
  mapping_id: 1
  address: 4198712
}
location {
  id: 12
  mapping_id: 1
  address: 4198707
}
location {
  id: 13
  mapping_id: 1
  address: 4198739
}
location {
  id: 14
  mapping_id: 1
  address: 4198743
}
location {
  id: 15
  mapping_id: 1
  address: 4198744
}
location {
  id: 16
  mapping_id: 1
  address: 4198719
}
string_table: ""
string_table: "samples"
string_table: "count"
string_table: "/vagrant/diy-parca-agent/fib"
string_table: "/usr/lib/x86_64-linux-gnu/libc.so.6"
string_table: "/usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2"
string_table: "[vdso]"
string_table: "[vsyscall]"
string_table: "cpu"
string_table: "nanoseconds"
time_nanos: 1679419143530427477
duration_nanos: 10000000000
period_type {
  type: 8
  unit: 9
}
period: 10000000
```

</details>

The example above shows a pprof sample that tells us
that some function (its memory `address: 4198694`) has been seen 138 times in stack traces.
That function is located in the mapping id 1 according to location id 10,
somewhere between `memory_start: 4198400` and `memory_limit: 4202496` addresses.
The mapping points to `file_offset: 4096` within some file whose path is defined by `filename: 3`.
I assume it's the third value in the pprof string table counting from zero:
`string_table: "/vagrant/diy-parca-agent/fib"`.

What does it all mean?
If we recall from [Linux process](https://marselester.com/linux-process.html) post,
Linux organizes the virtual memory of a process as a collection of areas (segments)
where an area is a chunk of related virtual pages.
An area can be associated (mapped) with a contiguous chunk of a file,
in this case it is `fib` executable object file starting at file offset 4096.
Let's look at the file's segments and find the one mentioned in the sample.

```console
﹩ readelf --segments --wide fib
```

<details>

```console
Elf file type is EXEC (Executable file)
Entry point 0x401040
There are 13 program headers, starting at offset 64

Program Headers:
  Type           Offset   VirtAddr           PhysAddr           FileSiz  MemSiz   Flg Align
  PHDR           0x000040 0x0000000000400040 0x0000000000400040 0x0002d8 0x0002d8 R   0x8
  INTERP         0x000318 0x0000000000400318 0x0000000000400318 0x00001c 0x00001c R   0x1
      [Requesting program interpreter: /lib64/ld-linux-x86-64.so.2]
  LOAD           0x000000 0x0000000000400000 0x0000000000400000 0x0004f0 0x0004f0 R   0x1000
  LOAD           0x001000 0x0000000000401000 0x0000000000401000 0x00019d 0x00019d R E 0x1000
  LOAD           0x002000 0x0000000000402000 0x0000000000402000 0x000104 0x000104 R   0x1000
  LOAD           0x002e10 0x0000000000403e10 0x0000000000403e10 0x000220 0x000228 RW  0x1000
  DYNAMIC        0x002e20 0x0000000000403e20 0x0000000000403e20 0x0001d0 0x0001d0 RW  0x8
  NOTE           0x000338 0x0000000000400338 0x0000000000400338 0x000020 0x000020 R   0x8
  NOTE           0x000358 0x0000000000400358 0x0000000000400358 0x000044 0x000044 R   0x4
  GNU_PROPERTY   0x000338 0x0000000000400338 0x0000000000400338 0x000020 0x000020 R   0x8
  GNU_EH_FRAME   0x002020 0x0000000000402020 0x0000000000402020 0x000034 0x000034 R   0x4
  GNU_STACK      0x000000 0x0000000000000000 0x0000000000000000 0x000000 0x000000 RW  0x10
  GNU_RELRO      0x002e10 0x0000000000403e10 0x0000000000403e10 0x0001f0 0x0001f0 R   0x1

 Section to Segment mapping:
  Segment Sections...
   00
   01     .interp
   02     .interp .note.gnu.property .note.gnu.build-id .note.ABI-tag .gnu.hash .dynsym .dynstr .gnu.version .gnu.version_r .rela.dyn .rela.plt
   03     .init .plt .text .fini
   04     .rodata .eh_frame_hdr .eh_frame
   05     .init_array .fini_array .dynamic .got .got.plt .data .bss
   06     .dynamic
   07     .note.gnu.property
   08     .note.gnu.build-id .note.ABI-tag
   09     .note.gnu.property
   10     .eh_frame_hdr
   11
   12     .init_array .fini_array .dynamic .got
```

</details>

```console
Program Headers:
  Type           Offset   VirtAddr           PhysAddr           FileSiz  MemSiz   Flg Align
  ...
  LOAD           0x001000 0x0000000000401000 0x0000000000401000 0x00019d 0x00019d R E 0x1000
  ...

 Section to Segment mapping:
  Segment Sections...
   ...
   03     .init .plt .text .fini
```

As we can see the ELF program header describes
`Flg=R E` readable and executable segment
loaded from `Offset=0x001000` (that's hex of `file_offset: 4096`) within the file
into memory starting at `0x0000000000401000` virtual address (that's hex of `memory_start: 4198400`).
According to `FileSiz=0x00019d` only 413 bytes are loaded from the file.
Given the alignment requirement `Align=0x1000`,
the memory area ends at `0x0000000000402000` virtual address (that's hex of `memory_limit: 4202496`).
So the area consists of a single page, i.e., 4096 bytes = memory_limit - memory_start.

Here is how a process's virtual address space looks
when the executable file is mapped.

```console
Executable object file        Virtual address space
 _______                       _______
|       | 0x3def              |       | N
|       |                     |       |
| ...   | 0x2000              | ...   | 0x402000
|-------| -----------> vm_end |-------|
| .fini | 0x1190              | .fini | 0x401190
| .text | 0x1040              | .text | 0x401040
| .plt  | 0x1020              | .plt  | 0x401020
| .init | 0x1000              | .init | 0x401000
|-------| ---------> vm_start |-------|
| ...   |                     | ...   |
|_______| 0                   |_______| 0
```

We can also see which ELF sections are placed within the segment: `.init .plt .text .fini`.
Our function in question should be somewhere in `.text` section,
because it's where machine code of the compiled program lives.

```console
﹩ readelf --sections --wide fib
```

<details>

```console
There are 30 section headers, starting at offset 0x3670:

Section Headers:
  [Nr] Name              Type            Address          Off    Size   ES Flg Lk Inf Al
  [ 0]                   NULL            0000000000000000 000000 000000 00      0   0  0
  [ 1] .interp           PROGBITS        0000000000400318 000318 00001c 00   A  0   0  1
  [ 2] .note.gnu.property NOTE            0000000000400338 000338 000020 00   A  0   0  8
  [ 3] .note.gnu.build-id NOTE            0000000000400358 000358 000024 00   A  0   0  4
  [ 4] .note.ABI-tag     NOTE            000000000040037c 00037c 000020 00   A  0   0  4
  [ 5] .gnu.hash         GNU_HASH        00000000004003a0 0003a0 00001c 00   A  6   0  8
  [ 6] .dynsym           DYNSYM          00000000004003c0 0003c0 000060 18   A  7   1  8
  [ 7] .dynstr           STRTAB          0000000000400420 000420 000050 00   A  0   0  1
  [ 8] .gnu.version      VERSYM          0000000000400470 000470 000008 02   A  6   0  2
  [ 9] .gnu.version_r    VERNEED         0000000000400478 000478 000030 00   A  7   1  8
  [10] .rela.dyn         RELA            00000000004004a8 0004a8 000030 18   A  6   0  8
  [11] .rela.plt         RELA            00000000004004d8 0004d8 000018 18  AI  6  23  8
  [12] .init             PROGBITS        0000000000401000 001000 00001b 00  AX  0   0  4
  [13] .plt              PROGBITS        0000000000401020 001020 000020 10  AX  0   0 16
  [14] .text             PROGBITS        0000000000401040 001040 00014e 00  AX  0   0 16
  [15] .fini             PROGBITS        0000000000401190 001190 00000d 00  AX  0   0  4
  [16] .rodata           PROGBITS        0000000000402000 002000 00001f 00   A  0   0  4
  [17] .eh_frame_hdr     PROGBITS        0000000000402020 002020 000034 00   A  0   0  4
  [18] .eh_frame         PROGBITS        0000000000402058 002058 0000ac 00   A  0   0  8
  [19] .init_array       INIT_ARRAY      0000000000403e10 002e10 000008 08  WA  0   0  8
  [20] .fini_array       FINI_ARRAY      0000000000403e18 002e18 000008 08  WA  0   0  8
  [21] .dynamic          DYNAMIC         0000000000403e20 002e20 0001d0 10  WA  7   0  8
  [22] .got              PROGBITS        0000000000403ff0 002ff0 000010 08  WA  0   0  8
  [23] .got.plt          PROGBITS        0000000000404000 003000 000020 08  WA  0   0  8
  [24] .data             PROGBITS        0000000000404020 003020 000010 00  WA  0   0  8
  [25] .bss              NOBITS          0000000000404030 003030 000008 00  WA  0   0  1
  [26] .comment          PROGBITS        0000000000000000 003030 00002b 01  MS  0   0  1
  [27] .symtab           SYMTAB          0000000000000000 003060 000348 18     28  18  8
  [28] .strtab           STRTAB          0000000000000000 0033a8 0001b1 00      0   0  1
  [29] .shstrtab         STRTAB          0000000000000000 003559 000116 00      0   0  1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings), I (info),
  L (link order), O (extra OS processing required), G (group), T (TLS),
  C (compressed), x (unknown), o (OS specific), E (exclude),
  D (mbind), l (large), p (processor specific)
```

</details>

```console
Section Headers:
  [Nr] Name              Type            Address          Off    Size   ES Flg Lk Inf Al
  ...
  [12] .init             PROGBITS        0000000000401000 001000 00001b 00  AX  0   0  4
  [13] .plt              PROGBITS        0000000000401020 001020 000020 10  AX  0   0 16
  [14] .text             PROGBITS        0000000000401040 001040 00014e 00  AX  0   0 16
  [15] .fini             PROGBITS        0000000000401190 001190 00000d 00  AX  0   0  4
  ...
```

## Symbol table

Information about functions and global variables (aka symbols)
defined and referenced in our program lives in a symbol table in `.symtab` section.

> An object file's symbol table holds information needed to locate and relocate a program's symbolic definitions and references. https://www.man7.org/linux/man-pages/man5/elf.5.html

This implies that we should be able to find
which symbol corresponds to memory `address: 4198694` or `0x401126` in hexadecimal form.
Let's inspect the symbol table.

```console
﹩ readelf --symbols fib
```

<details>

```console
Symbol table '.dynsym' contains 4 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND
     1: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND _[...]@GLIBC_2.34 (2)
     2: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND __gmon_start__
     3: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND [...]@GLIBC_2.3.4 (3)

Symbol table '.symtab' contains 35 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND
     1: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS crt1.o
     2: 000000000040037c    32 OBJECT  LOCAL  DEFAULT    4 __abi_tag
     3: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS crtstuff.c
     4: 0000000000401080     0 FUNC    LOCAL  DEFAULT   14 deregister_tm_clones
     5: 00000000004010b0     0 FUNC    LOCAL  DEFAULT   14 register_tm_clones
     6: 00000000004010f0     0 FUNC    LOCAL  DEFAULT   14 __do_global_dtors_aux
     7: 0000000000404030     1 OBJECT  LOCAL  DEFAULT   25 completed.0
     8: 0000000000403e18     0 OBJECT  LOCAL  DEFAULT   20 __do_global_dtor[...]
     9: 0000000000401120     0 FUNC    LOCAL  DEFAULT   14 frame_dummy
    10: 0000000000403e10     0 OBJECT  LOCAL  DEFAULT   19 __frame_dummy_in[...]
    11: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS main.c
    12: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS crtstuff.c
    13: 0000000000402100     0 OBJECT  LOCAL  DEFAULT   18 __FRAME_END__
    14: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS
    15: 0000000000403e20     0 OBJECT  LOCAL  DEFAULT   21 _DYNAMIC
    16: 0000000000402020     0 NOTYPE  LOCAL  DEFAULT   17 __GNU_EH_FRAME_HDR
    17: 0000000000404000     0 OBJECT  LOCAL  DEFAULT   23 _GLOBAL_OFFSET_TABLE_
    18: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND __libc_start_mai[...]
    19: 0000000000404020     0 NOTYPE  WEAK   DEFAULT   24 data_start
    20: 0000000000404030     0 NOTYPE  GLOBAL DEFAULT   24 _edata
    21: 0000000000401190     0 FUNC    GLOBAL HIDDEN    15 _fini
    22: 0000000000404020     0 NOTYPE  GLOBAL DEFAULT   24 __data_start
    23: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND __gmon_start__
    24: 0000000000404028     0 OBJECT  GLOBAL HIDDEN    24 __dso_handle
    25: 0000000000402000     4 OBJECT  GLOBAL DEFAULT   16 _IO_stdin_used
    26: 0000000000404038     0 NOTYPE  GLOBAL DEFAULT   25 _end
    27: 0000000000401070     5 FUNC    GLOBAL HIDDEN    14 _dl_relocate_sta[...]
    28: 0000000000401040    38 FUNC    GLOBAL DEFAULT   14 _start
    29: 0000000000404030     0 NOTYPE  GLOBAL DEFAULT   25 __bss_start
    30: 000000000040115a    52 FUNC    GLOBAL DEFAULT   14 main
    31: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND __printf_chk@GLI[...]
    32: 0000000000404030     0 OBJECT  GLOBAL HIDDEN    24 __TMC_END__
    33: 0000000000401000     0 FUNC    GLOBAL HIDDEN    12 _init
    34: 0000000000401126    52 FUNC    GLOBAL DEFAULT   14 fibNaive
```

</details>

```console
Symbol table '.symtab' contains 35 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
   ...
    34: 0000000000401126    52 FUNC    GLOBAL DEFAULT   14 fibNaive
```

So here you go, **fibNaive** C function we were looking for:

- `Value=0000000000401126` The symbol's absolute run-time address
  since it's an executable object file.
  For relocatable object files, it would be an offset from
  the beginning of the section defined by `Ndx=14`.
- `Size=52` The size of the function's code is 52 bytes.
- `Type=FUNC` The symbol is associated with a function.
- `Bind=GLOBAL` This indicates that the symbol is global,
  i.e., it can be referenced by other modules.
- `Vis=DEFAULT` Default symbol visibility rules apply:
  strong (functions and initialized global vars)
  and weak symbols (uninitialized global vars) are available to other modules.
- `Ndx=14` A symbol is "defined" in relation to `.text` section
  according to section header table index, see `readelf --sections fib`.
- `Name=fibNaive` The symbol name `fibNaive` was taken from
  string table in section `.strtab`, see `readelf --string-dump .strtab fib`.

😌

Great, let's check the remaining memory addresses we've got in the pprof file
(I've sorted them for our convenience).
Unfortunately, I couldn't find corresponding symbols in `.symtab` section.

```console
4198694 = 0x401126 fibNaive
4198700 = 0x40112c ?
4198705 = 0x401131 ?
4198706 = 0x401132 ?
4198707 = 0x401133 ?
4198708 = 0x401134 ?
4198712 = 0x401138 ?
4198715 = 0x40113b ?
4198719 = 0x40113f ?
4198724 = 0x401144 ?
4198727 = 0x401147 ?
4198736 = 0x401150 ?
4198739 = 0x401153 ?
4198743 = 0x401157 ?
4198744 = 0x401158 ?
4198745 = 0x401159 ?
```

What's interesting is that these addresses are within `Size=52` bytes
from `fibNaive` memory address, e.g.,
the distance to the highest address is 51 bytes = 4198745 - 4198694.

If we sort the symbols from `.symtab` section by `Value`,
we'll see that `fibNaive` is between `frame_dummy` and `main` symbols.
The rest of unresolved memory addresses from the samples
fit in between `fibNaive` and `main` symbols.

<details>

```console
0000000 = 0x000000 crt1.o
0000000 = 0x000000 crtstuff.c
0000000 = 0x000000 main.c
0000000 = 0x000000 crtstuff.c
0000000 = 0x000000
0000000 = 0x000000 __libc_start_main@GLIBC_2.34
0000000 = 0x000000 __gmon_start__
0000000 = 0x000000 __printf_chk@GLIBC_2.3.4
4195196 = 0x40037c __abi_tag
4198400 = 0x401000 _init
4198464 = 0x401040 _start
4198512 = 0x401070 _dl_relocate_static_pie
4198528 = 0x401080 deregister_tm_clones
4198576 = 0x4010b0 register_tm_clones
4198640 = 0x4010f0 __do_global_dtors_aux
4198688 = 0x401120 frame_dummy
4198694 = 0x401126 fibNaive
4198746 = 0x40115a main
4198800 = 0x401190 _fini
4202496 = 0x402000 _IO_stdin_used
4202528 = 0x402020 __GNU_EH_FRAME_HDR
4202752 = 0x402100 __FRAME_END__
4210192 = 0x403e10 __frame_dummy_init_array_entry
4210200 = 0x403e18 __do_global_dtors_aux_fini_array_entry
4210208 = 0x403e20 _DYNAMIC
4210688 = 0x404000 _GLOBAL_OFFSET_TABLE_
4210720 = 0x404020 data_start
4210720 = 0x404020 __data_start
4210728 = 0x404028 __dso_handle
4210736 = 0x404030 completed.0
4210736 = 0x404030 _edata
4210736 = 0x404030 __bss_start
4210736 = 0x404030 __TMC_END__
4210744 = 0x404038 _end
```

</details>

```console
0x401120 frame_dummy
0x401126 fibNaive
0x40112c ?
0x401131 ?
0x401132 ?
0x401133 ?
0x401134 ?
0x401138 ?
0x40113b ?
0x40113f ?
0x401144 ?
0x401147 ?
0x401150 ?
0x401153 ?
0x401157 ?
0x401158 ?
0x401159 ?
0x40115a main
```

Disassembling the program confirms that memory addresses in question
correspond to instructions of `fibNaive` function.

```console
﹩ objdump --disassemble fib
```

<details>

```objdump
...

0000000000401120 <frame_dummy>:
  401120:	f3 0f 1e fa          	endbr64
  401124:	eb 8a                	jmp    4010b0 <register_tm_clones>

0000000000401126 <fibNaive>:
  401126:	48 83 ff 02          	cmp    $0x2,%rdi
  40112a:	7f 06                	jg     401132 <fibNaive+0xc>
  40112c:	b8 01 00 00 00       	mov    $0x1,%eax
  401131:	c3                   	ret
  401132:	55                   	push   %rbp
  401133:	53                   	push   %rbx
  401134:	48 83 ec 08          	sub    $0x8,%rsp
  401138:	48 89 fb             	mov    %rdi,%rbx
  40113b:	48 8d 7f fe          	lea    -0x2(%rdi),%rdi
  40113f:	e8 e2 ff ff ff       	call   401126 <fibNaive>
  401144:	48 89 c5             	mov    %rax,%rbp
  401147:	48 8d 7b ff          	lea    -0x1(%rbx),%rdi
  40114b:	e8 d6 ff ff ff       	call   401126 <fibNaive>
  401150:	48 01 e8             	add    %rbp,%rax
  401153:	48 83 c4 08          	add    $0x8,%rsp
  401157:	5b                   	pop    %rbx
  401158:	5d                   	pop    %rbp
  401159:	c3                   	ret

000000000040115a <main>:
  40115a:	48 83 ec 08          	sub    $0x8,%rsp
  40115e:	bf 32 00 00 00       	mov    $0x32,%edi
  401163:	e8 be ff ff ff       	call   401126 <fibNaive>
  401168:	48 89 c1             	mov    %rax,%rcx
  40116b:	ba 32 00 00 00       	mov    $0x32,%edx
  401170:	be 04 20 40 00       	mov    $0x402004,%esi
  401175:	bf 01 00 00 00       	mov    $0x1,%edi
  40117a:	b8 00 00 00 00       	mov    $0x0,%eax
  40117f:	e8 ac fe ff ff       	call   401030 <__printf_chk@plt>
  401184:	b8 00 00 00 00       	mov    $0x0,%eax
  401189:	48 83 c4 08          	add    $0x8,%rsp
  40118d:	c3                   	ret

...
```

</details>

```objdump
0000000000401126 <fibNaive>:
  401126:	48 83 ff 02          	cmp    $0x2,%rdi
  40112a:	7f 06                	jg     401132 <fibNaive+0xc>
  40112c:	b8 01 00 00 00       	mov    $0x1,%eax
  401131:	c3                   	ret
  401132:	55                   	push   %rbp
  401133:	53                   	push   %rbx
  401134:	48 83 ec 08          	sub    $0x8,%rsp
  401138:	48 89 fb             	mov    %rdi,%rbx
  40113b:	48 8d 7f fe          	lea    -0x2(%rdi),%rdi
  40113f:	e8 e2 ff ff ff       	call   401126 <fibNaive>
  401144:	48 89 c5             	mov    %rax,%rbp
  401147:	48 8d 7b ff          	lea    -0x1(%rbx),%rdi
  40114b:	e8 d6 ff ff ff       	call   401126 <fibNaive>
  401150:	48 01 e8             	add    %rbp,%rax
  401153:	48 83 c4 08          	add    $0x8,%rsp
  401157:	5b                   	pop    %rbx
  401158:	5d                   	pop    %rbp
  401159:	c3                   	ret
```

It means that `bpf_get_stackid` we used in
[profiler](https://github.com/marselester/diy-parca-agent/blob/main/cmd/profiler/bpf/parca-agent.bpf.c)
collected values stored in `rip` register,
i.e., memory addresses of machine instructions.

## PC to symbol address

Since we have sorted symbol addresses from `.symtab`,
we can binary search them to find
whose instructions were executing when a sample was taken.

```console
0x401120 frame_dummy
0x401126 fibNaive
0x40112c ?
0x40115a main
```

When the program counter (PC) contains instruction address `0x401126`,
then `fibNaive` function symbol is found in the symbol table.
When PC points to `0x40112c` instruction,
there is no match in the symbol table,
so the `main` symbol's predecessor is resolved
because binary search stops at `0x40115a`.

```go
func PC2SymbolAddress(symbols []int, pc int) int {
	lft := 0
	var mid int
	rgt := len(symbols) - 1

	for lft <= rgt {
		mid = lft + (rgt-lft)/2
		symAddr := symbols[mid]

		switch {
		case symAddr == pc:
			return symAddr
		case symAddr < pc:
			lft = mid + 1
		case symAddr > pc:
			rgt = mid - 1
		}
	}

	// Since pc wasn't found in the array,
	// now mid points to symbol address > pc, i.e., 0x40115a.
	//
	// 0x401120 frame_dummy
	// 0x401126 fibNaive
	// 0x40112c ?
	// 0x40115a main
	//
	// Therefore the desired symbol's address is 0x401126.
	if mid >= 1 {
		return symbols[mid-1]
	}

	return -1
}
```

In reality symbolization is much more complicated.
Most likely you would have to resolve symbols in shared libraries,
and also deal with things like:

- Frame pointer or DWARF's Call Frame Information based stack walking, see
  [Reducing Go Execution Tracer Overhead With Frame Pointer Unwinding](https://blog.felixge.de/reducing-gos-execution-tracer-overhead-with-frame-pointer-unwinding/) and
  [DWARF-based Stack Walking Using eBPF](https://www.polarsignals.com/blog/posts/2022/11/29/profiling-without-frame-pointers/).
  Alas, frame pointers are disabled by default in gcc,
  but you can enable them with the `-fno-omit-frame-pointer` flag.
- Looking for symbols formatted in DWARF
  which could be part of an executable file or distributed separately,
  see [Fantastic Symbols and Where to Find Them](https://www.polarsignals.com/blog/posts/2022/01/13/fantastic-symbols-and-where-to-find-them/).
  The `-g` gcc flag produces debugging information in DWARF format.

When profiling JIT-compiled code you would have to
look for symbols in `/tmp/perf-$PID.map` file, see
[Fantastic Symbols and Where to Find Them - Part 2](https://www.polarsignals.com/blog/posts/2022/01/27/fantastic-symbols-and-where-to-find-them-part-2/).
