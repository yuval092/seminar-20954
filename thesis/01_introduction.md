# Chapter 1: Introduction

Smartphones are ubiquitous, centralizing personal and corporate data, which renders them persistent targets for exploitation. With Android dominating the global smartphone market, the timely discovery and mitigation of security vulnerabilities is paramount.

## 1.1 The Shifting Vulnerability Landscape

Android's security architecture has undergone significant transformation. Historically, vulnerabilities were frequently discovered within high-level applications or the Android framework itself, often manifesting as accessible memory corruption or logic flaws. In response, Android implemented robust mitigations, including strict application sandboxing, Mandatory Access Control (MAC) via SELinux, and compiler-level protections like Control Flow Integrity (CFI). These mechanisms substantially elevated the difficulty of exploiting userspace applications.

As the userspace became more secure, attackers redirected their focus deeper into the system stack. Between 2014 and 2016, the proportion of reported Android vulnerabilities residing in the Linux kernel and its device drivers escalated from 4% to nearly 40% [1]. 

More recently, this focus has expanded beyond the kernel to native system services, particularly those executing within the Hardware Abstraction Layer (HAL). Following architectural shifts such as Project Treble, these native daemons—typically implemented in C or C++ by hardware vendors—serve as the primary intermediaries between unprivileged applications and low-level hardware drivers. Because these services interact directly with the kernel, compromising a single service can yield a pathway to complete device compromise, positioning them at the forefront of Android security research.

## 1.2 The Fuzzing Bottleneck at Structured Interfaces

In proactive vulnerability discovery, fuzz testing (or fuzzing) is a foundational technique. Fuzzing explores a program's state space by supplying randomized or mutated inputs, monitoring for anomalies such as crashes or memory violations.

However, traditional fuzzing encounters significant limitations when applied to the privileged interfaces of an operating system. Interactions with a Linux kernel driver via an `ioctl` system call or with an Android native system service via Binder Inter-Process Communication (IPC) require data to conform to highly specific, predefined structures. 

For instance, an interface may expect a serialized byte stream (a Binder `Parcel`) containing exact sequences of integers, strings, and object handles, or an `ioctl` payload requiring nested C structures with valid pointers. When a standard fuzzer provides unstructured or mutated byte sequences, the input is typically rejected during the initial parsing phase. The target's `ioctl` dispatcher or deserialization routine fails basic structural validation, terminating execution before the fuzzer can access the core application logic. Consequently, the fuzzer expends substantial computational resources on shallow rejection paths.

## 1.3 The Evolution: From Static to Dynamic Analysis

To overcome these structural barriers, researchers developed **interface-aware fuzzing**. This methodology dictates that a fuzzer must comprehend the expected structure and grammar of a target interface prior to test generation. By synthesizing inputs that satisfy the initial parsing constraints, the fuzzer can bypass shallow rejections and systematically explore deeper logic.

This seminar thesis investigates the evolution of interface-aware fuzzing on Android, tracing the progression from early static source-code analysis techniques to contemporary dynamic binary instrumentation. The analysis focuses on three seminal research systems:

1.  **DIFUZE (2017) [1]:** A foundational system targeting the kernel layer. DIFUZE demonstrated that analyzing open-source kernel drivers could automatically recover complex `ioctl` structures, enabling the automated fuzzing of Android device drivers.
2.  **FANS (2020) [2]:** This system extended interface-awareness to the Android userspace. Targeting open-source native system services communicating via Binder IPC, FANS utilized Abstract Syntax Tree (AST) analysis to infer data types and dependencies across multi-stage transactions.
3.  **NASS (2025) [3]:** The primary focus of this thesis, NASS resolves a critical limitation of its predecessors: the dependence on source code. Employing dynamic binary instrumentation and deserialization-guided probing, NASS facilitates interface-aware, coverage-guided fuzzing on the proprietary, closed-source HAL services deployed on commercial devices.

While DIFUZE and FANS establish the historical and conceptual context—representing the "static analysis era"—this thesis predominantly examines NASS and how its dynamic methodology circumvents the "open-source blind spot."

## 1.4 Universal RPC Design Principles

A core conceptual advancement underpinning NASS [3] is the recognition that diverse Remote Procedure Call (RPC) frameworks share universal design principles. Frameworks such as Binder, gRPC, and Thrift generally adhere to three primary tenets:

1.  **Ab (Abstraction of IPC binding code):** IPC transport mechanisms are architecturally separated from business logic. Auto-generated or standard "stub" code processes the reception and validation of IPC requests.
2.  **Si (Single Entry Point):** Incoming remote requests for an interface are routed through a singular, predictable function signature (e.g., `onTransact` in Binder), providing a reliable location for traffic interception.
3.  **St (Standard Deserialization Routines):** Server stubs avoid custom parsing, relying instead on standard routines provided by the RPC framework's runtime library (e.g., `readInt32()` within `libbinder.so`) to extract payloads.

These principles dictate that the initial processing layer of an Android system service is highly predictable, independent of the underlying proprietary logic. Exploiting these principles facilitates the dynamic reverse-engineering of interfaces without source code access.

## 1.5 Document Structure

The remainder of this thesis is structured as follows:

*   **Chapter 2** provides the technical background, detailing fuzzing methodologies, Android's privilege architecture, and the mechanics of `ioctl` and Binder IPC, alongside a comparison of static and dynamic program analysis techniques.
*   **Chapter 3** examines kernel-level interface fuzzing through the DIFUZE approach, discussing its LLVM bitcode analysis pipeline and the challenges of recovering `ioctl` structures.
*   **Chapter 4** explores userspace system service fuzzing via FANS, detailing AST-based semantic analysis and the navigation of stateful Binder dependencies.
*   **Chapter 5** analyzes NASS, detailing Deserialization-Guided Interface Extraction (DGIE) and the dynamic unrolling of complex objects within proprietary HAL services.
*   **Chapter 6** continues with NASS, examining its methodology for capturing isolated, thread-specific coverage from multi-threaded system daemons via dynamic binary instrumentation.
*   **Chapter 7** synthesizes these findings, comparing the trade-offs between static and dynamic analysis and discussing the engineering implications of dynamic instrumentation.
*   **Chapter 8** concludes the thesis, summarizing the principal findings and identifying open challenges in security research.
