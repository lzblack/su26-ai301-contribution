# Contribution 1: PDF download in the presence of CSS with background colour

**Contribution Number:** 1
**Student:** Zhi Li
**Issue:** <https://github.com/marimo-team/marimo/issues/5832>
**Status:** Phase I Complete
---

## Why I Chose This Issue

When a marimo notebook uses a custom CSS theme with a background colour and is exported to PDF, the page margins and the full-page background conflict: with the default print margins the themed background leaves a white border around the page, and with zero margins the background fills the page but the text sits right against the edges. This matters because it makes themed notebooks look broken when shared as PDFs, and a clean fix would help both the in-app "Download as PDF" and the CLI export added in #7997.

I chose this issue because it is well-scoped, the maintainers marked it `help wanted` and offered to help, and the original reporter has stepped back, so it is genuinely open. I reproduced both failure modes on the current version (marimo 0.23.9) with the `wigwam` theme, so I already have a clear picture of what "fixed" looks like: move the spacing from the paper-margin layer to the content layer, so the background reaches the paper edge while the text keeps comfortable padding (ideally scaling with the width config). I have prior experience with HTML-to-PDF and print styling, and want to deepen my understanding of print CSS and marimo's export pipeline.

---

## Understanding the Issue

### Problem Description

Exporting a themed marimo notebook (custom CSS with a background colour) to PDF produces poor output because the page margins conflict with the full-page background.

### Expected Behavior

The background colour fills the whole page edge to edge, and the text keeps comfortable spacing from the paper edges.

### Current Behavior

- Default print margins: the background only fills the content area, leaving a white border around the page.
- Zero margins: the background reaches the paper edge, but title/body/dividers sit right against the edges with no breathing room.

### Affected Components

The shared print CSS (`@media print` / `@page`) used by the in-app "Download as PDF" (browser print) and the CLI PDF export added in #7997 (Chromium with `print_background` + `prefer_css_page_size`).

---

### Steps to Reproduce

1. Create a marimo notebook and apply a themed CSS with a background colour (e.g. `wigwam`).
2. Export to HTML, then print to PDF.
3. Default margins: a white border surrounds the themed background.
4. Zero margins: the background fills the page but the text touches the edges.

### Reproduction Evidence

- **Environment:** marimo 0.23.9
- **Artifacts:** `nb.py` (notebook), `wigwam.css` (theme), `nb.html` (export), `print_pdf.py` (print script), 2 PDFs (default vs zero margins), 2 screenshots.
- **My findings:** the premise holds on the current version; the spacing needs to move from the page-margin layer to the content layer.

---

## Solution Approach

### Analysis

[Your analysis of the root cause - what's causing the issue?]

### Proposed Solution

[High-level description of your fix approach]

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** [Restate the problem]

**Match:** [What similar patterns/solutions exist in the codebase?]

**Plan:** [Step-by-step implementation plan]

1. [Modify file X to do Y]
2. [Add function Z]
3. [Update tests]

**Implement:** [Link to your branch/commits as you work]

**Review:** [Self-review checklist - does it follow the project's contribution guidelines?]

**Evaluate:** [How will you verify it works?]

---

## Testing Strategy

### Unit Tests

- [ ] Test case 1: [Description]
- [ ] Test case 2: [Description]
- [ ] Test case 3: [Description]

### Integration Tests

- [ ] Integration scenario 1
- [ ] Integration scenario 2

### Manual Testing

[What you tested manually and results]

---

## Implementation Notes

### Week [X] Progress

[What you built this week, challenges faced, decisions made]

### Week [Y] Progress

[Continue documenting as you work]

### Code Changes

- **Files modified:** [List]
- **Key commits:** [Links to important commits]
- **Approach decisions:** [Why you chose certain approaches]

---

## Pull Request

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description - much of the content above can be adapted]

**Maintainer Feedback:**

- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

---

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically]

### Challenges Overcome

[What was hard and how you solved it]

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]
