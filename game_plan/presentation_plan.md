# Presentation Slide Game Plan & Storyboard

**Constraints (from Mentor):** 45 Minutes total. Visuals-First. Text-Minimal. No copy-pasting from the thesis. Sequential disclosure (animations) for complex logic.

## Design Philosophy
The slides will act as a visual backdrop to your oral explanation. If a slide can be understood without you speaking, it has too much text. We will rely heavily on the Mermaid diagrams we generated for the thesis, converted into polished graphics.

---

## Slide-by-Slide Storyboard with Speaker Notes

### Part 1: Hook & Motivation (4 mins)
*   **Slide 1: Title Slide.** 
    *   *Visual:* Title, Your Name, Date.
    *   *Speaker Notes:* Welcome the audience, introduce the seminar topic, and state the core problem.
*   **Slide 2: The Shifting Target.** 
    *   *Visual:* A line graph showing kernel/driver bugs rising from 4% (2014) to 39% (2016), with a new arrow pointing up to "System Services."
    *   *Speaker Notes:* 
        *   Discuss how mobile security has shifted over the years.
        *   Mention that as userspace apps get tightly sandboxed, attackers are forced to look deeper into the system.
*   **Slide 3: The Fuzzing Bottleneck.** 
    *   *Visual:* A fuzzer shooting random binary garbage at a brick wall labeled "Interface."
    *   *Speaker Notes:* 
        *   Briefly define fuzzing.
        *   Explain why naive fuzzing fails against strict interfaces: the inputs get rejected early by sanity checks before reaching the vulnerable logic.

### Part 2: Background (4 mins)
*   **Slide 4: The Android Attack Surface.**
    *   *Visual:* The 3-layer architecture diagram (App -> System Service -> HAL -> Kernel). 
    *   *Speaker Notes:* 
        *   Walk through the privilege layers.
        *   Introduce the "Camera Module" running example: how an app talks to cameraserver, which talks to the proprietary camera HAL, which talks to the v4l2 kernel driver.
*   **Slide 5: The Deserialization Firewall.**
    *   *Visual:* The sequence diagram of an `ioctl` call failing because of a bad struct pointer.
    *   *Speaker Notes:* 
        *   Explain the concept of the deserialization barrier.
        *   Emphasize that if the structure is wrong, the kernel panics or rejects it instantly, halting the fuzzing process.

### Part 3: DIFUZE - Kernel Drivers (9 mins)
*   **Slide 6: DIFUZE Overview.**
    *   *Visual:* Paper title and the LLVM Pipeline diagram.
    *   *Speaker Notes:* 
        *   Introduce DIFUZE (2017) as the foundational work in interface-aware fuzzing.
        *   Explain how it leverages LLVM static analysis on the kernel source code.
*   **Slide 7: Range Analysis.**
    *   *Visual:* A simplified C `switch(cmd)` statement on the left, an extracted "Valid Command Set" on the right.
    *   *Speaker Notes:* 
        *   Explain how DIFUZE extracts valid ioctl command IDs using range analysis and equality constraints.
*   **Slide 8: The Pointer Problem.**
    *   *Visual:* Two memory blocks. An animation showing the fuzzer allocating the sub-structure, then drawing an arrow (Pointer Fixup) from the parent structure to the child.
    *   *Speaker Notes:* 
        *   Explain the severe difficulty of handling nested pointers in kernel fuzzing.
        *   Detail the "pointer fixup" mechanism used during on-device execution to prevent immediate page faults.
*   **Slide 9: DIFUZE Results.**
    *   *Visual:* "36 Bugs. 7 Phones." A big red "Source Code Required" stamp.
    *   *Speaker Notes:* 
        *   Highlight the +54.5% bug rate increase achieved simply by adding structure info.
        *   Discuss the main limitation: we need the source code, which isn't always available for vendor drivers.

### Part 4: FANS - System Services (8 mins)
*   **Slide 10: Moving Up the Stack.**
    *   *Visual:* Zoom in on the "System Services" layer of the architecture diagram.
    *   *Speaker Notes:* 
        *   Transition to FANS (2020), which shifts focus to Binder IPC services.
*   **Slide 11: The Multi-Level Interface.**
    *   *Visual:* A tree graph showing an `open()` call returning a handle that allows a `startPreview()` call.
    *   *Speaker Notes:* 
        *   Explain that many interfaces are hidden (multi-level) and require sequential, stateful steps to reach.
*   **Slide 12: AST Extraction.**
    *   *Visual:* A snippet of Clang AST mapping a `readInt32()` call to a "Sequential Variable" node.
    *   *Speaker Notes:* 
        *   Explain why FANS chose AST over IR: to preserve variable names and types, which is critical for matching dependencies later.
*   **Slide 13: FANS Results & Limitation.**
    *   *Visual:* "30 Native Bugs." Red stamp: "Still needs source code. No coverage feedback."
    *   *Speaker Notes:* 
        *   Discuss the success on native services.
        *   Point out that closed-source, proprietary HALs remain a massive blind spot for this tool.

### Part 5: NASS - Proprietary HALs (10 mins)
*   **Slide 14: The Proprietary Blind Spot.**
    *   *Visual:* A pie chart showing 60% of services are closed-source.
    *   *Speaker Notes:* 
        *   Introduce NASS (2025). Emphasize that the most dangerous attack surface today is proprietary HALs.
*   **Slide 15: The 3 RPC Principles.**
    *   *Visual:* Quick diagram showing how an app interacts with a stub, not the business logic. Text: Abstraction, Single Entry, Standard Deserialization.
    *   *Speaker Notes:* 
        *   Explain how NASS leverages standard Binder libraries (St) to analyze black-box binaries without needing source code.
*   **Slide 16: DGIE Walkthrough (ANIMATED).**
    *   *Visual:* The iterative refinement loop.
    *   *Speaker Notes:* 
        *   (Click 1) Fuzzer sends empty Parcel.
        *   (Click 2) Observe `readInt32` via hook.
        *   (Click 3) Fuzzer updates interface and sends Int.
        *   (Click 4) Observe `readString`. Explain how this reconstructs the interface dynamically.
*   **Slide 17: Coverage Guided Feedback.**
    *   *Visual:* Frida Stalker hooking the `onTransact` thread, filtering by fuzzer PID.
    *   *Speaker Notes:* 
        *   Explain how NASS adds grey-box coverage feedback to reach deeper states, unlike the black-box approaches of DIFUZE and FANS.
*   **Slide 18: NASS Results.**
    *   *Visual:* "12 Bugs on COTS Devices (Pixel 9, Samsung S23)."
    *   *Speaker Notes:* 
        *   Mention the Samsung heap overflow case study to demonstrate real-world impact and effectiveness.

### Part 6: Synthesis & Conclusion (5 mins)
*   **Slide 19: The Evolution.**
    *   *Visual:* A comparison table of the three papers (Target, Analysis Type, Feedback, Source Dependency).
    *   *Speaker Notes:* 
        *   Summarize the historical shift from Static to Dynamic analysis, and from Black-box to Grey-box fuzzing.
*   **Slide 20: The Unifying Insight.**
    *   *Visual:* "The interface is both the barrier and the map."
    *   *Speaker Notes:* 
        *   Conclude the main narrative: understanding the interface is the absolute key to deep fuzzing on modern OSes.
*   **Slide 21: Open Problems & Q&A.**
    *   *Visual:* Bullet points on Asynchronous flows and Sanitizer-less detection.
    *   *Speaker Notes:* 
        *   Briefly touch on what's next in the field of fuzzing research.
        *   Open the floor for questions from the committee.
