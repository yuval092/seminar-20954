# Chapter 5: NASS — Fuzzing Proprietary Native Android System Services

Published at USENIX Security '25, **NASS [3]** (Native Android System Services) is the most advanced of the three papers in this study. It addresses the final and perhaps most significant hurdle in Android security research: the **proprietary service problem.**

## 5.1 The Proprietary Blind Spot: The Running Example

On a modern commercial device (e.g., a Samsung S23), the `cameraserver` daemon (from AOSP) does not communicate directly with the kernel. Instead, it talks to a proprietary vendor HAL service (e.g., `vendor.qti.hardware.camera`). This service is a binary executable with no available source code.

This service is a "blind spot" for DIFUZE and FANS. Since there is no source code to analyze, static analysis fails completely. However, these services are highly privileged—they run as native processes with direct access to hardware and the kernel. A vulnerability in a HAL service can be exploited to bypass Android's security sandbox and eventually compromise the entire system.

## 5.2 RPC Design Principles: The Foundation for Analysis

NASS is built on the discovery of three universal RPC design principles that enable the analysis of closed-source binaries:
1.  **Ab (Abstraction of IPC binding code):** All IPC-related logic is abstracted into a "stub" or "proxy" layer. The actual business logic of the service does not handle raw Parcel data.
2.  **Si (Single entry point):** All incoming transactions pass through a single, well-defined entry point (`onTransact` in Binder). This provides a predictable place to hook and monitor incoming requests.
3.  **St (Standard deserialization routines):** The server stub uses standard routines exported from `libbinder.so` (e.g., `readInt32`, `readString16`) to deserialize arguments. 

The NASS authors verified these principles empirically. They analyzed 528 native services across 5 commercial devices and found that 89% of proprietary services strictly complied with all three principles. The deviations were mostly minor (e.g., a service parsing a raw byte array instead of using standard types, which violates **St**, or placing business logic directly in the stub, violating **Ab**). Because the vast majority comply, dynamic analysis via standard `libbinder` hooks is highly effective.

## 5.3 DGIE: Iterative Interface Probing

The core innovation of NASS is **DGIE** (Deserialization-Guided Interface Extraction). Instead of analyzing source code, NASS observes the server's behavior as it processes transactions.

### 5.3.1 The Parcelable Unrolling Insight
A major challenge in Android is the `Parcelable` object—a high-level, complex data structure. Previous black-box tools failed because they couldn't guess the internal structure of a proprietary `Parcelable`. However, NASS observes that at the ABI level, a `Parcelable` is simply "unrolled" into a linear sequence of standard **St** routines (e.g., `readInt32`, `readString16`). Because DGIE operates at the level of these standard routines, it doesn't need to know that a `CameraConfig` object exists; it only needs to fulfill the linear sequence of reads that the object's `readFromParcel` method executes.

### 5.3.2 Iterative Refinement Algorithm
DGIE works by repeatedly probing the server with partially-correct inputs and observing which standard deserializers (**St**) are called:
1.  **Phase 1: Coverage-Based RPC Discovery:** It iterates through possible transaction IDs, watching for new code coverage via Frida Stalker to find valid functions.
2.  **Phase 2: Step-by-Step Refinement:** NASS uses an iterative feedback loop to build the signature:
    * It sends a minimal transaction. 
    * It observes which deserializer the server calls first (e.g., `readInt32`). 
    * It updates its internal interface model to expect an `int32`.
    * It sends a *new* transaction with a valid integer followed by an EOF marker. 
    * It then observes the *second* deserializer call (e.g., `readString16`). 
    * This process repeats until the server stops calling `read` methods.

```mermaid
graph LR
    Start([Start Probe]) --> Send1[Send empty Parcel]
    Send1 --> Obs1{Observe St hook}
    Obs1 -->|readInt32| Update1[Interface: {int}]
    Update1 --> Send2[Send Parcel: Int + EOF]
    Send2 --> Obs2{Observe St hook}
    Obs2 -->|readString16| Update2[Interface: {int, string}]
    Update2 --> Success[Signature Recovered]
```

This process is highly effective because the server's own stub logic acts as a "validator" that reveals the expected interface definition one call at a time. By unrolling complex objects and iteratively probing, DGIE achieves **88%** accuracy in recovering signatures, compared to just 53% for previous message-capture techniques.

## 5.4 Coverage-Guided Feedback: The Evolutionary Loop

Unlike DIFUZE and FANS, NASS is a grey-box fuzzer. It uses code coverage to guide its search for vulnerabilities, employing an evolutionary loop similar to LibFuzzer.

### 5.4.1 Instrumentation via Frida Stalker
To collect coverage without source code, NASS uses dynamic binary instrumentation (DBI) with Frida Stalker. It follows a precise protocol to ensure stable and request-correlated coverage:
1.  **Entry Point Hooking:** It hooks the `onTransact` entry point.
2.  **PID-Based Isolation:** It checks the caller's PID. Since NASS is the one sending the transaction, it only tracks coverage if the call originates from its own fuzzer process. This isolates the fuzzer's activity from background system noise.
3.  **Thread-Level Tracing:** It starts Frida Stalker only on the specific thread that received the `onTransact` call and stops it as soon as the function returns. This provides a clean execution trace for each individual input.

## 5.5 Evaluation and Real-World Impact

NASS was evaluated on five commercial devices, including the Google Pixel 9 and Samsung S23. It discovered 12 unique memory-corruption vulnerabilities, resulting in five assigned CVEs.

### 5.5.1 Case Study: The Samsung S23 Heap Overflow
A significant discovery was a heap overflow in a proprietary vendor HAL service on the Samsung S23. This vulnerability was found because NASS's **DGIE (Deserialization-Guided Interface Extraction)** correctly recovered the interface of a complex, closed-source binary.

1.  **Interface Recovery via DGIE:** NASS probed the HAL service and observed that its `onTransact` method repeatedly called `readInt32` and `readString16`, which it identified as part of a nested `Parcelable` object.
2.  **Coverage-Guided Feedback:** The evolutionary loop prioritised inputs that increased code coverage within the service's private address space, which NASS traced using Frida Stalker.
3.  **Vulnerability Trigger:** Eventually, the fuzzer generated a `Parcel` containing a very long string embedded within the recovered `Parcelable` structure. The proprietary service's deserialization logic failed to properly check the size of the string, leading to a heap overflow.

Crucially, this vulnerability was in a vendor-specific HAL service for which no source code was available. Prior systems like DIFUZE and FANS would have been entirely unable to analyze this service, and simple message-capture tools would have struggled to understand the nested structure required to reach the vulnerable code. This case study demonstrates that for modern, proprietary Android services, dynamic interface recovery is the only path forward for security research.

## 5.6 Limitations: DBI Overhead and Asynchronous Processing

Despite successfully targeting proprietary binaries, the dynamic approach pioneered by NASS introduces entirely new classes of limitations.

The most severe operational weakness is **DBI (Dynamic Binary Instrumentation) performance overhead.** Because NASS operates on closed-source binaries, it cannot compile lightweight coverage trackers (like ASan or LibFuzzer's default instrumentation) directly into the service. Instead, it must rely on Frida Stalker to dynamically translate and hook execution instructions at runtime. This introduces massive overhead, drastically reducing the number of executions per second compared to compiled static fuzzers. 

A second major limitation is its handling of **asynchronous processing.** Modern Android services are highly concurrent. If a service receives an `onTransact` call, quickly offloads the heavy processing to a background worker thread, and immediately returns, NASS's coverage collection fails. Because NASS only traces the specific thread handling the `onTransact` entry point, it becomes completely "blind" to any bugs or coverage changes occurring in the background threads.

Finally, while NASS adds coverage feedback, it inherits the same **stateful chain limitations** as FANS. It still struggles to discover bugs that require long, highly specific sequences of multiple different transactions, as its evolutionary loop is optimized for exploring the depth of a single transaction rather than the breadth of a multi-stage state machine.
