# Thesis Evaluation and Improvement Plan

## 1. Findings: Current State of the Thesis

After thoroughly reading the provided papers (DIFUZE, FANS, NASS) and reviewing the current draft in the `thesis/` directory, I have identified several critical areas for improvement to meet the mentor's standards.

### 1.1 Content and Quality
*   **Superficial Depth:** The current draft reads like a high-level executive summary rather than an in-depth academic seminar thesis. The mentor explicitly requested "deep understanding" and a document spanning "several dozen pages." The current draft glosses over the actual technical mechanics (e.g., it mentions DIFUZE uses LLVM passes and FANS uses AST extraction, but it doesn't explain *how* they work algorithmically).
*   **Lack of Empirical Evidence:** Instead of using the rich, real-world case studies provided in the papers, the thesis relies repeatedly on a single, hypothetical "Android Camera Subsystem" example. This weakens the academic rigor and fails to demonstrate a deep engagement with the source material.
*   **Bibliography Errors:** The current `bibliography.md` contains hallucinated authors for both the FANS and NASS papers. This is a critical academic error.

### 1.2 "AI-Like" Tone and Style
The thesis heavily exhibits characteristics typical of AI-generated text, which detracts from its credibility as a human-written academic paper:
*   **Hyperbole and Fluff:** Overuse of dramatic adjectives and adverbs such as "massive," "incredibly," "monumental," "fundamentally," and "paradigm shift."
*   **Colloquialisms:** Unprofessional phrasing for a thesis, e.g., "shook up the landscape," "slapped a coverage tracker," "bounced at the deserialization barrier," and "pretty common."
*   **Mechanical Signposting:** Repetitive and robotic transitions, such as "In this chapter, I will examine...", "The rest of this thesis is structured as follows...", and "To really understand...".
*   **Generic Framing:** Using "Let's use a hypothetical..." instead of citing empirical data from the studies.

---

## 2. Detailed Game Plan for Improvement

To elevate the thesis to a high-quality, human-sounding academic document that fulfills the seminar's requirements, we will execute the following strategy.

### Step 1: Technical Expansion (Adding Depth)
We must expand each core chapter to include the deep technical mechanics from the papers, replacing superficial summaries with rigorous explanations.
*   **DIFUZE (Chapter 3):** 
    *   Detail the GCC-to-LLVM bitcode conversion process.
    *   Explain the path-sensitive Range Analysis used to recover `cmd` constraints.
    *   Walk through the *Pointer Fixup* mechanism explicitly.
    *   **Action:** Replace the hypothetical camera example with the real `qseecom` (CVE-2017-0612) and `nve` (Honor 8 design flaw) case studies.
*   **FANS (Chapter 4):**
    *   Explain the four variable classes extracted from the AST (sequential, conditional, loop, return).
    *   Detail the Dependency Inference logic (Algorithm 1), explaining how type and name similarity are used to build the inter-transaction dependency graph.
    *   **Action:** Incorporate the real case studies: the `IDrm` new_capacity overflow, the `statsd` OOB access, and the multi-process `ip6tables-restore` stack overflow via `netd`.
*   **NASS (Chapters 4/5):**
    *   Deep dive into the two phases of Deserialization-Guided Interface Extraction (DGIE): Preliminary Fuzzing (discovery) and Iterative Refinement.
    *   Explain how NASS dynamically "unrolls" `Parcelable` objects into linear sequences of standard deserializers.
    *   Detail the dynamic binary instrumentation (Frida Stalker) and the specific PID-based filtering logic used to isolate coverage in multi-threaded daemons.
    *   **Action:** Add the actual bugs found by NASS, such as the `ISap` Use-After-Free (CVE-2024-47040) on the Pixel 9 and the `vendor.samsung.hardware.radio.network` heap overflow.

### Step 2: "De-AI-ing" the Tone (Humanization)
We will systematically rewrite the text to sound like an authoritative, human academic.
*   **Eradicate Hyperbole:** Strip out words like "massive," "huge," "incredibly," and "monumental." Replace them with precise, measured terms (e.g., "significant," "substantial," "notable").
*   **Formalize the Language:** Remove all colloquialisms. For instance, instead of "slapped a coverage tracker," use "applied dynamic binary instrumentation."
*   **Smooth Transitions:** Remove robotic signposting at the start and end of chapters. The narrative should flow logically from one concept to the next without explicitly telling the reader what the chapter will do.
*   **Voice and Perspective:** Use the first person ("I") sparingly and only when expressing a specific analytical opinion or structural choice, rather than for mechanical narration.

### Step 3: Fixing the Academic Apparatus
*   **Bibliography Correction:** Completely rewrite `bibliography.md` to reflect the exact authors, titles, publication venues, and years for DIFUZE (Corina et al., CCS '17), FANS (Liu et al., USENIX Security '20), and NASS (Mao, Busch, and Payer, USENIX Security '25).
*   **Inline Citations:** Ensure all claims, statistics (e.g., the 316 proprietary services metric), and case studies are properly cited inline.

### Step 4: Presentation Alignment
*   As we expand the text, we will explicitly mark sections that are prime candidates for visual diagrams (e.g., the DGIE state machine, the Android privilege architecture, the FANS dependency graph). This aligns directly with the mentor's emphasis on "Visuals-First, Text-Minimal" slide design for the upcoming presentation phases.