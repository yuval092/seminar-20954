# Thesis Improvement Strategy

## 1. Findings on the Current State of the Thesis

After a thorough review of the original research papers (DIFUZE, FANS, NASS) and the current draft chapters in the `thesis/` directory, here is an assessment of the thesis's current state:

### Structure and Content
*   **Alignment with TOC:** The draft strictly follows the planned structure outlined in `TOC.md`.
*   **Technical Accuracy:** The technical summaries of the papers are generally accurate and capture the main contributions, methodologies, and limitations of each system.
*   **Compliance with Mentor's Instructions:** The draft currently **fails** the mentor's primary directive: *"This is not a 'dry' summary; your writing must demonstrate that you have fully internalized and deeply understood the concepts."* It currently reads as a condensed, sequential summary of the three papers rather than an original synthesis.

### Length and Depth (The 20-30 Page Requirement)
*   **Critically Short:** The current markdown drafts total roughly 2,000 words, which equates to about 6-8 pages in a standard docx format. To reach the required 20-30 pages (approx. 6,000 - 10,000 words), the thesis requires a massive expansion in technical depth, original examples, and critical analysis. It is currently far too brief and high-level.

### "AI-ness" and Lack of Human Voice
The current draft exhibits strong signs of being AI-generated, which detracts from the required "originality" and "autonomy":
*   **Formulaic Structure:** Chapters begin and end with highly predictable, repetitive phrasing (e.g., "Published at [Conference], [Paper] addresses...", "In this seminar work, I will examine...").
*   **Superficial Use of the Running Example:** The "camera driver/service" running example is introduced at the beginning of chapters but is quickly abandoned in favor of summarizing the papers' original case studies (like `qseecom` or `netd`).
*   **Lack of Critical Synthesis:** The text presents the papers' claims as absolute facts without injecting the author's own critical perspective or comparing the papers' trade-offs organically.

### Missing Core Technical Sections
Several critical technical contributions from the papers were omitted or glossed over and must be included to achieve the required depth:
*   **DIFUZE:** The integration with Syzkaller vs. the bespoke MangoFuzz prototype, and the specific "Heartbeat" monitoring mechanism used for crash detection during on-device execution.
*   **FANS:** A detailed explanation of **Algorithm 1 (Name and Type Matching)** for inter-transaction dependency inference. Furthermore, the thesis misses how FANS handles AIDL-generated C++ code versus native C++ implementations during AST extraction.
*   **NASS:** The thesis completely glosses over NASS's **Coverage Collection mechanism** for multi-threaded RPC servers (using Frida Stalker hooked to `onTransact` and isolated by the caller's PID). This is a massive contribution that enables grey-box fuzzing of these complex daemons. 

### Overly Descriptive / "Dry" Sections
Conversely, some sections fall into the trap of being overly descriptive manuals rather than academic analysis:
*   **Background (`ioctl` encoding):** The detailed bit-level breakdown of the `ioctl` command ID (Direction, Size, Type/Magic, Number) is Linux kernel trivia that distracts from the core concept of structured interface parsing.
*   **FANS (AST Node Types):** Exhaustively listing the "seven kinds of sequential statements" reads like a textbook. This should be abstracted to explain *why* loop and conditional variables create semantic barriers for fuzzers, rather than just listing the AST nodes.

### State of the Technical Background (Chapter 2)
While structurally sound, Chapter 2 is currently too brief to anchor a 20-30 page thesis. 
*   **Missing Depth:** It needs a significantly deeper dive into the **Binder IPC Architecture** (the Proxy/Stub pattern, the `ServiceManager`, and exact `Parcel` serialization mechanisms). 
*   **Missing Context:** It needs a dedicated section on **Android's Security Model and SELinux** to explain *why* compromising a HAL service escalates privileges. 
*   **Universal Framing:** The three "RPC Design Principles" (Ab, Si, St) introduced in NASS are actually universal and should be introduced here in the background to frame the entire thesis's discussion on RPC interfaces.

---

## 2. Detailed Game Plan for Improvement

To elevate the thesis to a high academic standard that meets the mentor's specific criteria (Deep Understanding, Originality, Autonomy, Curation, and Length), we must execute the following plan:

### Phase 1: Expansion for Length and Depth
*   **Action:** Significantly expand the technical explanations of every core algorithm.
*   **Execution:** 
    *   Expand Chapter 2 (Background) to deeply cover Binder IPC, SELinux domains, and the universal RPC Design Principles.
    *   Add dedicated, deep-dive subsections for the missing technical components identified above (FANS Algorithm 1, NASS Coverage Collection, DIFUZE MangoFuzz/Heartbeat).
    *   Include extensive, original code snippets and diagrams (e.g., show a pseudo-code C++ Binder transaction, its AST representation, and how FANS parses it).

### Phase 2: Deep Integration of the Original Running Example
*   **Action:** Completely rewrite the technical explanation sections to use the "Camera Module" running example continuously.
*   **Execution:** 
    *   **DIFUZE:** Invent a hypothetical `v4l2` camera `ioctl` structure containing pointers, and walk through exactly how DIFUZE's Type Propagation and Pointer Fixup would handle it, rather than relying on the paper's `qseecom` example.
    *   **FANS:** Illustrate AST extraction and inter-transaction dependencies using a hypothetical `ICameraDevice` Binder interface (e.g., a `connect()` call returning a camera session handle used in `capture()`).
    *   **NASS:** Explain the DGIE unrolling process by showing how a proprietary camera HAL's `CameraConfig` Parcelable would be iteratively probed.
*   **Goal:** This directly addresses the mentor's requirement to provide "your own examples and adding explanations beyond what is found in the original source material," while organically expanding the page count.

### Phase 3: Tone Adjustment and De-Roboticization
*   **Action:** Strip out formulaic AI transitions and adopt a cohesive, human academic voice.
*   **Execution:** Remove repetitive chapter intros. Make the text flow logically as a single, unified argument about the evolution of fuzzing, rather than three separate book reports.

### Phase 4: Aggressive Curation of "Dry" Facts
*   **Action:** Cut exhaustive, list-like summaries of the papers' implementations and elevate the conceptual discussion.
*   **Execution:** 
    *   Remove the bit-level `ioctl` breakdown and the 7 types of sequential statements in FANS. 
    *   Instead of listing every bug found in the evaluations, focus the evaluation sections on *deep case study analyses* of one or two specific bugs per paper, showing exactly how the fuzzer's specific innovation allowed it to reach the vulnerable code.
*   **Goal:** Address the mentor's note: "Deciding how to curate the content—what to emphasize and what to omit—is a core part of the assignment." 

### Phase 5: Enhance Synthesis and Critique (Chapter 6)
*   **Action:** Inject the author's voice into the limitations and transition sections.
*   **Execution:** Expand Chapter 6 into a robust critical analysis. Compare the trade-offs of static vs. dynamic analysis. Critically analyze the "open-source blind spot" and discuss how the Android ecosystem's architectural shifts (like Project Treble) necessitated the evolution from FANS to NASS. Discuss the future of Android security (e.g., the planned rewrite of the Binder driver in Rust).