# Seminar Thesis: Interface-Aware Fuzzing on Android

## Table of Contents

*   [**Abstract**](00_abstract.md)
1.  [**Chapter 1: Introduction**](01_introduction.md)
    *   1.1 The Shifting Vulnerability Landscape
    *   1.2 The Fuzzing Bottleneck at Structured Interfaces
    *   1.3 The Methodological Evolution: From Static to Dynamic Analysis
    *   1.4 Universal RPC Design Principles
    *   1.5 Document Structure
2.  [**Chapter 2: Technical Background**](02_background.md)
    *   2.1 Fuzzing Methodologies and the Coverage Imperative
    *   2.2 Android's Privilege Architecture and SELinux
    *   2.3 The Kernel Boundary: ioctl
    *   2.4 The Userspace Boundary: Binder IPC Architecture
3. [**Chapter 3: The Origins of Interface-Aware Fuzzing (DIFUZE & FANS)**](03_origins.md)
    *   3.1 The Static Analysis Era
    *   3.2 DIFUZE: Overcoming the ioctl Barrier via Bitcode Analysis
    *   3.3 FANS: Navigating Binder Semantics via AST Extraction
    *   3.4 The "Open-Source Blind Spot" and the Limits of Static Analysis
4.  [**Chapter 4: NASS and Dynamic Interface Extraction**](04_nass_dgie.md)
    *   4.1 The Proprietary Vendor HAL: The Modern Attack Surface
    *   4.2 Exploiting RPC Design Principles (Ab, Si, St)
    *   4.3 Deserialization-Guided Interface Extraction (DGIE)
    *   4.4 Unrolling Complex Parcelables Dynamically
5.  [**Chapter 5: NASS and Grey-Box Coverage in Multi-Threaded Daemons**](05_nass_coverage.md)
    *   5.1 The Concurrency Problem in Android System Services
    *   5.2 Thread-Localized Tracing via Dynamic Binary Instrumentation
    *   5.3 Evolutionary Fuzzing Feedback Loop
    *   5.4 Real-World Case Studies and Discovered Vulnerabilities
6. [**Chapter 6: Synthesis and Critical Critique**](06_synthesis.md)
    *   6.1 The Interface as the Universal Multiplier
    *   6.2 Trade-offs: Static Precision vs. Dynamic Applicability
    *   6.3 The High Cost of Dynamic Binary Instrumentation
    *   6.4 The Challenges of In-Situ Hardware Fuzzing
    *   6.5 Future Directions: Memory Safety and Beyond
7.  [**Chapter 7: Conclusion**](07_conclusion.md)
8.  [**Bibliography**](bibliography.md)