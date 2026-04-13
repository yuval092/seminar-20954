# Chapter 1: Introduction

Mobile devices have fundamentally altered computing. Our smartphones store vast amounts of personal, financial, and corporate data, making them obvious targets for exploitation. Securing a platform like Android—which holds a dominant share of the global market—requires continuously identifying and patching vulnerabilities before they can be exploited.

## 1.1 The Shifting Vulnerability Landscape

Android's approach to security has evolved considerably. In the platform's early iterations, vulnerabilities were often found in high-level applications or the Android framework itself. Attackers frequently relied on straightforward memory corruption bugs or logic flaws in easily accessible components.

As Android matured, Google and the open-source community implemented numerous mitigations. These included strict app sandboxing, Mandatory Access Control (MAC) via SELinux, compiler-level protections like Control Flow Integrity (CFI), and a gradual shift toward memory-safe languages. Consequently, exploiting userspace applications became significantly more complex.

With the userspace increasingly hardened, attackers were forced to look deeper into the system stack to acquire necessary privileges. This shift is evident in historical data: between 2014 and 2016, the proportion of reported Android vulnerabilities located in the Linux kernel and its device drivers increased from 4% to nearly 40% [1].

More recently, this focus has broadened beyond the Linux kernel to include native system services, particularly those within the Hardware Abstraction Layer (HAL). Following architectural changes such as Project Treble, these native daemons—typically written in C or C++ by hardware vendors rather than Google—became the primary bridge between unprivileged apps and low-level hardware drivers. Because these services interact directly with the kernel, compromising one can provide an attacker with a clear path to system-wide control. This represents the modern battleground for Android security.

## 1.2 The Fuzzing Bottleneck at Structured Interfaces

When security researchers proactively seek vulnerabilities in complex software, they often rely on fuzz testing. Fuzzing involves feeding a large volume of randomized or mutated input to a target program and monitoring for unexpected behavior, such as crashes. It is an effective method for exploring a program's state space much faster than manual code review.

However, traditional fuzzing encounters a significant obstacle when testing the privileged interfaces of an operating system. Whether targeting a Linux kernel driver via the `ioctl` system call or an Android native system service via Binder Inter-Process Communication (IPC), these interfaces expect data structured in specific, predefined ways.

To illustrate this, consider a hypothetical **Android Camera Subsystem**, which will serve as a running example throughout this thesis. If an application needs to configure the camera sensor, it cannot manipulate hardware registers directly. It must construct a specific C-structure or a serialized byte stream (a Binder `Parcel`) containing exact sequences of integers, strings, and object handles. This payload is then passed across the privilege boundary.

If a standard fuzzer attempts to test this interface by sending random bytes, the input is typically rejected immediately. The driver's `ioctl` dispatcher or the camera service's deserialization routine will fail basic sanity checks, throwing an error before any complex logic is executed. The fuzzer wastes millions of cycles on these shallow rejection paths, missing the deeper, stateful code where vulnerabilities often reside.

## 1.3 The Evolution: From Static to Dynamic Analysis

To bypass these structural barriers, researchers developed **interface-aware fuzzing**. The core principle is that a fuzzer must understand the expected structure and grammar of its target interface *before* initiating testing. By generating inputs that are structurally correct enough to survive initial parsing, the fuzzer can bypass shallow rejection paths and explore deeper logic.

This seminar thesis examines the evolution of interface-aware fuzzing on Android, tracing the progression from early static source-code analysis to modern dynamic binary instrumentation. The analysis will focus on three major systems:

1.  **DIFUZE (2017) [1]:** A foundational tool focused on the kernel layer. DIFUZE demonstrated how analyzing the source code of open-source kernel drivers could automatically recover complex `ioctl` structures, enabling large-scale, automated fuzzing of Android device drivers.
2.  **FANS (2020) [2]:** This system brought interface-awareness to the Android userspace, targeting open-source native system services communicating via Binder IPC. FANS utilized Abstract Syntax Tree (AST) analysis to determine data types and infer dependencies between multi-stage transactions.
3.  **NASS (2025) [3]:** The primary focus of this thesis. NASS addresses the major limitation of its predecessors: the reliance on source code. By employing dynamic binary instrumentation and deserialization-guided probing, NASS brings interface-aware, coverage-guided fuzzing to the proprietary, closed-source HAL services prevalent on modern commercial devices.

While DIFUZE and FANS provide essential historical context—representing the "static analysis era"—the primary focus remains on NASS and how its dynamic approach overcomes the "open-source blind spot."

## 1.4 Universal RPC Design Principles

A key conceptual breakthrough enabling NASS [3] is the recognition that most Remote Procedure Call (RPC) frameworks share universal design principles. Regardless of whether the interface uses Binder, gRPC, or Thrift, they generally adhere to three main rules:

1.  **Ab (Abstraction of IPC binding code):** The low-level IPC transport specifics are separated from the core business logic. Auto-generated or standard "stub" code typically handles receiving and validating IPC requests.
2.  **Si (Single Entry Point):** All incoming remote requests for a given interface are routed through one predictable function signature (e.g., `onTransact` in Binder). This provides a reliable interception point for analysis.
3.  **St (Standard Deserialization Routines):** Server stubs generally avoid custom parsing logic. Instead, they rely on standard routines provided by the RPC framework's runtime library (e.g., `readInt32()` from `libbinder.so`) to unpack the payload.

Due to these principles, the initial processing layer of almost any Android system service is highly predictable, even if the underlying business logic is proprietary. As discussed in Chapter 4, exploiting these principles enables the dynamic reverse-engineering of interfaces without requiring source code access.

## 1.5 Document Structure

The remainder of this thesis is structured as follows:

*   **Chapter 2** provides the technical background, covering modern fuzzing techniques, Android's privilege architecture, and the mechanics of `ioctl` and Binder IPC.
*   **Chapter 3** examines the origins of interface-aware fuzzing through DIFUZE and FANS. It discusses their static analysis pipelines and why reliance on source code limits their applicability on modern devices.
*   **Chapter 4** focuses on NASS, detailing Deserialization-Guided Interface Extraction (DGIE) and its dynamic unrolling of complex objects.
*   **Chapter 5** continues with NASS, analyzing its method for collecting isolated, thread-specific coverage from multi-threaded system daemons using dynamic binary instrumentation.
*   **Chapter 6** synthesizes these findings, comparing the trade-offs between static and dynamic analysis and discussing the performance implications of dynamic instrumentation.
*   **Chapter 7** concludes the thesis, summarizing the main points and identifying open challenges for future security research.