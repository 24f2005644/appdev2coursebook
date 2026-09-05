# Topic 3: Markup Alternatives & Text-Based Representations

---

## 3.1 HTML in Modern Web Engineering

### What is HTML?

**HTML (HyperText Markup Language)** is the foundational **document markup language** of the web. It describes the **semantic structure** of content using a system of **tags** (elements) that browsers parse and render.

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8">
    <title>My Page</title>
  </head>
  <body>
    <h1>Welcome</h1>
    <p>This is a <strong>paragraph</strong> with <em>emphasis</em>.</p>
    <ul>
      <li>Item one</li>
      <li>Item two</li>
    </ul>
  </body>
</html>
```

### The HTML "Living Standard"

HTML is no longer a versioned specification (HTML4, XHTML, HTML5…). It is now maintained as a **"Living Standard"** by the **WHATWG (Web Hypertext Application Technology Working Group)**:

- Continuously updated — no more numbered versions
- Changes are incremental and backward-compatible
- Browser vendors (Google, Mozilla, Apple, Microsoft) collectively shape the standard
- You can read it live at: [html.spec.whatwg.org](https://html.spec.whatwg.org)

This means new elements, APIs, and behaviors are added continuously as the web evolves (e.g., `<dialog>`, `<details>`, `popover` attribute, etc.).

### Extensibility via Web Components

HTML's reach has been extended through **Web Components** — a suite of browser standards allowing developers to define their own custom HTML elements:

```html
<!-- Using a custom element defined by a developer -->
<user-avatar name="Jane Doe" src="/avatars/jane.jpg"></user-avatar>
<data-table sortable filterable src="/api/students"></data-table>
<modal-dialog title="Confirm Delete" id="confirm-modal"></modal-dialog>
```

Web Components consist of three browser APIs:

| API | Purpose |
|---|---|
| **Custom Elements** | Define new HTML tags with custom behavior (`customElements.define()`) |
| **Shadow DOM** | Encapsulate internal HTML/CSS so styles don't leak in or out |
| **HTML Templates** (`<template>`) | Define reusable, inert HTML fragments that are cloned at runtime |

**Benefit**: Web Components are framework-agnostic — a component built with vanilla Web Components works in React, Vue, Angular, or plain HTML pages. Libraries like Vue and Svelte compile down to Web Components.

### Separation of Concerns: Semantic HTML vs. CSS

A key principle of modern web development:

| Layer | Technology | Responsibility |
|---|---|---|
| **Structure / Semantics** | HTML | *What* the content is (heading, paragraph, list, navigation, article) |
| **Presentation / Style** | CSS | *How* the content looks (colors, fonts, layout, spacing) |
| **Behaviour / Logic** | JavaScript | *What* the content does (interactions, state, data fetching) |

**Semantic HTML** uses elements that describe meaning, not appearance:

```html
<!-- Non-semantic (wrong approach) -->
<div class="big-bold-text">Welcome to the Course</div>
<div class="clickable-thing" onclick="submit()">Submit</div>

<!-- Semantic (correct approach) -->
<h1>Welcome to the Course</h1>
<button type="submit">Submit</button>
```

Benefits of semantic HTML:
- **Accessibility**: Screen readers announce `<h1>` as a heading, `<button>` as interactive
- **SEO**: Search engines understand the document structure
- **Maintainability**: The HTML communicates intent, not styling
- **Styling flexibility**: Redesign by only changing CSS, not HTML

---

## 3.2 Inherent Limitations of HTML

Despite being universal, HTML has real shortcomings in several contexts.

### Inadequacy for Structured Data Interchange

HTML was designed to describe **documents for humans** — but modern systems often need to exchange **structured data between machines**.

```html
<!-- HTML: Designed for human reading, terrible for machine parsing -->
<table>
  <tr><th>Name</th><th>GPA</th></tr>
  <tr><td>Jane Doe</td><td>3.8</td></tr>
</table>
```

```json
// JSON: Designed for machine exchange, perfectly structured
{ "name": "Jane Doe", "gpa": 3.8 }
```

Parsing data out of HTML is called **web scraping** — it's fragile, error-prone, and breaks whenever the HTML layout changes. Structured formats like JSON and XML were created specifically because HTML is unsuitable for data exchange.

### Challenges in Immersive Environments

HTML/CSS is fundamentally a **2D, document-oriented** rendering model. It was not designed for:

- **3D environments**: VR/AR scenes with spatial positioning, lighting, physics
- **Game engines**: Real-time rendering, animation loops, asset management
- **Scientific visualization**: 3D molecular structures, data point clouds

Technologies created to fill this gap:
- **VRML (Virtual Reality Modeling Language)** — 1994, the first attempt at 3D on the web (largely abandoned)
- **X3D** — XML-based 3D graphics standard (successor to VRML)
- **WebXR** — Modern browser API for VR/AR headsets (Oculus, HoloLens)
- **WebGL / WebGPU** — Low-level graphics APIs (canvas-based, not HTML-based)
- **Three.js** — JavaScript library abstracting WebGL for 3D scenes

```html
<!-- HTML tries with Canvas, but it's an escape hatch from the DOM -->
<canvas id="scene" width="800" height="600"></canvas>
<script>
  const gl = canvas.getContext('webgl'); // Now you're in a completely different paradigm
</script>
```

### Verbosity and Author Overhead

HTML is verbose and repetitive. Writing long-form content in raw HTML is tedious:

```html
<!-- Writing a simple article in HTML -->
<h2>Introduction</h2>
<p>This is the first paragraph, with a <a href="https://example.com">link</a> and some <strong>bold text</strong>.</p>
<p>This is the second paragraph.</p>
<h3>Sub-section</h3>
<ul>
  <li>Item one</li>
  <li>Item two</li>
  <li>Item three</li>
</ul>
<blockquote>
  <p>A famous quote by someone important.</p>
</blockquote>
```

For content authors (technical writers, bloggers, documentation writers) who write thousands of words daily, this overhead is significant. This drove the creation of lightweight markup languages.

---

## 3.3 Text-Based Lightweight Markup Languages

### The Core Idea

Lightweight markup languages let you write **plain text** with **minimal, readable inline syntax** that can then be **compiled to HTML** (or other formats).

The philosophy: **the plain text source should be readable and look like the final output**, even before compilation.

### Markdown

Created by **John Gruber** in 2004. The most widely adopted lightweight markup language in the world.

**Basic Syntax:**

```markdown
# Heading 1
## Heading 2
### Heading 3

Regular paragraph text.

**Bold text** and *italic text* and ~~strikethrough~~

- Unordered list item
- Another item
  - Nested item

1. Ordered list item
2. Second item

[Link text](https://example.com)
![Alt text for image](image.jpg)

`inline code`

```python
# fenced code block with syntax highlighting
def hello():
    print("Hello, World!")
```

> Blockquote text

---   (horizontal rule)
```

**Why Markdown became dominant:**
- Incredibly simple to learn (15 minutes to basics)
- Source is readable as-is, even without rendering
- GitHub uses it for READMEs, Issues, PRs, Wikis
- Jupyter notebooks, Stack Overflow, Reddit, Discord, Slack — all use it
- Powers documentation sites (MkDocs, GitBook, Docusaurus)

**Markdown Dialects (Flavors):**

| Flavor | Used By | Notable Extensions |
|---|---|---|
| **CommonMark** | Standard spec | Tables, strikethrough |
| **GitHub Flavored Markdown (GFM)** | GitHub | Task lists `- [x]`, mentions `@user`, auto-links |
| **MultiMarkdown** | Academic writing | Footnotes, citations, math |
| **MDX** | React docs | Embeds JSX components inside Markdown |

### reStructuredText (RST)

Created as part of the **Python documentation toolchain**. More powerful than Markdown for technical documentation but more verbose.

```rst
Section Title
=============

Regular paragraph text.

**Bold** and *italic* text.

.. code-block:: python

   def hello():
       print("Hello, World!")

.. note::
   This is a special note admonition.

.. toctree::
   :maxdepth: 2

   installation
   usage
   api
```

**Key differences from Markdown:**
- **More consistent** — fewer ambiguities and dialect fragmentation than Markdown
- **Richer directives** — `.. note::`, `.. warning::`, `.. toctree::` for complex documentation
- **Python-native** — powers the official Python docs and Sphinx documentation system
- **Less readable as plain text** — underline-based headings feel unnatural to some

### AsciiDoc

A powerful markup language used heavily in technical publishing and the Java ecosystem:

```asciidoc
= Document Title
Author Name <author@example.com>
v1.0, 2024-01-15

== Section One

Some text with *bold* and _italic_.

[source,java]
----
public class HelloWorld {
    public static void main(String[] args) {
        System.out.println("Hello, World!");
    }
}
----

NOTE: This is an admonition note.

[cols="1,2,3"]
|===
| Col 1 | Col 2 | Col 3
| Cell  | Cell  | Cell
|===
```

**Key strengths:**
- Very rich feature set for books and long documents (cross-references, index, bibliography)
- Powers O'Reilly Media technical books and the official Git documentation
- **Asciidoctor** is the primary implementation (Ruby-based, also available for other languages)

### Readability Comparison

The same content in all four formats:

```
HTML:
<h2>Getting Started</h2>
<p>Install the package using <code>pip install mylib</code>.</p>

Markdown:
## Getting Started
Install the package using `pip install mylib`.

RST:
Getting Started
---------------
Install the package using ``pip install mylib``.

AsciiDoc:
== Getting Started
Install the package using `pip install mylib`.
```

Markdown wins on readability for simple content; RST and AsciiDoc win for complex technical documentation.

---

## 3.4 Philosophy of Plain Text: Advantages vs. Trade-offs

### Why Text?

#### Universal Character Encodings (ASCII → Unicode / UTF-8)

Plain text files use standard character encodings that every system on earth understands:

- **ASCII (1963)**: 128 characters — English alphabet, digits, basic punctuation. Every computer speaks ASCII.
- **Unicode**: A universal standard mapping 150,000+ characters from all world writing systems to unique code points
- **UTF-8**: The dominant encoding of Unicode — variable-width, backward-compatible with ASCII, used by ~98% of all websites

```
"Hello" in ASCII:   72 101 108 108 111
"Hello" in UTF-8:   Same bytes (UTF-8 is ASCII-compatible for basic Latin)
"नमस्ते" in UTF-8:  0xE0 0xA4 0xA8 0xE0 0xA4 0xAE... (3 bytes per character)
```

A plain text file written today will be readable on any device, in any operating system, in any programming language, in 100 years.

#### Longevity and Defense Against Format Obsolescence

Binary formats die. Plain text lives forever.

| Format | Status |
|---|---|
| `.doc` (old Word binary) | Requires Microsoft Word; early versions barely open in modern Word |
| `.pages` (Apple Pages) | Unusable on non-Apple devices without conversion |
| `.pdf` | Readable everywhere, but hard to edit and not diffable |
| **`.txt`, `.md`, `.rst`** | **Readable in any text editor on any OS, now and in 100 years** |

For long-term archiving (research, legal documents, historical records), plain text is the safest format.

#### Compact Payload Size

Plain text is compact. There's no binary overhead, no embedded fonts, no metadata bloat:

```
Word document:  15 KB for a one-page letter
PDF:             80 KB for the same letter
Markdown:         0.8 KB — pure text, no overhead
```

#### Human Inspectability and Diffability

Plain text works perfectly with:
- **`cat`, `grep`, `sed`, `awk`** — Unix text processing tools
- **`git diff`** — Line-by-line change tracking (impossible with binary formats)
- **`grep`** — Full-text search across thousands of files instantly

```diff
# git diff on a Markdown file — readable and meaningful
- The function returns an integer.
+ The function returns a float.
```

Contrast with a binary Word doc diff — you'd see binary garbage or no diff at all.

---

### Why Not Text?

#### Difficulty Encoding Deep Hierarchical / Nested Structures

Plain text is **linear** — it works well for flat or shallow document structures, but struggles with deeply nested or complex structured data:

```
# Fine in Markdown: shallow nesting
- Item 1
  - Sub-item 1.1
    - Sub-sub-item 1.1.1  ← Getting unwieldy

# Terrible in Markdown: relational/graph data
# How do you express: "Student 42 is enrolled in CS101 and MATH201,
# CS101 is taught by Professor 7, who is in Department 3..."
# → You can't. Use JSON or a database for this.
```

Deeply nested tree structures (like a software AST, a complex configuration, or relational data) need XML, JSON, YAML, or a proper database — not a markup language.

#### Parsing Ambiguities Across Dialect Implementations

Markdown, in particular, has **no single canonical spec** (CommonMark attempts to fix this but adoption is incomplete). Different implementations parse the same text differently:

```markdown
# Does this mean a code block or a quoted paragraph?
    some text with 4 spaces of indentation

# Is this bold or literal asterisks?
*text* with *asterisks* in the middle*
```

Different Markdown parsers (Python-Markdown, marked.js, pandoc, kramdown, goldmark) may render the same source differently. This is a real interoperability problem when moving content between systems.

#### Historical Bias Towards Latin / Roman Scripts

Early character encodings (especially ASCII and early HTML) were **designed around the Latin alphabet**:
- ASCII has no characters for Chinese, Arabic, Hindi, Japanese, etc.
- Early web pages in non-Latin scripts required hacks and custom encodings (Shift-JIS for Japanese, GB2312 for Chinese, etc.)
- Many older text processing tools assume one byte = one character (breaks with UTF-8 multi-byte characters)
- Right-to-left text (Arabic, Hebrew) requires special Unicode and CSS handling (the `dir` attribute)

Unicode has largely solved the *encoding* problem, but legacy tools, assumptions, and biases in markup language design still exist.

---

## 3.5 Compilation, Transpilation & Format Conversion

### The Compilation Pipeline

Lightweight markup languages are **compiled** to target formats via a systematic translation:

```
Source Text (Markdown/RST/AsciiDoc)
        │
        ▼  Parser
   Abstract Syntax Tree (AST)
        │
        ▼  Code Generator / Writer
   Target Format (HTML / PDF / DOCX / LaTeX / ePub / man page...)
```

The **AST (Abstract Syntax Tree)** is an intermediate, format-neutral representation of the document's structure — a tree of nodes like `Heading(level=2, text="Introduction")`, `Paragraph([Text("Hello "), Bold("world")])`, etc.

### Pandoc — The Universal Document Converter

**Pandoc** is often called the **"Swiss Army Knife" of document conversion**. It can convert between virtually any combination of document formats:

**Supported Input Formats** (partial list):
Markdown, CommonMark, GFM, reStructuredText, AsciiDoc, HTML, LaTeX, DocBook, ODT, DOCX, EPUB, MediaWiki, Jupyter notebooks, CSV, JSON, YAML...

**Supported Output Formats** (partial list):
HTML, PDF (via LaTeX/WeasyPrint/wkhtmltopdf), DOCX, ODT, EPUB, LaTeX, man pages, Markdown, RST, AsciiDoc, reveal.js (slideshow), Beamer (LaTeX slides), MediaWiki...

**Example conversions:**

```bash
# Markdown → HTML
pandoc README.md -o index.html

# Markdown → PDF (requires LaTeX)
pandoc thesis.md -o thesis.pdf

# Markdown → Word Document
pandoc report.md -o report.docx

# RST → Markdown
pandoc docs.rst -o docs.md

# Jupyter Notebook → HTML
pandoc notebook.ipynb -o notebook.html

# HTML → Markdown (reverse direction works too!)
pandoc article.html -o article.md
```

**Why Pandoc is remarkable:**
- A single tool replaces a whole ecosystem of format-specific converters
- Extends Markdown with footnotes, citations, math (LaTeX), cross-references
- Supports custom templates for every output format
- Widely used in academia for writing papers in Markdown/RST and generating LaTeX/PDF

### Custom Compiler Pipelines

Organizations often build custom pipelines that go beyond off-the-shelf tools:

```
Markdown source files
        │
        ▼ Custom Markdown parser (adds project-specific syntax)
   Custom AST (adds @include, {{variable}}, [[crossref]] nodes)
        │
        ├──▶ HTML generator → Static website
        ├──▶ PDF generator → Printable manual
        └──▶ JSON generator → API documentation
```

Examples:
- **Docusaurus** (Meta/Facebook): Markdown → React-powered documentation site
- **MkDocs**: Markdown → static documentation site
- **Sphinx** (Python): RST → HTML, PDF, ePub documentation

---

## 3.6 Blurring Code and Content: Mixed Functionality

### The Traditional Separation

Historically, there was a clean separation:
- **Content files** (`.md`, `.html`, `.txt`) — just text and markup
- **Code files** (`.py`, `.js`, `.java`) — just logic

Modern tools blur this boundary.

### Literate Programming

**Literate Programming** (coined by Donald Knuth, creator of TeX) is the idea that code and documentation should be **interleaved in a single document**, written for human readers first, computers second.

- **Web/Weave** (Knuth's tools): Extract code for compilation (`tangle`) or extract docs for typesetting (`weave`) from the same source file
- **Jupyter Notebooks** (`.ipynb`): The modern realization of literate programming — interleaved Markdown, code cells, and output (plots, tables, text)

```
Jupyter Notebook:
  [Markdown cell] ## Data Preprocessing
  [Markdown cell] We first normalize the input features.
  [Code cell]     df = pd.read_csv("data.csv"); df = (df - df.mean()) / df.std()
  [Output cell]   [rendered DataFrame table]
  [Markdown cell] The normalized data has mean ~0 and std ~1.
```

This is extremely powerful for data science, scientific papers, and educational content where the narrative and the computation are inseparable.

### Documentation Generators (Code → Docs)

Tools that **extract documentation from code comments**:

| Tool | Language | Output |
|---|---|---|
| **Doxygen** | C, C++, Java, Python | HTML, PDF, LaTeX |
| **JSDoc** | JavaScript | HTML |
| **Sphinx + autodoc** | Python | HTML, PDF, ePub |
| **Javadoc** | Java | HTML |
| **Rustdoc** | Rust | HTML |

```python
def calculate_gpa(grades: list[float]) -> float:
    """
    Calculate the Grade Point Average from a list of grades.

    Args:
        grades: A list of numeric grades in the range [0.0, 10.0].

    Returns:
        The arithmetic mean of all grades, rounded to 2 decimal places.

    Raises:
        ValueError: If the grades list is empty.

    Example:
        >>> calculate_gpa([8.5, 9.0, 7.5])
        8.33
    """
    if not grades:
        raise ValueError("Cannot compute GPA from empty grade list")
    return round(sum(grades) / len(grades), 2)
```

Doxygen or Sphinx reads this comment and generates a full, formatted API reference page — the **code is the source of truth for both logic and documentation**.

### Component Templates: Blending JavaScript and Markup

Modern frontend frameworks blur the line between markup and code at the component level.

#### JSX (React)

JSX embeds HTML-like markup **directly inside JavaScript**:

```jsx
// This is JavaScript, but it contains what looks like HTML
function StudentCard({ student }) {
  return (
    <div className={`card ${student.gpa >= 3.5 ? 'honors' : ''}`}>
      <img src={student.avatar} alt={student.name} />
      <h2>{student.name}</h2>
      <p>GPA: {student.gpa.toFixed(2)}</p>
      <ul>
        {student.courses.map(course => (
          <li key={course.id}>{course.title}</li>
        ))}
      </ul>
    </div>
  );
}
```

JSX is **not HTML** — it's syntactic sugar for `React.createElement()` calls. A Babel compiler transforms it to pure JavaScript. The key point: **markup and logic live in the same file, tightly coupled by design**.

#### Vue Single-File Components (SFCs)

Vue takes a different approach — a `.vue` file contains three sections in one file:

```vue
<template>
  <!-- HTML-like template with Vue directives -->
  <div class="student-card" :class="{ honors: student.gpa >= 3.5 }">
    <img :src="student.avatar" :alt="student.name" />
    <h2>{{ student.name }}</h2>
    <p>GPA: {{ student.gpa.toFixed(2) }}</p>
    <ul>
      <li v-for="course in student.courses" :key="course.id">
        {{ course.title }}
      </li>
    </ul>
  </div>
</template>

<script>
// JavaScript component logic
export default {
  props: ['student'],
  computed: {
    isHonors() { return this.student.gpa >= 3.5; }
  }
}
</script>

<style scoped>
/* CSS scoped to this component only */
.student-card { border: 1px solid #ccc; padding: 1rem; }
.honors { border-color: gold; }
</style>
```

A Vue SFC **unifies HTML, JavaScript, and CSS in a single file** — the opposite of the traditional "three separate files" approach. The Vue compiler (Vite/vue-loader) transforms `.vue` files into JavaScript modules.

#### The Pattern: Domain-Specific Markup Embedded in Code

Both JSX and Vue SFCs represent a broader trend: **embedding specialized markup languages inside general-purpose programming languages** for specific domains (UI components). The compiler bridges the two worlds.

Other examples:
- **Svelte**: Similar to Vue SFCs, compiles to minimal vanilla JS
- **Astro**: `.astro` files mix HTML, Markdown, and JavaScript for static site generation
- **MDX**: Write React components inside Markdown files
- **Jinja2 / Django Templates**: Python + HTML templating
