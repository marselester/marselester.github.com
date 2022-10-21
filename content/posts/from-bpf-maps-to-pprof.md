title: DIY CPU profiler: from BPF maps to pprof
date: 2022-10-20
tags: bpf, golang, monitoring
category: Go
slug: diy-cpu-profiler-from-bpf-maps-to-pprof

[In previous post](https://marselester.com/continuous-profiling-in-go.html)
I had a DIY BPF profiler printing stack trace IDs of a given process,
though they were not very helpful.

```console
﹩ sudo go run ./cmd/profiler/ -pid 15958
Waiting for stack traces...
{PID:15958 UserStackID:132 KernelStackID:114} seen 1 times
```

Let's try to show more useful information this time,
i.e., sampled memory addresses.
You can find the results of my experiments in
[github.com/marselester/diy-parca-agent](https://github.com/marselester/diy-parca-agent).

## Reading addresses from BPF maps

I continued to explore
[github.com/parca-dev/parca-agent/pkg/profiler/profiler.go](https://github.com/parca-dev/parca-agent/blob/1e935f4d5f7b4484f6cf3d4ee26340f3718ff37d/pkg/profiler/profiler.go#L336)
and found the place where stack trace samples are read from the BPF maps — `profileLoop()` method.
Here is its simplified version.

```go
const (
	// Always needs to be sync with MAX_STACK_DEPTH in parca-agent.bpf.c.
	stackDepth = 127
	// Twice the stack depth because we have a User and a potential Kernel stack.
	doubleStackDepth = 254
)

func (p *CgroupProfiler) profileLoop(ctx context.Context, captureTime time.Time) error {
	it := p.bpfMaps.counts.Iterator()
	for it.Next() {
		// Note, pid, userStackID, and kernelStackID are read from it.Key().
		stack := [doubleStackDepth]uint64{}

		// Collect user-space stack trace samples.
		stackBytes, err := p.bpfMaps.stackTraces.GetValue(
			unsafe.Pointer(&userStackID),
		)
		binary.Read(bytes.NewBuffer(stackBytes), byteOrder, stack[:stackDepth])
		for _, addr := range stack[:stackDepth] {
			m, err := mapping.PIDAddrMapping(pid, addr)
		}

		// Collect kernel-space stack trace samples.
		stackBytes, err = p.bpfMaps.stackTraces.GetValue(
			unsafe.Pointer(&kernelStackID),
		)
		binary.Read(bytes.NewBuffer(stackBytes), byteOrder, stack[stackDepth:])
		for _, addr := range stack[stackDepth:] {}
	}
}
```

From looking at it I can tell that PID, user-space stack ID, and kernel-space stack ID are read
from the `counts` BPF map on each iteration.

| { pid; userStackID; kernelStackID } | seen
| ---                                 | ---
| { 10342; 1253; 0234 }               | 45

A stack ID allows to fetch a stack trace
(an array of memory addresses that represent the code executed during profiling)
from the `stack_traces` BPF map.

| stack ID | memory addresses
| ---      | ---
| 1253     | [ 0xdeadbeef; 0x123abcde; ... ]
| 0234     | [ 0x597be95a; 0xae5ee03; ... ]

A user-space stack trace indexed by `userStackID=1253` is placed
in the beginning of the `stack [254]uint64` array.
The `kernelStackID=0234` kernel-space stack trace is placed in the middle of the array.

```go
// User-space stack trace is limited by stackDepth=127.
stack[0] = 0xdeadbeef
stack[1] = 0x123abcde
stack[126] = ...

// Kernel-space stack trace is limited by doubleStackDepth=254.
stack[127] = 0x597be95a
stack[128] = 0xae5ee03
stack[253] = ...
```

## Memory mapping

Alright, but what does
`m, err := mapping.PIDAddrMapping(pid, addr)` line do?
Basically the function call does the following
(see [maps.go](https://github.com/parca-dev/parca-agent/blob/1e935f4d5f7b4484f6cf3d4ee26340f3718ff37d/pkg/maps/maps.go#L85)):

1. opens a memory map file of a given process `/proc/PID/maps`
2. parses a memory map using `github.com/google/pprof/profile` package
3. looks up a mapping for a given memory address previously found in a stack trace (e.g., `0xdeadbeef`),
   i.e., it should be located between `m.Start` and `m.Limit` inclusively.
   For example, in `[0x55dd2d5eb000; 0x55dd2d65f000]` interval.

```go
f, err := os.Open(
	fmt.Sprintf("/proc/%d/maps", pid),
)

mm, err := profile.ParseProcMaps(f)

for _, m := range mm {
	if m.Start <= addr && m.Limit >= addr {
		return m
	}
}
```

Note, `mm` is a slice of `*profile.Mapping{}` structs
that could look as it's shown below.

```go
mm = []*profile.Mapping{
	{
		ID:                     0,
		Start:                  94408437313536, // 0x55dd2d5eb000
		Limit:                  94408437788672, // 0x55dd2d65f000
		Offset:                 45056,          // 0xb000
		File:                   "/usr/sbin/sshd",
		BuildID:                "",
		HasFunctions:           false,
		HasFilenames:           false,
		HasLineNumbers:         false,
		HasInlineFrames:        false,
		KernelRelocationSymbol: "",
	},
	{
		ID:                     0,
		Start:                  140349706260480, // 0x7fa5b662d000
		Limit:                  140349706448896, // 0x7fa5b665b000
		Offset:                 24576,           // 0x6000
		File:                   "/usr/lib/x86_64-linux-gnu/libnss_systemd.so.2",
		BuildID:                "",
		HasFunctions:           false,
		HasFilenames:           false,
		HasLineNumbers:         false,
		HasInlineFrames:        false,
		KernelRelocationSymbol: "",
	},
	// ...
}
```

Here is the corresponding memory map file.

```console
﹩ sudo cat /proc/4068/maps
55dd2d5e0000-55dd2d5eb000 r--p 00000000 08:01 2869         /usr/sbin/sshd
55dd2d5eb000-55dd2d65f000 r-xp 0000b000 08:01 2869         /usr/sbin/sshd
55dd2d65f000-55dd2d69d000 r--p 0007f000 08:01 2869         /usr/sbin/sshd
55dd2d69e000-55dd2d6a1000 r--p 000bd000 08:01 2869         /usr/sbin/sshd
55dd2d6a1000-55dd2d6a2000 rw-p 000c0000 08:01 2869         /usr/sbin/sshd
55dd2d6a2000-55dd2d6a6000 rw-p 00000000 00:00 0
55dd2e9d6000-55dd2ea68000 rw-p 00000000 00:00 0            [heap]
55dd2ea68000-55dd2ea8a000 rw-p 00000000 00:00 0            [heap]
7fa5b6627000-7fa5b662d000 r--p 00000000 08:01 4761         /usr/lib/x86_64-linux-gnu/libnss_systemd.so.2
7fa5b662d000-7fa5b665b000 r-xp 00006000 08:01 4761         /usr/lib/x86_64-linux-gnu/libnss_systemd.so.2
7fa5b665b000-7fa5b666a000 r--p 00034000 08:01 4761         /usr/lib/x86_64-linux-gnu/libnss_systemd.so.2
7fa5b666a000-7fa5b666e000 r--p 00042000 08:01 4761         /usr/lib/x86_64-linux-gnu/libnss_systemd.so.2
7fa5b666e000-7fa5b666f000 rw-p 00046000 08:01 4761         /usr/lib/x86_64-linux-gnu/libnss_systemd.so.2
7fa5b6675000-7fa5b6676000 r--p 00000000 08:01 3974         /usr/lib/x86_64-linux-gnu/security/pam_env.so

...

7fa5b72f4000-7fa5b72f6000 rw-p 00034000 08:01 3657         /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
7fff5522d000-7fff55271000 rw-p 00000000 00:00 0            [stack]
7fff55285000-7fff55289000 r--p 00000000 00:00 0            [vvar]
7fff55289000-7fff5528b000 r-xp 00000000 00:00 0            [vdso]
ffffffffff600000-ffffffffff601000 --xp 00000000 00:00 0    [vsyscall]
```

🤔

Let's do a similar thing in a new profiler called profiler2.
The v2 DIY profiler is able to print a memory map file of a given process
and show the traced memory addresses of user and kernel space.

```console
﹩ sudo go run ./cmd/profiler2 -pid=4068
/proc/4068/maps
start=0x55dd2d5eb000 limit=0x55dd2d65f000 offset=0xb000 /usr/sbin/sshd
...

Waiting for stack traces...
```

## pprof

It would be great to finally see symbols (function names) instead of memory addresses
and pprof is going to help us with that.
pprof is a tool for visualization and analysis of profiling data.
It reads a collection of profiling samples in
[profile.proto](https://github.com/google/pprof/blob/main/proto/profile.proto) format
and generates reports to visualize the collected profiles.

Let's get back to `profileLoop()` method and see what else it's doing:

1. it prepares `prof := profile.Profile{}` struct which is an in-memory representation of profile.proto
2. using addresses from BPF maps it fills `Profile.Location` — the set of program locations;
   each location describes function and line table debug information
3. it fills `Profile.Sample` — the set of samples recorded in a profile
4. it fills `Profile.Mapping` — mapping from address ranges to the object file
5. it uploads object files to Parca server
6. it uploads pprof profile

The first step is to create a CPU profile which will be saved somewhere by `prof.Write()`
after a profile was filled with samples.

```go
prof := profile.Profile{
	// SampleType is a description of the samples associated with each Sample.Value.
	// By convention the number of events should use unit "count" and
	// it should be at index 0.
	SampleType: []*profile.ValueType{{
		Type: "samples",
		Unit: "count",
	}},
	// TimeNanos is a time of collection (UTC) represented as nanoseconds past the epoch.
	TimeNanos: time.Now().UnixNano(),
	// DurationNanos is a profiling duration in nanoseconds, Parca uses 10s by default.
	DurationNanos: int64(10 * time.Second),
	// PeriodType is the kind of events between sampled ocurrences,
	// i.e., ["cpu", "nanoseconds"].
	//
	// ValueType describes the semantics and measurement units of a value.
	PeriodType: &profile.ValueType{
		Type: "cpu",
		Unit: "nanoseconds",
	},
	// Period is the number of events between sampled occurrences.
	// We sample at 100Hz (100 times per second),
	// which is every 0.01s or 10 million nanoseconds.
	Period: 10000000,
}
```

A simplified version of the steps 2 to 4 show
how user-space stack traces (samples) are added to a pprof profile.

For every non-zero unique address found in stack traces
we create a `profile.Location{}` to describe a function.
A corresponding memory mapping is added to that newly created location
("Memory mapping" section above shows how to find a mapping for an address).
All the locations are appended to `prof.Location` slice.

```go
prof.Location = append(prof.Location, &profile.Location{
	// ID is a unique nonzero id for the location.
	ID: uint64(locIndex + 1),
	// Address is the instruction address for this location.
	// It should be within [Mapping.memory_start...Mapping.memory_limit]
	// for the corresponding mapping.
	Address: addr,
	// Mapping is the corresponding profile.Mapping for this location.
	// It can be nil if the mapping is unknown.
	Mapping: m,
})
```

Locations related to a given stack trace are added to a sample.
Each `profile.Sample` records how many times a particular stack trace was seen.

```go
prof.Sample = append(prof.Sample, &profile.Sample{
	Value:    []int64{int64(seen)},
	Location: sampleLocations,
})
```

At the end, all the mappings get assigned IDs starting with one
because ID zero is reserved.
The pprof tool will show the following error if ID zero is used:
`malformed profile: found mapping with reserved ID=0`.
See the whole code snippet in the details below.

<details>

```go
type locationIndex struct {
	PID  int
	Addr uint64
}

// locationIndices maps {pid; addr} found in the sample
// to an index in the Profile.Location slice
// to look up a respective Location.
locationIndices := make(map[locationIndex]int)

for it.Next() {
	// Collect user-space stack trace samples.
	sampleLocations := []*profile.Location{}
	for _, addr := range userStack {
		if addr == 0 {
			continue
		}

		// Look up a location for the given PID and address
		// and append it to the current sample's locations.
		//
		// In case a location wasn't found, create one.
		key := locationIndex{pid, addr}
		locIndex, found := locationIndices[key]
		if !found {
			m := mappingForAddr(mm, addr)
			if m == nil {
				log.Printf("failed to get process mapping for addr: %x", addr)
			}

			locIndex = len(prof.Location)
			// Each Location describes function and line table debug information.
			prof.Location = append(prof.Location, &profile.Location{
				// ID is a unique nonzero id for the location.
				ID: uint64(locIndex + 1),
				// Address is the instruction address for this location.
				// It should be within [Mapping.memory_start...Mapping.memory_limit]
				// for the corresponding mapping.
				Address: addr,
				// Mapping is the corresponding profile.Mapping for this location.
				// It can be nil if the mapping is unknown.
				Mapping: m,
			})
			locationIndices[key] = locIndex
		}

		sampleLocations = append(sampleLocations, prof.Location[locIndex])
	}

	// Each Sample records how many times a particular stack trace was seen
	// along with the corresponding locations.
	prof.Sample = append(prof.Sample, &profile.Sample{
		Value:    []int64{int64(seen)},
		Location: sampleLocations,
	})
}

// Mappings in pprof must have IDs and need to start with 1.
for i, m := range mm {
	m.ID = uint64(i + 1)
}
prof.Mapping = mm
```

</details>

🧪

Putting those steps together gives us profiler3.
In v3 DIY profiler we write a CPU profile to `cpu.pprof` file
instead of uploading it to a Parca server.
Let's run profiler3 and make sure the profiled program does something instead of idling,
so there will be enough CPU samples.

```console
﹩ sudo go run ./cmd/profiler3 -pid=4068
/proc/4068/maps
start=0x55dd2d5eb000 limit=0x55dd2d65f000 offset=0xb000 /usr/sbin/sshd
...

Waiting for stack traces for 10s...
```

After ten seconds the profiler will terminate leaving you with `cpu.pprof` file.
You can inspect it with `go tool pprof cpu.pprof` command.

Exploring Parca Agent's internals and reconstructing it in the simplest form
helped me demystify some of aspects of BPF CPU profilers.
