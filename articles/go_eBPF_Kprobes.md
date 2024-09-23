# eBPF Security Monitoring with Golang: Leveraging Kprobes for Detection

## 1. Introduction

This article demonstrates how to implement advanced eBPF-based security monitoring using Golang. We'll create a sophisticated system that monitors `execve` system calls, providing real-time insights into process execution on a Linux system.

## 2. eBPF Program in C

First, let's define our eBPF program in C. This program will be compiled to eBPF bytecode and loaded into the kernel by our Golang program.

```c
#include <linux/bpf.h>
#include <linux/sched.h>
#include "bpf_helpers.h"

char LICENSE[] SEC("license") = "Dual BSD/GPL";

struct event {
    u32 pid;
    u32 ppid;
    u32 uid;
    u32 gid;
    char comm[TASK_COMM_LEN];
    char filename[256];
    u64 duration_ns;
};

struct {
    __uint(type, BPF_MAP_TYPE_RINGBUF);
    __uint(max_entries, 256 * 1024);
} events SEC(".maps");

struct {
    __uint(type, BPF_MAP_TYPE_HASH);
    __uint(max_entries, 8192);
    __type(key, u32);
    __type(value, u64);
} start_times SEC(".maps");

SEC("kprobe/sys_execve")
int kprobe_execve(struct pt_regs *ctx)
{
    u64 ts = bpf_ktime_get_ns();
    u32 pid = bpf_get_current_pid_tgid() >> 32;
    bpf_map_update_elem(&start_times, &pid, &ts, BPF_ANY);
    return 0;
}

SEC("kretprobe/sys_execve")
int kretprobe_execve(struct pt_regs *ctx)
{
    int ret = PT_REGS_RC(ctx);
    if (ret != 0)
        return 0;  // execve failed, don't report

    u32 pid = bpf_get_current_pid_tgid() >> 32;
    u64 *tsp, duration_ns;
    tsp = bpf_map_lookup_elem(&start_times, &pid);
    if (!tsp)
        return 0;  // missed entry, skip
    duration_ns = bpf_ktime_get_ns() - *tsp;

    struct event *e;
    e = bpf_ringbuf_reserve(&events, sizeof(*e), 0);
    if (!e)
        return 0;

    e->pid = pid;
    e->duration_ns = duration_ns;

    struct task_struct *task = (struct task_struct *)bpf_get_current_task();
    e->ppid = BPF_CORE_READ(task, real_parent, tgid);
    
    struct cred *cred = (struct cred *)BPF_CORE_READ(task, real_cred);
    e->uid = BPF_CORE_READ(cred, uid.val);
    e->gid = BPF_CORE_READ(cred, gid.val);
    
    bpf_get_current_comm(&e->comm, sizeof(e->comm));
    
    bpf_probe_read_user_str(e->filename, sizeof(e->filename), (void *)PT_REGS_PARM1(ctx));

    bpf_ringbuf_submit(e, 0);
    bpf_map_delete_elem(&start_times, &pid);
    return 0;
}
```

This eBPF program attaches to the `sys_execve` system call, capturing detailed information about each execution, including process details and execution duration.

## 3. Golang Implementation

Now, let's implement the Golang program that will load this eBPF program, attach it to the kernel, and process the collected data.

```go
package main

import (
    "bytes"
    "encoding/binary"
    "fmt"
    "log"
    "os"
    "os/signal"

    "github.com/cilium/ebpf"
    "github.com/cilium/ebpf/link"
    "github.com/cilium/ebpf/ringbuf"
    "golang.org/x/sys/unix"
)

//go:generate go run github.com/cilium/ebpf/cmd/bpf2go -cc clang -cflags "-O2 -g -Wall -Werror" bpf execve_monitor.c

type event struct {
    PID        uint32
    PPID       uint32
    UID        uint32
    GID        uint32
    Comm       [16]byte
    Filename   [256]byte
    DurationNS uint64
}

func main() {
    // Allow the current process to lock memory for eBPF resources.
    if err := unix.Setrlimit(unix.RLIMIT_MEMLOCK, &unix.Rlimit{
        Cur: unix.RLIM_INFINITY,
        Max: unix.RLIM_INFINITY,
    }); err != nil {
        log.Fatalf("Failed to set memlock limit: %v", err)
    }

    // Load pre-compiled programs and maps into the kernel.
    objs := bpfObjects{}
    if err := loadBpfObjects(&objs, nil); err != nil {
        log.Fatalf("Loading objects: %v", err)
    }
    defer objs.Close()

    // Attach the kprobe.
    kp, err := link.Kprobe("sys_execve", objs.KprobeExecve, nil)
    if err != nil {
        log.Fatalf("Opening kprobe: %v", err)
    }
    defer kp.Close()

    // Attach the kretprobe.
    krp, err := link.Kretprobe("sys_execve", objs.KretprobeExecve, nil)
    if err != nil {
        log.Fatalf("Opening kretprobe: %v", err)
    }
    defer krp.Close()

    // Open a ring buffer reader.
    rd, err := ringbuf.NewReader(objs.Events)
    if err != nil {
        log.Fatalf("Opening ringbuf reader: %v", err)
    }
    defer rd.Close()

    // Listen for events.
    go func() {
        for {
            record, err := rd.Read()
            if err != nil {
                if err == ringbuf.ErrClosed {
                    return
                }
                log.Printf("Reading from ringbuf: %v", err)
                continue
            }

            var e event
            if err := binary.Read(bytes.NewBuffer(record.RawSample), binary.LittleEndian, &e); err != nil {
                log.Printf("Parsing event: %v", err)
                continue
            }

            fmt.Printf("Process %s (PID: %d, PPID: %d, UID: %d, GID: %d) executed %s (duration: %d ns)\n",
                unix.ByteSliceToString(e.Comm[:]),
                e.PID,
                e.PPID,
                e.UID,
                e.GID,
                unix.ByteSliceToString(e.Filename[:]),
                e.DurationNS)
        }
    }()

    // Wait for a signal to quit.
    stopper := make(chan os.Signal, 1)
    signal.Notify(stopper, os.Interrupt)
    <-stopper

    fmt.Println("Received signal, exiting..")
}
```

This Golang program does the following:

1. Loads the pre-compiled eBPF program into the kernel.
2. Attaches kprobe and kretprobe to the `sys_execve` system call.
3. Opens a ring buffer to read events from the eBPF program.
4. Continuously reads and processes events from the ring buffer.
5. Gracefully handles program termination.

## 4. Building and Running

To build and run this program:

1. Save the C code as `execve_monitor.c` and the Go code as `main.go`.
2. Generate the Go bindings for the eBPF program:
   ```
   go generate
   ```
3. Build the Go program:
   ```
   go build -o execve_monitor
   ```
4. Run the program (requires root privileges):
   ```
   sudo ./execve_monitor
   ```

## Conclusion

This implementation demonstrates how to create a sophisticated eBPF-based security monitoring system using Golang. It provides real-time insights into process execution, with optimizations for performance and integration capabilities for enterprise environments. 

By leveraging eBPF's powerful in-kernel capabilities and Golang's efficient concurrency model, we've created a high-performance, low-overhead monitoring solution. This approach can be extended to monitor various other system calls and kernel functions, providing a comprehensive view of system activity for security analysis.
