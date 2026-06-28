# Contribution 1: PDF download in the presence of CSS with background colour

- **Contribution Number:** 1
- **Student:** Zhi Li
- **Issue:** <https://github.com/marimo-team/marimo/issues/5832>
- **Status:** Phase IV Complete

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

- **Branch:** <https://github.com/lzblack/marimo/tree/fix-issue-5832> (implementation now lives on the clean branch fix-5832-clean / [PR #9971](https://github.com/marimo-team/marimo/pull/9971))
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

#### Open items — resolved in Phase III

1. **In-app "Download as PDF" trigger — RESOLVED.** The current in-app "Download as PDF" is the *server-side* playwright path (#8121, Jan–Feb 2026), which renders headless and does **not** load `print.css`, so no nbconvert/playwright install was needed to validate this fix. When #5832 was filed (Jul 2025) that in-app button was the *browser-print* path (`print.css`) — the path this PR fixes. Verified via browser print (`Ctrl+P` → Save as PDF): `@page { margin: 0 }` takes effect as the dialog's default, so the white-border half is addressed at the default level (a user can still override margins manually, which is expected).
2. **Width-config selector — RESOLVED.** Width maps to `#App[data-config-width="…"]` (values: normal/compact/medium/full/columns; `config-width-full` class also exists). Hook available for per-width scaling, but deferred — see PR open question.
3. **Export (nbconvert) path** — confirmed out of scope.

---

### Automated Tests

Print-CSS layout (page margins, full-bleed background, content inset) isn't covered by an obvious existing unit/e2e pattern in the repo, so no automated test was added. I asked the maintainer in the PR whether an e2e / PDF-snapshot test is expected for a CSS-only print change. The existing suite is unaffected (no JS/TS logic changed). `pnpm exec stylelint` on the changed file reports only the two pre-existing violations already present on `main` — zero new violations.

### Manual Testing

- **Setup:** wigwam theme via `App(css_file="wigwam.css")` on `kitchen_sink.py`; `uv run marimo edit`, opened in the Windows browser (WSL2 localhost forwarding).
- **Method:** browser print — `Ctrl+P` → *Save as PDF*, with "Background graphics" on.
- **Result (Margins = Default):** themed background now fills the full page (no white border) **and** content keeps a ~2rem inset on all four sides — the middle ground the issue asks for. Screenshot attached in the PR.
- **Before-state:** matches the issue's own `margins.pdf` (white border) and `no-margins.pdf` (edge-to-edge text).

---

### Week 3 Progress (Phase III — Build)

**What I built:** a single-file change in `frontend/src/css/app/print.css`, scoped to the browser-print path:

- `@page { margin: 0 }` inside `@media print` — removes the page-box margin so the themed background reaches the paper edge.
- A print-only `padding: 2rem !important` merged into the existing `#App` rule. `#App` carries the theme background (`bg-background` → `var(--background)`) and is `w-full h-full`, so the padding sits *inside* the background box: background fills the page edge-to-edge while content is inset on all four sides.

**Key decisions:**

- **Padded `#App`, not `.output-area`.** `#App` holds the theme background and fills the page, so one rule gives a uniform four-side inset with full-bleed background — cleaner than restoring `.output-area` L/R padding and separately handling top/bottom.
- **Left `body { background-color: #fff }` untouched** — changing it would regress un-themed/default-white notebooks.
- **Did not build width-config scaling.** The DOM hook exists (`#App[data-config-width="compact|medium|full"]`), but a single inset already solves the issue; per-width insets are raised as an open question in the PR rather than implemented speculatively. (Keeps the diff minimal and mergeable.)

**Challenges faced:**

- **Two distinct "PDF export" paths.** Verified via issue date + PR history that the in-app "Download as PDF" was browser-print-based when #5832 was filed (Jul 2025; dropdown added #1919), then moved to a server-side playwright path (#8036/#8121, Jan–Feb 2026) that doesn't load `print.css`. So this fix targets the path the reporter actually hit; documented the scope boundary in the PR. (Avoided installing nbconvert/playwright to chase the server path — out of scope.)
- **Minimal stylelint diff.** `--fix` rewrote a pre-existing longhand `overflow` unrelated to my change; a second `#App` block tripped `no-duplicate-selectors`. Resolved by merging `padding` into the existing `#App` block, adding the required blank line before the comment, and verifying without `--fix`.
- **Tangled branch history.** The original branch was based on a stale checkout; rather than rebase through unrelated upstream doc changes, rebuilt a clean branch (`fix-5832-clean`) off latest `upstream/main` with only the 8-line change.

### Code Changes

- **Files modified:** `frontend/src/css/app/print.css` (2 insertions, 8 lines)
- **Key commit:** `4f92e6c2` — fix(frontend): remove print page margin and inset content for themed PDF export
- **Branch:** <https://github.com/lzblack/marimo/tree/fix-5832-clean>
- **Approach decisions:** see "Key decisions" above.

---

## Pull Request

**PR Link:** <https://github.com/marimo-team/marimo/pull/9971>

**PR Description:**
What does this PR do?: Fixes themed-PDF export in marimo's browser print path. Inside `@media print` in `frontend/src/css/app/print.css`, it adds `@page { margin: 0 }` and sets `padding: 2rem !important` on `#App`, so notebooks using a CSS theme with a background colour render full-bleed when printed (Ctrl+P) or exported via the in-app "Download as PDF", while keeping content inset by 2rem.

Why was this PR needed?: Issue #5832 reported that printing/exporting a themed notebook left an unwanted white page margin instead of the theme's background colour. The default browser `@page` margin was clipping the background; removing it and re-introducing the inset on `#App` restores the intended full-bleed look.

Scope: This targets only the browser-print path. The server-side Playwright rendering path (added early 2026) does not load `print.css` and was explicitly kept out of scope; related issues #8036 and #8121 are not addressed here.

What are the relevant issue numbers?: Closes #5832

Does this PR meet the acceptance criteria?:

- [x] Change validated against the reported case (themed notebook now full-bleed with 2rem inset)
- [x] No regression on the default/un-themed notebook (verified via Chrome Ctrl+P)
- [x] Follows project style (single-file, 8-line CSS change; stylelint clean)
- [x] No breaking changes

**Maintainer Feedback:**

- Jun 24: @Light2Dark confirmed the approach made sense and noted the Playwright export path uses a different flow and styling altogether — validating the scope boundary I'd drawn (browser-print only).
- Jun 25: @mscolnick approved and merged the PR. No change requests were raised.

**Status:** Merged (merged Jun 25, 2026 by mscolnick into `marimo-team/marimo:main`, commit `c2971bc2`)

## Learnings & Reflections

Biggest lesson: scoping is a contribution skill in its own right, not a step before the "real" work. The bug touched two superficially similar surfaces — the browser print path (`print.css`) and the newer server-side Playwright export — and the cheap, mergeable win was the former. I deliberately excluded the Playwright path (#8036, #8121) up front and stated that boundary explicitly in the PR. A maintainer then independently confirmed the boundary was correct before approving. Drawing the line early is what turned this into a single-commit, zero-rework merge rather than an open-ended refactor.

Hardest part: trusting a small diff. The final change is 8 lines in one file, which felt almost too small to be "the fix." Verifying it the right way — reproducing the themed case, confirming the full-bleed-plus-inset result, and separately confirming the default notebook didn't regress — was what gave me confidence that the smallness was correct, not incomplete.

What I'd do differently: nothing major on the technical path, but I'd convert from draft to ready-for-review sooner. The fix and its validation were done well before I marked it ready; the instinct to keep polishing added days without adding value.

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
