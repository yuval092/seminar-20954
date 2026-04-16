# Chapter 2: Technical Background

## 2.1 Fuzzing Methodologies and the Coverage Imperative

Modern fuzz testing has moved well past blind random mutation. Fuzzers generally fall into two categories, differentiated by how they generate test cases:

*   **Mutation-Based Fuzzing** starts with a "seed corpus" of known-good inputs and applies random modifications, such as bit flips. While easy to scale, it struggles with programs expecting rigidly structured data. A minor mutation—breaking a magic header or corrupting an offset—causes the target's parser to reject the input immediately.
*   **Generation-Based Fuzzing** builds inputs from scratch based on structural models or grammars. This ensures they pass initial validation, but the manual effort required to reverse-engineer targets and write grammars is often prohibitive.

A major breakthrough in the field was **grey-box fuzzing**, popularized by tools like AFL and Syzkaller. These use lightweight instrumentation to track code paths executed by an input. If a mutated input triggers a new path, the fuzzer saves it as "interesting" to mutate further. This evolutionary loop lets the fuzzer explore complex state spaces without exhaustive manual modeling.

That said, coverage guidance is ineffective if the fuzzer cannot bypass initial parsing checks. If 99% of inputs are rejected during validation, the fuzzer discovers no new code and stalls. This is where **interface-aware fuzzing** becomes arguably the most important requirement for auditing modern OS components. By automatically learning expected input models, these systems satisfy initial parsing constraints so that coverage-guided fuzzing can finally reach the core application logic.

## 2.2 Android's Privilege Architecture and SELinux

Android's security model, built on a modified Linux kernel, relies on the principle of least privilege. The system uses sandboxes to keep untrusted code isolated from critical resources. Standard Linux permissions are reinforced by Mandatory Access Control (MAC) policies through SELinux; every process and file is assigned a distinct context, and policies dictate their interaction.

Third-party apps run in restricted sandboxes and are denied direct access to hardware device nodes. Consequently, a malicious app cannot exploit a vulnerability in a kernel driver directly, because SELinux policies prevent it from even opening the required device file. Legitimately accessing hardware requires communication with a **system service**. These services operate within elevated SELinux domains and have the authority to talk to specific hardware drivers. 

## 2.3 Hardware Abstraction Layers (HAL) and Project Treble

To standardize the ecosystem and simplify updates, Google introduced Project Treble in Android 8. Before Treble, framework code and vendor-specific hardware drivers were tangled together within the same processes. Project Treble modularized this by introducing the **Vendor Hardware Abstraction Layer (HAL)**.

The Vendor HAL acts as a boundary between the open-source Android framework (AOSP) and proprietary hardware implementations written by Original Equipment Manufacturers (OEMs). Hardware-specific logic was isolated into distinct vendor processes, and communication across this boundary was standardized over Binder IPC. Modern devices run numerous proprietary HAL services (e.g., `vendor.camera.hal`) with high privileges. If an attacker can exploit a privileged system service through a legitimate channel, they hijack its elevated SELinux context, providing a clear path toward the kernel.

## 2.4 The Kernel Boundary: `ioctl`

When a userspace process needs to talk directly to the Linux kernel—for example, a HAL service controlling a hardware driver—it typically uses the `ioctl` (Input/Output Control) system call. The `ioctl` interface is a catch-all for generic, device-specific operations. Its signature is simple:

```c
int ioctl(int fd, unsigned long request, ...);
```

The real complexity lies in the `request` parameter and the variadic third argument, which is almost always a pointer to a userspace data structure. The driver uses the command identifier to select an internal handler, then copies data into kernel memory using functions like `copy_from_user()`.

A simplified vulnerable `ioctl` handler demonstrates the "deserialization barrier":

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

If a fuzzer supplies a pointer to a randomly generated buffer, the driver interprets it as a `sensor_config` struct. When it validates the embedded `user_buffer` pointer—likely full of random values—it causes an invalid memory access and triggers a kernel panic. The system crashes before the fuzzer can explore deeper logic.

## 2.5 The Userspace Boundary: Binder IPC Architecture

Communication *between* userspace processes relies on **Binder**, a custom Remote Procedure Call (RPC) mechanism. When an app talks to a service, it asks the `ServiceManager` for a handle. The client-side Proxy object packs arguments into a linear container called a `Parcel`, which is routed through the `/dev/binder` kernel driver to the target service.

On the receiving end, the service's Stub object unpacks the `Parcel` inside a dispatch function:

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

This is the semantic barrier. The content a `Parcel` should carry is not fixed—it depends on values the service reads earlier in the same transaction. Misalign this sequence by even one field and the entire transaction is silently dropped.

### 2.5.1 Android Interface Definition Language (AIDL)

The reason this process is so rigid is because it is machine-generated. Developers define interfaces using **AIDL**, and the AIDL compiler generates the Proxy and Stub classes. The generated Stub contains the `onTransact` method, filled with standard library calls to unpack arguments. Consequently, interface structures are predictable and follow universal design principles.

### 2.5.2 Universal RPC Design Principles

Research on modern RPC frameworks reveals architectural similarities that aid systematic analysis. These converge on three **RPC Design Principles**:

1.  **St (Standard Deserialization Routines)** ensure that different processes can understand each other. Server stubs rely on shared runtime libraries—like `libbinder.so`—to call standard routines (like `readInt32`), rather than requiring custom parsers for every developer. NASS relies heavily on this principle to observe signatures dynamically.
2.  **Ab (Abstraction of IPC binding code)** isolates IPC transport from business logic. Standard "stub" layers handle the validation and receipt of data.
3.  **Si (Single Entry Point)** funnels remote requests through a predictable function signature, such as `onTransact` in Binder, providing a reliable interception point.

## 2.6 Program Analysis Techniques: Static vs. Dynamic

To figure out interface models automatically, researchers use program analysis techniques categorized as static or dynamic.

**Static Analysis** examines code or binaries without execution. **AST Extraction** analyzes the Abstract Syntax Tree generated by a compiler; this preserves human context, like variable names, but requires source code. **IR/Bitcode Analysis** looks at intermediate representations, such as LLVM bitcode, providing a platform-independent view of data flow across functions.

**Dynamic Analysis** monitors behavior during execution. **Dynamic Binary Instrumentation (DBI)** frameworks, such as Frida, inject a tracing engine into a running process. DBI intercepts instructions just before execution, monitoring memory accesses and control flow. While DBI works on closed-source code, it introduces performance overhead. 
