# Contribution 1: PDF download in the presence of CSS with background colour

- **Contribution Number:** 1
- **Student:** Zhi Li
- **Issue:** <https://github.com/marimo-team/marimo/issues/5832>
- **Status:** Phase II Complete

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

`frontend/src/css/app/print.css` — the print CSS used by the in-app **"Download as PDF" (browser print)** path. This is the reporter's original target and the path this PR addresses.

> **Scope note (verified on current `main`):** the `marimo export pdf` CLI/UI path is **out of scope**. On current `main` it goes through nbconvert (PDFExporter/LaTeX, fallback WebPDFExporter) and does **not** use `print.css`. (`prefer_css_page_size` survives only in the reveal.js *slides* export path, not regular-notebook export — so an earlier assumption that one print-CSS fix would also cover the CLI export does not hold on current `main`.)

---

## Reproduction Process

### Environment Setup

Started on native Windows but hit the frontend-build wall: `make fe` runs `scripts/buildfrontend.sh`, which native PowerShell can't execute (marimo's CONTRIBUTING also recommends WSL for Windows). Migrated to **WSL2 (Ubuntu)** and re-cloned into the Linux filesystem (`~/repos/marimo`, *not* `/mnt/c`, to avoid slow IO / line-ending issues; new SSH key generated in WSL, remote switched to SSH).

Toolchain (all met marimo's requirements): `uv` 0.11.19, Node v24.16.0 (≥22), pnpm 10.28.2 (≥10), GNU Make 4.3, gcc 13.3.

Issues hit and resolved:

- **`ModuleNotFoundError: No module named 'msgspec'`** — editable install was missing a core dep. `uv run` auto-created/synced the project `.venv` (equivalently `uv pip install --group=dev -e .`), pulling in `msgspec` and the rest.
- **Fork behind upstream** — synced `main` to `marimo-team/marimo` and confirmed the working branch was on the latest base, so the issue is verified against current code, not a stale checkout.
- **`gio: ... Operation not supported`** — not an error; marimo tries to auto-open a browser, which WSL has none. Opened `http://localhost:2718` manually in the Windows browser (WSL2 forwards localhost).
Frontend was already built (`make fe` → "Nothing to be done"); backend ran via `uv run marimo edit --no-token`.

### Steps to Reproduce

The trigger condition is in the issue title: a custom CSS theme **with a background colour**. Reproduced with the `wigwam` theme on the repo's `kitchen_sink.py`:

1. Copy the demo notebook out of the repo (keeps the repo's git status clean):
   `cp marimo/_smoke_tests/slides_examples/kitchen_sink.py ~/repro/`
2. Download a background-colour theme into the same dir:
   `curl -sL https://raw.githubusercontent.com/Haleshot/marimo-themes/main/themes/wigwam/wigwam.css -o ~/repro/wigwam.css`
   (wigwam sets `--background: light-dark(#e6ffe9, #0b1e26)` — mint green.)
3. Attach the CSS: `app = marimo.App(width="medium", css_file="wigwam.css")`, or via **Notebook Settings → Custom Files → Custom CSS = `wigwam.css`**.
4. `uv run marimo edit ~/repro/kitchen_sink.py --no-token`, open `http://localhost:2718`. Background turns mint green — confirms the CSS applied.
5. Browser print to PDF (`Ctrl+P` → *Save as PDF*).
6. **Margins = Default** → theme background leaves a white border around the page (matches the reporter's `margins.pdf`).
7. **Margins = None** → background fills the page, but text sits against the paper edge (matches `no-margins.pdf`).
**Expected:** background fills the page *and* the content keeps a comfortable inset.
**Actual:** only one or the other is achievable — no middle state. Both modes reproduce consistently and match the PDFs attached to the issue.

### Reproduction Evidence

- **Branch:** <https://github.com/lzblack/marimo/tree/fix-issue-5832>
- **Version:** current `main` (synced from upstream), originally on 0.23.9.
- **Artifacts:** `wigwam.css` (theme), `kitchen_sink.py` (test notebook), 2 screenshots (Default-margins white border; None-margins text-against-edge) matching the issue's `margins.pdf` / `no-margins.pdf`.
- **Finding:** the premise holds on current `main`; `print.css` has no `@page` rule (margin delegated to the browser) and zeroes `.output-area` padding in print — the two halves of the problem.

---

## Solution Approach

### Analysis

Root cause is in `frontend/src/css/app/print.css`: in print, marimo neither sets a `@page` margin (so the browser controls it → white border on Default margins) nor gives the content any inset (`.output-area` padding is zeroed → text against the edge on None margins). Background printing is already handled (`print-color-adjust: exact` is present). So the "spacing" is on the wrong layer — it should move from the page to the content.

### Proposed Solution

In the print CSS: set `@page { margin: 0 }` so the background reaches the paper edge, and give `.output-area` a real print padding so text keeps an inset — with that padding scaling by width config. Keep the existing `print-color-adjust: exact`.

### Implementation Plan (UMPIRE)

**Understand:** A themed notebook (background-colour CSS) can't export to a good PDF — default margins give a white border, zero margins give edge-to-edge text, no middle state. Desired: background fills the page, content keeps padding that scales with the width config (`compact`/`medium`/`full`). Enhancement to the in-app "Download as PDF" (browser-print) path.

**Match:** `frontend/src/css/app/print.css` already has an `@media print` block. It implements `print-color-adjust: exact` (background printing — keep), has **no `@page` rule**, and explicitly zeroes `.output-area` padding. The fix extends this existing pattern.

**Plan:**

1. Add `@page { margin: 0 }` (eliminate the white border).
2. Replace `.output-area`'s zeroed padding with a meaningful print padding (eliminate edge-to-edge text).
3. Scale that padding with the width config.
4. Leave `print_background` (`print-color-adjust: exact`) as-is.
   - **File(s):** `frontend/src/css/app/print.css` (possibly the `@media print` block in `md.css`).
**Implement:** (Phase III) Branch: <https://github.com/lzblack/marimo/tree/fix-issue-5832>

**Review:** Self-review against `CONTRIBUTING.md` — `make fe-check`/`make check` (lint/typecheck/format) must pass; tests expected for code changes (see Testing Strategy); CLA signed on the PR; follow commit/PR conventions. Maintainer approval already obtained (Myles welcomed it + `help wanted`; Light2Dark reviewed the approach, "sounds reasonable", suggested validating on `kitchen_sink.py`).

**Evaluate:** Re-run the reproduction; confirm background fills the page *and* text keeps an inset across `compact`/`medium`/`full` and against Default/None browser margins; visually compare to the issue's reference PDFs.

#### Open items to verify in Phase III

1. **In-app "Download as PDF" trigger.** If it opens the browser print *dialog* (user picks margins), the dialog's choice overrides `@page margin` — the reliable fix is the content padding; the white-border half may be limited. If it prints programmatically (honours `@page`), `@page margin: 0` also takes effect. Verify with the actual in-app button after installing `nbconvert[webpdf]`.
2. **Width-config selector.** Locate how `compact`/`medium`/`full` map to the DOM:
   `grep -rniE "data-width|width-(compact|medium|full)|appWidth" frontend/src | grep -v node_modules`
3. **Export (nbconvert) path** — confirmed out of scope.

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
