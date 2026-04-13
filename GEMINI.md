# Seminar: Interface-Aware Fuzzing on Android

This project is an academic seminar for a Bachelor's degree, focused on the research and explanation of advanced fuzzing techniques for the Android operating system.

## Project Overview

The seminar centers on a deep dive into three seminal research papers that trace the evolution of interface-aware fuzzing:
1.  **DIFUZE (2017):** Focuses on recovering `ioctl` interfaces for kernel drivers using static analysis.
2.  **FANS (2020):** Extends interface-awareness to Android system services (Binder IPC) using AST-based extraction and dependency modeling.
3.  **NASS (2025):** The core paper, which addresses proprietary/closed-source services through dynamic deserialization-guided interface extraction (DGIE) and grey-box fuzzing.

The ultimate goal is to produce a comprehensive written thesis and a 45-minute oral presentation that demonstrates a deep, original understanding of these concepts and their relationship to the Android attack surface.

## Directory Structure

- **`README.md`**: Provides the official seminar objectives, mentor's specific instructions for the thesis and presentation, and the meeting/exam schedule.
- **`game_plan/plan.md`**: A high-signal document containing:
    - Detailed summaries and key takeaways for DIFUZE, FANS, and NASS.
    - A proposed table of contents and narrative arc for the written thesis.
    - A storyboard and segment-by-segment breakdown for the presentation.
    - A phased task list and timeline for completion.
- **`papers/`**: Contains the source PDF files for the three research papers.

## Usage and Context

This workspace is used for:
- **Research & Analysis**: Synthesizing information from the provided PDFs.
- **Writing**: Drafting the written thesis according to the structure in `game_plan/plan.md`.
- **Presentation Design**: Creating a slide deck that follows the "Visuals-First, Text-Minimal" guidelines in the `README.md`.

When assisting with this project, prioritize maintaining the "chronological arc" identified in the game plan: **DIFUZE → FANS → NASS**. This narrative highlights the progression from kernel drivers to open-source services, and finally to proprietary HAL services.
