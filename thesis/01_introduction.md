# Chapter 1: Introduction

Mobile devices have become the most pervasive and personal computing platform in history. They manage our communication, financial transactions, and highly sensitive personal data. Because of this, the security of mobile operating systems—particularly Android, which holds the largest global market share—is a critical area of research.

## 1.1 The Shifting Vulnerability Landscape

The security posture of Android has evolved significantly since its inception. In its early years, vulnerabilities were often found in high-level applications and common libraries. Over time, as the Android framework has become more hardened through modern security features like sandboxing, hardware-backed Keystores, and improved memory safety, attackers have been forced to shift their focus to deeper, more privileged layers of the system.

Data from the last decade reveals a striking trend: a massive increase in vulnerabilities discovered in the Linux kernel and its associated device drivers. For example, between 2014 and 2016, the percentage of reported Android vulnerabilities residing in the kernel and drivers jumped from 4% to 39%. More recently, this focus has further expanded to include the Hardware Abstraction Layer (HAL) and native system services. These are the privileged daemons that bridge the gap between high-level apps and low-level hardware, and they represent the modern frontier of Android exploitation.

## 1.2 The Fuzzing Bottleneck

Fuzz testing, or fuzzing, is widely considered the most effective automated tool for discovering vulnerabilities in complex software. By providing randomized or mutated inputs and monitoring for crashes, fuzzers explore a program's state space far more exhaustively than manual testing ever could.

Traditional fuzzing techniques, however, face a severe bottleneck when targeting the privileged interfaces of an operating system. Whether it is a kernel driver's `ioctl` call or a system service's Binder IPC transaction, these interfaces expect highly structured, semantically correct data. If a fuzzer just sends random bytes, the input is almost instantly rejected by the target's initial sanity checks. This "Interface Problem" means that a naïve fuzzer wastes nearly all of its execution cycles hitting a "front door" and never reaches the deep, complex code paths where critical memory-corruption bugs are actually hidden.

## 1.3 Interface-Aware Fuzzing: The Solution

To overcome this bottleneck, security researchers have developed what we now call **interface-aware fuzzing.** Instead of generating random data, these systems first "recover" the expected structure and grammar of the targeted interface. By generating inputs that are semantically correct enough to pass the initial validation, these fuzzers can finally reach and test the deep-seated logic of the system.

In this seminar work, I will examine the evolution of interface-aware fuzzing on Android through three seminal research papers:
1.  **DIFUZE (2017) [1]:** The first fully-automated system for recovering structured interfaces from kernel driver source code.
2.  **FANS (2020) [2]:** A system that moved the target upward to Android's native system services, using AST-based extraction and dependency modeling to navigate the complex Binder IPC layer.
3.  **NASS (2025) [3]:** The most recent advancement, which eliminates the requirement for source code through dynamic, deserialization-guided interface extraction. This finally enables the analysis of the vast proprietary "blind spot" on modern commercial devices.

## 1.4 Document Structure

This document is structured as follows. In Chapter 2, I provide the necessary technical background on fuzzing methodologies, Android's architecture, and the specific IPC mechanisms involved. Chapters 3, 4, and 5 dive into the technical details of DIFUZE, FANS, and NASS, exploring their design, innovations, and real-world results. In Chapter 6, I synthesize these findings to analyze overarching trends in the field, such as the shift from static to dynamic analysis. Finally, Chapter 7 concludes the seminar and highlights open problems for future research.