# Chapter 1: Introduction

Smartphones centralize personal and corporate data, making them a high-value target for attackers. With Android's dominance in the global market, finding and patching vulnerabilities before exploitation is essential for user safety.

## 1.1 The Shifting Vulnerability Landscape

Android's security architecture has evolved significantly over a decade. In the early days, researchers frequently discovered bugs in high-level applications or the framework, relying on easily reachable memory corruptions or logic flaws. In response, Google and the open-source community implemented robust mitigations: application sandboxing, Mandatory Access Control (MAC) via SELinux, and compiler-level protections like Control Flow Integrity (CFI). Exploiting standard userspace applications became difficult.

As userspace hardened, the focus moved deeper into the system stack. Between 2014 and 2016, the proportion of reported Android vulnerabilities in the Linux kernel and its drivers jumped from 4% to nearly 40% [1]. 

More recently, attackers have pivoted again. They now look beyond the kernel to native system services, particularly those in the Hardware Abstraction Layer (HAL). Following Project Treble, these native daemons—written in C or C++ by vendors—serve as the primary bridge between unprivileged apps and low-level hardware. Because these services interact directly with the kernel, a single compromise can lead to total device takeover.

## 1.2 The Fuzzing Bottleneck at Structured Interfaces

Fuzz testing (fuzzing) is among the most effective techniques for hunting vulnerabilities. It works by feeding randomized or mutated inputs to a program to trigger crashes or memory violations.

Traditional fuzzers fail when testing privileged operating system interfaces. Interactions with a kernel driver via `ioctl`, or with an Android service via Binder, expect data in rigid formats. A fuzzer providing unstructured or mutated byte sequences is almost always rejected during the initial parsing phase. The `ioctl` dispatcher or deserialization routine fails basic structural validation, ending execution before the fuzzer accesses core application logic. This wastes nearly all execution cycles on shallow rejection paths.

## 1.3 The Evolution: From Static to Dynamic Analysis

To bypass these structural barriers, researchers developed **interface-aware fuzzing**. The fuzzer must understand the expected grammar and structure of its target interface before generating tests. Synthesizing inputs that satisfy parsing constraints allows the fuzzer to explore deeper logic where critical bugs hide.

This thesis examines the evolution of interface-aware fuzzing on Android, from early static techniques to modern dynamic instrumentation. Our analysis focuses on three systems:

1.  **DIFUZE (2017) [1]:** DIFUZE demonstrated that analyzing open-source kernel drivers could automatically recover `ioctl` structures, enabling automated fuzzing of Android device drivers at scale.
2.  **FANS (2020) [2]:** Targeting open-source native system services communicating via Binder IPC, FANS used Abstract Syntax Tree (AST) analysis to determine data types and map dependencies across multi-stage transactions.
3.  **NASS (2025) [3]:** The primary focus here. NASS resolves a limitation of its predecessors: the need for source code. Using dynamic binary instrumentation and deserialization-guided probing, NASS brings interface-aware, coverage-guided fuzzing to the proprietary, closed-source HAL services that run on most modern devices.

While DIFUZE and FANS provide historical and conceptual background—the "static analysis era"—this thesis emphasizes NASS and its dynamic approach to the "open-source blind spot."

## 1.4 Universal RPC Design Principles

NASS's key insight [3] is that most Remote Procedure Call (RPC) frameworks share a set of universal design principles. Whether it is Binder, gRPC, or Thrift, these systems tend to separate low-level IPC transport from the actual business logic using an abstraction layer. This "stub" code handles the boring work of receiving and validating data, which means all incoming remote requests route through a single, predictable function—like `onTransact` in Binder. Furthermore, these stubs rarely use custom parsing; instead, they rely on standard runtime libraries to call routines like `readInt32()`. 

The initial processing layer of an Android system service is therefore highly predictable, even if the business logic behind it is proprietary. Exploiting these principles makes it possible to reverse-engineer interfaces dynamically without source code.

## 1.5 Document Structure

Chapter 2 establishes the technical background on fuzzing and Android IPC. Chapter 3 examines kernel-level interface fuzzing through DIFUZE, while Chapter 4 explores userspace service fuzzing via FANS. Chapters 5 and 6 analyze NASS's dynamic extraction and coverage collection techniques. Chapter 7 synthesizes these findings, comparing trade-offs between static and dynamic analysis before Chapter 8 concludes.
