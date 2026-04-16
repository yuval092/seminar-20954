# Chapter 2: Technical Background

## 2.1 The Problem with Structured Targets

Modern fuzz testing has moved well past blind random mutation. The real breakthrough was grey-box fuzzing—AFL, Syzkaller, and tools like that. They use lightweight instrumentation to track code paths, saving anything that triggers a new path to mutate later. Simple idea. Surprisingly effective.

But fuzzers usually fall into two categories based on how they generate test cases:

*   **Mutation-Based Fuzzing** starts with a "seed corpus"—basically just valid, known-good inputs. The fuzzer takes these seeds and applies random changes, like flipping bits. While easy to scale, this approach struggles with targets that expect rigid structures. One wrong byte causes the parser to reject the input immediately.
*   **Generation-Based Fuzzing** builds inputs from scratch based on a model. This ensures the inputs are valid, but the manual effort to reverse-engineer targets and write these grammars is usually way too high.

Coverage guidance only works if the fuzzer actually gets through the front door. If 99% of inputs fail validation, no new code paths are ever reached—the fuzzer just stalls. This is where **interface-aware fuzzing** becomes the main requirement. By automatically learning the expected input models, these systems satisfy the initial parsing checks. Only then can the fuzzer reach the actual application logic.

## 2.2 Security and SELinux

To understand why these attacks work, it helps to know how Android separates processes from each other. The core idea is least privilege—no process gets more access than it needs. The system uses sandboxes to keep untrusted code away from critical resources. Standard Linux permissions are reinforced by SELinux; every process and file is assigned a context, and policies dictate how they interact.

Third-party apps run in very restricted sandboxes. They can't access hardware nodes directly. A malicious app cannot exploit a vulnerability in a kernel driver directly, because SELinux policies prevent it from even opening the required device file. Legitimately accessing hardware requires talking to a system service. That process holds the elevated SELinux context needed to open the hardware node on behalf of the requester.

That is actually by design—not a flaw. But it also means that the system service becomes the boundary an attacker has to cross to get to the hardware.

## 2.3 HALs and Project Treble

To understand why so much code became proprietary, it helps to look at Project Treble. Google introduced it in Android 8 to standardize the ecosystem and simplify updates. Before Treble, framework code and vendor-specific drivers were tangled together in the same processes. Treble modularized this by introducing the **Vendor Hardware Abstraction Layer (HAL)**.

The Vendor HAL is the boundary between the open-source Android framework and proprietary hardware code written by vendors. Hardware logic was moved into distinct processes, and communication across this boundary was standardized over Binder IPC. Modern phones run a lot of proprietary HAL services with very high privileges. 

If an attacker can exploit one of these services, they hijack its elevated context, providing a direct path toward the kernel.

## 2.4 The `ioctl` Boundary

When a userspace process needs to talk directly to the kernel—like a HAL service controlling a driver—it usually uses the `ioctl` (Input/Output Control) system call. The `ioctl` interface is basically a catch-all for generic, device-specific operations. Its signature is simple:

```c
int ioctl(int fd, unsigned long request, ...);
```

The real complexity is in the `request` parameter and the variadic third argument. That third argument is almost always a pointer to a userspace structure. The driver uses the command ID to select a handler, then copies data into kernel memory.

A simplified vulnerable `ioctl` handler shows why this is a barrier:

```c
struct sensor_config {
    int sensor_id;
    int data_length;
    char *user_buffer; // Embedded userspace pointer
};

long sensor_ioctl_handler(struct file *file, unsigned int cmd, unsigned long arg) {
    struct sensor_config config;

    if (cmd == SENSOR_CONFIG_CMD) {
        if (copy_from_user(&config, (void __user *)arg, sizeof(config))) {
            return -EFAULT;
        }

        char *kernel_buffer = kmalloc(config.data_length, GFP_KERNEL);
        if (copy_from_user(kernel_buffer, config.user_buffer, config.data_length)) {
            kfree(kernel_buffer);
            return -EFAULT;
        }

        return 0;
    }
    return -ENOTTY;
}
```

If a fuzzer sends a pointer to a random buffer, the driver tries to read it as a `sensor_config` struct. When it tries to validate the `user_buffer` pointer—which is just random values—it causes an invalid memory access. The system crashes before the fuzzer can even explore the logic.

## 2.5 Binder and the Semantic Barrier

When two userspace processes need to talk to each other on Android, they go through Binder — a custom RPC mechanism built right into the kernel. When an app talks to a service, it asks for a handle. The client-side Proxy object packs arguments into a container called a `Parcel`, which is routed through the kernel to the target service.

On the receiving end, the service unpacks the `Parcel` inside a dispatch function:

```cpp
status_t TargetService::onTransact(uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags) {
    switch (code) {
        case TARGET_TRANSACTION: {
            CHECK_INTERFACE(ITargetService, data, reply);

            int32_t session_id;
            if (data.readInt32(&session_id) != NO_ERROR) return BAD_VALUE;

            String16 package_name;
            if (data.readString16(&package_name) != NO_ERROR) return BAD_VALUE;

            status_t res = this->executeLogic(session_id, package_name);
            reply->writeInt32(res);
            return NO_ERROR;
        }
        default:
            return BBinder::onTransact(code, data, reply, flags);
    }
}
```

This is what we call the semantic barrier. The content a `Parcel` should carry isn't fixed—it depends on values the service reads earlier in the same transaction. If you misalign this sequence by even one field, the whole transaction is just silently dropped.

### 2.5.1 AIDL and Machine-Generated Code

The reason this process is so rigid is that it is machine-generated. Developers define interfaces with **AIDL**, and the compiler generates the code. Because of this, interface structures are very predictable and follow standard design principles.

### 2.5.2 Universal RPC Principles

Research on modern RPC frameworks reveals architectural similarities that aid systematic analysis. These converge on three **RPC Design Principles**:

1.  **St (Standard Deserialization Routines)** ensure that different processes can understand each other. Server stubs rely on shared runtime libraries—like `libbinder.so`—to call standard routines (like `readInt32`), rather than requiring custom parsers for every developer. NASS relies heavily on this principle to observe signatures dynamically.
2.  **Ab (Abstraction of IPC binding code)** isolates IPC transport from business logic. Standard "stub" layers handle the validation and receipt of data.
3.  **Si (Single Entry Point)** funnels remote requests through a predictable function signature, such as `onTransact` in Binder, providing a reliable interception point.

## 2.6 Static vs. Dynamic Analysis

To figure out interface models automatically, researchers use either static or dynamic analysis.

**Static Analysis** looks at code or binaries without actually running them. FANS uses AST extraction, which works from the Abstract Syntax Tree generated by Clang. This preserves real human context: variable names and custom type names that usually disappear in compiled binaries. Other static tools analyze Intermediate Representation (IR) or bitcode; while IR analysis is platform-agnostic, it strips away that semantic richness and human context.

**Dynamic Analysis** monitors behavior during execution. **Dynamic Binary Instrumentation (DBI)** frameworks, like Frida, inject a tracing engine into a running process. While DBI works on closed-source code, it does introduce quite a bit of performance overhead. 
 
