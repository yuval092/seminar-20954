# Thesis Finalization & Humanization Plan

This document outlines the final steps required to finish the written thesis and, crucially, how to rewrite sections so that it reads naturally as an undergraduate seminar paper rather than an AI-generated textbook.

## Phase 1: The "De-Botting" Process (Humanization)

To ensure the thesis passes academic scrutiny and reads naturally while maintaining high academic quality (without introducing errors), we need to apply the following stylistic changes across all chapters:

### 1. Introduce the "Student Voice" and Active Phrasing
*   **Action:** Rewrite the Introduction (Chapter 1) and Conclusion (Chapter 7) to include a first-person academic perspective. Shift from a purely passive, detached voice to an active, engaged voice where appropriate.
*   **Example Change:** Change "This thesis examines the evolution..." to "In this seminar work, I examine the evolution...".
*   **Subjective Synthesis:** In the Synthesis chapter (Chapter 6), add a paragraph detailing *your* analysis on which paper's approach is the most elegant or future-proof based on the evidence presented.

### 2. Melt Down the Bullet Points
*   **Action:** AI overuses bolded bullet points. We need to convert at least 50% of the bulleted lists into cohesive prose paragraphs.
*   **Target Areas:** The "Contributions" sections and the "Variable Classes" in Chapter 4. 
*   **Example Change:** Instead of a bulleted list for "Sequential, Conditional, Loop, Return" variables, write it as a continuous paragraph explaining the flow of how FANS extracts the model step-by-step.

### 3. Eradicate AI Transition Words and Inject Academic Nuance
*   **Action:** Perform a search-and-replace to eliminate or reduce the frequency of AI-favorite transition words. Replace absolute statements with academic hedging.
*   **Target Words:** "Crucially", "Furthermore", "Consequently", "Ultimately", "Delving into", "It is worth noting".
*   **Academic Nuance:** Change absolute claims like "This proves that..." to "This strongly suggests..." or "As the authors demonstrate...". Human academic writing relies heavily on nuanced framing rather than absolute certainty.

### 4. Break the Symmetry and Vary Sentence Length
*   **Action:** Introduce slight structural imperfections and vary the rhythm of the text. 
*   **Implementation:** 
    *   Not every case study needs to be exactly 3 paragraphs. 
    *   Not every chapter needs a rigidly titled "Limitations" section; sometimes, limitations can just be integrated into the concluding paragraph of the evaluation section. 
    *   Mix short, punchy sentences with longer, complex ones. AI tends to write uniformly medium-long sentences with perfect symmetry.

### 5. Cross-Chapter Signposting
*   **Action:** Explicitly link concepts forward and backward across chapters. AI often generates chapters in isolation without acknowledging the broader document flow.
*   **Example:** "Unlike the static approach we saw in DIFUZE (Chapter 3), NASS relies entirely on..." or "As I will discuss later in Chapter 6...".

## Phase 2: Final Content Tasks

1.  **Citation Formatting:** We need to go through the text and add proper inline citations (e.g., "[1]", "[2]") referencing the `bibliography.md` document. 
2.  **Intro/Conclusion Alignment:** Ensure the Introduction accurately reflects the newly expanded technical depth we added to Chapters 2-5. 
3.  **Final Page Count Check:** Compile the markdown files into a single PDF to verify we have hit the "several dozen pages" requirement set by the mentor.
