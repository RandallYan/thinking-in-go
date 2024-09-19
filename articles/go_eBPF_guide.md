# eBPF: A Comprehensive Guide from Basics to Practice

## 1. Introduction to eBPF

eBPF (Extended Berkeley Packet Filter) is a revolutionary technology that allows running custom programs in the Linux kernel safely and efficiently. Originally designed for network packet filtering, eBPF has evolved into a general-purpose framework for various system-level programming tasks.

### Key Features of eBPF:

1. **High Performance**: eBPF programs run directly in the kernel, avoiding frequent context switches.
2. **Safety**: eBPF programs undergo strict verification before loading, ensuring system stability.
3. **Flexibility**: Used in networking, security, tracing, and performance analysis.

## 2. eBPF Working Principle

The lifecycle of an eBPF program includes:

1. **Writing**: Develop eBPF programs using C or a C-like language.
2. **Compilation**: Use LLVM/Clang to compile the program into eBPF bytecode.
3. **Loading**: Load the bytecode into the kernel.
4. **Verification**: The kernel verifier checks the program's safety.
5. **JIT Compilation**: Convert bytecode to native machine code.
6. **Execution**: The program runs when triggered by specific events (e.g., network packet arrival).

## 3. eBPF Application Scenarios

eBPF finds application in various domains:

- **Networking**: Packet filtering, load balancing, DDoS protection
- **Security**: System call monitoring, container security
- **Performance Analysis**: CPU, memory, I/O profiling
- **Observability**: Distributed tracing, custom metric collection

## 4. eBPF Development Toolchain

### 4.1 Main Tools

1. **BCC (BPF Compiler Collection)**
   - Provides Python and Lua frontends, simplifying eBPF program development.
   - Suitable for rapid prototyping and scripting.

2. **libbpf**
   - C library offering APIs for loading and manipulating eBPF programs.
   - Supports CO-RE (Compile Once – Run Everywhere) technology.

3. **bpftrace**
   - High-level tracing language for writing short eBPF programs.
   - Ideal for one-off debugging and performance analysis tasks.

### 4.2 Go Language eBPF Libraries

1. **cilium/ebpf**
   - Offers pure Go implementation of eBPF loading and management functions.
   - Supports CO-RE, suitable for developing complex eBPF applications.

2. **iovisor/gobpf**
   - Go bindings for BCC.
   - Suitable for Go projects requiring BCC functionality.

3. **libbpfgo**
   - Go wrapper for libbpf.
   - Combines the power of libbpf with Go's ease of use.

## 5. eBPF CO-RE Technology

CO-RE (Compile Once – Run Everywhere) is a significant advancement in eBPF technology, solving portability issues across different kernel versions.

### 5.1 Core Components of CO-RE

1. **BTF (BPF Type Format)**
   - Describes the format of kernel data structures.
   - Allows eBPF programs to adapt to different structure layouts at runtime.

2. **Relocation**
   - Uses BTF information to adjust field offsets at runtime.
   - Ensures programs can correctly access data structures on different kernel versions.

### 5.2 CO-RE vs Traditional eBPF

| Feature | Traditional eBPF | CO-RE eBPF |
|---------|------------------|------------|
| Compilation | Needs recompilation for each kernel version | Compile once, run everywhere |
| Kernel header dependency | Required | Not required |
| Portability | Low | High |
| Runtime performance | Good | Good |
| Development complexity | Higher | Lower |

## 6. Practice: Developing eBPF Programs with Go and cilium/ebpf

Let's create a simple example that traces the `execve` system call using Go and cilium/ebpf.

### 6.1 eBPF Program (kprobe.c)

```c
#include <linux/bpf.h>
#include <bpf/bpf_helpers.h>

SEC("kprobe/sys_execve")
int kprobe_execve(struct pt_regs *ctx)
{
    char msg[] = "Hello, eBPF!";
    bpf_trace_printk(msg, sizeof(msg));
    return 0;
}

char LICENSE[] SEC("license") = "GPL";
```

### 6.2 Go Program

```go
package main

import (
    "log"
    "time"

    "github.com/cilium/ebpf"
    "github.com/cilium/ebpf/link"
    "github.com/cilium/ebpf/rlimit"
)

//go:generate go run github.com/cilium/ebpf/cmd/bpf2go bpf kprobe.c -- -I/usr/include/bpf

func main() {
    // Allow the current process to lock memory for eBPF resources
    if err := rlimit.RemoveMemlock(); err != nil {
        log.Fatal(err)
    }

    // Load pre-compiled programs
    objs := bpfObjects{}
    if err := loadBpfObjects(&objs, nil); err != nil {
        log.Fatalf("loading objects: %v", err)
    }
    defer objs.Close()

    // Attach kprobe
    kp, err := link.Kprobe("sys_execve", objs.KprobeExecve)
    if err != nil {
        log.Fatalf("opening kprobe: %s", err)
    }
    defer kp.Close()

    log.Println("Successfully started! Please run \"sudo cat /sys/kernel/debug/tracing/trace_pipe\" to see output")

    // Keep the program running
    for {
        time.Sleep(time.Second)
    }
}
```

### 6.3 Running the Program

1. Compile the eBPF program:
   ```
   go generate
   ```

2. Compile and run the Go program:
   ```
   go build -o ebpf_example
   sudo ./ebpf_example
   ```

3. View the output in another terminal:
   ```
   sudo cat /sys/kernel/debug/tracing/trace_pipe
   ```

You'll see "Hello, eBPF!" output whenever a new program is executed on the system.

## 7. Best Practices

1. **Use CO-RE**: Whenever possible, use CO-RE technology to improve program portability.
2. **Error Handling**: Handle and log errors in detail, as eBPF program issues are often difficult to debug.
3. **Performance Considerations**: eBPF programs run in critical paths, so minimize program overhead.
4. **Security**: Follow the principle of least privilege, only requesting necessary permissions.
5. **Testing**: Test your programs on multiple kernel versions to ensure compatibility.

## 8. Advanced Resources

- [Linux Kernel BPF Documentation](https://www.kernel.org/doc/html/latest/bpf/index.html)
- [BPF Performance Tools](https://www.brendangregg.com/bpf-performance-tools-book.html) by Brendan Gregg
- [cilium/ebpf Documentation](https://github.com/cilium/ebpf)
- [eBPF.io](https://ebpf.io/) - Official eBPF website

## Conclusion

eBPF is a powerful technology that has brought revolutionary changes to system observation, networking, and security. This guide provides you with a basic understanding of eBPF and the ability to start developing simple eBPF programs using Go. As you delve deeper, you'll be able to leverage eBPF to solve more complex problems and bring unprecedented observability and programmability to Linux systems.
