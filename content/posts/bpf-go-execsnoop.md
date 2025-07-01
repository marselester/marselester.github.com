title: BPF: Go frontend for execsnoop
date: 2021-10-26
tags: bpf, golang
category: Go
slug: bpf-go-frontend-for-execsnoop

After reading Brendan Gregg's books about BPF I was excited to try and implement something in Go.
At a first glance it turned out I would need to install LLVM, Clang, and kernel header dependencies to run a simple program.
Fortunately [BTF, CO-RE technologies](https://www.brendangregg.com/blog/2020-11-04/bpf-co-re-btf-libbpf.html)
eliminate those dependencies though kernel >=5.8 should be used.

Since I don't know C, I resorted to [BCC libbpf-tools](https://github.com/iovisor/bcc/tree/master/libbpf-tools),
from which I picked a few BPF C programs (tcpconnect.bpf.c, execsnoop.bpf.c) to write Go frontends.
Below I will be focusing on execsnoop,
and you can find my experiments at [github.com/marselester/libbpf-tools](https://github.com/marselester/libbpf-tools).

## Preparing the environment

First things first, I ran Ubuntu 21.04 in a virtual machine and installed Clang, Go.

```console
﹪ cat > Vagrantfile <<CFG
Vagrant.configure("2") do |config|
    config.vm.box = "ubuntu/hirsute64"
end
CFG
﹪ vagrant up
﹪ vagrant ssh
﹩ sudo apt-get update
﹩ sudo apt-get install clang
﹩ sudo snap install go --classic
﹩ uname -nr
ubuntu-hirsute 5.11.0-37-generic
﹩ clang -v
Ubuntu clang version 12.0.0-3ubuntu1~21.04.2
```

Then I made sure the execsnoop program from BCC toolkit showed new processes
(it prints one line for every `execve` syscall),
so I could compare its output later with the Go implementation.

The ELF binary `execsnoop` was copied from a Docker container to the virtual machine as follows.
In case you wanted to build the image yourself,
here is a [Dockerfile](https://gist.github.com/marselester/ac8e4262742c90539c8c53f37d9a6965).

```console
﹪ docker run --name libbpf marselester/libbpf-tools:latest
﹪ docker cp libbpf:/opt/execsnoop .
```

The execsnoop captured `sshd` and `ls` processes, nice!

```console
﹩ sudo /vagrant/execsnoop
PCOMM            PID    PPID   RET ARGS
sshd             21254  832      0 /usr/sbin/sshd -D -R
ls               21361  21351    0 /usr/bin/ls --color=auto
```

It can be useful to check for short-lived processes that consume CPU resources,
e.g., slow or failing application/container startup.

## Embedding BPF in Go

My search for a BPF Go library stopped at [github.com/cilium/ebpf](https://github.com/cilium/ebpf)
since it had several examples to build upon, and folks from Cilium were [very helpful](https://github.com/cilium/ebpf/issues/303).

The repository also contains [bpf2go](https://github.com/cilium/ebpf/tree/master/cmd/bpf2go) tool to compile a BPF C source file into BPF bytecode.
It emits Go files containing compiled BPF for little and big endian systems.

I copied [execsnoop.bpf.c](https://github.com/iovisor/bcc/blob/master/libbpf-tools/execsnoop.bpf.c),
[execsnoop.h](https://github.com/iovisor/bcc/blob/master/libbpf-tools/execsnoop.h) files from BCC toolkit
and tried to generate Go files.

```console
﹩ cd /vagrant/
﹩ mkdir -p cmd/execsnoop/bpf/
﹩ curl -o cmd/execsnoop/bpf/execsnoop.bpf.c https://raw.githubusercontent.com/iovisor/bcc/master/libbpf-tools/execsnoop.bpf.c
﹩ curl -o cmd/execsnoop/bpf/execsnoop.h https://raw.githubusercontent.com/iovisor/bcc/master/libbpf-tools/execsnoop.h
﹩ cat > cmd/execsnoop/main.go <<<'''
package main

//go:generate go run github.com/cilium/ebpf/cmd/bpf2go -cflags $BPF_CFLAGS -cc clang-12 ExecSnoop ./bpf/execsnoop.bpf.c -- -I../../headers

func main() {}
'''
﹩ go mod init execsnoop
﹩ go get github.com/cilium/ebpf/cmd/bpf2go
﹩ go generate ./cmd/execsnoop/
/vagrant/cmd/execsnoop/bpf/execsnoop.bpf.c:2:10: fatal error: 'vmlinux.h' file not found
```

Unfortunately that didn't work because the `vmlinux.h` header was missing.
This file contains kernel's type definitions that can be generated from the installed kernel using bpftool tool.

```console
﹩ sudo apt-get install linux-tools-$(uname -r) linux-tools-common
﹩ bpftool btf dump file /sys/kernel/btf/vmlinux format c > ./headers/vmlinux.h
```

There were three more header files missing that I copied from ubuntu-kernel and libbpf repositories.

```console
﹩ mkdir -p headers/bpf/
﹩ curl -o ./headers/bpf/bpf_helpers.h https://git.launchpad.net/~ubuntu-kernel/ubuntu/+source/linux/+git/hirsute/plain/tools/lib/bpf/bpf_helpers.h
﹩ curl -o ./headers/bpf/bpf_helper_defs.h https://raw.githubusercontent.com/libbpf/libbpf/master/src/bpf_helper_defs.h
﹩ curl -o ./headers/bpf/bpf_core_read.h https://git.launchpad.net/~ubuntu-kernel/ubuntu/+source/linux/+git/hirsute/plain/tools/lib/bpf/bpf_core_read.h
```

Finally the Go files were generated and the BPF program was loaded into the kernel from an ELF.

```console
﹩ go generate ./cmd/execsnoop/
Compiled /vagrant/cmd/execsnoop/execsnoop_bpfel.o
Wrote /vagrant/cmd/execsnoop/execsnoop_bpfel.go
Compiled /vagrant/cmd/execsnoop/execsnoop_bpfeb.o
Wrote /vagrant/cmd/execsnoop/execsnoop_bpfeb.go
﹩ sudo go run ./cmd/execsnoop/
```

Note, bpf2go tool uses Clang to compile the BPF program:

- `-cflags $BPF_CFLAGS` are flags passed to the compiler from the `$BPF_CFLAGS` env var,
  for example, `BPF_CFLAGS='-D__TARGET_ARCH_x86' go generate` is expected in
  [bpf_tracing.h](https://elixir.bootlin.com/linux/latest/source/tools/lib/bpf/bpf_tracing.h#L5) header
  (it is not used in execsnoop though)
- `-cc clang-12` is a Clang binary name
- `ExecSnoop` is a name used by a code generation
- `./bpf/execsnoop.bpf.c` is a path to the BPF C file that is compiled
- `-- -I../../headers` are compiler options that indicate where to find header files

```go
package main

import (
	"log"

	"golang.org/x/sys/unix"
)

//go:generate go run github.com/cilium/ebpf/cmd/bpf2go -cflags $BPF_CFLAGS -cc clang-12 ExecSnoop ./bpf/execsnoop.bpf.c -- -I../../headers

func main() {
	// Increase the resource limit of the current process to provide sufficient space
	// for locking memory for the BPF maps.
	if err := unix.Setrlimit(unix.RLIMIT_MEMLOCK, &unix.Rlimit{
		Cur: unix.RLIM_INFINITY,
		Max: unix.RLIM_INFINITY,
	}); err != nil {
		log.Printf("failed to set temporary RLIMIT_MEMLOCK: %v", err)
		return
	}

	// Load the BPF program into the kernel from an ELF.
	// ExecSnoopObjects contains all objects (BPF programs and maps) after they have been loaded into the kernel:
	// TracepointSyscallsSysEnterExecve and TracepointSyscallsSysExitExecve BPF programs,
	// Events and Execs BPF maps.
	objs := ExecSnoopObjects{}
	if err := LoadExecSnoopObjects(&objs, nil); err != nil {
		log.Printf("failed to load BPF programs and maps: %v", err)
		return
	}
	defer objs.Close()
}
```

## Tracepoints

The BPF program is executed on events such as tracepoints.
Tracepoint is a static instrumentation (hard-coded to the source code) of the Linux kernel,
see `/sys/kernel/debug/tracing/events` to find available tracepoints.

From looking at BPF C source file `execsnoop.bpf.c` we can see two tracepoints instrumented:
the entry and the return of the `execve` syscall so that
the process ID, command name, arguments, and the return value can be printed.

```cpp
SEC("tracepoint/syscalls/sys_enter_execve")
int tracepoint__syscalls__sys_enter_execve(struct trace_event_raw_sys_enter *ctx) { ... }

SEC("tracepoint/syscalls/sys_exit_execve")
int tracepoint__syscalls__sys_exit_execve(struct trace_event_raw_sys_exit *ctx) { ... }
```

`SEC` declares an ELF section named `tracepoint/syscalls/sys_enter_execve`, followed by a BPF program.
A user-level loader may use this section header to determine where to attach a program.

`tracepoint__syscalls__sys_enter_execve` function is called for the tracepoint event.
The struct `trace_event_raw_sys_enter` argument contains register state and BPF context.
From the registers, function arguments `ctx->args` and return value `ctx->ret` can be read.

On Go side I attached the BPF program to `syscalls/sys_enter_execve` tracepoint using `github.com/cilium/ebpf/link` package.
The `link.Tracepoint()` function returns a link (it represents a program attached to a BPF hook)
which is later used to detach the program.

```go
tpEnter, err := link.Tracepoint("syscalls", "sys_enter_execve", objs.ExecSnoopPrograms.TracepointSyscallsSysEnterExecve)
if err != nil {
	log.Printf("failed to attach the BPF program to sys_enter_execve tracepoint: %v", err)
	return
}
defer tpEnter.Close()

tpExit, err := link.Tracepoint("syscalls", "sys_exit_execve", objs.ExecSnoopPrograms.TracepointSyscallsSysExitExecve)
if err != nil {
	log.Printf("failed to attach the BPF program to sys_exit_execve tracepoint: %v", err)
	return
}
defer tpExit.Close()
```

The `bpf_perf_event_output()` C function emits records to user space via a BPF map `events`
that accesses perf per-CPU ring buffers.

```cpp
struct {
	__uint(type, BPF_MAP_TYPE_PERF_EVENT_ARRAY);
	__uint(key_size, sizeof(u32));
	__uint(value_size, sizeof(u32));
} events SEC(".maps");

struct event e;
bpf_perf_event_output(ctx, &events, BPF_F_CURRENT_CPU, &e, sizeof(e));
```

In user space the events are read using `github.com/cilium/ebpf/perf` package.

```go
// Open a perf event reader from user space on the PERF_EVENT_ARRAY map
// defined in the BPF C program.
rd, err := perf.NewReader(objs.ExecSnoopMaps.Events, os.Getpagesize())
if err != nil {
	log.Printf("failed to create perf event reader: %v", err)
	return
}
defer rd.Close()

for {
	record, err := rd.Read()
	if err != nil {
		if perf.IsClosed(err) {
			break
		}
		log.Printf("failed to read from perf ring buffer: %v", err)
	}
	log.Printf("received from perf ring buffer: %s", record.RawSample)
}
```

In order to verify that the events are captured, I ran the Go program and `ls` in another terminal.
Luckily it worked!

```console
﹩ sudo go run ./cmd/execsnoop/
received from perf ring buffer: ?????ls/usr/bin/ls--color=auto
```

## Decoding events

I wanted to match the output of the original execsnoop program, for that I needed to decode `record.RawSample`
which contains bytes of the `event` C struct into a Go struct.

```cpp
struct event {
	pid_t pid;
	pid_t ppid;
	uid_t uid;
	int32 retval; // it's int in https://raw.githubusercontent.com/iovisor/bcc/master/libbpf-tools/execsnoop.h
	int32 args_count; // int
	u32 args_size; // unsigned int
	char comm[16];
	char args[7680];
};
```

Go `event` struct represents a perf event sent to user space from the BPF program running in the kernel.
Note, that it should match the C event struct, and both C and Go structs must be aligned the same way.

```go
type event struct {
	// PID is the process ID.
	PID int32
	// PPID is the process ID of the parent of this process.
	PPID int32
	// UID is the process user ID, e.g., 1000.
	UID uint32
	// Retval is the return value of the execve().
	Retval int32
	// ArgsCount is a number of arguments.
	ArgsCount int32
	// ArgSize is a size of arguments in bytes.
	ArgsSize uint32
	// Comm is the parent process/command name, e.g., bash.
	Comm [16]byte
}
```

The variable length `args` field is omitted on Go side and decoded manually,
because I kept getting `failed to parse perf event: unexpected EOF error`.
Perhaps the Go program got trailing garbage from the ring buffer which couldn't be unmarshalled.

```go
// The data submitted via bpf_perf_event_output.
// Due to a kernel bug, this can contain between 0 and 7 bytes of trailing
// garbage from the ring depending on the input sample's length.
// See https://pkg.go.dev/github.com/cilium/ebpf/perf#Record.
RawSample []byte
```

Timo Beckers in Cilium's Slack channel recommended to check alignment and hand-write an unmarshaller.
Here you can find my attempts to verify alignments [#3](https://github.com/marselester/libbpf-tools/pull/3)
with alignchecker program.

> Everything up until ArgsCount should be aligned on 8 bytes, but I'd expect the Go compiler to insert 4 bytes of padding between ArgsSize and Args, which would lead to an underrun on the C side. I've found hand-writing binary unmarshalers for these structs to be the most robust, but also the most annoying if your event format changes frequently.
>
> Cilium has a few alignchecker.go files if you're looking to dive deeper, we compare the DWARF info from the eBPF ELFs to Go struct fields annotated with align tags to make sure C and Go alignment corresponds

`binary.Read()` is used to read structured binary data from `record.RawSample` into the `event` struct.
The remaining bytes belong to `args` field and just printed.

```go
// eventSize is the event struct size in bytes, i.e., unsafe.Sizeof(e).
// It's used to obtain the variable length args from the perf record.
const eventSize = 40

fmt.Printf("%-16s %-6s %-6s %3s %s\n", "PCOMM", "PID", "PPID", "RET", "ARGS")
for {
	record, err := rd.Read()
	if err != nil {
		if perf.IsClosed(err) {
			break
		}
		log.Printf("failed to read from perf ring buffer: %v", err)
	}

	if record.LostSamples != 0 {
		log.Printf("ring event perf buffer is full, dropped %d samples", record.LostSamples)
		continue
	}

	var e event
	err = binary.Read(
		bytes.NewBuffer(record.RawSample),
		binary.LittleEndian,
		&e,
	)
	if err != nil {
		log.Printf("failed to parse perf event: %v", err)
		continue
	}

	args := record.RawSample[eventSize:]

	fmt.Printf(
		"%-16s %-6d %-6d %3d %s\n",
		bytes.TrimRight(e.Comm[:], "\x00"),
		e.PID,
		e.PPID,
		e.Retval,
		strings.ReplaceAll(
			string(bytes.Trim(args, "\x00")),
			"\x00",
			" ",
		),
	)
}
```

The resulting output of the Go execsnoop program is very close to the original 🎉.

```console
﹩ sudo go run ./cmd/execsnoop/
PCOMM            PID    PPID   RET ARGS
ls               69672  60093    0 /usr/bin/ls --color=auto
sh               69783  69782    0 /bin/sh -c    cd / && run-parts --report /etc/cron.hourly
run-parts        69784  69783    0 /usr/bin/run-parts --report /etc/cron.hourly
```

You can find all the source code at
[github.com/marselester/libbpf-tools](https://github.com/marselester/libbpf-tools).

## Appendix

[BPF performance tools](https://www.brendangregg.com/bpf-performance-tools-book.html) book
aided to navigate BPF C code.
Here are a few notes that are relevant to execsnoop program.

A BPF program has to use [helpers](https://elixir.bootlin.com/linux/v5.11/source/include/uapi/linux/bpf.h#L712)
because it can't access arbitrary memory (outside of BPF) and can't call arbitrary kernel functions.
The `execsnoop.bpf.c` makes use of the following helpers.

`bpf_probe_read_user(dst, size, unsafe_ptr)` safely attempts to read *size* bytes from user space address
*unsafe_ptr* and stores the data in *dst*.

`bpf_get_current_pid_tgid()` returns an unsigned 64-bit integer containing the current TGID (what user space calls the PID) in the upper bits
and the current PID (what user space calls the kernel thread ID) in the lower bits.

```cpp
u64 id = bpf_get_current_pid_tgid();
pid_t pid = (pid_t)id;
pid_t tgid = id >> 32;
e->pid = tgid;
```

`bpf_get_current_uid_gid()` returns a u64 integer containing the current GID in the upper bits
and UID in the lower bits.

```cpp
uid_t uid = (u32)bpf_get_current_uid_gid();
```

`bpf_get_current_task()` returns a pointer to the current (on-CPU)
[task struct](https://elixir.bootlin.com/linux/v5.11/source/include/linux/sched.h#L649) (a process).
This contains many details about the running process and links to other structs containing system state.

```cpp
struct task_struct *t;
t = (struct task_struct*)bpf_get_current_task();
```

`bpf_get_current_comm(buf, buf_size)` copies the task name to the buffer.

```cpp
bpf_get_current_comm(&e->comm, sizeof(e->comm));
```

`bpf_perf_event_output(ctx, map, data, size)` writes data to the `perf_event` ring buffers;
this is used for per-event output.

`bpf_map_lookup_elem(map, key)` finds a key in a map and returns its value (pointer).

```cpp
struct event *e;
e = bpf_map_lookup_elem(&execs, &pid);
```

`bpf_map_update_elem(map, key, value, flags)` atomically updates the value of the entry selected by key.

```cpp
static const struct event empty_event = {};

struct {
	__uint(type, BPF_MAP_TYPE_HASH);
	__uint(max_entries, 10240);
	__type(key, pid_t);
	__type(value, struct event);
} execs SEC(".maps");

if (bpf_map_update_elem(&execs, &pid, &empty_event, BPF_NOEXIST))
	return 0;
```

The *execs* is declared of type `BPF_MAP_TYPE_HASH` (a hash table).
The key is a `pid_t`, and the value is struct `event`.
*BPF_NOEXIST* flag specifies that the entry for key must not exist in the map.

BPF supports the following map types:

- `BPF_MAP_TYPE_HASH` is a hash table: key/value pairs
- `BPF_MAP_TYPE_ARRAY` is an array of elements
- `BPF_MAP_TYPE_PERF_EVENT_ARRAY` is an interface to the `perf_event` ring buffers
  for emitting trace records to user space
- `BPF_MAP_TYPE_PERCPU_HASH` is a faster hash table maintained on a per-CPU basis
- `BPF_MAP_TYPE_STACK_TRACE` is a storage for stack traces, indexed by stack IDs
- `BPF_MAP_TYPE_STACK` is a storage for stack traces

There are three ways to output data from kernel to user:

- `BPF_PERF_OUTPUT()` is a way to send per-event details to user space, via a custom struct you define.
- BPF maps using `bpf_perf_event_output()` (periodically read from user space)
- `bpf_trace_printk()` writes to trace_pipe (debugging only).
  I wasn't sure about `pid_t` size so I used `bpf_printk("the PID is %d", sizeof(pid));`
  and printed the trace buffer as follows (see [BPF tips & tricks](https://nakryiko.com/posts/bpf-tips-printk/)).

```console
﹩ sudo cat /sys/kernel/debug/tracing/trace_pipe
<...>-487422  [001] .... 544705.474270: 0: the PID is 4
```
