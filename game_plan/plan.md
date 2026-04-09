# Seminar Game Plan: Interface-Aware Fuzzing on Android

This is a rich and well-chosen set of papers. The three form a natural chronological arc — DIFUZE (2017) → FANS (2020) → NASS (2025) — each building on the previous one's limitations. Let me lay out a complete game plan.

---

## Part 1: Key Takeaways From Each Paper

### DIFUZE (CCS '17) — The Foundation

**Central problem:** The `ioctl` interface is the primary way userspace talks to kernel device drivers on POSIX systems. Its third argument is an _arbitrary, driver-defined structure_, making naive fuzzing nearly useless — a fuzzer sending random bytes will almost always fail at the `switch(cmd)` dispatch and never reach the real logic.

**Key contributions:**

- First fully-automated system for interface recovery from kernel driver source code, using LLVM static analysis
- Identifies ioctl handler functions by looking for assignments to known `file_operations`-type structures
- Recovers device file paths (e.g. `/dev/example_device`) by tracing from registration functions
- Recovers valid command IDs via path-sensitive, inter-procedural analysis of equality constraints on the `cmd` argument
- Recovers argument types by tracing paths to `copy_from_user`, handling wrappers, casts, and nested structures
- Handles pointer-containing structures (the hardest case) by generating sub-structures independently and doing pointer fixup on-device
- Results: 36 bugs on 7 Android phones, 32 previously unknown, ranging from DoS to arbitrary code execution
- Key insight from evaluation: adding structure type information on top of command IDs increased bug count by 54.5% — the interface really matters

**What DIFUZE cannot do:** It requires source code. It has no coverage feedback. It targets `ioctl` only, not Android's higher-level IPC mechanisms.

---

### FANS (USENIX Security '20) — Moving Up the Stack

**Central problem:** Above the kernel driver layer, Android's attack surface consists of _native system services_ — C++ daemons that apps communicate with via the Binder IPC mechanism. These services expose transactions over a unified interface `IBinder::transact(code, data, reply, flags)`. The `data` parcel contains serialized, service-specific, structured arguments. Fuzzing this blindly fails for the same reason as ioctl: inputs that don't match the expected format are rejected immediately.

**Key contributions:**

- Recognizes _multi-level interfaces_ — interfaces not directly registered in the ServiceManager but retrievable via top-level interfaces (some buried 5 levels deep). Prior work missed 37% of the attack surface.
- Interface model extraction from the **AST** (not IR), which preserves variable names and types — crucial for semantic correctness. Extracts sequential, conditional, and loop variables; handles return-path pruning.
- **Dependency inference**: intra-transaction (conditional and loop dependencies between variables) and inter-transaction (output of one transaction feeds input of another, inferred by type + name similarity)
- Handles the AIDL-generated code case by recording compilation commands and scanning all produced files
- Results: 30 native vulnerabilities, 138 unique Java exceptions, on 6 phones with Android 9; 20 confirmed by Google
- Key insight: without correct interface model, deep code paths are unreachable; semantic input generation (e.g., generating a valid `packageName`) is necessary

**What FANS cannot do:** Requires source code — cannot fuzz proprietary services. No coverage feedback — fuzzer is black-box. Cannot handle binary-only (closed-source) HAL services.

---

### NASS (USENIX Security '25) — Closing the Blind Spot

**Central problem:** On 5 modern COTS devices, 316 out of 528 native services (60%) are entirely proprietary, and 92% of proprietary services run in the HAL layer with direct kernel access. FANS cannot touch any of them. Meanwhile, a pattern is emerging where attackers compromise a system service _first_ (e.g., `cameraserver` in the wild CVE-2024-44068) and then exploit the kernel. The attack surface is shifting upward.

**Key contributions:**

- Identifies 3 universal RPC design principles that hold for Binder (and verified for gRPC and Thrift): **Ab** (abstraction of IPC binding code), **Si** (single entry point), **St** (standard deserialization routines)
- **DGIE (Deserialization-Guided Interface Extraction):** Dynamically hooks standard deserialization routines (exported by `libbinder.so`) and iteratively probes the server, observing which deserializers are called in sequence. Recovers the complete "deserializer-level" function signature without source code. Achieves 88% accuracy vs. 53% for BinderCracker's message-capture approach.
- **Coverage collection:** Hooks the `onTransact` entry point (Si), checks the caller PID to isolate NASS's own requests, and runs Frida Stalker on that thread only until `onTransact` returns. This gives stable, request-correlated coverage from multi-threaded services.
- Full grey-box evolutionary fuzzing loop: seed corpus, interface-aware mutators (23 argument types, Parcelable-aware), LibFuzzer scheduler
- Results: 12 unique memory-corruption bugs on 5 up-to-date COTS devices (Google Pixel 9, Samsung S23, Xiaomi, OnePlus, Infinix), 5 CVEs assigned. Found bugs FANS cannot find (proprietary services) and bugs FANS missed even on open-source services.
- 89% of proprietary COTS services are compliant with all 3 RPC design principles — DGIE works.

---

## Part 2: Written Document Game Plan

### Main Narrative

The document tells a coherent story of **the evolution of interface-aware fuzzing on Android**, motivated by a single unifying insight: _the interface between privilege levels is both the attack surface and the barrier to effective testing_. Knowing the interface is the key. Each paper solves this problem at a different layer and with different constraints, each building on the previous one's shortcomings.

A concrete running example can thread through the whole document: _imagine you want to fuzz an Android camera driver._ At the kernel layer, DIFUZE recovers the `ioctl` interface. At the service layer, FANS handles the `cameraserver` Binder interface — but only if you have source code. At the HAL layer, NASS handles the proprietary camera HAL service that neither prior tool can touch.

### Table of Contents

**1. Introduction**

- The security of mobile devices — why it matters deeply
- Android's vulnerability landscape: the shift from userspace to kernel/driver bugs (4% in 2014 → 39% in 2016), and the emerging shift to system services
- Why fuzzing is the right tool, and why it fails naively at structured interfaces
- Overview of the three papers and the document's structure

**2. Background**

- 2.1 Fuzzing: concepts, mutation vs. generation, coverage-guided fuzzing
- 2.2 Android's architecture: the Linux kernel, monolithic design, kernel modules and device drivers
- 2.3 The `ioctl` interface: how it works, why it's powerful, why it's dangerous (arbitrary structures, no type safety, complex nesting, pointers)
- 2.4 Android system services: Java vs. native, the framework/HAL layering (Project Treble, Android 8+), the Binder IPC mechanism, serialization via Parcel, the `onTransact` dispatcher pattern
- 2.5 The interface problem: why interface-unawareness causes fuzzing to waste almost all its cycles on rejected inputs

**3. DIFUZE: Interface-Aware Fuzzing for Kernel Drivers**

- 3.1 The ioctl challenge in depth — with the running example from the paper (DriverStructOne, DriverStructTwo, the switch/case handler)
- 3.2 System architecture and pipeline overview
- 3.3 Build system instrumentation: GCC → LLVM bitcode, consolidation per driver
- 3.4 ioctl handler identification: scanning for known `file_operations` structure assignments
- 3.5 Device file detection: tracing from ioctl handler through registration functions
- 3.6 Command value determination: path-sensitive equality constraint analysis and Range Analysis
- 3.7 Argument type identification: path-sensitive type propagation through `copy_from_user`, handling wrappers and casts (with Table 1 from the paper explained in depth)
- 3.8 Structure definition recovery: GCC → c2xml pipeline
- 3.9 Structure generation: type-specific value creation (power-of-two heuristics), sub-structure generation
- 3.10 On-device execution: pointer fixup, execution, heartbeat monitoring, crash logging and replay
- 3.11 Evaluation: results on 7 phones, the qseecom bug case study (why full structure instantiation was required), the Honor 8 design bug case study
- 3.12 Limitations and what comes next

**4. FANS: Extending Interface-Awareness to Android System Services**

- 4.1 Why ioctl-level fuzzing isn't enough: the Binder attack surface
- 4.2 The multi-level interface problem: why 37% of the attack surface is invisible without dependency traversal
- 4.3 Design choices: RPC-centric testing, generation-based fuzzing, learning from source code AST
- 4.4 Interface collection: scanning compilation commands, AIDL tool handling
- 4.5 Interface model extraction from the AST: the four variable classes (sequential, conditional, loop, return), the seven sequential statement kinds, type definition extraction, type aliases
- 4.6 Dependency inference: interface dependency (generation/use relationships via `writeStrongBinder`/`readStrongBinder`), intra-transaction dependency (conditional, loop, array size), inter-transaction dependency (Algorithm 1 explained and analyzed)
- 4.7 The fuzzer engine: transaction generation order (constraint first → dependency second → type and name third), interface acquisition
- 4.8 Evaluation: interface statistics (top-level vs. multi-level, AIDL-generated), the interface dependency graph, the three case studies (IDrm overflow, statsd OOB, ip6tables-restore stack overflow via `netd`)
- 4.9 Limitations: source code requirement, no coverage, incomplete state machine modeling

**5. NASS: Fuzzing Proprietary Native Android System Services**

- 5.1 The proprietary service problem: 316 out of 528 native services on real devices are closed-source; this is where the kernel attack surface lives
- 5.2 The threat model: an attacker in the app sandbox escalating via a HAL service (the cameraserver 0-day in the wild as motivation)
- 5.3 RPC design principles: Ab, Si, St — why they exist, why they hold, why they enable systematic analysis. Verification that 89% of real COTS services comply.
- 5.4 DGIE: Deserialization-Guided Interface Extraction
    - Intuition: the server stub _enforces_ the interface definition through its deserialization calls — we can observe those calls dynamically
    - Phase 1: Coverage-based RPC function discovery (iterating identifiers, watching for new coverage)
    - Phase 2: Iterative refinement — sending partially-correct inputs, observing which deserializers are reached next, building up the signature incrementally
    - The Parcelable unrolling insight: high-level objects flatten to linear sequences of standard deserializers
    - Handling Ab violations (application logic in server stub): the preliminary fuzzing phase explores these probabilistically
    - Results: 88% correct vs. 53% for message capture
- 5.5 Coverage collection: entry-point hooking, PID-based request isolation, Frida Stalker thread tracing
- 5.6 Interface-aware fuzzing: the 23 supported argument types, type-specific mutators, the special handling of file descriptors and Binder references, Parcelable-vector mutations
- 5.7 Implementation: FRIDA-based DBI, on-device fuzzing in-situ, crash detection via PID monitoring, automatic service restart
- 5.8 Evaluation: the ground truth study (Table 3 analyzed), coverage comparison with FANS and NASS(NI) (Figure 3 discussed per service), the real-world COTS results, the three case studies (use-after-free on Pixel 9, arbitrary write on OnePlus, heap overflow on Samsung)
- 5.9 Limitations: nested interface handles, cross-thread async processing, DBI overhead, sanitizer challenges

**6. Synthesis: The Evolution of Interface-Aware Fuzzing**

- 6.1 The common insight across all three systems: interface knowledge is the multiplier
- 6.2 Static vs. dynamic analysis for interface recovery — trade-offs (precision, automation, source code dependency)
- 6.3 Black-box vs. grey-box: what coverage buys you even when the interface is known
- 6.4 The attack surface in motion: from kernel drivers → open-source services → proprietary services → (future: TEEs, secure monitors, other RPC frameworks)
- 6.5 Open problems: nested proprietary interfaces, cross-thread fuzzing, sanitizer-less heap corruption detection

**7. Conclusion**

---

## Part 3: Presentation Game Plan

### Main Narrative for the Presentation

The presentation should feel like a _guided tour of a research journey_, not a paper summary. The audience should leave understanding: (1) why naïve fuzzing fails on structured interfaces, (2) how the field solved this problem in three progressively more powerful steps, and (3) what remains open.

### Proposed Presentation Table of Contents (45 min)

|Segment|Duration|Content|
|---|---|---|
|Hook + Motivation|4 min|Mobile phones as the most attacked devices; the escalating share of kernel/service bugs; what fuzzing is and why it matters|
|The Interface Problem|5 min|A single concrete example: you want to fuzz a camera driver. Show what a random-input fuzzer hits at the `switch(cmd)` wall. Introduce the core insight visually.|
|Background|4 min|ioctl mechanics (visual), Binder IPC architecture (layer diagram), what a transaction looks like|
|DIFUZE|9 min|Architecture diagram → walk through the pipeline stage by stage with visuals → the running example → one case study (qseecom) → key result: +54.5% bugs from structure info → limitation|
|FANS|8 min|The new attack surface (Binder) → multi-level interface graph visual → AST extraction concept → dependency graph → key case study → key result → limitation (source code, no coverage)|
|NASS|10 min|The proprietary blind spot (pie chart: 60% proprietary) → the 3 RPC principles → DGIE walkthrough (animated: observe deserializers iteratively) → coverage collection visual → key result → CVEs found on real phones|
|Comparison + Takeaways|4 min|Side-by-side comparison table; the single unifying insight; open problems|
|Conclusion|1 min|Summary|

### Key Slide Principles

- **Architecture diagrams** for every system (not bullet points)
- **Animated DGIE walkthrough**: show the iterative probing visually (send bytes → observe readString[] → add to interface → send again → observe readInt → etc.)
- **Layer diagram** showing how DIFUZE, FANS, and NASS each operate at different layers of the Android stack — this is a powerful single image
- **Coverage comparison graphs** from NASS paper (Figure 3) discussed briefly
- **No code on slides** unless it's the running example with 2-3 highlighted lines max
- **Code examples** (like Listing 3 from DIFUZE) should be shown as a visual with annotated call-flow arrows, not raw text

### What the Presentation Skips (Relative to the Document)

- Full details of command value determination (Range Analysis internals)
- Full Algorithm 1 from FANS (inter-transaction dependency inference)
- V4L2 driver handling (Appendix B of DIFUZE)
- FANS implementation details (ASan build, logcat)
- NASS Appendix (gRPC/Thrift discussion)
- Most evaluation tables (summarize in one comparison slide)

---

## Part 4: Task Plan and Timeline

### All Tasks, In Order

|#|Task|Depends On|Estimated Time|
|---|---|---|---|
|1|Deep re-read of DIFUZE with structured notes|—|3–4 hours|
|2|Deep re-read of FANS with structured notes|1|3–4 hours|
|3|Deep re-read of NASS with structured notes|1, 2|4–5 hours|
|4|Write Background chapter (§2)|1–3|4–5 hours|
|5|Write DIFUZE chapter (§3)|4|6–8 hours|
|6|Write FANS chapter (§4)|4, 5|6–8 hours|
|7|Write NASS chapter (§5)|4, 6|8–10 hours|
|8|Write Introduction (§1)|5–7|2–3 hours|
|9|Write Synthesis and Conclusion (§6, §7)|5–7|3–4 hours|
|10|Full document review and edit pass|5–9|3–4 hours|
|11|Design presentation structure + storyboard|10|1–2 hours|
|12|Create all slides|11|8–12 hours|
|13|Practice first presentation (first half material)|12|3–4 hours|
|14|First Zoom meeting with mentor|13|—|
|15|Revise based on mentor feedback|14|2–4 hours|
|16|Practice second presentation (second half)|15|3–4 hours|
|17|Second Zoom meeting with mentor|16|—|
|18|Full document finalization|15, 17|2–3 hours|
|19|Full presentation rehearsal|15–18|4–5 hours|
|20|Third Zoom meeting (rehearsal)|19|—|
|21|Final polish and exam presentation|20|—|

**Total estimated working time before first meeting:** ~50–65 hours of focused work

### Recommended Order of Writing (Within the Document)

Write **DIFUZE first**, even though it's not the main paper — it's the most concrete and self-contained, and it establishes the vocabulary (interface recovery, structure generation, on-device execution) that FANS and NASS both inherit. Then FANS, then NASS. Write the Introduction **last**, after you fully understand where the narrative is going.

### A Note on First Meeting Preparation

For the **first Zoom meeting**, per the mentor's instructions you need to present the **first half of the material**. A natural split is: Background + DIFUZE + the first half of FANS (through dependency inference). This is a good split because DIFUZE is a clean self-contained story and provides excellent foundation for discussing FANS.

---

## Summary of the Core Narrative (One Paragraph)

The central argument of your seminar is this: **modern mobile devices expose multiple privilege-separated layers — kernel drivers, open-source system services, and proprietary HAL services — each of which communicates through a structured interface. Naïve fuzzing fails at structured interfaces because inputs that don't conform to the expected format are silently rejected before reaching any interesting logic. DIFUZE (2017) proved that recovering and respecting the interface transforms fuzzing effectiveness for kernel drivers. FANS (2020) extended this insight to Android's Binder IPC layer, adding the crucial dimensions of multi-level interface discovery, AST-based semantic extraction, and dependency modeling. NASS (2025) completed the picture by eliminating the source-code requirement through dynamic deserialization-guided interface extraction, adding coverage-guided feedback, and targeting the 60% of native services that were entirely invisible to prior work.** The throughline is that the interface is simultaneously the barrier to effective testing and the map to the attack surface — whoever can read it, wins.