# Chapter 1: Introduction

Between 2014 and 2016, the number of Android vulnerabilities in the Linux kernel and its drivers jumped from 4% to nearly 40% [1]. That is a huge leap. It tells a story about how quickly an attack surface shifts when defenders harden one layer and leave another one exposed. Android's story since then has been a recurring version of the same pattern: you close one door, and the pressure just builds behind another.

## 1.1 Where the Vulnerabilities are Moving

To understand why these attacks work, it helps to know how Android separates processes from each other. The core idea is least privilege—no process gets more access than it needs. In the early days, researchers found bugs in high-level apps easily, usually through memory corruption or simple logic flaws. Because of this, Google implemented strong mitigations like application sandboxing and SELinux. It made exploiting standard apps much harder.

So, the focus moved deeper into the stack. More recently, attackers have pivoted again. They are looking beyond the kernel to native system services, especially those in the Hardware Abstraction Layer (HAL). Following Project Treble, these native daemons serve as the primary bridge between unprivileged apps and low-level hardware. Because these services interact directly with the kernel, a single compromise can lead to a total device takeover.

## 1.2 The Real Fuzzing Problem

Fuzz testing (fuzzing) is one of the best ways to hunt for vulnerabilities. It works by feeding random or mutated inputs to a program to trigger a crash. Simple idea. Surprisingly effective.

However, traditional fuzzers fail when they hit privileged interfaces. Coverage guidance only works if the fuzzer actually gets through the front door. If 99% of inputs fail validation, no new code paths are ever reached—the fuzzer just stalls. And that is exactly what happens here. Interactions with a kernel driver via `ioctl`, or with a Binder service, expect data in very rigid formats. If the bytes aren't exactly what the service expects, it just drops the request. This wastes almost all your time on shallow paths that don't find anything.

## 1.3 Static vs. Dynamic Analysis

To get past these barriers, researchers came up with **interface-aware fuzzing**. The fuzzer has to actually understand the grammar of its target before it starts. If you can satisfy the parsing constraints, you can explore the deeper logic where the real bugs are hiding.

This thesis looks at how interface-aware fuzzing on Android evolved from early static techniques to modern dynamic ones. This thesis focuses on three main systems:

1.  **DIFUZE (2017) [1]:** This tool showed that you could analyze open-source kernel drivers to automatically recover `ioctl` structures. 
2.  **FANS (2020) [2]:** This one targeted native system services. It used AST analysis to map out data types and dependencies across complex transactions.
3.  **NASS (2025) [3]:** This is the main focus here. NASS solves a huge problem: the need for source code. By using dynamic instrumentation, it can fuzz the proprietary, closed-source HAL services that run on most real devices.

While DIFUZE and FANS provide the background—the "static analysis era"—this thesis emphasizes NASS and its approach to the "open-source blind spot."

## 1.4 How RPC Frameworks are Designed

NASS's key insight is that most Remote Procedure Call (RPC) frameworks actually share the same design principles. Whether it is Binder, gRPC, or Thrift, these systems separate the IPC transport from the actual logic. There is usually "stub" code that handles the boring work of validating data. Because of this, all remote requests go through a single function—like `onTransact` in Binder. Also, these stubs rarely use custom parsing; they just use standard libraries to read integers or strings. 

The initial processing layer of an Android system service is therefore highly predictable, even if the business logic behind it is proprietary. We can use these principles to reverse-engineer interfaces dynamically without needing source code at all.

Without a map of these input structures, auditing the proprietary layers of modern phones is—in practice—effectively impossible. At least, no automated tool has managed it yet.

## 1.5 How this Thesis is Structured

Chapter 2 covers the technical background on fuzzing and Android IPC. Chapter 3 looks at kernel-level fuzzing through DIFUZE, and Chapter 4 explores userspace services via FANS. Chapters 5 and 6 analyze NASS's dynamic techniques. Then, Chapter 7 synthesizes the findings and compares the trade-offs before Chapter 8 wraps everything up.
