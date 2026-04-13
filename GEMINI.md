# Seminar: Interface-Aware Fuzzing on Android

This workspace is dedicated to an academic seminar for a Bachelor's degree, focused on researching and synthesizing advanced fuzzing techniques for the Android operating system.

## Project Overview

The seminar follows the evolution of "interface-aware" fuzzing through three seminal research papers. The core objective is to produce a high-quality written thesis (several dozen pages) and a 45-minute oral presentation demonstrating deep conceptual understanding and original analysis.

### The Research Arc
1.  **DIFUZE (2017):** Focuses on the kernel layer. It uses static analysis (LLVM) to recover `ioctl` interfaces for device drivers, enabling structured input generation where random bytes fail.
2.  **FANS (2020):** Moves up to Android system services. It extracts interface models from ASTs to handle Binder IPC, including complex multi-level interfaces and transaction dependencies.
3.  **NASS (2025):** Addresses the "proprietary blind spot." It uses Deserialization-Guided Interface Extraction (DGIE) to fuzz closed-source HAL services without source code, adding coverage-guided feedback.

## Directory Structure

- **`thesis/`**: Contains the draft chapters of the written thesis in Markdown format.
    - `TOC.md`: The master Table of Contents.
    - `01_introduction.md` through `07_conclusion.md`: Individual chapters tracing the chronological and technical arc.
- **`game_plan/`**: Strategy documents and task lists.
    - `plan.md`: The primary strategy document containing deep summaries, the narrative arc, and the thesis/presentation outlines.
    - `presentation_plan.md`: Specific strategy for the 45-minute lecture and slide design.
    - `expansion_plan.md` & `thesis_finalization_plan.md`: Detailed task lists for project completion.
- **`papers/`**: Source materials.
    - Contains the original PDFs and Markdown summaries (`DIFUZE.md`, `FANS.md`) of the research papers.
- **`README.md`**: Contains the mentor's specific instructions, grading criteria, and the official project objectives.

## Usage and Context

### Research & Analysis
When assisting with research, refer to `game_plan/plan.md` for high-signal summaries of the core papers. Ensure all technical explanations align with the "chronological arc" (DIFUZE → FANS → NASS).

### Thesis Writing
Adhere to the structure defined in `thesis/TOC.md`. Maintain a professional, academic tone that prioritizes conceptual depth and original examples over "dry" summaries, as mandated by the mentor in `README.md`.

### Presentation Design
Follow the "Visuals-First, Text-Minimal" guidelines in `README.md`. Focus on architecture diagrams, animated walkthroughs of algorithms (like DGIE), and layer diagrams showing the Android attack surface.

## Key Technical Concepts
- **Interface-Awareness:** The ability of a fuzzer to understand and conform to the expected input structure of a target interface (ioctl, Binder, etc.).
- **Dependency Modeling:** Inferring how different IPC transactions or arguments rely on each other (FANS).
- **DGIE (Deserialization-Guided Interface Extraction):** Dynamically probing a service to learn its interface by observing which deserialization routines it calls (NASS).
- **Attack Surface Shift:** The migration of vulnerabilities from the Linux kernel to system services and proprietary HALs.
