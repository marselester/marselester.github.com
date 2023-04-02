title: DIY CPU profiler: position independent executable 🥧
date: 2023-04-02
tags: bpf, profiling
category: Profiler
slug: diy-cpu-profiler-position-independent-executable

In the [simplest case of symbolization](https://marselester.com/diy-cpu-profiler-the-simplest-case-of-symbolization.html)
we used `-fno-pie -no-pie` flags, so
gcc wouldn't produce a position independent executable (PIE).
It is produced by default for security measures such as
address space layout randomization (ASLR).
That means each time the program runs,
its segments are loaded into different regions of virtual memory,
but their relative positions stay the same.

Let's see what happens when those flags are omitted.

```console
﹩ gcc -Og -fcf-protection=none -o fib main.c
﹩ ./fib &
﹩ sudo go run ./cmd/profiler3 -pid=$(pidof fib)
/proc/19978/maps
start=0x555806f2c000 limit=0x555806f2d000 offset=0x1000 /vagrant/diy-parca-agent/fib
start=0x7f4e21595000 limit=0x7f4e2172a000 offset=0x28000 /usr/lib/x86_64-linux-gnu/libc.so.6
start=0x7f4e2179f000 limit=0x7f4e217c9000 offset=0x2000 /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
start=0x7fffe4d9e000 limit=0x7fffe4da0000 offset=0x0 [vdso]
start=0xffffffffff600000 limit=0xffffffffff601000 offset=0x0 [vsyscall]

Waiting for stack traces for 10s...
{PID:19978 UserStackID:288 KernelStackID:-14} seen 254 times
	288 [555806f2c139]
...
```

The first thing I noticed when looking at samples is much higher addresses,
e.g., we've just sampled `93836562055481` address vs `4198694` from
the [previous post](https://marselester.com/diy-cpu-profiler-the-simplest-case-of-symbolization.html).
The size of the mapping id 1 is still one page,
i.e., 4096 bytes = memory_limit - memory_start.

```console
﹩ zcat cpu.pprof | protoc --decode perftools.profiles.Profile ./pprof/proto/profile.proto
```

<details>

```toml
sample_type {
  type: 1
  unit: 2
}
sample {
  location_id: 1
  value: 2
}
sample {
  location_id: 2
  value: 75
}
sample {
  location_id: 3
  value: 26
}
sample {
  location_id: 4
  value: 1
}
sample {
  location_id: 5
  value: 1
}
sample {
  location_id: 4
  value: 48
}
sample {
  location_id: 6
  value: 1
}
sample {
  location_id: 7
  value: 1
}
sample {
  location_id: 8
  value: 79
}
sample {
  location_id: 9
  value: 1
}
sample {
  location_id: 10
  value: 2
}
sample {
  location_id: 11
  value: 99
}
sample {
  location_id: 12
  value: 23
}
sample {
  location_id: 13
  value: 4
}
sample {
  location_id: 9
  value: 87
}
sample {
  location_id: 10
  value: 203
}
sample {
  location_id: 5
  value: 61
}
sample {
  location_id: 7
  value: 254
}
sample {
  location_id: 14
  value: 14
}
sample {
  location_id: 15
  value: 15
}
sample {
  location_id: 14
  value: 1
}
mapping {
  id: 1
  memory_start: 93836562055168
  memory_limit: 93836562059264
  file_offset: 4096
  filename: 3
}
mapping {
  id: 2
  memory_start: 139973543677952
  memory_limit: 139973545336832
  file_offset: 163840
  filename: 4
}
mapping {
  id: 3
  memory_start: 139973545816064
  memory_limit: 139973545988096
  file_offset: 8192
  filename: 5
}
mapping {
  id: 4
  memory_start: 140737032871936
  memory_limit: 140737032880128
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
  address: 93836562055526
}
location {
  id: 2
  mapping_id: 1
  address: 93836562055495
}
location {
  id: 3
  mapping_id: 1
  address: 93836562055506
}
location {
  id: 4
  mapping_id: 1
  address: 93836562055494
}
location {
  id: 5
  mapping_id: 1
  address: 93836562055499
}
location {
  id: 6
  mapping_id: 1
  address: 93836562055487
}
location {
  id: 7
  mapping_id: 1
  address: 93836562055481
}
location {
  id: 8
  mapping_id: 1
  address: 93836562055531
}
location {
  id: 9
  mapping_id: 1
  address: 93836562055523
}
location {
  id: 10
  mapping_id: 1
  address: 93836562055511
}
location {
  id: 11
  mapping_id: 1
  address: 93836562055493
}
location {
  id: 12
  mapping_id: 1
  address: 93836562055518
}
location {
  id: 13
  mapping_id: 1
  address: 93836562055492
}
location {
  id: 14
  mapping_id: 1
  address: 93836562055502
}
location {
  id: 15
  mapping_id: 1
  address: 93836562055530
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
time_nanos: 1680393594260736123
duration_nanos: 10000000000
period_type {
  type: 8
  unit: 9
}
period: 10000000
```

</details>

```toml
...
mapping {
  id: 1
  memory_start: 93836562055168 # 4198400 before
  memory_limit: 93836562059264 # 4202496 before
  file_offset: 4096
  filename: 3
}
...
location {
  id: 7
  mapping_id: 1
  address: 93836562055481      # 4198694 before
}
...
```

The second thing is a mismatch between
where the segment should be loaded `VirtAddr=0x0000000000001000`
and where it was actually loaded `memory_start: 93836562055168`.

```console
﹩ readelf --segments --wide fib
```

<details>

```console
Elf file type is DYN (Position-Independent Executable file)
Entry point 0x1050
There are 13 program headers, starting at offset 64

Program Headers:
  Type           Offset   VirtAddr           PhysAddr           FileSiz  MemSiz   Flg Align
  PHDR           0x000040 0x0000000000000040 0x0000000000000040 0x0002d8 0x0002d8 R   0x8
  INTERP         0x000318 0x0000000000000318 0x0000000000000318 0x00001c 0x00001c R   0x1
      [Requesting program interpreter: /lib64/ld-linux-x86-64.so.2]
  LOAD           0x000000 0x0000000000000000 0x0000000000000000 0x000638 0x000638 R   0x1000
  LOAD           0x001000 0x0000000000001000 0x0000000000001000 0x0001b1 0x0001b1 R E 0x1000
  LOAD           0x002000 0x0000000000002000 0x0000000000002000 0x00010c 0x00010c R   0x1000
  LOAD           0x002db8 0x0000000000003db8 0x0000000000003db8 0x000258 0x000260 RW  0x1000
  DYNAMIC        0x002dc8 0x0000000000003dc8 0x0000000000003dc8 0x0001f0 0x0001f0 RW  0x8
  NOTE           0x000338 0x0000000000000338 0x0000000000000338 0x000020 0x000020 R   0x8
  NOTE           0x000358 0x0000000000000358 0x0000000000000358 0x000044 0x000044 R   0x4
  GNU_PROPERTY   0x000338 0x0000000000000338 0x0000000000000338 0x000020 0x000020 R   0x8
  GNU_EH_FRAME   0x002020 0x0000000000002020 0x0000000000002020 0x000034 0x000034 R   0x4
  GNU_STACK      0x000000 0x0000000000000000 0x0000000000000000 0x000000 0x000000 RW  0x10
  GNU_RELRO      0x002db8 0x0000000000003db8 0x0000000000003db8 0x000248 0x000248 R   0x1

 Section to Segment mapping:
  Segment Sections...
   00
   01     .interp
   02     .interp .note.gnu.property .note.gnu.build-id .note.ABI-tag .gnu.hash .dynsym .dynstr .gnu.version .gnu.version_r .rela.dyn .rela.plt
   03     .init .plt .plt.got .text .fini
   04     .rodata .eh_frame_hdr .eh_frame
   05     .init_array .fini_array .dynamic .got .data .bss
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
  LOAD           0x001000 0x0000000000001000 0x0000000000001000 0x0001b1 0x0001b1 R E 0x1000
  ...

 Section to Segment mapping:
  Segment Sections...
   ...
   03     .init .plt .plt.got .text .fini
   ...
```

The third thing is a difference in the segment's sections: the `.plt.got` section was added.

## How PIE affects symbolization

Clearly none of the sampled addresses matches `fibNaive` function
when we look at the symbol table.
The function's address `0x1139` is very far from
the beginning of the loaded segment `0x555806f2c000`
(that's hex of `memory_start: 93836562055168`).

```console
﹩ readelf --symbols fib
```

<details>

```console
Symbol table '.dynsym' contains 7 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND
     1: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND _[...]@GLIBC_2.34 (2)
     2: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND _ITM_deregisterT[...]
     3: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND __gmon_start__
     4: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND [...]@GLIBC_2.3.4 (3)
     5: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND _ITM_registerTMC[...]
     6: 0000000000000000     0 FUNC    WEAK   DEFAULT  UND [...]@GLIBC_2.2.5 (4)

Symbol table '.symtab' contains 37 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND
     1: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS Scrt1.o
     2: 000000000000037c    32 OBJECT  LOCAL  DEFAULT    4 __abi_tag
     3: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS crtstuff.c
     4: 0000000000001080     0 FUNC    LOCAL  DEFAULT   15 deregister_tm_clones
     5: 00000000000010b0     0 FUNC    LOCAL  DEFAULT   15 register_tm_clones
     6: 00000000000010f0     0 FUNC    LOCAL  DEFAULT   15 __do_global_dtors_aux
     7: 0000000000004010     1 OBJECT  LOCAL  DEFAULT   25 completed.0
     8: 0000000000003dc0     0 OBJECT  LOCAL  DEFAULT   21 __do_global_dtor[...]
     9: 0000000000001130     0 FUNC    LOCAL  DEFAULT   15 frame_dummy
    10: 0000000000003db8     0 OBJECT  LOCAL  DEFAULT   20 __frame_dummy_in[...]
    11: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS main.c
    12: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS crtstuff.c
    13: 0000000000002108     0 OBJECT  LOCAL  DEFAULT   19 __FRAME_END__
    14: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS
    15: 0000000000003dc8     0 OBJECT  LOCAL  DEFAULT   22 _DYNAMIC
    16: 0000000000002020     0 NOTYPE  LOCAL  DEFAULT   18 __GNU_EH_FRAME_HDR
    17: 0000000000003fb8     0 OBJECT  LOCAL  DEFAULT   23 _GLOBAL_OFFSET_TABLE_
    18: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND __libc_start_mai[...]
    19: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND _ITM_deregisterT[...]
    20: 0000000000004000     0 NOTYPE  WEAK   DEFAULT   24 data_start
    21: 0000000000004010     0 NOTYPE  GLOBAL DEFAULT   24 _edata
    22: 00000000000011a4     0 FUNC    GLOBAL HIDDEN    16 _fini
    23: 0000000000004000     0 NOTYPE  GLOBAL DEFAULT   24 __data_start
    24: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND __gmon_start__
    25: 0000000000004008     0 OBJECT  GLOBAL HIDDEN    24 __dso_handle
    26: 0000000000002000     4 OBJECT  GLOBAL DEFAULT   17 _IO_stdin_used
    27: 0000000000004018     0 NOTYPE  GLOBAL DEFAULT   25 _end
    28: 0000000000001050    38 FUNC    GLOBAL DEFAULT   15 _start
    29: 0000000000004010     0 NOTYPE  GLOBAL DEFAULT   25 __bss_start
    30: 000000000000116d    54 FUNC    GLOBAL DEFAULT   15 main
    31: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND __printf_chk@GLI[...]
    32: 0000000000004010     0 OBJECT  GLOBAL HIDDEN    24 __TMC_END__
    33: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND _ITM_registerTMC[...]
    34: 0000000000000000     0 FUNC    WEAK   DEFAULT  UND __cxa_finalize@G[...]
    35: 0000000000001000     0 FUNC    GLOBAL HIDDEN    12 _init
    36: 0000000000001139    52 FUNC    GLOBAL DEFAULT   15 fibNaive
```

</details>

```console
Symbol table '.symtab' contains 37 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
    ...
    36: 0000000000001139    52 FUNC    GLOBAL DEFAULT   15 fibNaive
```

Let's try to figure out where sections are mapped in the virtual address space.
For that, we need to know the distance from the segment to each section,
i.e., the section's offset minus the segment's offset:

- 0 bytes from `.init` section = 0x1000 - 0x1000
- 0x20 bytes from `.plt` section = 0x1020 - 0x1000
- 0x40 bytes from `.plt.got` section = 0x1040 - 0x1000
- 0x50 bytes from `.text` section = 0x1050 - 0x1000
- 0x1a4 bytes from `.fini` section = 0x11a4 - 0x1000

```console
﹩ readelf --sections --wide fib
```

<details>

```console
There are 30 section headers, starting at offset 0x36b8:

Section Headers:
  [Nr] Name              Type            Address          Off    Size   ES Flg Lk Inf Al
  [ 0]                   NULL            0000000000000000 000000 000000 00      0   0  0
  [ 1] .interp           PROGBITS        0000000000000318 000318 00001c 00   A  0   0  1
  [ 2] .note.gnu.property NOTE            0000000000000338 000338 000020 00   A  0   0  8
  [ 3] .note.gnu.build-id NOTE            0000000000000358 000358 000024 00   A  0   0  4
  [ 4] .note.ABI-tag     NOTE            000000000000037c 00037c 000020 00   A  0   0  4
  [ 5] .gnu.hash         GNU_HASH        00000000000003a0 0003a0 000024 00   A  6   0  8
  [ 6] .dynsym           DYNSYM          00000000000003c8 0003c8 0000a8 18   A  7   1  8
  [ 7] .dynstr           STRTAB          0000000000000470 000470 0000a1 00   A  0   0  1
  [ 8] .gnu.version      VERSYM          0000000000000512 000512 00000e 02   A  6   0  2
  [ 9] .gnu.version_r    VERNEED         0000000000000520 000520 000040 00   A  7   1  8
  [10] .rela.dyn         RELA            0000000000000560 000560 0000c0 18   A  6   0  8
  [11] .rela.plt         RELA            0000000000000620 000620 000018 18  AI  6  23  8
  [12] .init             PROGBITS        0000000000001000 001000 00001b 00  AX  0   0  4
  [13] .plt              PROGBITS        0000000000001020 001020 000020 10  AX  0   0 16
  [14] .plt.got          PROGBITS        0000000000001040 001040 000008 08  AX  0   0  8
  [15] .text             PROGBITS        0000000000001050 001050 000153 00  AX  0   0 16
  [16] .fini             PROGBITS        00000000000011a4 0011a4 00000d 00  AX  0   0  4
  [17] .rodata           PROGBITS        0000000000002000 002000 00001f 00   A  0   0  4
  [18] .eh_frame_hdr     PROGBITS        0000000000002020 002020 000034 00   A  0   0  4
  [19] .eh_frame         PROGBITS        0000000000002058 002058 0000b4 00   A  0   0  8
  [20] .init_array       INIT_ARRAY      0000000000003db8 002db8 000008 08  WA  0   0  8
  [21] .fini_array       FINI_ARRAY      0000000000003dc0 002dc0 000008 08  WA  0   0  8
  [22] .dynamic          DYNAMIC         0000000000003dc8 002dc8 0001f0 10  WA  7   0  8
  [23] .got              PROGBITS        0000000000003fb8 002fb8 000048 08  WA  0   0  8
  [24] .data             PROGBITS        0000000000004000 003000 000010 00  WA  0   0  8
  [25] .bss              NOBITS          0000000000004010 003010 000008 00  WA  0   0  1
  [26] .comment          PROGBITS        0000000000000000 003010 00002b 01  MS  0   0  1
  [27] .symtab           SYMTAB          0000000000000000 003040 000378 18     28  18  8
  [28] .strtab           STRTAB          0000000000000000 0033b8 0001eb 00      0   0  1
  [29] .shstrtab         STRTAB          0000000000000000 0035a3 000111 00      0   0  1
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
  [12] .init             PROGBITS        0000000000001000 001000 00001b 00  AX  0   0  4
  [13] .plt              PROGBITS        0000000000001020 001020 000020 10  AX  0   0 16
  [14] .plt.got          PROGBITS        0000000000001040 001040 000008 08  AX  0   0  8
  [15] .text             PROGBITS        0000000000001050 001050 000153 00  AX  0   0 16
  [16] .fini             PROGBITS        00000000000011a4 0011a4 00000d 00  AX  0   0  4
  ...
```

Then we can add those distances to `memory_start: 0x555806f2c000`
to get an idea how the process's virtual address space looked like
when the executable file was mapped.

```console
Executable object file           Virtual address space
 __________                       __________
|          | 0x3e37              |          | N
|          |                     |          |
| ...      | 0x2000              | ...      | 0x555806f2d000
|----------| -----------> vm_end |----------|
| .fini    | 0x11a4              | .fini    | 0x555806f2c1a4
| .text    | 0x1050              | .text    | 0x555806f2c050
| .plt.got | 0x1040              | .plt.got | 0x555806f2c040
| .plt     | 0x1020              | .plt     | 0x555806f2c020
| .init    | 0x1000              | .init    | 0x555806f2c000
|----------| ---------> vm_start |----------|
| ...      |                     | ...      |
|__________| 0                   |__________| 0
```

The sampled addresses should be in the `.text` section
which starts at `0x1050` offset within the file.
Since the segment itself starts at `0x1000` offset,
the distance between them is 0x50 bytes = 0x1050 - 0x1000.
Therefore, in virtual address space the `.text` section starts at
`0x555806f2c050` address = memory_start + distance = 0x555806f2c000 + 0x50.

Our sampled address `0x555806f2c139` (that's `address: 93836562055481` in hexadecimal form)
can be adjusted to the ELF file's address range as follows:

1. Find the distance between the sampled memory address and
  beginning of the loaded segment:
  313 bytes = 0x555806f2c139 - 0x555806f2c000.
2. Add the segment's offset to the distance
  to find the address used in the `.symtab` section:
  0x1139 = 0x1000 + 313.

The `0x1139` address belongs to `fibNaive` function as we've seen
in the symbol table.
That's nice! 🙌

```go
// Distance between the sampled memory address and
// beginning of the loaded segment (vm_start memory address).
d := sampledAddr - mappingMemoryStart
// Sampled address adjusted to .symtab address range.
adjustedAddr := mappingFileOffset + d
```

Here are all the addresses from `cpu.pprof` file sorted in ascending order,
their distances from `memory_start: 0x555806f2c000`,
and addresses adjusted to the file's range.

| sampled address | distance  | adjusted address
| ---             | ---       | ---
| 0x555806f2c139  | 313 bytes | 0x1139
| 0x555806f2c13f  | 319 bytes | 0x113f
| 0x555806f2c144  | 324 bytes | 0x1144
| 0x555806f2c145  | 325 bytes | 0x1145
| 0x555806f2c146  | 326 bytes | 0x1146
| 0x555806f2c147  | 327 bytes | 0x1147
| 0x555806f2c14b  | 331 bytes | 0x114b
| 0x555806f2c14e  | 334 bytes | 0x114e
| 0x555806f2c152  | 338 bytes | 0x1152
| 0x555806f2c157  | 343 bytes | 0x1157
| 0x555806f2c15e  | 350 bytes | 0x115e
| 0x555806f2c163  | 355 bytes | 0x1163
| 0x555806f2c166  | 358 bytes | 0x1166
| 0x555806f2c16a  | 362 bytes | 0x116a
| 0x555806f2c16b  | 363 bytes | 0x116b

Disassembling `fib` program confirms that adjusted memory addresses
correspond to instructions of `fibNaive` function.

```objdump
﹩ objdump --disassemble=fibNaive fib
...
0000000000001139 <fibNaive>:
    1139:	48 83 ff 02          	cmp    $0x2,%rdi
    113d:	7f 06                	jg     1145 <fibNaive+0xc>
    113f:	b8 01 00 00 00       	mov    $0x1,%eax
    1144:	c3                   	ret
    1145:	55                   	push   %rbp
    1146:	53                   	push   %rbx
    1147:	48 83 ec 08          	sub    $0x8,%rsp
    114b:	48 89 fb             	mov    %rdi,%rbx
    114e:	48 8d 7f fe          	lea    -0x2(%rdi),%rdi
    1152:	e8 e2 ff ff ff       	call   1139 <fibNaive>
    1157:	48 89 c5             	mov    %rax,%rbp
    115a:	48 8d 7b ff          	lea    -0x1(%rbx),%rdi
    115e:	e8 d6 ff ff ff       	call   1139 <fibNaive>
    1163:	48 01 e8             	add    %rbp,%rax
    1166:	48 83 c4 08          	add    $0x8,%rsp
    116a:	5b                   	pop    %rbx
    116b:	5d                   	pop    %rbp
    116c:	c3                   	ret
...
```

Based on these observations, I made a symbolization tool
[addr2func](https://github.com/marselester/diy-parca-agent/blob/main/cmd/addr2func/main.go)
and proposed a change in Parca, see [#2910](https://github.com/parca-dev/parca/pull/2910).
