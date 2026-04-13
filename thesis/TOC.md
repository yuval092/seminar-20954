# Seminar Thesis: Interface-Aware Fuzzing on Android

## Table of Contents

*   [**Abstract**](00_abstract.md)
1.  [**Chapter 1: Introduction**](01_introduction.md)
    *   1.1 The Shifting Vulnerability Landscape
    *   1.2 The Fuzzing Bottleneck at Structured Interfaces
    *   1.3 The Evolution: From Static to Dynamic Analysis
    *   1.4 Universal RPC Design Principles
    *   1.5 Document Structure
2.  [**Chapter 2: Technical Background**](02_background.md)
    *   2.1 Fuzzing Methodologies and the Coverage Imperative
    *   2.2 Android's Privilege Architecture and SELinux
    *   2.3 Hardware Abstraction Layers (HAL) and Project Treble
    *   2.4 The Kernel Boundary: `ioctl`
    *   2.5 The Userspace Boundary: Binder IPC Architecture
    *   2.6 Program Analysis Techniques: Static (LLVM/AST) vs. Dynamic (DBI)
3.  [**Chapter 3: Kernel-Level Interface Fuzzing: The DIFUZE Approach**](03_difuze.md)
    *   3.1 The `ioctl` Fuzzing Challenge
    *   3.2 Static Interface Extraction Pipeline
    *   3.3 GCC-to-LLVM Bitcode Compilation
    *   3.4 Handler and Device Identification
    *   3.5 Recovering Command Values via Range Analysis
    *   3.6 Argument Type Identification and Type Propagation
    *   3.7 The Pointer Fixup Mechanism
    *   3.8 Real-World Case Studies (qseecom, Honor 8 nve)
4.  [**Chapter 4: Userspace System Service Fuzzing: The FANS Approach**](04_fans.md)
    *   4.1 The Semantic Barrier of Binder IPC
    *   4.2 Abstract Syntax Tree (AST) Extraction
    *   4.3 Modeling Parcel Semantics (Sequential, Conditional, Loop, Return)
    *   4.4 Resolving the Multi-Level Interface Problem
    *   4.5 Inter-Transaction Dependency Inference
    *   4.6 State Navigation and Deep Vulnerability Discovery
    *   4.7 Case Studies (ip6tables-restore, IDrm, statsd)
    *   4.8 The "Open-Source Blind Spot"
5.  [**Chapter 5: Proprietary Service Fuzzing: NASS and Dynamic Extraction**](05_nass_dgie.md)
    *   5.1 The Vendor HAL: The Modern Attack Surface
    *   5.2 Universal RPC Design Principles (Ab, Si, St)
    *   5.3 Deserialization-Guided Interface Extraction (DGIE)
    *   5.4 The Probing State Machine and Refinement Heuristics
    *   5.5 Dynamic Unrolling of Complex Parcelables
6.  [**Chapter 6: NASS and Grey-Box Coverage in Multi-Threaded Daemons**](06_nass_coverage.md)
    *   6.1 The Concurrency Problem in Android System Services
    *   6.2 Thread-Localized Tracing via Dynamic Binary Instrumentation
    *   6.3 The Evolutionary Fuzzing Feedback Loop
    *   6.4 Real-World Case Studies (Samsung S23 Heap Overflow, Pixel 9 UAF)
7.  [**Chapter 7: Synthesis and Critical Critique**](07_synthesis.md)
    *   7.1 The Interface as the Universal Multiplier
    *   7.2 Trade-offs: Static Precision vs. Dynamic Applicability
    *   7.3 The High Cost of Dynamic Binary Instrumentation
    *   7.4 The Challenges of In-Situ Hardware Fuzzing
    *   7.5 Future Directions: Memory Safety and Beyond
8.  [**Chapter 8: Conclusion**](08_conclusion.md)
*   [**Bibliography**](bibliography.md)