# Thesis Expansion Plan: Deepening Technical Rigor

This document outlines the strategy for expanding the seminar thesis to meet the required length ("several dozen pages") by increasing technical depth and accuracy, while strictly adhering to the content of the three source papers (DIFUZE, FANS, NASS).

## 1. Tactical Expansion Strategy

### A. Strict Scope Adherence
*   **Included:** Any mechanism, algorithm, result, or architectural detail mentioned in the three papers.
*   **Excluded:** General Android security topics (exploit chains, app-level vulnerabilities) not directly relevant to the papers' threat models.
*   **Consistency:** The "Camera Module" running example will be used to explain all three systems.

### B. Depth Over Breadth
Instead of adding new topics, we will "unroll" the existing ones:
1.  **Algorithm Breakdown:** Instead of saying "FANS infers dependencies," we will describe the logic of the dependency types (Intra- vs. Inter-transaction).
2.  **Data Structures:** Describe the physical layout of the interfaces (ioctl structs vs. Binder Parcels).
3.  **Evaluation Analysis:** Instead of summarizing results, we will analyze the *specific reasons* the authors give for those results (e.g., why DIFUZE's structure recovery led to 54% more bugs).

---

## 2. Chapter-by-Chapter Tasks

### Chapter 2: Background
*   **Task:** Define the "Interface Problem" technically.
*   **Depth:** Explain the `ioctl` command encoding structure (type, number, size, direction) from DIFUZE. Explain the Binder `Parcel` serialization sequence (how `writeInt32` etc. creates a linear stream) from FANS/NASS.
*   **Visual:** Mermaid diagram of the Binder IPC stack.

### Chapter 3: DIFUZE
*   **Task:** Deepen the LLVM Static Analysis section.
*   **Depth:** Detail the "Pointer Fixup" mechanism—how the fuzzer manages userspace-to-kernel address translation. Explain the "Heartbeat" mechanism used for crash detection during on-device execution.
*   **Case Study:** Expand the `qseecom` analysis with specific structure examples from the paper.

### Chapter 4: FANS
*   **Task:** Expand the "Extraction" and "Dependency" sections.
*   **Depth:** Detail the 7 kinds of sequential statements and 4 variable classes extracted from the Clang AST (§4.1). Describe the logic for Inter-transaction dependency (output of one function serving as the input for another).
*   **Example:** Show how a Camera Service's `open()` call creates a dependency for a `startPreview()` call.

### Chapter 5: NASS
*   **Task:** Expand the "DGIE" and "Coverage" sections.
*   **Depth:** Explain the "Parcelable Unrolling" logic—how high-level objects are decomposed into standard primitives. Describe the Frida Stalker thread-isolation technique (PID filtering and entry/exit hooking).
*   **Principle:** Detail the "Three RPC Design Principles" (Ab, Si, St) and the evidence provided for their universality.

---

## 3. Immediate Roadmap

1.  **[T1] Background Depth:** Expand `02_background.md` with technical Binder/ioctl layout details.
2.  **[T2] FANS Extraction Depth:** Expand `04_fans.md` with the AST variable classification logic.
3.  **[T3] NASS Principle Depth:** Expand `05_nass.md` with the RPC design principle verification.
4.  **[T4] Running Example Sync:** Audit all chapters for the Camera Module thread.
5.  **[T5] Mermaid Integration:** Add architecture and logic diagrams.
