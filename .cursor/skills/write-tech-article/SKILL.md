---
name: write-tech-article
description: Write a deep technical architecture analysis blog post for this Hugo site. Use when the user asks to write, draft, or create a new technical blog article, or when asked to analyze a framework, library, or system and produce a blog post about it.
---

# Write Technical Article

This skill encodes the workflow for producing a deep technical architecture analysis article for the blog. Style constraints are defined in the project's Rule file and apply automatically — this skill focuses on the **research → plan → write → verify** procedure.

## Prerequisites

- The user provides a technical analysis target: a framework, library, codebase, system design, documentation URL, or GitHub repository
- If the target is not yet provided, ask the user to specify it before proceeding

## Workflow

### Phase 1: Research

1. **Gather source material** using web fetch, web search, and code exploration:
   - Official documentation pages
   - GitHub repository structure (README, key source files, directory layout)
   - Core abstractions: data structures, protocols, base classes
   - Configuration and extension points
2. **Identify the core problem domain**: what problem does this system solve, and what are its design goals?
3. **Map the architecture layers**: from infrastructure to application, identify 4-6 distinct layers or modules
4. **Locate source code for key mechanisms**: find the specific files, classes, and functions that implement core behaviors — not just module names

### Phase 2: Plan

1. **Draft the overall architecture diagram** (ASCII art) showing the layered structure
2. **Outline 4-6 main sections**, each covering one architectural layer or design concern
3. **For each section, identify**:
   - The motivating problem or question
   - The design choice made and its rationale
   - Key source code or data structures to cite
   - Personal evaluation or comparison with alternatives
4. **Decide on a closing section**: usually "设计权衡与待改进之处" discussing trade-offs, not a generic summary

### Phase 3: Write

1. **Opening paragraph**: establish the target's background, ecosystem positioning, and core design intent. Include comparison with related systems if relevant
2. **Architecture overview**: present the overall layered diagram immediately after the opening
3. **Body sections**: write each section following the pattern:
   - Motivating problem → Design analysis → Source-level mechanism → Evaluation/extension
4. **Code and diagrams**: include source code snippets for key abstractions, and ASCII diagrams for data flows and component relationships
5. **Design trade-offs section**: discuss 3-5 concrete design decisions that involve meaningful trade-offs, with the author's own analysis
6. **Front matter**: generate Hugo front matter with title, current date (ISO 8601, +08:00), `draft: false`, and appropriate category
7. **File placement**: save to `content/posts/<category>/<slug>.md`

### Phase 4: Verify

Run through this checklist before presenting the article:

- [ ] Article opens with a contextual paragraph, not a bullet list or definition
- [ ] Overall architecture diagram appears within the first few paragraphs
- [ ] Sections use numbered headings (## 1, ### 1.1)
- [ ] Each core argument follows the 4-step pattern (problem → choice → mechanism → evaluation)
- [ ] Source code citations reference specific files/classes, not vague module names
- [ ] Prose-driven narrative, not bullet-list-heavy
- [ ] Bold (**) only used for first introduction of key terms
- [ ] No redundant "总结" or "结语" section at the end
- [ ] Article length matches complexity (typically 2000+ words for architecture analysis)
- [ ] Personal judgments are clearly framed as author's perspective
