[![build-release](https://github.com/psylector/openhtmltopdf/workflows/release/badge.svg)](https://github.com/psylector/openhtmltopdf/actions?query=workflow%3Arelease)

# OPEN HTML TO PDF (Psylector Fork)

## WHY THIS FORK?

This is a fork of [openhtmltopdf/openhtmltopdf](https://github.com/openhtmltopdf/openhtmltopdf) maintained by [Psylector](https://github.com/psylector) to fix PDF/UA-1 (accessible PDF) issues that are not yet addressed upstream.

Changes in this fork focus on improving PDF/UA compliance and accessibility support.

## CONSUMING FROM GITHUB PACKAGES

Add the GitHub Packages repository and dependency to your `pom.xml`:

```xml
<repositories>
  <repository>
    <id>github</id>
    <url>https://maven.pkg.github.com/psylector/openhtmltopdf</url>
  </repository>
</repositories>

<dependency>
  <groupId>cz.psylector.openhtmltopdf</groupId>
  <artifactId>openhtmltopdf-pdfbox</artifactId>
  <version>LATEST_VERSION</version> <!-- See https://github.com/psylector/openhtmltopdf/packages -->
</dependency>
```

You also need to authenticate with GitHub Packages. Add this to your `~/.m2/settings.xml`:

```xml
<servers>
  <server>
    <id>github</id>
    <username>YOUR_GITHUB_USERNAME</username>
    <password>YOUR_GITHUB_TOKEN</password>
  </server>
</servers>
```

The token needs the `read:packages` scope.

## OVERVIEW

Open HTML to PDF is a pure-Java library for rendering a reasonable subset of well-formed XML/XHTML (and even some HTML5)
using CSS 2.1 (and later standards) for layout and formatting, outputting to PDF or images.

## UPSTREAM

+ Upstream repository: [openhtmltopdf/openhtmltopdf](https://github.com/openhtmltopdf/openhtmltopdf)
+ [1.0.10 Online Sandbox](https://sandbox.openhtmltopdf.com/)
+ [Documentation wiki](https://github.com/danfickle/openhtmltopdf/wiki)

## DIFFERENCES WITH FLYING SAUCER

+ Uses the well-maintained and open-source (LGPL compatible) PDFBOX as PDF library, rather than iText.
+ Proper support for generating accessible PDFs (Section 508, PDF/UA, WCAG 2.0).
+ Proper support for generating PDF/A standards compliant PDFs.
+ New, faster renderer means this project can be several times faster for very large documents.
+ Better support for CSS3 transforms.
+ Automatic visual regression testing of PDFs, with many end-to-end tests.
+ Ability to insert pages for cut-off content.
+ Built-in plugins for SVG and MathML.
+ Font fallback support.
+ Limited support for RTL and bi-directional documents.
+ On the negative side, no support for OpenType fonts.
+ Footnote support.

## LICENSE

Open HTML to PDF is distributed under the LGPL. Open HTML to PDF itself is licensed
under the GNU Lesser General Public License, version 2.1 or later, available at
https://www.gnu.org/copyleft/lesser.html. You can use Open HTML to PDF in any
way and for any purpose you want as long as you respect the terms of the
license. A copy of the LGPL license is included as license-lgpl-2.1.txt or license-lgpl-3.txt
in our distributions and in our source tree.

An exception to this is the pdf-a testing module, which is licensed under the GPL. This module is not published to GitHub Packages and is for testing only.

## KNOWN ISSUES (PDF/UA-1)

### CSS `float` breaks structure tree hierarchy

When `usePdfUaAccessibility(true)` is enabled, floated elements (`float: left`, `float: right`) are placed at the wrong level in the PDF structure tree. The CSS renderer paints floats in a separate layer, and the accessibility helper attaches them as siblings of the document root instead of keeping them inside their DOM parent.

**Example:** For this HTML:
```html
<body>
  <h1>Title</h1>
  <div style="float: left; width: 50%"><p>Left column</p></div>
  <div style="float: right; width: 50%"><p>Right column</p></div>
  <p>Footer</p>
</body>
```

The structure tree becomes:
```text
Document
├── Div (body) → H1, P(Footer)   ← normal flow only
├── Div → P(Left column)          ← float escaped from body
└── Div → P(Right column)         ← float escaped from body
```

This violates PDF/UA-1 rule 7.4.2-1 (logical reading order).

**Workaround:** Replace `float` with `display: table` / `table-cell` layout, which produces a correct structure tree:

```html
<div style="display: table; width: 100%">
  <div style="display: table-cell; width: 50%"><p>Left column</p></div>
  <div style="display: table-cell; width: 50%"><p>Right column</p></div>
</div>
```

This is an upstream issue inherited from [openhtmltopdf/openhtmltopdf](https://github.com/openhtmltopdf/openhtmltopdf).

## FAQ

+ OPEN HTML TO PDF is tested with Temurin 8, 11, 17 and 21. It requires at least Java 8 to run.
+ No, you can not use it on Android.
+ No, it's not a web browser. Specifically, it does not run javascript or implement many modern standards such as flex
  and grid layout.

## TEST CASES

Test cases, failing or working are welcome, please place them
in ````/openhtmltopdf-examples/src/main/resources/testcases/````
and run them
from ````/openhtmltopdf-examples/src/main/java/com/openhtmltopdf/testcases/TestcaseRunner.java````.
