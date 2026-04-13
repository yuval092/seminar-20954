# Chapter 1: Introduction

Because smartphones centralize so much of our personal and corporate lives, they are a constant target for attackers. With Android dominating the global smartphone market, finding and patching security vulnerabilities before they can be exploited is essential to keeping users safe.

## 1.1 The Shifting Vulnerability Landscape

Android's security architecture has evolved significantly over the past decade. In the early days, researchers often found bugs directly in high-level applications or the Android framework, relying on easily reachable memory corruptions or simple logic flaws. In response, Google and the open-source community implemented robust mitigations, including strict application sandboxing, Mandatory Access Control (MAC) via SELinux, and compiler-level protections like Control Flow Integrity (CFI). These changes made exploiting standard userspace applications much more difficult.

As the userspace became more secure, attackers redirected their focus deeper into the system stack. Between 2014 and 2016, the proportion of reported Android vulnerabilities residing in the Linux kernel and its device drivers jumped from 4% to nearly 40% [1]. 

More recently, this focus has shifted again. Attackers are looking beyond the kernel to native system services, particularly those running within the Hardware Abstraction Layer (HAL). Following architectural changes like Project Treble, these native daemons—typically written in C or C++ by hardware vendors—serve as the primary bridge between unprivileged apps and low-level hardware drivers. Because these services often interact directly with the kernel, compromising just one of them can open a path to total device takeover. This makes them a prime target for both attackers and security researchers.

## 1.2 The Fuzzing Bottleneck at Structured Interfaces

When hunting for software vulnerabilities, fuzz testing (or fuzzing) is one of the most effective techniques. It works by feeding a program a massive number of randomized or mutated inputs to see if any of them cause crashes or memory violations.

However, traditional fuzzers hit a major bottleneck when testing the privileged interfaces of an operating system. Interactions with a Linux kernel driver via an `ioctl` system call, or with an Android native system service via Binder Inter-Process Communication (IPC), expect data to arrive in very specific, rigid formats. 

For instance, an interface might expect a serialized byte stream (a Binder `Parcel`) containing exact sequences of integers, strings, and object handles. An `ioctl` payload might require nested C structures with valid memory pointers. When a standard fuzzer provides unstructured or mutated byte sequences, the input is almost always rejected during the initial parsing phase. The target's `ioctl` dispatcher or deserialization routine fails basic structural validation, ending the execution before the fuzzer can access the core application logic. As a result, the fuzzer wastes almost all of its execution cycles on these shallow rejection paths.

## 1.3 The Evolution: From Static to Dynamic Analysis

To get past these structural barriers, the security community developed **interface-aware fuzzing**. The core idea is that the fuzzer needs to understand the expected grammar and structure of its target interface before it starts generating tests. By synthesizing inputs that satisfy the initial parsing constraints, the fuzzer can bypass the shallow rejections and systematically explore the deeper logic where the real bugs tend to hide.

This thesis explores how interface-aware fuzzing evolved on Android. It traces the journey from early static analysis techniques that relied on source code, to modern dynamic binary instrumentation that can analyze closed-source binaries on the fly. The analysis focuses on three foundational research systems:

1.  **DIFUZE (2017) [1]:** A system targeting the kernel layer. DIFUZE demonstrated that analyzing open-source kernel drivers could automatically recover complex `ioctl` structures, enabling the first automated fuzzing of Android device drivers at scale.
2.  **FANS (2020) [2]:** This system extended interface-awareness to the Android userspace. Targeting open-source native system services that communicate via Binder IPC, FANS used Abstract Syntax Tree (AST) analysis to figure out data types and map out dependencies across multi-stage transactions.
3.  **NASS (2025) [3]:** The primary focus of this thesis. NASS resolves a critical limitation of its predecessors: the need for source code. By using dynamic binary instrumentation and deserialization-guided probing, NASS brings interface-aware, coverage-guided fuzzing to the proprietary, closed-source HAL services that run on most modern commercial devices.

While DIFUZE and FANS provide the necessary historical and conceptual background—representing the "static analysis era"—this thesis focuses heavily on NASS and how its dynamic approach solves the "open-source blind spot."

## 1.4 Universal RPC Design Principles

One of the key insights behind NASS [3] is that most Remote Procedure Call (RPC) frameworks share a set of universal design principles. Whether it's Binder, gRPC, or Thrift, they generally adhere to three main rules:

1.  **Ab (Abstraction of IPC binding code):** Low-level IPC transport mechanics are kept separate from the actual business logic. Auto-generated or standard "stub" code handles receiving and validating the incoming requests.
2.  **Si (Single Entry Point):** All incoming remote requests for a given interface are routed through a single, predictable function signature (like `onTransact` in Binder). This provides a reliable place to intercept traffic.
3.  **St (Standard Deserialization Routines):** Server stubs generally avoid custom parsing logic. Instead, they rely on standard routines provided by the RPC framework's runtime library (like `readInt32()` within `libbinder.so`) to unpack the payload.

Because of these rules, the initial processing layer of an Android system service is actually highly predictable, even if the business logic behind it is proprietary and obfuscated. Exploiting these principles makes it possible to dynamically reverse-engineer interfaces without ever seeing the source code.

## 1.5 Document Structure

The remainder of this thesis is structured as follows:

*   **Chapter 2** provides the technical background, covering modern fuzzing methods, Android's privilege architecture, and the mechanics of `ioctl` and Binder IPC, as well as a comparison of static and dynamic program analysis techniques.
*   **Chapter 3** examines kernel-level interface fuzzing through the DIFUZE approach, discussing its LLVM bitcode analysis pipeline and the challenges of mapping out `ioctl` structures.
*   **Chapter 4** explores userspace system service fuzzing via FANS, explaining AST-based semantic analysis and how to navigate stateful Binder dependencies.
*   **Chapter 5** analyzes NASS, detailing Deserialization-Guided Interface Extraction (DGIE) and how it dynamically unrolls complex objects within proprietary HAL services.
*   **Chapter 6** continues with NASS, examining its approach to capturing isolated, thread-specific coverage from multi-threaded system daemons using dynamic binary instrumentation.
*   **Chapter 7** synthesizes these findings, comparing the trade-offs between static and dynamic analysis, and discussing the real-world engineering costs of dynamic instrumentation.
*   **Chapter 8** concludes the thesis, summarizing the main takeaways and pointing out open challenges in security research.
