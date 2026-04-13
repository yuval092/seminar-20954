# Chapter 1: Introduction

Smartphones have fundamentally changed how we use computers. Because they store our personal messages, banking details, and work emails, they are an obvious target for malicious hackers. When a platform like Android dominates the global smartphone market, finding and patching security vulnerabilities before attackers can exploit them becomes incredibly important.

## 1.1 The Shifting Vulnerability Landscape

Android's approach to security has changed a lot over the last decade. In the platform's early years, it was pretty common to find bugs in high-level applications or the Android framework itself. Attackers would often rely on straightforward memory corruption bugs or simple logic flaws that were relatively easy to reach.

As Android grew up, Google and the open-source community started pushing back. They introduced strict app sandboxing, Mandatory Access Control (MAC) via SELinux, compiler-level protections like Control Flow Integrity (CFI), and started shifting toward memory-safe languages. Exploiting a regular app running in userspace went from being relatively simple to extremely difficult.

With the userspace locked down, attackers were forced to look deeper into the system stack to get the privileges they wanted. The data backs this up: between 2014 and 2016, the number of reported Android vulnerabilities located in the Linux kernel and its device drivers jumped from 4% to nearly 40% [1].

Recently, this focus has shifted again. Attackers aren't just looking at the Linux kernel anymore; they're also targeting native system services, particularly those inside the Hardware Abstraction Layer (HAL). After architectural updates like Project Treble, these native daemons—which are usually written in C or C++ by hardware vendors rather than Google—became the primary bridge between unprivileged apps and low-level hardware drivers. Because these services interact directly with the kernel, compromising just one can give an attacker a clear path to taking over the entire device. This is where the modern battleground for Android security really lies.

## 1.2 The Fuzzing Bottleneck at Structured Interfaces

When security researchers proactively hunt for vulnerabilities in complex software, they almost always use fuzz testing. Fuzzing involves throwing a massive amount of randomized or mutated input at a target program and watching to see if it crashes. It's a great way to explore a program's state space much faster than a human could by reading the code line by line.

However, traditional fuzzing hits a massive roadblock when testing the privileged interfaces of an operating system. Whether you are targeting a Linux kernel driver through an `ioctl` system call or an Android native system service via Binder Inter-Process Communication (IPC), these interfaces expect data to be structured in highly specific, predefined ways.

To help illustrate this, let's use a hypothetical **Android Camera Subsystem** as a running example throughout this thesis. If an app wants to configure the camera sensor, it can't just manipulate the hardware registers directly. It has to build a specific C-structure or a serialized byte stream (called a Binder `Parcel`) containing exact sequences of integers, strings, and object handles. This structured payload is then passed across the privilege boundary to the camera service.

If a standard fuzzer tries to test this interface by sending a bunch of random bytes, the input usually gets rejected immediately. The driver's `ioctl` dispatcher or the camera service's deserialization routine will fail basic sanity checks. It throws an error before any of the actual complex logic is executed. The fuzzer ends up wasting millions of cycles on these shallow rejection paths, completely missing the deeper, stateful code where the real bugs are usually hiding.

## 1.3 The Evolution: From Static to Dynamic Analysis

To get past these structural barriers, researchers came up with **interface-aware fuzzing**. The core idea is simple: a fuzzer has to understand the expected structure and grammar of its target interface *before* it starts testing. By generating inputs that are structurally correct enough to survive that initial parsing stage, the fuzzer can bypass the shallow rejections and start exploring the deeper logic.

In this seminar thesis, I will examine how interface-aware fuzzing has evolved on Android. The story traces the progression from early static source-code analysis techniques to modern dynamic binary instrumentation. I'll focus the analysis on three major research systems:

1.  **DIFUZE (2017) [1]:** A foundational tool that focused on the kernel layer. DIFUZE showed how analyzing the source code of open-source kernel drivers could automatically recover complex `ioctl` structures, which enabled the first large-scale, automated fuzzing of Android device drivers.
2.  **FANS (2020) [2]:** This system brought interface-awareness to the Android userspace. It targeted open-source native system services communicating via Binder IPC. FANS utilized Abstract Syntax Tree (AST) analysis to figure out data types and infer dependencies between multi-stage transactions.
3.  **NASS (2025) [3]:** This is the primary focus of my thesis. NASS addresses the biggest limitation of its predecessors: the reliance on source code. By using dynamic binary instrumentation and deserialization-guided probing, NASS brings interface-aware, coverage-guided fuzzing to the proprietary, closed-source HAL services that run on most commercial devices today.

While DIFUZE and FANS provide essential historical context—representing what we can call the "static analysis era"—my primary focus will be on NASS and how its dynamic approach overcomes the "open-source blind spot."

## 1.4 Universal RPC Design Principles

One of the key conceptual breakthroughs that makes NASS possible [3] is the realization that most Remote Procedure Call (RPC) frameworks share universal design principles. Regardless of whether the interface uses Binder, gRPC, or Thrift, they generally follow three main rules:

1.  **Ab (Abstraction of IPC binding code):** Good software engineering separates low-level IPC transport details from the core business logic. Auto-generated or standard "stub" code usually handles receiving and validating the IPC requests.
2.  **Si (Single Entry Point):** All incoming remote requests for a given interface are routed through one predictable function signature (like `onTransact` in Binder). This gives us a reliable place to intercept and monitor traffic.
3.  **St (Standard Deserialization Routines):** Server stubs generally avoid using custom parsing logic. Instead, they rely on standard routines provided by the RPC framework's runtime library (like calling `readInt32()` from `libbinder.so`) to unpack the payload.

Because of these principles, the initial processing layer of almost any Android system service is highly predictable, even if the underlying business logic is completely proprietary. As I will discuss in Chapter 4, exploiting these principles allows us to dynamically reverse-engineer interfaces without ever needing to look at the source code.

## 1.5 Document Structure

The rest of this thesis is structured as follows:

*   **Chapter 2** covers the technical background, explaining modern fuzzing techniques, Android's privilege architecture, and the mechanics of `ioctl` and Binder IPC.
*   **Chapter 3** examines the origins of interface-aware fuzzing through DIFUZE and FANS. I'll discuss their static analysis pipelines and why relying on source code limits their usefulness on modern devices.
*   **Chapter 4** shifts the focus to NASS, detailing Deserialization-Guided Interface Extraction (DGIE) and how it unrolls complex objects dynamically.
*   **Chapter 5** continues with NASS, analyzing its method for collecting isolated, thread-specific coverage from noisy, multi-threaded system daemons using dynamic binary instrumentation.
*   **Chapter 6** synthesizes these findings. I'll compare the trade-offs between static and dynamic analysis and discuss the real-world performance implications of dynamic instrumentation.
*   **Chapter 7** concludes the thesis, summarizing the main points and pointing out some open challenges for future security research.