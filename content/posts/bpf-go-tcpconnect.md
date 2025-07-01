title: BPF: Go frontend for tcpconnect
date: 2021-11-01
tags: bpf, golang
category: Go
slug: bpf-go-frontend-for-tcpconnect

[In the previos BPF post](https://marselester.com/bpf-go-frontend-for-execsnoop.html)
I shared my experiment of writing Go frontend for execsnoop.
Here I would like to focus on tcpconnect, a BCC tool to trace new TCP active connections.
It is useful for determining who is connecting to whom.
This works by tracing the `tcp_v4_connect()` and `tcp_v6_connect()` kernel functions.

As before, I tried to make Go tcpconnect's output match the original program,
so I copied the ELF binary `tcpconnect` from a Docker container to the virtual machine
(Ubuntu 21.04 from the previos post).

```console
﹪ docker run --name libbpf marselester/libbpf-tools:latest
﹪ docker cp libbpf:/opt/tcpconnect .
```

The tcpconnect captured `curl example.com` running in another terminal:

- PID *93336* is the process ID that opened the connection
- COMM *curl* is the process name that opened the connection, e.g., via a `connect()` syscall
- IP *4* stands for IP v4 address protocol
- SADDR *10.0.2.15* is the source address
- DADDR *93.184.216.34* is the destination address
- DPORT *80* is the destination port

```console
﹩ sudo /vagrant/tcpconnect
PID    COMM         IP SADDR            DADDR            DPORT
93336  curl         4  10.0.2.15        93.184.216.34    80
```

## Embedding BPF in Go

The tcpconnect BPF program comprises of
[tcpconnect.bpf.c](https://github.com/iovisor/bcc/blob/master/libbpf-tools/tcpconnect.bpf.c),
[tcpconnect.h](https://github.com/iovisor/bcc/blob/master/libbpf-tools/tcpconnect.h),
and [maps.bpf.h](https://github.com/iovisor/bcc/blob/master/libbpf-tools/maps.bpf.h) files.
Let's copy them from BCC toolkit and generate Go files.

```console
﹩ cd /vagrant/
﹩ mkdir -p cmd/tcpconnect/bpf/
﹩ curl -o cmd/tcpconnect/bpf/tcpconnect.bpf.c https://raw.githubusercontent.com/iovisor/bcc/master/libbpf-tools/tcpconnect.bpf.c
﹩ curl -o cmd/tcpconnect/bpf/tcpconnect.h https://raw.githubusercontent.com/iovisor/bcc/master/libbpf-tools/tcpconnect.h
﹩ curl -o cmd/tcpconnect/bpf/maps.bpf.h https://raw.githubusercontent.com/iovisor/bcc/master/libbpf-tools/maps.bpf.h
﹩ cat > cmd/tcpconnect/main.go <<<'''
package main

//go:generate go run github.com/cilium/ebpf/cmd/bpf2go -cflags $BPF_CFLAGS -cc clang-12 TCPConnect ./bpf/tcpconnect.bpf.c -- -I../../headers

func main() {}
'''
﹩ go generate ./cmd/tcpconnect/
/vagrant/cmd/tcpconnect/bpf/tcpconnect.bpf.c:9:10: fatal error: 'bpf/bpf_tracing.h' file not found
```

Since `bpf_tracing.h` file is missing, let's get it from ubuntu-kernel repository
and run `go generate` again,
but this time with `BPF_CFLAGS='-D__TARGET_ARCH_x86'` env var (a compiler flag) because
[bpf_tracing.h](https://elixir.bootlin.com/linux/latest/source/tools/lib/bpf/bpf_tracing.h#L5) expects it.

```console
﹩ curl -o ./headers/bpf/bpf_tracing.h https://git.launchpad.net/~ubuntu-kernel/ubuntu/+source/linux/+git/hirsute/plain/tools/lib/bpf/bpf_tracing.h
```

It worked out 😬.

```console
﹩ BPF_CFLAGS='-D__TARGET_ARCH_x86' go generate ./cmd/tcpconnect/
Compiled /vagrant/cmd/tcpconnect/tcpconnect_bpfel.o
Wrote /vagrant/cmd/tcpconnect/tcpconnect_bpfel.go
Compiled /vagrant/cmd/tcpconnect/tcpconnect_bpfeb.o
Wrote /vagrant/cmd/tcpconnect/tcpconnect_bpfeb.go
﹩ sudo go run ./cmd/tcpconnect/
```

BPF programs loading looks very similar to Go execsnoop version,
though instead of calling `LoadTCPConnectObjects()` the `LoadTCPConnect()` is called to
parse an ELF file into `spec` struct (a collection of BPF maps and programs) and
replace constants in the C program to filter connections by PID or UID.
Here is how they are defined in `tcpconnect.bpf.c`.

```cpp
const volatile uid_t filter_uid = -1;
const volatile pid_t filter_pid = 0;
```

After the constants are rewritten, the BPF programs get loaded into the kernel with `spec.LoadAndAssign()`.

```go
package main

import (
	"flag"
	"log"

	"golang.org/x/sys/unix"
)

//go:generate go run github.com/cilium/ebpf/cmd/bpf2go -cflags $BPF_CFLAGS -cc clang-12 TCPConnect ./bpf/tcpconnect.bpf.c -- -I../../headers

func main() {
	var (
		filterUID = flag.Int("uid", -1, "trace this UID only")
		filterPID = flag.Int("pid", 0, "trace this PID only")
	)
	flag.Parse()

	// Increase the resource limit of the current process to provide sufficient space
	// for locking memory for the BPF maps.
	if err := unix.Setrlimit(unix.RLIMIT_MEMLOCK, &unix.Rlimit{
		Cur: unix.RLIM_INFINITY,
		Max: unix.RLIM_INFINITY,
	}); err != nil {
		log.Printf("failed to set temporary RLIMIT_MEMLOCK: %v", err)
		return
	}

	// Replace constants in the BPF C program to filter connections by PID or UID.
	spec, err := LoadTCPConnect()
	if err != nil {
		log.Printf("failed to load collection spec: %v", err)
		return
	}
	bpfConst := make(map[string]interface{})
	if *filterUID > -1 {
		bpfConst["filter_uid"] = uint32(*filterUID)
	}
	if *filterPID > 0 {
		bpfConst["filter_pid"] = int32(*filterPID)
	}
	if err := spec.RewriteConstants(bpfConst); err != nil {
		log.Printf("failed to rewrite BPF constants: %v", err)
		return
	}

	// Load the BPF program into the kernel from an ELF.
	// TCPConnectObjects contains all objects (BPF programs and maps) after they have been loaded into the kernel:
	// - TcpV4Connect, TcpV4ConnectRet, TcpV6Connect, TcpV6ConnectRet BPF programs,
	// - Events, Ipv4Count, Ipv6Count, Sockets BPF maps.
	objs := TCPConnectObjects{}
	if err := spec.LoadAndAssign(&objs, nil); err != nil {
		log.Printf("failed to load BPF programs and maps: %v", err)
		return
	}
	defer objs.Close()
}
```

## Kprobes

A BPF program is executed on events such as kprobes.
Kprobes provide kernel dynamic instrumentation for any kernel function,
and they can instrument instructions within functions.
There is also an interface called kretprobes for instrumenting when functions return, and their return values.

The `tcpconnect.bpf.c` lists four BPF programs that trace events related to creating TCP sessions:
enter and exit of `tcp_v4_connect()` and `tcp_v6_connect()` kernel functions.

```cpp
SEC("kprobe/tcp_v4_connect")
int BPF_KPROBE(tcp_v4_connect, struct sock *sk) {
	return enter_tcp_connect(ctx, sk);
}

SEC("kretprobe/tcp_v4_connect")
int BPF_KRETPROBE(tcp_v4_connect_ret, int ret) {
	return exit_tcp_connect(ctx, ret, 4);
}

SEC("kprobe/tcp_v6_connect")
int BPF_KPROBE(tcp_v6_connect, struct sock *sk) {
	return enter_tcp_connect(ctx, sk);
}

SEC("kretprobe/tcp_v6_connect")
int BPF_KRETPROBE(tcp_v6_connect_ret, int ret) {
	return exit_tcp_connect(ctx, ret, 6);
}
```

On Go side I attached the BPF programs to kprobes/kretprobes.
The `link.Kprobe()`, `link.Kretprobe` functions return a link (it represents a program attached to a BPF hook)
which is later used to detach a BPF program.

```go
tcpv4kp, err := link.Kprobe("tcp_v4_connect", objs.TCPConnectPrograms.TcpV4Connect)
if err != nil {
	log.Printf("failed to attach the BPF program to tcp_v4_connect kprobe: %s", err)
	return
}
defer tcpv4kp.Close()

tcpv4krp, err := link.Kretprobe("tcp_v4_connect", objs.TCPConnectPrograms.TcpV4ConnectRet)
if err != nil {
	log.Printf("failed to attach the BPF program to tcp_v4_connect kretprobe: %s", err)
	return
}
defer tcpv4krp.Close()

tcpv6kp, err := link.Kprobe("tcp_v6_connect", objs.TCPConnectPrograms.TcpV6Connect)
if err != nil {
	log.Printf("failed to attach the BPF program to tcp_v6_connect kprobe: %s", err)
	return
}
defer tcpv6kp.Close()

tcpv6krp, err := link.Kretprobe("tcp_v6_connect", objs.TCPConnectPrograms.TcpV6ConnectRet)
if err != nil {
	log.Printf("failed to attach the BPF program to tcp_v6_connect kretprobe: %s", err)
	return
}
defer tcpv6krp.Close()
```

The `bpf_perf_event_output()` C function emits records to user space via a BPF map `events`
that accesses perf per-CPU ring buffers.

```cpp
struct {
	__uint(type, BPF_MAP_TYPE_PERF_EVENT_ARRAY);
	__uint(key_size, sizeof(u32));
	__uint(value_size, sizeof(u32));
} events SEC(".maps");

struct event e = {};
bpf_perf_event_output(ctx, &events, BPF_F_CURRENT_CPU, &e, sizeof(e));
```

In user space the events are read using `github.com/cilium/ebpf/perf` package.

```go
// Open a perf event reader from user space on the PERF_EVENT_ARRAY map
// defined in the BPF C program.
rd, err := perf.NewReader(objs.TCPConnectMaps.Events, os.Getpagesize())
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

Running `curl` shows that the TCP connect events are captured.

```console
﹩ sudo go run ./cmd/tcpconnect/
received from perf ring buffer:
]??"curl???0x??P
```

## Decoding events

Let's decode the events and print them as the original tcpconnect program does.
As mentioned before, the `event` Go struct should match its C counterpart.

```cpp
struct event {
	union {
		__u32 saddr_v4;
		__u8 saddr_v6[16];
	};
	union {
		__u32 daddr_v4;
		__u8 daddr_v6[16];
	};
	char task[16];
	__u64 ts_us;
	__u32 af; // AF_INET or AF_INET6
	__u32 pid;
	__u32 uid;
	__u16 dport;
};
```

I wasn't sure how to represent `union { ... }` in Go, so allocating 16-byte slice worked well,
since the union is only as big as necessary to hold its largest member (IPv6 address).

```go
// event represents a perf event sent to user space from the BPF program running in the kernel.
// Note, that it must match the C event struct, and both C and Go structs must be aligned the same way.
type event struct {
	// SrcAddr is the source address.
	SrcAddr [16]byte
	// DstAddr is the destination address.
	DstAddr [16]byte
	// Comm is the process name that opened the connection.
	Comm [16]byte
	// Timestamp is the timestamp in microseconds.
	Timestamp uint64
	// AddrFam is the address family, 2 is AF_INET (IPv4), 10 is AF_INET6 (IPv6).
	AddrFam uint32
	// PID is the process ID that opened the connection.
	PID uint32
	// UID is the process user ID.
	UID uint32
	// DstPort is the destination port (uint16 in C struct).
	// Note, network byte order is big-endian.
	DstPort [2]byte
}
```

The source and destination addresses were pretty-printed using `net.IP` type
(either 4-byte (IPv4) or 16-byte (IPv6) slice) depending on the address family
(`2` is IPv4, `10` is IPv6).

The destination port (unsigned 16-bit integer in C struct) was translated with
`binary.BigEndian.Uint16()` function because `DstPort` byte order is big-endian
(most significant bits first).

```go
func printEvent(w io.Writer, e *event) {
	var (
		srcAddr, dstAddr net.IP
		ipVer            byte
	)
	switch e.AddrFam {
	case 2:
		srcAddr = net.IP(e.SrcAddr[:4])
		dstAddr = net.IP(e.DstAddr[:4])
		ipVer = 4
	case 10:
		srcAddr = net.IP(e.SrcAddr[:])
		dstAddr = net.IP(e.DstAddr[:])
		ipVer = 6
	}

	fmt.Fprintf(
		w,
		"%-6d %-12s %-2d %-16s %-16s %d\n",
		e.PID,
		bytes.TrimRight(e.Comm[:], "\x00"),
		ipVer,
		srcAddr,
		dstAddr,
		binary.BigEndian.Uint16(e.DstPort[:]),
	)
}
```

Putting it all together we've got a Go frontend for tcpconnect with ability
to filter connections by PID and UID 🎉.

```console
﹩ sudo go run ./cmd/tcpconnect/
PID    COMM         IP SADDR            DADDR            DPORT
189217 curl         4  10.0.2.15        93.184.216.34    80
﹩ ./tcpconnect -h
Usage of tcpconnect:
  -pid int
    	trace this PID only
  -uid int
    	trace this UID only (default -1)
```

You can find all the source code at
[github.com/marselester/libbpf-tools](https://github.com/marselester/libbpf-tools).

```go
fmt.Printf("%-6s %-12s %-2s %-16s %-16s %s\n", "PID", "COMM", "IP", "SADDR", "DADDR", "DPORT")
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
		log.Printf("failed to parse perf event: %#+v", err)
		continue
	}

	printEvent(os.Stdout, &e)
}
```
