# Fork Changes — psylector/openhtmltopdf

This document summarizes all changes made in the [psylector/openhtmltopdf](https://github.com/psylector/openhtmltopdf) fork relative to the upstream [nicehash/openhtmltopdf](https://github.com/nicehash/openhtmltopdf) (commit `9dc941bc`).

These changes are intended for contribution back to the upstream project.

## Summary

The fork focuses on **PDF/UA-1 (ISO 14289-1) accessibility compliance**. Three bugs were fixed and validated with veraPDF and manual PDF structure tree inspection.

---

## Release 1.1.38 — Fork setup

**PR:** [#1](https://github.com/psylector/openhtmltopdf/pull/1)
**Commit:** `b3f2bb53`
**Ticket:** PSY-76

Build/CI-only changes to repackage the fork. **No code changes relevant for upstream.**

- Changed Maven groupId from `io.github.openhtmltopdf` to `cz.psylector.openhtmltopdf`
- Replaced Maven Central publishing with GitHub Packages
- Added JaCoCo code coverage reporting to CI
- Added CodeRabbit configuration

---

## Release 1.1.39 — Fix structure tree DOM ordering

**PR:** [#2](https://github.com/psylector/openhtmltopdf/pull/2)
**Commit:** `9450e513`
**Ticket:** PSY-77

### Problem

The PDF structure tree was built in CSS paint order (floats first, then normal flow) instead of DOM document order. This violates **PDF/UA-1 rule 7.4.2-1** which requires the structure tree to reflect the logical reading order of the document.

For example, a document with `<h1>`, `<h2>`, `<h3>` could produce a structure tree with H3 before H1 if CSS floats reordered the paint sequence.

### Fix

Added `sortChildrenByDomOrder()` in `PdfBoxAccessibilityHelper` that recursively reorders structure tree children using `Node.compareDocumentPosition()` before the tree is finalized. Only items with a DOM node are sorted; anonymous boxes and text runs stay in their original positions to preserve inline content ordering.

### Files changed (upstream-relevant)

- `openhtmltopdf-pdfbox/src/main/java/com/openhtmltopdf/pdfboxout/PdfBoxAccessibilityHelper.java`

### Tests added

- `NonVisualRegressionTest.testStructureTreeFollowsDomOrder` — verifies H1→H2→H3 ordering in structure tree

### Known limitation

CSS floats can escape from their DOM parent in the structure tree. Workaround: use `display: table-cell` instead of `float: left` for multi-column layouts. This is consistent with the [OpenHTMLtoPDF wiki recommendation](https://github.com/danfickle/openhtmltopdf/wiki/PDF-Accessibility-(PDF-UA,-WCAG,-Section-508)-Support).

---

## Release 1.1.40 (pending) — Fix links in running footers

**PR:** [#3](https://github.com/psylector/openhtmltopdf/pull/3)
**Commit:** `c836d8b3`
**Ticket:** PSY-78

### Problem

Links inside running headers/footers (`position: running()`) were painted entirely as pagination artifacts. Link annotations existed in the PDF but had no corresponding `/Link` structure elements, violating:

- **PDF/UA-1 rule 7.18.1-2** — Link annotations shall be nested within a `/Link` structure element
- **PDF/UA-1 rule 7.18.5-1** — `/Link` shall contain an OBJR (object reference to the annotation)
- **PDF/UA-1 rule 7.18.5-2** — `/Link` shall include tagged content identifying the link text

### Root cause

`SimplePainter` (used for running content) does not participate in the structure tree — it paints everything as artifacts. The `DisplayListPainter` (used for normal content) calls `startStructure()`/`endStructure()` but `SimplePainter` does not.

### Fix

Implemented a hybrid artifact/structure approach in `PdfBoxAccessibilityHelper`:

1. When painting running content and an anchor element is detected (via DOM ancestry walk), temporarily close the artifact BMC
2. Emit MCID-tagged content under a `/Link` structure element
3. Reopen the artifact BMC
4. A DOM-element-based cache ensures the `/Link` structure is reused when the same running element appears on multiple pages

Additional fixes in the same PR:

- **SimplePainter**: Added `startStructure(REPLACED)`/`endStructure()` calls for replaced elements (images), matching `DisplayListPainter` behavior
- **Image links**: Route REPLACED content through `FigureStructualElement` to preserve alt text and BBox
- **OBJR for image-only links**: Fall back to `_runningLinkCache` in `addLink()` when anchor box lacks accessibility object
- **CI**: Switched coverage reporting to JaCoCo `report-aggregate` for accurate cross-module data

### Files changed (upstream-relevant)

- `openhtmltopdf-pdfbox/src/main/java/com/openhtmltopdf/pdfboxout/PdfBoxAccessibilityHelper.java`
- `openhtmltopdf-core/src/main/java/com/openhtmltopdf/render/simplepainter/SimplePainter.java`

### Tests added

- `NonVisualRegressionTest.testRunningFooterLinksGetLinkStructureElements` — /Link with OBJR + MCID content
- `NonVisualRegressionTest.testRunningFooterLinkAnnotationPosition` — annotation rect in bottom margin
- `NonVisualRegressionTest.testRunningFooterLinkAcrossMultiplePages` — single /Link with 2 OBJRs across pages
- `NonVisualRegressionTest.testRunningFooterMultipleLinks` — 2 distinct /Link elements
- `NonVisualRegressionTest.testRunningFooterImageLinkGetsFigureStructure` — /Figure with alt text under /Link
- `NonVisualRegressionTest.testRunningFooterLinkMultipleSpansReusesStructure` — cache reuse verification

---

## Current branch (PSY-82) — veraPDF PDF/UA-1 validation + page-break heading test

**Branch:** `psylector/psy-82-add-verapdf-validation-and-page-break-structure-tree-tests`
**Ticket:** PSY-82

### Changes

#### 1. veraPDF PDF/UA-1 validation tests

New test class `PdfUaTester` in `openhtmltopdf-pdfa-testing` validates generated PDFs against the `PDFUA_1` veraPDF profile (already available in veraPDF 1.18.8). Two tests:

- `testAllInOnePdfUa1` — validates the existing comprehensive `all-in-one.html` test document
- `testStructureWithRunningFooterLinks` — validates a new HTML with heading hierarchy (H1→H2→H3) and running footer links

#### 2. Link annotation Contents key (PDF/UA-1 fix)

veraPDF validation revealed that link annotations were missing the `Contents` key required by PDF/UA-1 (ISO 32000-1:2008, §14.9.3). Added `setLinkContents()` in `PdfBoxFastLinkManager` that sets the alternate description from the `title` attribute (fallback: element text content).

#### 3. Page-break heading structure tree test

New test `testHeadingsAcrossPageBreaksPreserveOrder` in `NonVisualRegressionTest` verifies that headings split across page breaks preserve H1→H2→H3 reading order in the structure tree.

### Files changed (upstream-relevant)

- `openhtmltopdf-pdfbox/src/main/java/com/openhtmltopdf/pdfboxout/PdfBoxFastLinkManager.java` — `setLinkContents()` for PDF/UA-1 compliance

### Files changed (test-only)

- `openhtmltopdf-pdfa-testing/src/test/java/com/openhtmltopdf/pdfa/testing/PdfUaTester.java` (new)
- `openhtmltopdf-pdfa-testing/src/test/resources/html/pdfua-structure.html` (new)
- `openhtmltopdf-examples/src/test/java/com/openhtmltopdf/nonvisualregressiontests/NonVisualRegressionTest.java`

---

## PDF/UA-1 rules addressed

| Rule | Description | Fixed in |
|------|-------------|----------|
| 7.4.2-1 | Structure tree shall follow logical reading order | 1.1.39 (PSY-77) |
| 7.18.1-2 | Link annotations shall be nested in /Link structure elements | 1.1.40 (PSY-78) |
| 7.18.5-1 | /Link shall contain OBJR (annotation object reference) | 1.1.40 (PSY-78) |
| 7.18.5-2 | /Link shall include tagged content identifying the link | 1.1.40 (PSY-78) |
| §14.9.3 | Link annotations shall have alternate description (Contents key) | PSY-82 |

## Upstream PR scope

For the upstream contribution, the following changes are relevant (excluding fork-specific build/CI changes):

1. **`PdfBoxAccessibilityHelper.java`** — DOM order sorting + running footer link structure
2. **`SimplePainter.java`** — `startStructure(REPLACED)` for replaced elements
3. **`PdfBoxFastLinkManager.java`** — `setLinkContents()` for link annotation alt text
4. **All test files** — 9 new tests covering the above fixes
