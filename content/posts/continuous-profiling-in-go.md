title: Continuous profiling in Go
date: 2022-04-22
tags: bpf, golang, monitoring
category: Go
slug: continuous-profiling-in-go

In this post I explore possibilities of continuous profiling of Go programs
using Parca and also peek under the hood of its BPF agent.
You can find the results of my experiments in
[github.com/marselester/diy-parca-agent](https://github.com/marselester/diy-parca-agent).

## Ad hoc profiling

I was looking for a way to profile Go programs running in Kubernetes
to see where a program spends its CPU time (which functions consume most CPU)
or where memory allocations happen.
Allocations in a frequently called function could cause GC pressure
which in turn could cause latency spikes in HTTP requests.

Latency of the Bookstore service is taken seriously,
so its API server already had `import _ "net/http/pprof"` to enable profiling with `go tool pprof` program.
In order to get access to the HTTP server running in the pod
a port forwarder could be used.

```console
﹩ kubectl get pods -A | grep bookstore
default                  bookstore-574897d6-e18e
﹩ kubectl port-forward -n default bookstore-574897d6-e18e 6060 &
Forwarding from 127.0.0.1:6060 -> 6060
Forwarding from [::1]:6060 -> 6060
﹩ fg 1
```

The following URLs under `/debug/pprof/` provide profiling data.

```console
＃ 30-second CPU profile.
﹩ curl http://0.0.0.0:6060/debug/pprof/profile --output ./cpu.pprof
＃ Heap allocations profile since the process started
＃ including garbage-collected bytes (useful when optimising code).
﹩ curl http://0.0.0.0:6060/debug/pprof/allocs --output ./heap-all.pprof
＃ Heap allocations of live objects (useful when looking for memory leaks).
﹩ curl http://0.0.0.0:6060/debug/pprof/heap --output ./heap-live.pprof
```

In order to inspect the heap profile `heap-all.pprof`
I would need to compile the API `server` program and fix the search path for source files:

- `source_path` is an absolute path to your source code (the Bookstore's git repository),
  e.g., on Mac it could be `/Users/bob/code/bookstore/`.
- `trim_path` is a path the profile expects the code to be at, e.g.,
  `/opt/bookstore/` because the binary was compiled in Docker by CI/CD

```console
﹩ GOOS=linux go build ./cmd/server/
﹩ go tool pprof -source_path="$(pwd)" -trim_path=/opt/bookstore/ ./server ./heap-all.pprof
﹪ web
﹪ top10 -cum
```

The `web` command opens an allocation graph of function calls.
The `top10 -cum` command shows top ten functions by allocations in that function
or a function it called (cumulative) down the stack:

- `flat` is a memory allocated by that function and is held by it
- `cum` is a memory allocated by that function or a function it called down the stack.
  When `flat` and `cum` numbers match,
  this might indicate the allocated memory is retained.

## Continuous profiling

I could be lucky and discover some obvious problem,
though most likely by the time the profiles are collected it is already too late
because a Kubernetes pod in question is already gone.

A continuous profiler could have collected the data
when a problem was occuring on production,
so I was curious to see what open source tooling is available.
Here is what I found so far:

- [Parca](https://www.parca.dev/)
- [Pyroscope](https://pyroscope.io/)
- [profefe](https://github.com/profefe/profefe)

All of them are capable, moreover Parca and Pyroscope have BPF agents.
I experimented with Parca more because I stumbled on it earlier.
So here is how Parca Server configuration looked like
when I set it up to collect profiles every minute from the Bookstore service.

```yaml
debug_info:
  bucket:
    type: "FILESYSTEM"
    config:
      directory: "./tmp"
  cache:
    type: "FILESYSTEM"
    config:
      directory: "./tmp"

scrape_configs:
  - job_name: "bookstore"
    scrape_interval: "1m"
    scrape_timeout: "15s"
    static_configs:
      - targets: [ 'bookstore.default:6060' ]
```

Note, the config above is part of
[kubernetes-manifest.yaml](https://github.com/parca-dev/parca/releases/download/v0.10.0/kubernetes-manifest.yaml).
Check out [Parca tutorial](https://www.parca.dev/docs/kubernetes) to learn more.

## BPF profiling agent

The cool things about BPF profiling agents are that they incur low overhead,
can capture user/kernel-space stack traces, and don't need much configuration
because they discover containers on the Kubernetes node where an agent is running.

A quick summary about BPF.
In the kernel, a BPF program is executed on each event.
The event types could be kprobes, uprobes, tracepoints, sockets, perf_events.
It can use BPF helpers to fetch kernel state, and BPF maps for storage.
In user space BPF maps or a perf buffer are periodically read and some output (summary) is shown to a user.

I was curious how Parca Agent was implemented.
According to its [design doc](https://github.com/parca-dev/parca-agent/blob/main/docs/design.md)
it attaches the BPF program to a Linux cgroup using
[perf_event_open()](https://man7.org/linux/man-pages/man2/perf_event_open.2.html) system call.
The call creates a file descriptor that allows measuring performance information.
It instructs the kernel to call the BPF program 100 times per second.

Here I annotated
[github.com/parca-dev/parca-agent/pkg/profiler/profiler.go](https://github.com/parca-dev/parca-agent/blob/1e935f4d5f7b4484f6cf3d4ee26340f3718ff37d/pkg/profiler/profiler.go#L270) code around the syscall.

```go
// To discover cgroups to profile in Kubernetes,
// Parca Agent first discovers all pods running on the node it is on,
// then discovers the primary PID of the cgroup using Kubernetes CRI (container runtime interface).
// For profiling purposes a perf_event cgroup is required, which is read from /proc/PID/cgroup.
cgroup, err := os.Open(...)
// The pid and cpu arguments allow specifying which process and CPU to monitor.
pid := int(cgroup.Fd())

for cpu := 0; cpu < runtime.NumCPU(); cpu++ {
	fd, err := unix.PerfEventOpen(
		&unix.PerfEventAttr{
			// PERF_TYPE_SOFTWARE event type indicates that
			// we are measuring software events provided by the kernel.
			Type: unix.PERF_TYPE_SOFTWARE,
			// Config is a Type-specific configuration.
			// PERF_COUNT_SW_CPU_CLOCK reports the CPU clock, a high-resolution per-CPU timer.
			Config: unix.PERF_COUNT_SW_CPU_CLOCK,
			// Size of attribute structure for forward/backward compatibility.
			Size: uint32(unsafe.Sizeof(unix.PerfEventAttr{})),
			// Sample could mean sampling period (expressed as the number of occurrences of an event)
			// or frequency (the average rate of samples per second).
			// See https://perf.wiki.kernel.org/index.php/Tutorial#Period_and_rate.
			// In order to use frequency PerfBitFreq flag is set below.
			// The kernel will adjust the sampling period to try and achieve the desired rate.
			Sample: 100,
			Bits: unix.PerfBitDisabled | unix.PerfBitFreq,
		},
		pid,
		cpu,
		// This argument allows event groups to be created.
		// A single event on its own is created with groupFd = -1
		// and is considered to be a group with only 1 member.
		-1,
		// PERF_FLAG_PID_CGROUP flag activates per-container system-wide monitoring.
		// In this mode, the event is measured only if the thread running on the monitored CPU
		// belongs to the designated container (cgroup).
		unix.PERF_FLAG_PID_CGROUP,
	)
}
```

### BPF C code

From looking at BPF C code
[parca-agent.bpf.c](https://github.com/parca-dev/parca-agent/blob/31253527651ebebb74c6200eb68fe9251479ed6b/parca-agent.bpf.c)
we can see the BPF program `do_sample`.
As mentioned above it's executed 100 times per second.
Each time it captures stack traces of the thread that is currently running on CPU.

```cpp
SEC("perf_event")
int do_sample(struct bpf_perf_event_data *ctx) { ... }
```

`SEC("perf_event")` denotes `BPF_PROG_TYPE_PERF_EVENT` BPF program type
which specifies the type of events that the BPF program attaches to (perf_events),
and the arguments for the events.

```cpp
u64 id = bpf_get_current_pid_tgid();
u32 tgid = id >> 32;
u32 pid = id;
```

[bpf_get_current_pid_tgid()](https://elixir.bootlin.com/linux/latest/source/include/uapi/linux/bpf.h#L1777)
BPF helper returns an unsigned 64-bit integer containing
the current TGID (what user space calls the PID) in the upper bits and
the current PID (what user space calls the kernel thread ID) in the lower bits.

```cpp
// Max amount of different stack trace addresses to buffer in the map.
#define MAX_STACK_ADDRESSES 1024
// Max depth of each stack trace to track.
#define MAX_STACK_DEPTH 127
// Stack trace value is 1 big byte array of the stack addresses.
typedef __u64 stack_trace_type[MAX_STACK_DEPTH];

// The stack_traces map holds an array of memory addresses,
// e.g., stack_traces[1253] = [0xdeadbeef, 0x123abcde]
// where 1253 is a stack ID.
struct {
    __uint(type, BPF_MAP_TYPE_STACK_TRACE);
    __uint(max_entries, MAX_STACK_ADDRESSES);
    __type(key, u32);
    __type(value, stack_trace_type);
} stack_traces SEC(".maps");

struct stack_count_key_t {
    u32 pid;
    int32 user_stack_id;
    int32 kernel_stack_id;
};

// The counts map keeps track of how many times a stack trace has been seen,
// e.g., counts[{10342, 1253, 0234}] = 45 times.
struct {
    __uint(type, BPF_MAP_TYPE_HASH);
    __uint(max_entries, 10240);
    __type(key, struct stack_count_key_t);
    __type(value, u64);
} counts SEC(".maps");
```

The BPF program records stack traces in special `stack_traces` BPF map
which is indexed by stack IDs.
The map value is an array of memory addresses that represent the code executed.

| stack ID | memory addresses
| ---      | ---
| 1253     | [ 0xdeadbeef; 0x123abcde; ... ]
| 0234     | [ 0x597be95a; 0xae5ee03; ... ]

The `counts` BPF map keeps track of how many times a stack trace has been seen:

- its key is a triple of PID, user-space stack ID, and kernel-space stack ID, see `struct stack_count_key_t { ... }`
- its value is the amount of times that stack trace has been observed

| stack_count_key_t     | seen
| ---                   | ---
| { 10342; 1253; 0234 } | 45

The maps are populated using
[bpf_get_stackid()](https://elixir.bootlin.com/linux/latest/source/include/uapi/linux/bpf.h#L2080)
and `bpf_map_lookup_or_try_init()` functions.

```cpp
// Create a key for "counts" map.
struct stack_count_key_t key = {.pid = tgid};
// Read user-space stack ID and insert memory addresses into stack_traces map.
// The positive or null stack id is returned on success,
// or a negative error in case of failure.
key.user_stack_id = bpf_get_stackid(ctx, &stack_traces, BPF_F_USER_STACK);
// Read kernel-space stack ID and insert memory addresses into stack_traces map.
key.kernel_stack_id = bpf_get_stackid(ctx, &stack_traces, 0);

u64 zero = 0;
u64 *seen;
seen = bpf_map_lookup_or_try_init(&counts, &key, &zero);
if (!seen)
    return 0;
// Atomically increments the seen counter.
__sync_fetch_and_add(seen, 1);
```

On Go side the BPF maps are read every 10 seconds,
get processed, and then purged to reset for the next iteration.

### DIY profiler

Parca Agent relies on [cgo bindings for libbpf](https://github.com/aquasecurity/libbpfgo) from Aqua Security.
I wanted to make something similar using
[github.com/cilium/ebpf](https://github.com/cilium/ebpf) which doesn't need cgo.

In order to prepare the environment I ran Ubuntu 21.10 in a virtual machine and installed Clang with Go.

```console
﹪ cat > Vagrantfile <<CFG
Vagrant.configure("2") do |config|
    config.vm.box = "ubuntu/impish64"
end
CFG
﹪ vagrant up
﹪ vagrant ssh
﹩ sudo apt-get update
﹩ sudo apt-get install clang
﹩ sudo snap install go --classic
﹩ uname -nr
ubuntu-impish 5.13.0-39-generic
﹩ clang -v
Ubuntu clang version 13.0.0-2
```

Then I copied
[parca-agent.bpf.c](https://github.com/parca-dev/parca-agent/blob/31253527651ebebb74c6200eb68fe9251479ed6b/parca-agent.bpf.c)
from parca-agent repository and tried to generate Go files using bpf2go tool.

```console
﹩ cd /vagrant/
﹩ mkdir -p cmd/profiler/bpf/
﹩ curl -o cmd/profiler/bpf/parca-agent.bpf.c https://raw.githubusercontent.com/parca-dev/parca-agent/31253527651ebebb74c6200eb68fe9251479ed6b/parca-agent.bpf.c
﹩ cat > cmd/profiler/main.go <<<'''
package main

//go:generate go run github.com/cilium/ebpf/cmd/bpf2go -cflags $BPF_CFLAGS -cc clang-13 ParcaAgent ./bpf/parca-agent.bpf.c -- -I../../headers

func main() {}
'''
﹩ go mod init diy-parca-agent
﹩ go get github.com/cilium/ebpf/cmd/bpf2go
﹩ go generate ./cmd/profiler/
/vagrant/cmd/profiler/bpf/parca-agent.bpf.c:11:10: fatal error: 'vmlinux.h' file not found
```

`vmlinux.h` can be generated from the installed kernel using bpftool tool.

```console
﹩ mkdir headers
﹩ sudo apt-get install linux-tools-$(uname -r) linux-tools-common
﹩ bpftool btf dump file /sys/kernel/btf/vmlinux format c > ./headers/vmlinux.h
```

The remaining missing headers I copied from ubuntu-kernel, libbpf, and cilium repositories.

```console
﹩ curl -o ./headers/bpf_core_read.h https://git.launchpad.net/~ubuntu-kernel/ubuntu/+source/linux/+git/impish/plain/tools/lib/bpf/bpf_core_read.h
﹩ curl -o ./headers/bpf_endian.h https://raw.githubusercontent.com/cilium/ebpf/master/examples/headers/bpf_endian.h
﹩ curl -o ./headers/bpf_helpers.h https://git.launchpad.net/~ubuntu-kernel/ubuntu/+source/linux/+git/impish/plain/tools/lib/bpf/bpf_helpers.h
﹩ curl -o ./headers/bpf_tracing.h https://git.launchpad.net/~ubuntu-kernel/ubuntu/+source/linux/+git/impish/plain/tools/lib/bpf/bpf_tracing.h
﹩ curl -o ./headers/bpf_helper_defs.h https://raw.githubusercontent.com/libbpf/libbpf/master/src/bpf_helper_defs.h
```

Finally, all the dependencies were resolved and Go code was generated.

```console
﹩ go generate ./cmd/profiler/
Compiled /vagrant/cmd/profiler/parcaagent_bpfel.o
Stripped /vagrant/cmd/profiler/parcaagent_bpfel.o
Wrote /vagrant/cmd/profiler/parcaagent_bpfel.go
Compiled /vagrant/cmd/profiler/parcaagent_bpfeb.o
Stripped /vagrant/cmd/profiler/parcaagent_bpfeb.o
Wrote /vagrant/cmd/profiler/parcaagent_bpfeb.go
```

Next step was to see whether `DoSample` BPF program is loaded into the kernel from an ELF.
`ParcaAgentObjects{}` contains all objects (BPF program and BPF maps) after they have been loaded into the kernel.

```go
objs := ParcaAgentObjects{}
if err := LoadParcaAgentObjects(&objs, nil); err != nil {
	log.Printf("failed to load BPF program and maps: %v", err)
	return
}
defer objs.Close()
```

It worked, no errors were printed.

```console
﹩ sudo go run ./cmd/profiler/
```

> A call to perf_event_open() creates a file descriptor
> that allows measuring performance information.
> Each file descriptor corresponds to one event that is measured.
> Events can be enabled and disabled via ioctl(2).
> https://man7.org/linux/man-pages/man2/perf_event_open.2.html

It's time to attach `DoSample` BPF program to perf event using
`unix.PerfEventOpen()` and `unix.IoctlSetInt()` functions.
Note, Parca Agent doesn't directly use `unix.IoctlSetInt()` and relies on
[AttachPerfEvent()](https://pkg.go.dev/github.com/aquasecurity/libbpfgo#BPFProg.AttachPerfEvent)
function from [github.com/aquasecurity/libbpfgo](http://github.com/aquasecurity/libbpfgo).
I found cilium related explanation [#548](https://github.com/cilium/ebpf/discussions/548)
how to achieve the same thing:

1. Load the BPF program into the kernel: `LoadParcaAgentObjects(&objs, nil)`
2. Open the perf event: `fd, _ := unix.PerfEventOpen(...)`.
3. Set the BPF program on the perf event:
  `unix.IoctlSetInt(fd, unix.PERF_EVENT_IOC_SET_BPF, objs.DoSample.FD())`
4. Enable the perf event:
   `unix.IoctlSetInt(fd, unix.PERF_EVENT_IOC_ENABLE, 0)`

Those steps worked as well.
I was hopeful to see whether stack traces were collected,
so I periodically printed the content of `Counts` BPF map.

```go
type stackCountKey struct {
	PID           uint32
	UserStackID   int32
	KernelStackID int32
}

var (
	key   stackCountKey
	value uint64
)
it := objs.ParcaAgentMaps.Counts.Iterate()
for it.Next(&key, &value) {
	fmt.Printf("%+v seen %d times\n", key, value)
}
if err := it.Err(); err != nil {
	log.Printf("failed to read from Counts map: %v", err)
}
```

The easiest way to reliably get stack traces was to run `top` in another terminal.
I made the profiler focus on its PID 15958 and it worked 🎉.

```console
﹩ top
﹩ sudo go run ./cmd/profiler/ -pid 15958
Waiting for stack traces...
{PID:15958 UserStackID:132 KernelStackID:114} seen 1 times
{PID:15958 UserStackID:709 KernelStackID:-14} seen 1 times # -14 indicates bpf_get_stackid() error.
{PID:15958 UserStackID:366 KernelStackID:30} seen 2 times
{PID:15958 UserStackID:674 KernelStackID:943} seen 1 times
```

The end of my post is an disappointing anticlimax because
I haven't looked yet at how to show the actual stack traces.
I will update [the repository](https://github.com/marselester/diy-parca-agent) when I have time.

Update: finally I figured out how to show stack traces, see
[DIY CPU profiler: from BPF maps to pprof](https://marselester.com/diy-cpu-profiler-from-bpf-maps-to-pprof.html) post.
