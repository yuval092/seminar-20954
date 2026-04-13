# Seminar Thesis: Interface-Aware Fuzzing on Android

## Table of Contents

1.  [**Chapter 1: Introduction**](01_introduction.md)
    *   1.1 The Shifting Vulnerability Landscape
    *   1.2 The Fuzzing Bottleneck
    *   1.3 Interface-Aware Fuzzing: The Solution
    *   1.4 Document Structure
2.  [**Chapter 2: Background**](02_background.md)
    *   2.1 Fuzzing Concepts
    *   2.2 Android’s Architecture and Attack Surface
    *   2.3 The ioctl Interface
    *   2.4 Android System Services and Binder
    *   2.5 The Running Example: The Camera Module
3.  [**Chapter 3: DIFUZE — Interface-Aware Fuzzing for Kernel Drivers**](03_difuze.md)
    *   3.1 The ioctl Challenge: The Running Example
    *   3.2 System Architecture: The LLVM Pipeline
    *   3.3 ioctl Handler Identification and Device Mapping
    *   3.4 Signature Extraction: The Core Algorithms
    *   3.5 Handling the Pointer Problem: Structure Fixup
    *   3.6 Evaluation and Key Results (Case Study: qseecom)
    *   3.7 Limitations: The Source Code Barrier
4.  [**Chapter 4: FANS — Moving Up the Stack to System Services**](04_fans.md)
    *   4.1 The Binder Challenge: The Running Example
    *   4.2 The Multi-Level Interface Problem
    *   4.3 AST-Based Extraction: Preserving Semantics
    *   4.4 Dependency Inference: The Multi-Stage Fuzzing Key
    *   4.5 The Fuzzer Engine and Evaluation (Case Study: netd)
    *   4.6 Limitations: The Open-Source Blind Spot
5.  [**Chapter 5: NASS — Fuzzing Proprietary Native Android System Services**](05_nass.md)
    *   5.1 The Proprietary Blind Spot: The Running Example
    *   5.2 RPC Design Principles: The Foundation for Analysis
    *   5.3 DGIE: Iterative Interface Probing
    *   5.4 Coverage-Guided Feedback: The Evolutionary Loop
    *   5.5 Evaluation and Real-World Impact (Case Study: Samsung S23)
    *   5.6 Limitations: DBI and Asynchrony
6.  [**Chapter 6: Synthesis — The Evolution of Interface-Aware Fuzzing**](06_synthesis.md)
    *   6.1 The Interface as the Multiplier
    *   6.2 Comparison of Fuzzing Systems
    *   6.3 Static vs. Dynamic Analysis: The Source Code Divide
    *   6.4 The Role of Coverage Feedback
    *   6.5 The Attack Surface in Motion
    *   6.6 Future Directions and Open Problems
7.  [**Chapter 7: Conclusion**](07_conclusion.md)
8.  [**Bibliography**](bibliography.md)
