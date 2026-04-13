# Presentation Deck Outline: First Zoom Meeting (45 Minutes)

## Objective
This deck is designed for the first Zoom meeting with the mentor. According to the `README.md` instructions, this presentation should cover the first half of the material (Background, DIFUZE, and FANS) before shifting to NASS in a later presentation. 

The mentor's grading criteria for presentations emphasize:
1.  **Visuals-First, Text-Minimal:** Max 3 bullet points per slide. Use diagrams.
2.  **Sequential Disclosure:** Use animations to walk through complex concepts step-by-side.
3.  **No Copy-Pasting:** The presentation must summarize concepts concisely, not read like a script.

---

## Slide 1: Title Slide
*   **Title:** Breaking the Front Door: The Evolution of Interface-Aware Fuzzing on Android
*   **Subtitle:** Part 1: The Static Analysis Era (Background, DIFUZE & FANS)
*   **Presenter:** [Your Name]
*   **Visual:** A stylized lock (representing the interface barrier) being opened by a gear (representing automated fuzzing).

---

## Slide 2: The Android Attack Surface is Shifting
*   **Visual:** A layered diagram of Android OS (App -> Framework -> HAL -> Kernel). Arrow animating downward from Kernel to HAL over time.
*   **Bullet 1:** Userspace mitigations (SELinux, Sandboxing) drove attackers deeper.
*   **Bullet 2:** Early 2010s: Monolithic Linux Kernel was the prime target.
*   **Bullet 3:** Modern Android: Project Treble shifted the attack surface to highly privileged, native vendor system services (HAL).

---

## Slide 3: The Fuzzing Bottleneck
*   **Visual:** A flowchart showing "Random Bytes (Fuzzer)" hitting a "Brick Wall (Deserialization Stub)" before it can reach "Vulnerable Logic".
*   **Bullet 1:** Fuzzers find bugs through mass mutation of inputs.
*   **Bullet 2:** Complex OS interfaces (`ioctl`, Binder) expect strictly formatted data structures.
*   **Bullet 3:** "Dumb" fuzzing fails instantly; inputs are rejected before reaching vulnerable logic.

---

## Slide 4: The Core Problem: The Camera Subsystem Example
*   **Visual:** A code snippet of a complex C-struct (`camera_sensor_config`) containing a nested pointer (`lens_calibration_data *`).
*   **Bullet 1:** Userspace apps must pass this exact structure to the kernel via `ioctl`.
*   **Bullet 2:** If a fuzzer mutates the nested pointer into garbage bytes...
*   **Animation (Sequential Disclosure):** ...a red "KERNEL PANIC" error box appears over the pointer deference. The system crashes instantly, hiding deeper bugs.

---

## Slide 5: The Interface-Aware Solution
*   **Visual:** The previous flowchart, but now the fuzzer is generating "Structured Bytes", which cleanly bypasses the "Deserialization Stub" and hits "Vulnerable Logic".
*   **Bullet 1:** A fuzzer must first learn the "language" of the interface.
*   **Bullet 2:** Method 1: Static Analysis (Compiling Source Code).
*   **Bullet 3:** Method 2: Dynamic Analysis (Observing Binary Execution).

---

## Slide 6: DIFUZE (2017) - Overcoming the `ioctl` Barrier
*   **Visual:** A pipeline diagram: [Kernel C Source] -> [GCC to LLVM Converter] -> [LLVM Bitcode] -> [DIFUZE Analysis Passes].
*   **Bullet 1:** Target: Open-source Linux kernel device drivers.
*   **Bullet 2:** Technique: Converts GCC kernel to LLVM bitcode for deep static analysis.
*   **Bullet 3:** Result: Automatically maps devices, command IDs, and expected C-structures.

---

## Slide 7: DIFUZE Innovation: Pointer Fixup
*   **Visual:** An animation showing a fuzzer creating a child object in memory, getting its virtual address, and dynamically injecting that address into the parent object before calling `ioctl`.
*   **Bullet 1:** Solves the "Nested Pointer Crash" problem.
*   **Bullet 2:** Fuzzer allocates valid userspace memory for nested objects dynamically.
*   **Bullet 3:** Forces the kernel to safely dereference fuzzed data.

---

## Slide 8: The Attack Surface Migrates: Welcome to Binder
*   **Visual:** Diagram showing an App sending a serialized `Parcel` through the `/dev/binder` driver to a System Service's `onTransact()` method.
*   **Bullet 1:** `ioctl` is for userspace-to-kernel. Binder is for userspace-to-userspace (App -> HAL).
*   **Bullet 2:** Binder relies on serialized byte streams (`Parcels`), not flat C-structures.
*   **Bullet 3:** Semantics matter: Data parsing often depends on runtime state.

---

## Slide 9: FANS (2020) - Navigating Binder Semantics
*   **Visual:** A tree diagram representing an Abstract Syntax Tree (AST), highlighting preserved variable names and complex type aliases.
*   **Bullet 1:** Target: Open-source Android Native System Services.
*   **Bullet 2:** Technique: Analyzes Clang ASTs instead of LLVM bitcode.
*   **Bullet 3:** Result: Preserves exact variable names and high-level C++ semantics lost during compilation.

---

## Slide 10: FANS Innovation: Dependency Inference (Algorithm 1)
*   **Visual:** A state graph showing `openConnection()` returning `session_id`, with an arrow pointing to `captureImage()` requiring `session_id` as input.
*   **Bullet 1:** Android services are highly stateful (transactions rely on previous transactions).
*   **Bullet 2:** FANS uses AST variable names (e.g., matching "out_session_id" to "in_session_id") to map relationships.
*   **Bullet 3:** Fuzzer can now organically navigate multi-stage logic flaws.

---

## Slide 11: The Fatal Flaw of the Static Era
*   **Visual:** A pie chart showing "Open Source (40%)" and "Proprietary Closed Source (60%)" with a magnifying glass only looking at the Open Source slice.
*   **Bullet 1:** DIFUZE and FANS require exact C/C++ source code to function.
*   **Bullet 2:** Reality: Over 60% of native services on modern commercial devices (Samsung, Google) are proprietary HAL binaries.
*   **Bullet 3:** The "Open-Source Blind Spot": Static analysis tools are blind to the most critical attack surface.

---

## Slide 12: Preview: The Shift to Dynamic Analysis
*   **Visual:** A black box (Proprietary Binary) with a fuzzer probing it dynamically.
*   **Bullet 1:** How do we fuzz what we cannot compile?
*   **Bullet 2:** Next Meeting: We will introduce NASS (2025) and dynamic binary instrumentation.
*   **Bullet 3:** Questions and Discussion.