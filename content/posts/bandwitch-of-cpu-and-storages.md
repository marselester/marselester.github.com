title: Bandwitch 🧙‍♀️ of CPU and storages
date: 2023-01-08
tags: architecture, performance
category: Performance
slug: bandwitch-of-cpu-and-storages

Let us begin with a definition of a CPU clock rate ⏰.
It refers to the frequency at which the clock generator (an oscillator crystal)
of a processor can generate pulses, which are used to synchronize the operations of its components.
For example, 1GHz CPU implies that its clock runs at 10^9 cycles per second,
therefore the time required for each clock cycle is 1 nanosecond.
Faster the clock -- more instructions a processor can execute per second.

Performance of a program depends on how fast the data gets accessed,
so the speed of a storage is paramount.
The CPU has the fastest storage device called the register file
which consists of 16 word-sized (64 bits) registers.
The data stored in a register requires 0 cycles to access.

The program counter (PC) `rip` register contains the memory address of a machine instruction.
CPU reads the instruction from memory pointed at by the PC,
executes the instruction, and updates the PC to point to the next instruction.
In general, CPU performs the following operations as per instruction:

- load -- copy a byte/word from main memory into register
- store -- copy a byte/word from register into main memory
- operate -- copy the contents of two registers to the arithmetic/logic unit (ALU),
  perform an arithmetic operation, and store the result in a register
- jump -- copy a word from the instruction into the PC

The memory addresses used by a program are virtual addresses,
so the main memory appears to be a byte array where array index is a memory address.
It can range from 0 to 2^63-1, but currently is
[limited to 2^48 or 256TB](https://en.wikipedia.org/wiki/X86-64#Virtual_address_space_details).
The lower-addressed half (user space) is occupied by a process,
and the rest is left to kernel virtual memory (it is identical for each process).
Here is how a 4-byte integer might be stored in memory (little endian byte ordering).

```go
// The variable's address &x is 0x100.
var x int32 = 0x00001e61
```

| memory address | byte
| ---            | ---
| 0x100          | 61
| 0x101          | 1e
| 0x102          | 00
| 0x103          | 00

Arrays and structures are stored as a contiguous sequence of bytes.
The smallest memory address of the bytes used is a variable's address.

Memory access is often the slowest of CPU operations, as it may take ~200 cycles to access it,
during which instruction execution has stalled.
Memory caches blocks of virtual memory (4-KB pages) and parts of files (buffer cache).
Disk controller caches disk sectors.
Its access time is ~100,000 cycles, and disk's access time is ~10,000,000 cycles.
Disks read and write data in sector-size blocks (512 bytes) which takes ~10ms.
CPU L1-L3 caches cache 64-byte blocks (cache lines) of the main memory
to significantly reduce the number of cycles needed for memory access.
L1 cache can be accessed in ~4 clock cycles, L2 in ~10 cycles, and L3's access time is ~50 cycles.

Buses transfer word-sized information between CPU, main memory, disk, etc.
It takes ~256ns for L1 cache to read 512 bytes (access time is ~4ns for a 64-bit word).

```console
Register file -- L1-3 caches -- Bus interface
                                 |
                                 | System bus
                                 |
Disk -- Disk controller --------- I/O bridge
                         I/O bus |
                                 | Memory bus
                                 |
                                Main memory
```

Each CPU core has its own L1 and L2 caches.
L1 is split into i-cache to read recently fetched instructions, and d-cache to read/write data.
All cores share L3 cache and the interface to main memory.

```console
Registers                  Registers
    |                          |
L1 d-cache  L1 i-cache ... L1 d-cache  L1 i-cache
          \/                         \/
        L2 cache                  L2 cache
                \                /
                 \-- L3 cache --/
```

An instruction requires ~20 clock cycles to execute,
but a processor manages to execute more of them per clock cycle using instruction pipelining,
i.e., the actions are partitioned into ~15 steps which are performed in parallel
working on different instructions, e.g.,

- fetching the instruction from memory
- determining the instruction type
- reading from memory
- arithmetic operation
- writing to memory
- updating the program counter

CPU can perform out-of-order execution of the pipeline,
where later instructions can be completed while earlier instructions are stalled,
improving instruction throughput.
The instruction control unit (ICU) fetches instructions from L1 i-cache ahead of time.
It generates a sequence of primitive operations from instructions
and sends them to the execution unit (EU).
EU executes the operations using multiple functional units
(e.g., store to L1 d-cache) some of which of the same type
and tells ICU whether branches were correctly predicted.
Mispredicting a conditional jump can incur ~15-30 clock cycles of wasted effort, hurting performance.

Memory management unit (MMU) uses translation look-aside buffer (TLB) which is a cache
that provides a fast translation (0 cycles access time) from virtual to physical addresses.
On TLB miss, MMU searches the address mapping in the page table which is maintained by the kernel.
When it fails, a page fault exception is signalled to get a referenced memory address from the disk.
The OS's exception handler sets up a transfer from disk to memory (several thousands cycles).
Once this completes (millions of cycles),
the OS will return to the original program to retry the instruction that caused a page fault.
Direct memory access (DMA) technique allows data to travel directly from disk to memory avoiding CPU.

## General advice

Intel Vtune defines CPI (cycles per instruction retired) as a metric indicating how much time
each executed instruction took, in units of cycles.
The CPU issues up to four instructions per cycle, suggesting a theoretical best CPI of 0.25.
A CPI < 1 is typical for instruction bound code,
while a CPI > 1 may show up for a stall cycle bound application, also likely memory bound.
If you're curious, check out a [Vtune perf report](https://github.com/klauspost/compress/discussions/717) example.

Hyper-threading allows the CPU core to run more than one thread,
effectively scheduling between them when one instruction stalls on memory I/O.
Workloads that are stall cycle-heavy (low IPC instructions per cycle)
could have better performance than those that are instruction-heavy
because stall cycles reduce core contention.

IPC < 1 (or CPI > 1) indicates that the CPU is often stalled,
typically for memory access, so a general advice is to:

- keep intermediate calculation values in registers (i.e., local variables) instead of the memory
  and store results in memory when the final value was computed
- use a value as often as possible once it has been read from memory (temporal locality),
  i.e., it will be read from L1 cache
- access data (arrays, structs) sequentially in the order they are stored in memory (spatial locality),
  so multiple values are read from the same L1 cache line
- reduce the data dependencies in a loop between different parts of a computation.
  Otherwise the CPU would have to wait for dependent load/store operations that form a critical path.
  Similar to databases, partitioning helps, e.g., introduce more variables to accumulate computed values.

Beware of false sharing -- a usage pattern occurring when two CPU cores read from and write to
unrelated variables that happen to share the same L1 cache line,
e.g., two pointers declared in the same struct next to each other.
When write happens in one core,
the cache controller invalidates that cache line in other cores,
so they have to reload non-stale data to do their reads.
The solution is to add 64-byte padding between those variables so they end up in different cache lines.
Check out an interesting case study:
[Cache invalidation really is one of the hardest problems in computer science](https://surfingcomplexity.blog/2022/11/25/cache-invalidation-really-is-one-of-the-hardest-things-in-computer-science/)
and [Seeing through hardware counters: a journey to threefold performance increase](https://netflixtechblog.com/seeing-through-hardware-counters-a-journey-to-threefold-performance-increase-2721924a2822).
That post also mentions true sharing -- a usage pattern occurring
when multiple cores read from and write to the same variable.
The CPU-enforced memory ordering causes slowdown
([coherency-limited scalability](https://go-talks.appspot.com/github.com/marselester/scalability/scalability.slide#3) due to inconsistent copies of data).

If you're curious about systems performance, I recommend Brendan Gregg's books,
Chris Kanich's [YouTube channel](https://www.youtube.com/@ChrisKanich),
Computer Systems book by Bryant and O'Hallaron.
