# Chapter 4: FANS — Moving Up the Stack to System Services

While DIFUZE successfully dismantled the structural barriers guarding the Linux kernel, the Android platform's architecture was simultaneously undergoing a massive shift. The introduction of Project Treble in Android 8 fundamentally reorganized the operating system, isolating hardware vendors' code into a heavily sandboxed Hardware Abstraction Layer (HAL). As the kernel became increasingly difficult to reach directly from unprivileged applications, the attack surface naturally migrated upward to the native system services communicating via Binder IPC.

Published in 2020 at the USENIX Security Symposium, **FANS** (Fuzzing Android Native System Services) recognized this shift. The researchers understood that the techniques used to fuzz POSIX `ioctl` calls could not be trivially ported to Binder. The challenge was no longer just about filling C-structures with data; it was about navigating deep, stateful, object-oriented communication graphs. FANS pioneered the use of Abstract Syntax Tree (AST) analysis to not only extract the structural format of Binder messages but to automatically infer the complex semantic dependencies required to reach vulnerable code.

## 4.1 The Binder Challenge: The Camera Module IPC

To understand why Binder IPC breaks traditional fuzzers, we return to our **Camera Module** example. At the userspace level, an application does not talk to the camera hardware directly. Instead, it queries the `ServiceManager` for a handle to the `cameraserver` daemon and initiates a Remote Procedure Call (RPC).

The payload for this RPC is serialized into a specialized, linear byte buffer called a `Parcel`. When the `cameraserver` receives the IPC request, its internal dispatcher (the `onTransact` method) must deserialize this buffer.

```cpp
// A conceptual Binder Server Stub for an ICameraDevice interface
status_t CameraDevice::onTransact(uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags) {
    switch (code) {
        case CAPTURE_IMAGE: {
            int32_t session_id = data.readInt32();
            String16 package_name = data.readString16();
            int32_t is_burst_mode = data.readInt32();
            
            if (is_burst_mode) {
                int32_t burst_count = data.readInt32();
                // ... process burst capture
            }
            return NO_ERROR;
        }
    }
}
```

A fuzzer cannot simply send a random chunk of bytes. The `Parcel` must perfectly mimic the expected serialization sequence: a 32-bit integer, followed by a 16-bit string, followed by another 32-bit integer. If the structure is misaligned by even a single byte, the `readString16()` call will fail, the transaction will abort, and the actual image capture logic will never be reached. 

Furthermore, this interface exhibits **semantic barriers**. Notice the `is_burst_mode` variable. If it evaluates to true, the deserialization logic suddenly expects an additional integer (`burst_count`). A purely structural fuzzer like DIFUZE, analyzing compiled bitcode, would struggle to cleanly map this conditional dependency. This realization led the FANS authors to analyze the code at a higher level of abstraction: the Abstract Syntax Tree.

## 4.2 AST-Based Model Extraction

To extract the expected `Parcel` layout, FANS analyzes the source code of the Android operating system. However, instead of operating on compiler intermediate representations (like LLVM bitcode), FANS uses Clang to parse the C++ source code into an Abstract Syntax Tree (AST).

The AST preserves vital high-level semantics that are optimized away during compilation, specifically **variable names** and **exact type aliases** (e.g., preserving a `typedef` of `pid_t` rather than reducing it to a generic `int32`). This semantic context is crucial for generating inputs that make sense to the application logic.

### 4.2.1 Handling AIDL vs. Native Implementations
A significant engineering challenge for FANS is that Android system services are implemented in two distinctly different ways. Some services are written purely in native C++, with manual `onTransact` deserialization routines. However, many modern services define their interfaces using the Android Interface Definition Language (AIDL). During the Android build process, the AIDL compiler automatically generates the C++ proxy and stub classes.

If FANS merely scanned the static AOSP source tree, it would completely miss the interfaces defined by AIDL, as the corresponding C++ code does not exist until compile time. To solve this, the FANS interface collector hooks into the actual AOSP compilation pipeline. By tracking the `cc1` compilation commands as they are executed, FANS ensures that it parses the ASTs of both the manually written native services and the dynamically generated AIDL C++ stubs, guaranteeing comprehensive coverage of the attack surface.

### 4.2.2 The Semantic Barriers: Loops and Conditions
As FANS traverses the AST of the `onTransact` function, it categorizes variables based on how they are read from the `Parcel`. While simple, unconditional reads are straightforward, FANS specifically targets the semantic barriers that block traditional fuzzers:
*   **Conditional Variables:** As seen in our Camera Module example, a read operation nested inside an `if` statement implies that the variable's existence in the `Parcel` depends entirely on the value of a previously read variable.
*   **Loop Variables:** Often, a `Parcel` contains a variable-length vector or array. The server will first read an integer representing the size of the array, and then enter a `for` loop, calling a `read` function multiple times. 

By analyzing the AST control flow, FANS correlates these loop bounds and conditional checks directly to their triggering variables. When the generation engine builds a fuzzing payload, it understands that if it generates an array of 5 elements, it must prepend the array with the integer 5 to satisfy the loop constraint in the server's deserialization logic.

## 4.3 Navigating the State Machine: Dependency Inference

Extracting the structural layout of a single `Parcel` is only half the battle. Android system services are heavily stateful. The most critical vulnerabilities are rarely triggered by a single transaction; they require a specific sequence of API calls. FANS' most significant contribution is its ability to automatically map these multi-stage execution chains through **Dependency Inference**.

### 4.3.1 The Multi-Level Interface Problem
FANS uncovered that a massive portion of the Binder attack surface is essentially invisible to a naive fuzzer. Many interfaces are not registered with the centralized `ServiceManager`. Instead, they are "multi-level" interfaces. 

In our Camera Module, an app cannot simply request an `ICameraDevice` interface. It must first connect to the top-level `ICameraService` and execute an `openCamera()` transaction. The return value of that transaction is an `IBinder` reference to a newly instantiated, deeply nested `ICameraDevice` object. FANS tracks the generation and use of these `IBinder` objects across the AST, mapping the exact sequence of transactions required to retrieve and interact with these hidden multi-level interfaces, effectively exposing the remaining 37% of the native attack surface.

### 4.3.2 Algorithm 1: Inter-Transaction Dependency Mapping
The most complex semantic challenge occurs when the output of one transaction is required as the input for a subsequent, seemingly unrelated transaction. For example, a `connect()` transaction might return a unique integer `session_id`. A subsequent `capture()` transaction might require that exact `session_id` to be passed in its `Parcel` to authorize the action.

If a fuzzer generates a random integer for the `capture()` call, the transaction will fail authorization. FANS solves this using a heuristic **Name and Type Matching Algorithm** (formally defined as Algorithm 1 in their methodology).

The algorithm operates in a conceptually elegant manner:
1.  **Extraction:** It scans the AST of all transactions within an interface, maintaining a global list of all "Output Variables" (data written into a `reply` Parcel) and all "Input Variables" (data read from a request `Parcel`).
2.  **Type Filtering:** It iterates through the list of Input Variables. For each input, it filters the Output Variables, retaining only those that perfectly match the C++ data type. 
3.  **Semantic Matching:** Because type matching alone is too broad (dozens of transactions might return an `int32`), the algorithm relies on the semantic data preserved by the AST: the variable names. It calculates the string similarity between the name of the Input Variable (e.g., `target_session_id`) and the Output Variable (e.g., `active_session_id`). 
4.  **Graph Construction:** If the type matches and the names exhibit high similarity, FANS formally registers an inter-transaction dependency graph edge.

During the execution phase, the FANS fuzzing engine consults this graph. Before attempting to fuzz the `capture()` transaction, it knows it must first successfully execute the `connect()` transaction, harvest the returned `session_id` from the reply `Parcel`, and inject that exact value into the request `Parcel` for the `capture()` call. This automated state-machine navigation is what allows FANS to penetrate deep logic flaws.

## 4.4 Real-World Evaluation and Deep Vulnerabilities

Deployed against six commercial Android devices, FANS uncovered 30 native memory-corruption vulnerabilities and over a hundred Java-layer logic exceptions. The true power of its dependency inference was highlighted by the complexity of the bugs it found.

In one notable case study, FANS discovered a critical stack overflow in the `ip6tables-restore` binary, reachable via the `netd` (network daemon) system service. This bug was buried deep within a stateful execution chain. The vulnerability resided in a transaction that strictly expected an active Binder reference to a previously configured network interface. A standard fuzzer would have been incapable of generating a valid network configuration object out of thin air. However, because FANS' Name and Type Matching Algorithm successfully linked the output of an interface-creation transaction to the input of the vulnerable transaction, the fuzzer organically generated the multi-stage prerequisite sequence required to deliver the malicious payload.

## 4.5 The "Open-Source Blind Spot"

Despite proving that AST-based extraction and dependency inference could master the complexities of Binder IPC, FANS suffers from a fatal architectural limitation: it requires full access to the source code.

While AOSP components like the `cameraserver` are open-source, the hardware vendors (e.g., Qualcomm, Samsung) that implement the actual physical drivers deliver their code as proprietary, pre-compiled binaries operating within the HAL. Because FANS relies entirely on Clang to parse C++ ASTs, it is completely blind to these proprietary binaries. 

As the Android ecosystem evolved and vendors pushed more critical, highly privileged logic down into these closed-source HAL services, the security community realized that static, source-reliant analysis was no longer sufficient. To truly secure the modern Android platform, fuzzing methodologies had to find a way to map complex IPC interfaces without ever seeing the source code. This critical necessity paved the way for dynamic, binary-only approaches.
