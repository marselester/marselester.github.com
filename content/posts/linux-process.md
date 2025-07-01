title: Linux process
date: 2023-01-31
tags: architecture, linux
category: Infrastructure
slug: linux-process

Being curious about BPF, I studied source code of several programs from the BCC libbpf-tools.
BPF performance tools book aided me to navigate BPF C code.
For example, it explained that a BPF program has to use helpers because it can't access arbitrary memory (outside of BPF) and can't call arbitrary Linux kernel functions.

BPF has opened a door into the kernel for me, but I quickly realized that I don't know much about it.
Take, for example, `bpf_get_current_pid_tgid()` and `bpf_get_current_task()` helpers.
What are `tgid`, `pid`, and `task` in the function names?
It turns out that `tgid` is what user space calls a process ID,
and `pid` is what user space calls a thread ID.

```cpp
u64 id = bpf_get_current_pid_tgid();
pid_t pid = (pid_t)id;
pid_t tgid = id >> 32;

struct task_struct *t;
t = (struct task_struct*)bpf_get_current_task();
```

I found the terminology a bit confusing, so I spent some time trying to clarify it for myself.

## Task

A process is an instance of a program in execution
that provides a program an illusion that it is the only one currently running
with exclusive use of CPU and memory.
A thread (thread of execution) is a unit of execution (sequence of machine instructions)
that can be managed by an OS scheduler.
Typically threads share the process's resources, e.g., memory and file descriptors.

The process and the thread are abstractions whose implementation depends on the OS.
Linux implements those abstractions with "light weight processes" which can share resources with each other,
thus in a way blending the process and the thread concepts.
For example, `ps` shows parca-agent Go program running as 8 light weight processes (also referred to as threads in the man page).
As we can see, they all have unique light weight process ID (LWP column), and the same process ID 93015 (PID column).
Together they form the thread group with ID 93015 which acts as a whole,
e.g., we can terminate it by sending a TERM signal `kill 93015`.
Note, the light weight process whose PID and LWP columns contain 93015 indicates that it was created first (the main thread).

```console
﹩ ps -aL
    PID     LWP TTY          TIME CMD
  93013   93013 pts/0    00:00:00 sudo
  93015   93015 pts/3    00:00:33 parca-agent
  93015   93016 pts/3    00:00:24 parca-agent
  93015   93017 pts/3    00:00:30 parca-agent
  93015   93018 pts/3    00:00:00 parca-agent
  93015   93019 pts/3    00:00:00 parca-agent
  93015   93020 pts/3    00:00:00 parca-agent
  93015   93021 pts/3    00:00:30 parca-agent
  93015   93022 pts/3    00:00:32 parca-agent
  94130   94130 pts/1    00:00:00 ps
```

Linux kernel maintains a [task_struct](https://elixir.bootlin.com/linux/v6.1.8/source/include/linux/sched.h#L737) for each light weight process.
Therefore there should be 8 such structures with `tgid = 93015`
(thread group ID or PID column)
and `pid` (LWP column) in a range from 93015 to 93022.
The maximum PID value is 4194304,
see [PID_MAX_LIMIT](https://elixir.bootlin.com/linux/v6.1.8/source/include/linux/threads.h#L34)
and `/proc/sys/kernel/pid_max`.

```cpp
struct task_struct {
    // Possible states: running, interruptible (sleeping),
    // uninterruptible, stopped, traced.
    unsigned int                __state;
    // Pointer to kernel stack (each task has one),
    // see https://www.kernel.org/doc/html/latest/x86/kernel-stacks.html.
    // General-purpose registers (e.g., rax, rbx)
    // used by a task in user mode are saved on
    // the kernel stack before performing process switching.
    void                        *stack;
    // Points to the previous and to the next task_struct
    // thus representing a list of all tasks in the system.
    struct list_head            tasks;
    // Points to a structure that describes
    // the current state of virtual memory.
    struct mm_struct            *mm;
    // Possible exit states: zombie, dead.
    int                         exit_state;
    int                         exit_code;
    // ID of this light weight process.
    pid_t                       pid;
    // ID of the thread group (PID column in ps).
    pid_t                       tgid;
    // Points to the task that created this task
    // or to the init task if the parent no longer exists.
    struct task_struct __rcu    *real_parent;
    // A list of all children created by this task.
    struct list_head            children;
    // Points to the next and previous sibling tasks.
    struct list_head            sibling;
    // Points to the process group leader (PGID),
    // e.g., of "sleep 10 | sleep 20".
    struct task_struct          *group_leader;
    // Executable name, excluding path.
    char                        comm[TASK_COMM_LEN];
    // Open file information (contains pointers to file descriptors).
    struct files_struct         *files;
    // CPU-specific state of this task is stored here
    // when the task is being switched out, i.e.,
    // most of the CPU registers (es, ds, fs, gs, FPU registers),
    // except the general-purpose registers.
    // The sp0 and sp fields are the user and
    // kernel stack pointers respectively.
    struct thread_struct        thread;
}
```

All `task_struct` that currently present in the system are linked using a double linked list.
When we run `kill 93015` to terminate 8 light weight processes by `tgid`,
the kernel looks up the thread group's leader by ID in
a [hash table](https://elixir.bootlin.com/linux/v3.19.8/source/kernel/pid.c#L44)
and then walks the list of the group.
That applies at least to the kernel v3, and the v6 seems to be using a radix tree,
see [pid.c](https://elixir.bootlin.com/linux/v6.1.8/source/kernel/pid.c),
[idr.c](https://elixir.bootlin.com/linux/v6.1.8/source/lib/idr.c),
[ID allocation](https://www.kernel.org/doc/html/latest/core-api/idr.html).

With namespaces each PID may have several values, with each one being valid in one namespace.
Linux kernel has a [pid](https://elixir.bootlin.com/linux/v6.1.8/source/include/linux/pid.h) structure
that refers to individual tasks, process groups, and sessions.
Check out [#26779416](https://stackoverflow.com/questions/26779416/what-is-the-relation-between-task-struct-and-pid-namespace) stackoverflow answer for more details.

<details>

```cpp
/*
 * struct upid is used to get the id of the struct pid, as it is
 * seen in particular namespace. Later the struct pid is found with
 * find_pid_ns() using the int nr and struct pid_namespace *ns.
 */
struct upid {
    // The pid value.
    int nr;
    // The namespace this value is visible in.
    struct pid_namespace *ns;
};

/*
 * What is struct pid?
 *
 * A struct pid is the kernel's internal notion of a process identifier.
 * It refers to individual tasks, process groups, and sessions.  While
 * there are processes attached to it the struct pid lives in a hash
 * table, so it and then the processes that it refers to can be found
 * quickly from the numeric pid value.  The attached processes may be
 * quickly accessed by following pointers from struct pid.
 *
 * Storing pid_t values in the kernel and referring to them later has a
 * problem.  The process originally with that pid may have exited and the
 * pid allocator wrapped, and another process could have come along
 * and been assigned that pid.
 *
 * Referring to user space processes by holding a reference to struct
 * task_struct has a problem.  When the user space process exits
 * the now useless task_struct is still kept.  A task_struct plus a
 * stack consumes around 10K of low kernel memory.  More precisely
 * this is THREAD_SIZE + sizeof(struct task_struct).  By comparison
 * a struct pid is about 64 bytes.
 *
 * Holding a reference to struct pid solves both of these problems.
 * It is small so holding a reference does not consume a lot of
 * resources, and since a new struct pid is allocated when the numeric pid
 * value is reused (when pids wrap around) we don't mistakenly refer to new
 * processes.
 */
struct pid {
    // Reference counter.
    refcount_t count;
    // The number of upids.
    unsigned int level;
    // Lists of tasks that use this pid.
    struct hlist_head tasks[PIDTYPE_MAX];
    struct hlist_head inodes;
    struct upid numbers[1];
};
```

</details>

## Address space

A process provides a program an illusion that
it has exclusive use of the whole memory address space in the system
using virtual memory abstraction.

Physical memory is organized as an array of bytes and
it's partitioned into fixed-size blocks
(usually 4KB depending on CPU architecture) called page frames.
A virtual page can be:

- cached in DRAM — backed by a page frame, i.e.,
  4KB of data is already loaded to physical memory from disk
- uncached in DRAM — doesn't occupy physical memory yet,
  but is already associated with a 4KB part of a file on disk.
  Once some virtual address within this virtual page is accessed by the CPU
  causing a page fault exception (DRAM cache miss),
  the 4KB of data will be loaded from disk to memory.
- unallocated — NULL, doesn't point to physical memory or to disk

This mapping is stored in a data structure called page table.
A memory management unit (MMU) relies on page tables
to translate virtual addresses to physical addresses.
Here is how a process's virtual address space could approximately look.

```console
 _________________________
| page tables,            |
| task and mm structs     |
|-------------------------|
| kernel stack            | ⬇️
|-------------------------| rsp (stack pointer in kernel mode)
| thread info struct      |
|-------------------------|
| physical memory         |
|-------------------------|
| kernel code & data      |
|_________________________| top part is reserved for the kernel
| arg, env                |
|-------------------------|
| stack for main thread   | ⬇️
|-------------------------| rsp (stack pointer in user mode)
| ...                     |
|-------------------------|
| stack for thread 3      | ⬇️
|-------------------------|
| stack for thread 2      | ⬇️
|-------------------------|
| stack for thread 1      | ⬇️
|-------------------------|
| shared libraries        |
|-------------------------|
| ...                     |
|-------------------------| brk (top of the heap)
| heap                    | ⬆️
|-------------------------|
| uninitialized data .bss |
|-------------------------| vm_end
| initialized data .data  |
|-------------------------| vm_start
| code .text              | thread 3 executing
|                         | main thread executing
|                         | thread 1 executing
|                         | thread 2 executing
|                         |
|-------------------------| 0x400000
| ...                     |
|_________________________| 0
```

The bottom part describes user space addresses of a process.
The top part is a kernel virtual memory:

- process-specific structures such as page tables, task and mm structs,
  kernel stack, thread info structure.
- physical memory part which is identical for each process.
  Linux maps a set of virtual pages equal in size to DRAM
  to the corresponding set of physical pages
  to access any specific location in physical memory,
  e.g., to access page tables.
- kernel code and data which is identical for each process

Linux organizes the virtual memory as a collection of areas (segments)
where an area is a chunk of related pages, e.g., code, data,
heap, shared libraries, user stack segments.
A process can create arbitrary number of areas using `mmap` function.

The `vm_area_struct` structures keep track of virtual memory areas of a task.
They can be reached via `task->mm->mm_mt` tree.
The fields `vma.vm_start` and `vma.vm_end` point to the beginning and the end of an area.

<details>

```cpp
// See https://elixir.bootlin.com/linux/v6.1.8/source/include/linux/mm_types.h#L512.

struct mm_struct {
    struct {
        // A tree to look up vm_area_struct by user address.
        struct maple_tree mm_mt;
        // Memory mapping.
        unsigned long mmap_base;
        // Points to a page global directory,
        // i.e., to the first entry of the level 1 page table.
        // The physical address of PGD in use is stored in the cr3 control register.
        pgd_t *pgd;
        // Addresses of text and data segments.
        unsigned long start_code, end_code, start_data, end_data;
        // Addresses of heap and stack segments.
        unsigned long start_brk, brk, start_stack;
        unsigned long arg_start, arg_end, env_start, env_end;
    }
}

/*
 * This struct describes a virtual memory area. There is one of these
 * per VM-area/task. A VM area is any part of the process virtual memory
 * space that has a special rule for the page-fault handlers (ie a shared
 * library, the executable area).
 */
struct vm_area_struct {
    // Points to the beginning of the area within vm_mm.
    unsigned long vm_start;
    // Points to the end of the area within vm_mm.
    unsigned long vm_end;
    // Points to the address space this area belongs to.
    struct mm_struct *vm_mm;
    // Access permissions of this VMA, e.g., read/write.
    pgprot_t vm_page_prot;
    // Describes whether the pages in the area are shared with other tasks
    // or private to this task.
    unsigned long vm_flags;
    // The offset (within vm_file) in PAGE_SIZE units (number of pages).
    unsigned long vm_pgoff;
    // The file backing this mapping.
    // It can be NULL, e.g., anonymous mapping (stack, heap, bss).
    struct file *vm_file;
}
```

</details>

Linux can associate a virtual memory area with a contiguous section of a file,
e.g., executable object file.
That section (e.g., code segment) is divided into page-size chunks.
When the CPU accesses a virtual address that is within some page's region,
that virtual page (e.g., some content of an executable file)
is loaded to the physical memory from disk.
You can check out an example of a memory mapping in the context of
[CPU profiler](https://marselester.com/diy-cpu-profiler-from-bpf-maps-to-pprof.html).

An area can also be mapped (associated) to
an anonymous file (it doesn't exist on disk).
When the CPU accesses a virtual page within that area (e.g., heap or user stack),
the kernel finds a free page in physical memory, zeroes it,
and marks it as resident in the page table.
If no free pages exist (run out of physical memory),
then some dirty page gets swapped out to disk.

A file can be mapped as the shared object into areas of different processes.
The memory mappings are unique for each process, but the underlying physical pages are the same.
So if one process writes to its area, the changes are visible to other processes,
and the file is updated on disk as well.
Shared libraries are mapped as shared objects by many processes thus saving physical memory pages.

If a file is mapped as the private object,
the writes to that area are not visible to other processes and
the changes are not written to disk.
For example, two processes mapped a private object into their areas of virtual memories.
The same physical memory pages will be used as long as both processes only read data from those areas.
Once a process writes to its area, a new copy of pages is created in physical memory.
This copy-on-write allows to save memory, e.g., multiple processes share .text segment which is never modified
(recall 8 parca-agent light weight processes).

---

To wrap up, I must say that I might have misunderstood some of the parts
and posted inaccurate information, sorry about that.
The Linux kernel is a complicated project 🐧,
and I wouldn't be able to navigate it without books such as:

- Computer Systems, Randal E. Bryant, David R. O'Hallaron
- Understanding the Linux Kernel, Daniel P. Bovet, Marco Cesati
- The Linux Programming Interface: A Linux and UNIX System Programming Handbook, Michael Kerrisk
