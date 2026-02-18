# SEDMAT: Sed-Expression-Driven Markdown Annotation & Transformation

**Version**: 3.5  
**Last Updated**: 2026-02-18  
**Status**: Draft

## Abstract

SEDMAT (Sed-Expression-Driven Markdown Annotation & Transformation) is a specification for document manipulation using sed-inspired syntax combined with a compact brace-based formatting DSL (`{flags}`). SEDMAT provides a portable, human-readable DSL for batch text transformations, formatting operations, and structural modifications targeting rich document formats (Google Docs, Word, etc.).

**Brace Syntax** is the canonical formatting system. Markdown-style formatting (`**bold**`, `*italic*`) is supported as a convenience layer for familiarity but is not the primary syntax.

## Introduction

SEDMAT enables:

- **Familiar Syntax**: sed-style `s/pattern/replacement/flags` expressions
- **Rich Formatting**: Brace syntax (`{b}`, `{c=red}`, `{h=1}`) as the canonical format
- **Markdown Shortcuts**: Optional convenience layer (`**bold**`, `*italic*`) for those familiar with Markdown
- **Structural Operations**: Headings, lists, tables, horizontal rules, blockquotes, code blocks
- **Extended sed Commands**: `d/` (delete), `a/` (append), `i/` (insert), `y/` (transliterate)
- **Regex Power**: Full Extended Regular Expression (ERE) support with back-references
- **Batch Processing**: Transform multiple elements via pipelines or `-f` batch files
- **Dry-Run Mode**: Preview changes before applying

## Conventions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119).

## Quick Start

### Minimal Expression

```bash
s/pattern/replacement/
```

**That's it!** A minimal SEDMAT expression:
- Searches for `pattern`
- Replaces with `replacement`
- Affects first match only

### With Brace Formatting

```bash
# Make the first occurrence of warning bold
s/warning/{b}/

# Bold and preserve matched text explicitly
s/warning/{b t=$0}/

# or shorter, because t defaults to $0
s/warning/{b t}/

# or even shorter, because t defaults to being present
s/warning/{b}/

# Bold + red
s/error/{b c=red}/

# Wrap in link
s/Google/{u=https://google.com}/

# Delete paragraphs containing "DRAFT"
d/DRAFT/

# Transliterate vowels
y/aeiou/AEIOU/
```

## Core Concepts

### Expression Anatomy

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#333333', 'primaryTextColor': '#ffffff', 'primaryBorderColor': '#555555', 'lineColor': '#aaaaaa', 'secondaryColor': '#2a2a2a', 'tertiaryColor': '#222222', 'background': '#111111', 'edgeLabelBackground': '#222222', 'clusterBkg': '#1a1a1a', 'clusterBorder': '#444444' }}}%%
graph LR
    subgraph "SEDMAT Expression"
        CMD["s|d|a|i|y"] --> SEP1["/"]
        SEP1 --> PAT["pattern"]
        PAT --> SEP2["/"]
        SEP2 --> REP["replacement"]
        REP --> SEP3["/"]
        SEP3 --> FLAGS["flags"]
    end
    
    style CMD fill:#1a4a1a,color:#ffffff,stroke:#2a7a2a
    style PAT fill:#4a3a1a,color:#ffffff,stroke:#7a5a2a
    style REP fill:#1a2a4a,color:#ffffff,stroke:#2a4a7a
    style FLAGS fill:#4a1a3a,color:#ffffff,stroke:#7a2a5a
```

### Processing Model

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#333333', 'primaryTextColor': '#ffffff', 'primaryBorderColor': '#555555', 'lineColor': '#aaaaaa', 'secondaryColor': '#2a2a2a', 'tertiaryColor': '#222222', 'background': '#111111', 'edgeLabelBackground': '#222222' }}}%%
flowchart TB
    INPUT[Input Document] --> PARSE[Parse Expression]
    PARSE --> CMD{Command<br/>Type?}
    CMD -->|s| HASFORMAT{Contains<br/>Formatting?}
    CMD -->|d| DELETE[Delete Matching<br/>Paragraphs]
    CMD -->|a/i| INSERT[Insert/Append<br/>Paragraphs]
    CMD -->|y| XLAT[Transliterate<br/>Characters]
    HASFORMAT -->|No| NATIVE[Native Regex API<br/>Fast Path]
    HASFORMAT -->|Yes| BRACEPARSE[Parse Brace Syntax]
    BRACEPARSE --> WALK[Document Walk<br/>Full Processing]
    NATIVE --> OUTPUT[Output Document]
    WALK --> OUTPUT
    DELETE --> OUTPUT
    INSERT --> OUTPUT
    XLAT --> OUTPUT
    
    style NATIVE fill:#1a4a1a,color:#ffffff,stroke:#2a7a2a
    style WALK fill:#4a3a1a,color:#ffffff,stroke:#7a5a2a
    style BRACEPARSE fill:#3a2a4a,color:#ffffff,stroke:#5a4a7a
```

SEDMAT processors SHOULD detect plain-text replacements and use native regex APIs for optimal performance.

---

## Commands

### Substitute: `s/pattern/replacement/[flags]` ✅ STABLE

The core command. Matches `pattern` in the document and replaces with `replacement`.

```bash
s/old/new/      # Replace first match
s/old/new/g     # Replace all matches
s/old/new/2     # Replace 2nd match only
```

### Delete: `d/pattern/` ✅ STABLE

Deletes entire paragraphs matching `pattern`.

```bash
d/DRAFT/        # Delete paragraphs containing "DRAFT"
d/^$/           # Delete empty paragraphs
d/TODO.*/       # Delete paragraphs matching regex
```

### Append: `a/pattern/text/` ✅ STABLE

Appends `text` as a new paragraph after each paragraph matching `pattern`.

```bash
a/Introduction/This text appears after the Introduction paragraph/
a/^Chapter/--- end of chapter ---/
```

### Insert: `i/pattern/text/` ✅ STABLE

Inserts `text` as a new paragraph before each paragraph matching `pattern`.

```bash
i/Conclusion/This text appears before the Conclusion paragraph/
i/^Chapter/=== start of chapter ===/
```

### Transliterate: `y/source/dest/` ✅ STABLE

Character-for-character transliteration, like sed's `y` command. Each character in `source` is replaced with the corresponding character in `dest`. Both strings MUST be the same length.

```bash
y/aeiou/AEIOU/          # Uppercase vowels
y/abc/xyz/              # a→x, b→y, c→z
y/ABCDEFGHIJKLMNOPQRSTUVWXYZ/abcdefghijklmnopqrstuvwxyz/  # Lowercase
```

---

## Flags and Delimiters

### Command Flags ✅ STABLE

Flag | Meaning | Status
--- | --- | ---
`g` | Replace ALL matches | ✅ STABLE
`2`..`9` | Replace Nth occurrence only | ✅ STABLE
`m` | Multiline mode (`^`/`$` match line boundaries) | ✅ STABLE
*(none)* | Replace first match only | ✅ STABLE

**Flag combinations**: The `n` (nth occurrence) flag is a digit appended after the delimiter. The `g` and `m` flags MAY be combined: `s/foo/bar/gm`.

```bash
s/foo/bar/g     # All occurrences
s/foo/bar/2     # 2nd occurrence only
s/^line/LINE/m  # Multiline: ^ matches start of each line
s/^line/LINE/gm # Multiline + global
```

**Conformance**: Implementations MUST support `g`, `n` (nth occurrence), and `m` (multiline) flags.

### Delimiters ✅ STABLE

The delimiter `/` MAY be replaced with any consistent character not appearing in the pattern or replacement:

```bash
# Standard
s/path/to/file/replacement/

# Alternate delimiter (when pattern contains /)
s#path/to/file#replacement#
s|path/to/file|replacement|
```

Implementations MUST support at minimum `/`, `#`, and `|` as delimiters.

---

## Brace Syntax — Canonical Formatting DSL ✅ STABLE

> **Status**: ✅ STABLE — Brace Syntax is the canonical formatting system in SEDMAT. All brace syntax features are REQUIRED for conformant implementations.

Brace Syntax is the **primary formatting system** in SEDMAT replacement strings. It uses `{key=value}` pairs and `{flag}` toggles inside curly braces to specify formatting, structural, and semantic attributes.

### Overview

Where Markdown uses wrapper characters (`**bold**`, `*italic*`), Brace Syntax uses flag-based declarations:

```bash
# Brace Syntax (canonical)
s/error/{b}/g

# Brace Syntax with color (no Markdown equivalent)
s/error/{b c=red}/g

# Markdown style (convenience alternative)
s/error/**error**/g
```

Brace Syntax is particularly useful for expressing attributes that have no Markdown equivalent (color, font, size, background, headings, breaks, comments, bookmarks, smart chips).

### Design Rules

1. **`t=` defaults to `$0`** — The implicit text flag preserves the matched text. Writing `{b}` is equivalent to `{b t=$0}`. The matched text is automatically preserved unless you explicitly override it with `t=something`.

2. **Boolean flags need no `=y`** — Simply `{b i _}` for bold + italic + underline.

3. **Bare value flags reset to defaults** — `{f s c z}` resets font to Arial, size to 11pt, color to black, background to clear.

4. **`{0}` resets ALL formatting** — Acts like CSS `all: unset`. MAY be combined: `{0 b}` resets everything then applies bold.

5. **Inline scoping with `{flag=text}`** — Boolean flags accept optional `=text` to scope the style to just that text inline (see [Inline Scoping](#inline-scoping-flagtext--stable)).

6. **Structural flags target containers** — Most flags target text ranges, but structural flags (`cols=`, `+=`, and future page-level properties) target the containing section or document. The matched text acts as a locator to identify which section to modify. Both text flags and structural flags coexist in the same `{}` group — implementations apply each to its appropriate scope.

### The Implicit `t=` Default

This is the most important design rule. When you write a brace expression, the matched text (`$0`) is implicitly preserved:

```bash
# These are equivalent:
s/warning/{b}/g
s/warning/{b t=$0}/g

# The replacement is "warning" with bold applied
```

If you want to replace the text, use `t=`:

```bash
# Replace "error" with "ERROR" and make it bold
s/error/{b t=ERROR}/g

# Replace with back-reference
s/(\w+)/{b t=[$1]}/g   # "hello" → "[hello]" in bold
```

If you omit brace syntax entirely, you're doing a plain text replacement:

```bash
# Plain replacement (no formatting)
s/old/new/g
```

### Boolean Flags ✅ STABLE

Boolean flags are activated by presence alone. No `=y` is needed.

| Short | Long | Effect |
|-------|------|--------|
| `b` | `bold` | Bold |
| `i` | `italic` | Italic |
| `_` | `underline` | Underline |
| `-` | `strike` | Strikethrough |
| `#` | `code` | Monospace/code |
| `^` | `sup` | Superscript (entire replacement) |
| `,` | `sub` | Subscript (entire replacement) |
| `w` | `smallcaps` | Small caps |
| `check` | — | Checkbox (tri-state: absent/`=n`/`=y`) |

**Examples:**

```bash
s/important/{b}/g              # Bold
s/emphasis/{i}/g               # Italic
s/code/{#}/g                   # Monospace
s/deleted/{-}/g                # Strikethrough
s/term/{_ i}/g                 # Underline + italic
```

### Inline Scoping: `{flag=text}` ✅ STABLE

Boolean flags accept optional `=text` for inline scoping. This allows you to apply a style to just a portion of the replacement text, rather than the entire match.

#### How It Works

- **`{b}` alone as entire replacement** = bold the whole match (implicit `{b=$0}`)
- **`{b=text}` embedded in replacement** = bold just "text" inline

#### Supported Flags

All 8 boolean flags support inline scoping:

| Flag | Inline Form | Effect |
|------|-------------|--------|
| `b` | `{b=text}` | Bold just "text" |
| `i` | `{i=text}` | Italicize just "text" |
| `_` | `{_=text}` | Underline just "text" |
| `-` | `{-=text}` | Strikethrough just "text" |
| `#` | `{#=text}` | Monospace just "text" |
| `^` | `{^=text}` | Superscript just "text" |
| `,` | `{,=text}` | Subscript just "text" |
| `w` | `{w=text}` | Small caps just "text" |

#### Examples

**Chemistry (subscripts):**
```bash
s/H2O/H{,=2}O/g               # H₂O — subscript only the "2"
s/CO2/CO{,=2}/g               # CO₂
s/C6H12O6/C{,=6}H{,=12}O{,=6}/g  # Glucose formula
```

**Math (superscripts):**
```bash
s/x2/x{^=2}/g                 # x² — superscript only the "2"
s/E=mc2/E=mc{^=2}/g           # E=mc²
s/x2\+y2=z2/x{^=2}+y{^=2}=z{^=2}/g  # Pythagorean theorem
```

**Prose (emphasis):**
```bash
s/Warning: read this/{b=Warning}: read this/g     # Bold only "Warning"
s/Note: important/{i=Note}: important/g           # Italic only "Note"
s/Action Required: do it/{b=Action} {i=Required}: do it/g  # Mix inline styles
```

**Code spans:**
```bash
s/Run npm install to start/{#=npm install}/g      # Monospace just the command
s/Use the --force flag/{#=--force}/g              # Monospace just the flag
```

**Strikethrough (pricing, edits):**
```bash
s/\$\$99 now \$\$49/{-=$$99} now $$49/g           # Strike the old price
s/was 100 now 75/{-=was 100} now 75/g             # Strike the old text
```

#### Complex Multi-Style Inline Spans

For applying multiple styles to the same inline text, use `t=`:

```bash
# Bold + red for "WARNING"
s/WARNING: danger/{b c=red t=WARNING}: danger/g

# Bold + italic + underline for a term
s/important term/{b i _ t=important term}/g
```

**Note**: `{b=text}` is syntactic sugar for `{b t=text}` when used inline. The `t=` form is required when combining multiple styles on one span.

### Negation ✅ STABLE

Use `!` prefix to explicitly turn off a flag:

```bash
s/already bold/{!b}/g         # Remove bold
s/styled/{!b !i !_}/g         # Remove multiple styles
```

> **Note**: The `=n` suffix for negation (e.g., `{b=n}`) is no longer supported as of v3.3. The `=` on boolean flags now indicates inline scoping (e.g., `{b=text}`). Use the `!` prefix for all negation.

### Combining Boolean Flags ✅ STABLE

Multiple boolean flags MAY appear in a single brace group:

```bash
s/critical/{b i _}/g      # Bold + italic + underline
s/code/{# -}/g            # Monospace + strikethrough
s/warning/{b i c=red}/g   # Bold + italic + red color
```

### Value Flags ✅ STABLE

Value flags use `key=value` syntax. When a value flag appears **bare** (without `=value`), it resets to its default.

| Short | Long | Default (bare) | Effect |
|-------|------|----------------|--------|
| `t` | `text` | `$0` (matched text) | Replacement text |
| `c` | `color` | black | Text color (hex or named) |
| `z` | `bg` | clear | Background/highlight color |
| `f` | `font` | Arial | Font family |
| `s` | `size` | 11pt | Font size in points |
| `u` | `url` | (strip link) | Link/URL/URI |
| `h` | `heading` | HEADING_1 | Heading level |
| `l` | `leading` | 1.15 | Line height |
| `a` | `align` | left | Text alignment |
| `o` | `opacity` | 100 | Opacity (percentage) |
| `n` | `indent` | 0 | Indent level |
| `k` | `kerning` | 0 | Letter spacing |
| `x` | `width` | — | Width in pixels |
| `y` | `height` | — | Height in pixels |
| `p` | `spacing` | default | Paragraph spacing above/below (pt or above,below) |
| `e` | `effect` | — | Shadow/glow/blur |
| `cols` | — | 1 | Number of columns in containing section |
| `toc` | — | — | Insert table of contents (optional `=N` for max depth) |

**Examples:**

```bash
# Colors
s/error/{c=red}/g
s/warning/{c=#FFA500}/g        # Orange (hex)
s/highlight/{z=yellow}/g       # Yellow background

# Fonts and sizes
s/heading/{f=Georgia s=18}/g
s/caption/{s=9 i}/g

# Combined
s/CRITICAL/{b c=red z=yellow s=14}/g
```

### Named Colors ✅ STABLE

SEDMAT defines a standard named color palette for the `c=` (color) and `z=` (background) flags. Implementations MUST support hex notation (`#RRGGBB`). Named colors are RECOMMENDED.

| Name | Hex | Name | Hex |
|------|-----|------|-----|
| `black` | #000000 | `white` | #FFFFFF |
| `red` | #FF0000 | `green` | #00FF00 |
| `blue` | #0000FF | `yellow` | #FFFF00 |
| `cyan` | #00FFFF | `magenta` | #FF00FF |
| `orange` | #FF8C00 | `purple` | #800080 |
| `pink` | #FF69B4 | `brown` | #8B4513 |
| `gray` / `grey` | #808080 | `lightgray` | #D3D3D3 |
| `darkgray` | #404040 | `navy` | #000080 |
| `teal` | #008080 | | |

**Examples:**

```bash
# These are equivalent:
s/error/{c=red}/g
s/error/{c=#FF0000}/g

# Named colors for readability
s/warning/{c=orange z=yellow}/g
s/info/{c=navy}/g
s/success/{c=green b}/g
s/muted/{c=gray}/g
```

### Reset: `{0}` ✅ STABLE

The `{0}` flag resets ALL formatting to defaults, like CSS `all: unset`:

```bash
s/messy text/{0}/g            # Strip all formatting
s/messy text/{0 b}/g          # Reset then apply bold only
s/weird font/{0 f=Arial}/g    # Reset then set font
```

You can also reset individual attributes by using bare value flags:

```bash
s/text/{f s c z}/g            # Reset font, size, color, background
```

### Heading Shorthands ✅ STABLE

The `h` flag controls heading level using compact shorthands:

| Syntax | Maps to |
|--------|---------|
| `{h=t}` | TITLE |
| `{h=s}` | SUBTITLE |
| `{h=1}` | HEADING_1 |
| `{h=2}` | HEADING_2 |
| `{h=3}` | HEADING_3 |
| `{h=4}` | HEADING_4 |
| `{h=5}` | HEADING_5 |
| `{h=6}` | HEADING_6 |
| `{h}` (bare) | HEADING_1 (default heading) |
| `{h=0}` | NORMAL_TEXT (reset heading) |

**Examples:**

```bash
s/My Document/{h=t}/g         # Set as Title
s/Overview/{h=s}/g            # Set as Subtitle
s/Chapter 1/{h=1 b}/g         # Heading 1 + bold
s/Section/{h=2}/g             # Heading 2
s/heading text/{h}/g          # Apply default heading (HEADING_1)
s/normal paragraph/{h=0}/g    # Reset to normal text
```

### Paragraph Spacing: `{p=}` ✅ STABLE

The `p` flag controls paragraph spacing (space above and below the paragraph).

| Syntax | Effect |
|--------|--------|
| `{p=12}` | 12pt spacing both above and below |
| `{p=12,6}` | 12pt above, 6pt below (comma-separated) |
| `{p=0}` | No spacing (tight paragraphs) |
| `{p=0,12}` | No spacing above, 12pt below |
| `{p}` (bare) | Reset to default spacing |

**Examples:**

```bash
# Tight title with space below
s/Title/{h=1 p=0,24}/g

# Generous spacing for readability
s/Section/{h=2 p=24,12}/g

# No spacing for compact lists
s/item/{p=0}/g

# Reset to defaults
s/paragraph/{p}/g
```

### Superscript and Subscript ✅ STABLE

SEDMAT provides boolean flags for super/subscript with inline scoping support.

#### Boolean Flags: `{^}` and `{,}`

The `{^}` (superscript) and `{,}` (subscript) flags apply to the **entire replacement text** when used alone:

```bash
s/TM/{^}/g                    # "TM" as superscript → ᵀᴹ
s/2/{,}/g                     # "2" as subscript → ₂
s/note/{,}/g                  # "note" entirely as subscript
```

#### Inline Scoping for Partial Formatting

Use `{^=text}` and `{,=text}` to apply super/subscript to just part of the replacement:

```bash
# Chemistry
s/H2O/H{,=2}O/g               # H₂O
s/CO2/CO{,=2}/g               # CO₂
s/H2SO4/H{,=2}SO{,=4}/g       # H₂SO₄

# Math
s/E=mc2/E=mc{^=2}/g           # E=mc²
s/x2\+y2=z2/x{^=2}+y{^=2}=z{^=2}/g  # x²+y²=z²

# Ordinals
s/1st/1{^=st}/g               # 1ˢᵗ
s/2nd/2{^=nd}/g               # 2ⁿᵈ
```

#### Unicode Alternative

For simple cases, Unicode super/subscript characters work directly:

```bash
s/H2O/H₂O/g                   # Unicode subscript 2
s/E=mc2/E=mc²/g               # Unicode superscript 2
```

Unicode superscript: ⁰¹²³⁴⁵⁶⁷⁸⁹⁺⁻⁼⁽⁾ⁿ  
Unicode subscript: ₀₁₂₃₄₅₆₇₈₉₊₋₌₍₎

### Breaks: `{+}` ✅ STABLE

The `+` key inserts structural breaks. When used alone, it inserts a horizontal rule. With a value, it specifies the break type:

| Syntax | Effect |
|--------|--------|
| `{+}` | Horizontal rule (default) |
| `{+=p}` | Page break |
| `{+=c}` | Column break |
| `{+=s}` | Section break |

**Examples:**

```bash
s/---/{+}/g                   # Horizontal rule
s/END/{+=p}/g                 # Page break
s/COLUMN_BREAK/{+=c}/g        # Column break
s/SECTION_END/{+=s}/g         # Section break
```

### Columns: `{cols=N}` ✅ STABLE

The `cols=` flag is a **structural flag** that sets the number of columns in the section containing the match. The matched text acts as a **locator** to identify which section to modify.

| Syntax | Effect |
|--------|--------|
| `{cols=2}` | 2-column layout |
| `{cols=1}` | Single column (default) |
| `{cols=3}` | 3-column layout |
| `{cols}` (bare) | Reset to 1 column |

#### Structural vs Text Flags

| Flag Type | Target | Examples |
|-----------|--------|----------|
| Text flags | Matched text range | `{b}`, `{c=red}`, `{h=1}` |
| Structural flags | Containing section | `{cols=2}`, `{+=s}`, `{+=p}` |

Both coexist in the same `{}` group:

```bash
# h=1 and b apply to text; cols=2 applies to section
s/Chapter/{h=1 b cols=2}/g
```

#### Pairing with Section Breaks

```bash
# Section break + set new section to 2 columns
s/NEWSLETTER_START/{+=s cols=2}/g

# Return to single column for footer
s/FOOTER_START/{+=s cols=1}/g
```

#### Column Breaks

```bash
# Force content to next column
s/NEXT_COLUMN/{+=c}/g
```

#### Example: Newsletter Layout

```bash
s/^/{h=t t=Monthly Newsletter}\n/
s/ARTICLES/{+=s cols=2 h=2 t=Articles}/g
s/ARTICLE_BREAK/{+=c}/g
s/FOOTER/{+=s cols=1 h=2 t=Contact}/g
```

### Checkboxes: `{check}` ✅ STABLE

The `check` flag creates native checklist items (Google Docs checkboxes). Unlike other boolean flags, `check` has **tri-state semantics**:

| Syntax | Effect |
|--------|--------|
| (absent) | No checkbox |
| `{check}` or `{check=n}` | Unchecked checkbox ☐ |
| `{check=y}` | Checked checkbox ☑ |

**Examples:**

```bash
# Convert TODO markers to unchecked checkboxes
s/TODO/{check=n}/g

# Convert DONE markers to checked checkboxes
s/DONE/{check=y}/g

# Checkbox + bold
s/task/{check=n b}/g

# Build a checklist from placeholders
s/\[ \]/{check=n}/g
s/\[x\]/{check=y}/g
```

#### Batch Checklist Creation

```bash
# Transform a list into a checklist
s/- pending/{check=n t=pending}/g
s/- complete/{check=y t=complete}/g

# Add checkbox to all list items
s/^- (.+)/{check=n t=$1}/gm
```

### Table of Contents: `{toc}` ✅ STABLE

The `toc` flag inserts a table of contents generated from document headings (h=1 through h=6).

| Syntax | Effect |
|--------|--------|
| `{toc}` | Full TOC (all heading levels) |
| `{toc=N}` | TOC with max depth N |

**Examples:**

```bash
# Insert TOC at placeholder
s/{{TOC}}/{toc}/

# Prepend TOC to document
s/^/{toc}\n/

# TOC with only h1 and h2
s/{{TOC}}/{toc=2}/

# TOC after title
s/Introduction/{toc}\n\nIntroduction/
```

Implementations SHOULD auto-link TOC entries to their corresponding headings.

### Comments: `{"=text}` ✅ STABLE

The `"` key attaches a comment (annotation) to the matched text:

```bash
s/TODO/{"=needs review}/g                 # Just a comment
s/TODO/{"=needs review b c=red}/g         # Comment + bold + red
s/FIXME/{"=assigned to viz b}/g           # Comment with formatting
```

### Bookmarks: `{@=name}` and `{u=#name}` ✅ STABLE

Create bookmark anchors and link to them:

| Syntax | Effect |
|--------|--------|
| `{@=name}` | Creates a bookmark anchor on the matched text |
| `{u=#name}` | Links to a bookmark by name |

**Examples:**

```bash
# Create a bookmark anchor
s/Chapter 1/{@=ch1 h=1}/g            # Bookmark + heading

# Link to the bookmark
s/see Chapter 1/{u=#ch1 c=blue _}/g  # Link to bookmark + styling

# Create anchor and style
s/Definition/{@=def1 b}/g

# Reference it elsewhere
s/as defined above/{u=#def1}/g
```

### URI Schemes for `{u=}` ✅ STABLE

The `u` flag accepts any valid URI. SEDMAT defines standard schemes and custom `chip://` schemes for Google Docs smart chips.

#### Standard Schemes

| Scheme | Example |
|--------|---------|
| `https://` | `{u=https://deft.md}` |
| `http://` | `{u=http://example.com}` |
| `mailto:` | `{u=mailto:viz@example.com}` |
| `tel:` | `{u=tel:+14076163470}` |
| `geo:` | `{u=geo:28.69,-81.31}` |
| `sms:` | `{u=sms:+14076163470}` |
| `webcal:` | `{u=webcal://feed.ics}` |
| `file:` | `{u=file:///path}` |

**Examples:**

```bash
s/Google/{u=https://google.com}/g
s/contact/{u=mailto:hello@example.com}/g
s/call us/{u=tel:+14075551234}/g
s/location/{u=geo:28.5383,-81.3792}/g
```

#### Custom `chip://` Schemes ✅ STABLE

For Google Docs smart chip insertion:

| Scheme | Example | Effect | Notes |
|--------|---------|--------|-------|
| `chip://person/` | `{u=chip://person/viz@example.com}` | Person smart chip | |
| `chip://date/` | `{u=chip://date/2026-03-15}` | Date smart chip | |
| `chip://file/` | `{u=chip://file/DOC_ID}` | File link chip | |
| `chip://place/` | `{u=chip://place/Orlando, FL}` | Place chip | |
| `chip://dropdown/` | `{u=chip://dropdown/Draft\|Review\|Done}` | Dropdown chip | |
| `chip://chart/` | `{u=chip://chart/SHEET_ID/0}` | Chart embed | |
| `chip://bookmark/` | `{u=chip://bookmark/section-1}` | Internal doc link | Prefer `{u=#name}` shorthand |

**Examples:**

```bash
s/viz/{u=chip://person/jtaylor@zendicate.com}/g
s/deadline/{u=chip://date/2026-03-15}/g
s/project doc/{u=chip://file/1BxiMVs0XRA5nFMdKvBdBZjgmUUqptlbs74OgvE2upms}/g
s/status/{u=chip://dropdown/Draft|Review|Done}/g
```

### Charts ✅ STABLE

Two approaches for embedding charts from Google Sheets:

#### 1. Direct chip:// Reference

```bash
s/CHART/{u=chip://chart/SHEET_ID/0 x=600 y=400}/g
```

Pulls chart at index `0` from the linked Sheet and embeds it at 600×400 pixels.

#### 2. Unix Pipe Composition

```bash
gog sheets chart SHEET_ID 0 --png | gog docs sed DOC_ID 's/PLACEHOLDER/{img=-1}/g'
```

Export chart as image, then insert via pipe.

### Brace Syntax Examples

```bash
# Simple styling
s/error/{b c=red}/g
s/TODO/{b c=red z=yellow}/g
s/note/{i c=gray s=10}/g

# Text replacement with formatting
s/old text/{b t=new text}/g
s/(\w+)/{i t=[$1]}/g

# Inline scoping
s/H2O/H{,=2}O/g               # Subscript just "2"
s/E=mc2/E=mc{^=2}/g           # Superscript just "2"
s/Warning: stop/{b=Warning}: stop/g  # Bold just "Warning"
s/Run npm install/{#=npm install}/g  # Monospace the command

# Reset formatting
s/messy text/{0}/g
s/messy text/{0 b}/g          # Reset then bold

# Links and chips
s/viz/{u=chip://person/jtaylor@zendicate.com}/g
s/deadline/{u=chip://date/2026-03-15}/g
s/call me/{u=tel:+14075551234}/g
s/website/{u=https://example.com c=blue _}/g

# Headings
s/My Doc/{h=t}/g
s/Overview/{h=s}/g
s/Chapter 1/{h=1 b}/g

# Superscript and subscript
s/TM/{^}/g
s/note/{,}/g
s/CO2/CO{,=2}/g
s/x2/x{^=2}/g

# Breaks
s/END/{+=p}/g
s/---/{+}/g
s/SECTION/{+=s}/g

# Columns
s/NEWSLETTER/{+=s cols=2}/g   # Section break + 2 columns
s/FOOTER/{+=s cols=1}/g       # Back to single column
s/NEXT_COL/{+=c}/g            # Column break

# Checkboxes
s/TODO/{check=n}/g            # Unchecked
s/DONE/{check=y}/g            # Checked
s/task/{check=n b}/g          # Checkbox + bold

# Table of contents
s/{{TOC}}/{toc}/              # Full TOC
s/{{TOC}}/{toc=2}/            # TOC depth 2

# Comments and bookmarks
s/TODO/{@=todo1 "=needs review b c=red}/g
s/see above/{u=#todo1 c=blue _}/g

# Combined
s/WARNING/{b i c=red s=14 z=yellow}/g
s/https\S+/{u=$0 c=blue _}/g
```

---

## Back-References ✅ STABLE

SEDMAT supports capture groups using `(...)` with references via `$1`-`$9` or `\1`-`\9`:

```bash
# Capture and format
s/(important)/{b t=$1}/g

# Bold all-caps words (2+ chars)
s/([A-Z]{2,})/{b t=$1}/g

# Wrap names in links
s/(@\w+)/{u=https://twitter.com/$1 t=$1}/g
```

### Reference Syntax

| Reference | Alternative | Meaning |
|-----------|-------------|---------|
| `$0` | `&` | Entire match |
| `$1` | `\1` | First capture group |
| `$2` | `\2` | Second capture group |
| `$n` | `\n` | Nth capture group (1-9) |

### Whole-Match: `&`

The `&` represents the entire matched text (like sed). Use `\&` for a literal ampersand.

```bash
# Wrap every match in brace formatting
s/important/{b}/g             # Implicit: {b t=$0}

# Using & explicitly in plain replacement
s/important/(**&**)/g         # Markdown-style with parens

# Literal ampersand
s/rock/rock \& roll/          # \& = literal &
```

### Dollar Sign Escaping

Use `$$` for a literal dollar sign in replacements:

```bash
s/PRICE/$$49.99/g             # Outputs: $49.99
s/COST/$$100/g                # Outputs: $100
```

**Conformance**: Implementations MUST support `$1`-`$9`, `$0`, `&`, and `$$` notation. Support for `\1`-`\9` is RECOMMENDED for sed compatibility.

---

## Links ✅ STABLE

### Link Syntax

| Syntax | Effect | Status |
|--------|--------|--------|
| `[text](url)` | Hyperlink with text | REQUIRED |
| `<url>` | Auto-link (bare URL) | REQUIRED |
| `[text](url "title")` | Link with title attribute | RECOMMENDED |
| `{u=url}` | Brace syntax link | REQUIRED |

**Examples:**

```bash
# Markdown-style link
s/Google/[Google](https:\/\/google.com)/

# Brace syntax link (canonical)
s/Google/{u=https://google.com}/

# Link with styling
s/Google/{u=https://google.com c=blue _}/

# Link email addresses
s/(\S+@\S+)/[$1](mailto:$1)/g

# Auto-link bare URLs
s/(https:\/\/\S+)/<$1>/g
```

---

## Images ✅ STABLE

### Standard Markdown Image Insertion (RECOMMENDED)

| Syntax | Effect | Status |
|--------|--------|--------|
| `![](url)` | Image, no alt text | ✅ RECOMMENDED |
| `![alt](url)` | Image with alt text | ✅ RECOMMENDED |
| `![alt](url "caption")` | Image with caption | ✅ RECOMMENDED |
| `![](url){x=N}` | Width in pixels | ✅ STABLE |
| `![](url){y=N}` | Height in pixels | ✅ STABLE |
| `![](url){x=N y=M}` | Both dimensions | ✅ STABLE |

**Examples:**

```bash
# Insert at placeholder (standard markdown)
s/{{LOGO}}/![](https:\/\/example.com\/logo.png)/

# With alt text
s/{{LOGO}}/![Company Logo](https:\/\/example.com\/logo.png)/

# With dimensions (brace syntax)
s/{{HERO}}/![](https:\/\/example.com\/hero.jpg){x=600}/
s/{{BANNER}}/![](https:\/\/example.com\/banner.png){x=800 y=200}/
```

### Brace Image References ✅ STABLE

> **Status**: ✅ STABLE — The `{img=}` brace syntax is the canonical way to reference existing images by position or pattern.

| Syntax | Meaning | Status |
|--------|---------|--------|
| `{img=1}` | First image | ✅ STABLE |
| `{img=2}` | Second image | ✅ STABLE |
| `{img=-1}` | Last image | ✅ STABLE |
| `{img=-2}` | Second to last | ✅ STABLE |
| `{img=*}` | All images | ✅ STABLE |
| `{img=regex}` | Images matching alt text pattern | ✅ STABLE |

**Examples:**

```bash
# Replace first image
s/{img=1}/![](https:\/\/new-image.png)/

# Delete first image
s/{img=1}//

# Replace last image
s/{img=-1}/![](https:\/\/replacement.png)/

# Replace by alt text pattern
s/{img=old-logo}/![new-logo](https:\/\/new.png)/

# Apply to all images
s/{img=*}/{x=400}/
```

### Image Dimensions Grammar

```
dimensions     = "{" dimension-list "}"
dimension-list = dimension ( " " dimension )*
dimension      = width | height
width          = ("width" | "w" | "x") "=" number
height         = ("height" | "h" | "y") "=" number
number         = DIGIT+
```

---

## Tables ✅ STABLE

### Unified Brace Addressing: `{T=}` ✅ STABLE

> **Status**: ✅ STABLE — The `{T=}` key is the canonical brace-based addressing syntax for tabular data across Google Docs tables and Google Sheets. This is the primary syntax for all table operations.

The `{T=}` key uses `!` as the mode switch: the `!` separator (following Excel/Sheets convention) enters cell context. An optional `DOC_ID:` prefix enables cross-document references.

#### Syntax

```
{T=table!cell}
{T=table!range}
{T=table!row-or-col-op}
{T=[doc:]sheet!cell}        # For Sheets
```

| Component | Meaning | Examples |
|-----------|---------|----------|
| `table` | Table reference (1-indexed, negative for reverse) | `1`, `-1`, `*` |
| `doc:` | Document ID (optional, for cross-doc refs) | `1f1W9Wd...:`  |
| `sheet` | Sheet tab by name or 0-based index | `Sales`, `0`, `Budget` |
| `!` | Table/cell separator | |
| `cell` | Cell address (Excel-style) | `A1`, `B3` |
| `range` | Cell range (rectangular) | `A1:C3` |
| `*` | Wildcard (entire table/row/column) | `1!*`, `1!1,*`, `1!*,2` |
| `row=`/`col=` | Row/column operations | `row=$+`, `col=+3` |

#### Table References

```bash
# Reference tables by position
s/{T=1!A1}/Name/              # First table, cell A1
s/{T=-1!B2}/Value/            # Last table, cell B2
s/{T=*!1,*}/{b}/              # All tables, entire row 1
```

#### Cell & Range Access

```bash
# Cell in first table
s/{T=1!A1}/{t=Revenue b}/

# Cell by row,col notation (1-indexed)
s/{T=1!2,3}/{t=Data}/

# Rectangular range
s/{T=1!A1:C3}/{b}/

# Entire table (all cells)
s/{T=1!*}/{0}/

# Wildcard: entire row
s/{T=1!1,*}/{b}/              # Bold entire row 1

# Wildcard: entire column
s/{T=1!*,2}/{i}/              # Italicize entire column 2
```

#### Row & Column Operations

```bash
# Append row to first table
s/{T=1!row=$+}//

# Insert row before row 2
s/{T=1!row=+2}//

# Delete row 5
s/{T=1!row=5}//

# Insert column before column 3
s/{T=1!col=+3}//

# Delete last column
s/{T=1!col=-1}//

# Append column at end
s/{T=1!col=$+}//
```

#### Table Creation

```bash
# Create 3×4 table (3 rows, 4 columns)
s/{{TABLE}}/{T=3x4}/

# With header flag
s/{{TABLE}}/{T=3x4:header}/

# Create table in empty document
s/^$/{T=3x4}/
```

#### Table Deletion

```bash
# Delete first table
s/{T=1}//

# Delete last table
s/{T=-1}//

# Delete all tables
s/{T=*}//
```

#### Table Merge

```bash
# Merge cells in a table
s/{T=1!A1:C1}/merge/

# Merge a 2x2 block
s/{T=1!2,1:3,2}/merge/
```

#### Cross-Document/Sheet References

```bash
# Reference cell in another doc's sheet
s/{T=1f1W9Wd...:Sales!A1}/{t=Updated}/

# Reference named sheet tab
s/{T=Budget!C5}/{t=$$1,234 b c=green}/

# Format entire sheet header row
s/{T=0!1,*}/{b z=#333 c=white}/
```

#### Combined with Brace Formatting

```bash
# Set value + bold + color
s/{T=1!B2}/{t=$$42,000 b c=green}/

# Bold an entire header row
s/{T=1!1,*}/{b z=#333 c=white}/

# Format column as currency
s/{T=1!*,3}/{c=green b}/
```

---

## Positional Insert ✅ STABLE

Special patterns for inserting content at document positions without matching existing text:

| Pattern | Meaning | Status |
|---------|---------|--------|
| `^$` | Empty document only | REQUIRED |
| `^` | Beginning of document (prepend) | REQUIRED |
| `$` | End of document (append) | REQUIRED |

### Behavior

- `^$` — Implementations MUST check that the document body is empty (no non-whitespace content). If the document is not empty, the expression MUST be a no-op.
- `^` — Implementations MUST insert the replacement at the beginning of the document body (index 1).
- `$` — Implementations MUST insert the replacement at the end of the document body.

### Examples

```bash
# Insert text into an empty document
s/^$/Hello world/

# Prepend title to any document
s/^/{h=t t=Document Title}\n/

# Append footer
s/$/\nGenerated on 2026-02-17/

# Insert table at start
s/^/{T=3x4}\n/
```

Positional insert MUST support all replacement features (formatting, images, tables, headings, lists, etc.).

---

## CLI Modes ✅ STABLE

### Dry-Run Mode

The `--dry-run` (or `-n`) flag shows what changes would be made without modifying the document:

```bash
gog docs sed --dry-run 's/foo/bar/g' <doc-id>
gog docs sed -n 's/old/new/' <doc-id>
```

Implementations MUST display the matches and proposed changes, and MUST NOT modify the document.

### Batch Processing

The `-f` flag reads expressions from a file (one per line). Lines starting with `#` are comments:

```bash
gog docs sed -f transforms.sed <doc-id>
```

**transforms.sed:**
```sed
# Header formatting
s/TITLE/{h=t}/
s/SUBTITLE/{h=s}/

# Clean up
s/DRAFT//g
d/TODO/

# Add footer
s/$/\nGenerated automatically/
```

---

## Processing Modes

### Native Mode (Fast Path)

When replacement contains **no formatting markers**, implementations SHOULD use native regex APIs:

```bash
s/colour/color/g          # Plain text
s/\bfoo\b/bar/g           # Word boundary
s/2023/2024/g             # Numeric
```

### Document Walk Mode

Required when formatting is present:

```bash
s/foo/{b}/g                            # Brace syntax
s/bar/{u=https://bar.com}/g            # Link
s/{{IMG}}/![](https://img.png)/        # Image
```

---

## Escaping ✅ STABLE

### Special Characters

Characters with special meaning MUST be escaped with backslash:

| Character | Escape | Context |
|-----------|--------|---------|
| `/` | `\/` | When used as delimiter |
| `*` | `\*` | Literal asterisk |
| `[` | `\[` | Literal bracket |
| `]` | `\]` | Literal bracket |
| `(` | `\(` | Literal paren (in replacement) |
| `)` | `\)` | Literal paren (in replacement) |
| `$` | `$$` | Literal dollar sign (in replacement) |
| `&` | `\&` | Literal ampersand (in replacement) |
| `\` | `\\` | Literal backslash |
| `{` | `\{` | Literal brace (in replacement) |
| `}` | `\}` | Literal brace (in replacement) |

---

## Markdown Formatting — Convenience Alternatives

> **Note**: This section describes Markdown shortcuts that serve as a convenience layer. The canonical formatting system is [Brace Syntax](#brace-syntax--canonical-formatting-dsl--stable). Implementations MUST support brace syntax; Markdown shortcuts are RECOMMENDED for standard CommonMark syntax.

For users familiar with Markdown, SEDMAT supports common Markdown formatting syntax as an alternative to brace syntax. These produce identical output.

### Inline Styles (RECOMMENDED — Standard CommonMark)

| Syntax | Brace Equivalent | Effect | Status |
|--------|------------------|--------|--------|
| `**text**` | `{b t=text}` | Bold | ✅ RECOMMENDED |
| `*text*` | `{i t=text}` | Italic | ✅ RECOMMENDED |
| `_text_` | `{i t=text}` | Italic | ✅ RECOMMENDED |
| `***text***` | `{b i t=text}` | Bold + Italic | ✅ RECOMMENDED |
| `~~text~~` | `{- t=text}` | Strikethrough | ✅ RECOMMENDED |
| `` `text` `` | `{# t=text}` | Monospace | ✅ RECOMMENDED |

**Examples:**

```bash
# These pairs are equivalent:
s/warning/**warning**/g       # Markdown
s/warning/{b}/g               # Brace (preferred)

s/emphasis/*emphasis*/g       # Markdown
s/emphasis/{i}/g              # Brace (preferred)

s/deprecated/~~deprecated~~/g # Markdown
s/deprecated/{-}/g            # Brace (preferred)
```

### Headings (RECOMMENDED — Standard CommonMark)

| Syntax | Brace Equivalent | Effect |
|--------|------------------|--------|
| `# text` | `{h=1 t=text}` | Heading 1 |
| `## text` | `{h=2 t=text}` | Heading 2 |
| `### text` | `{h=3 t=text}` | Heading 3 |
| `#### text` | `{h=4 t=text}` | Heading 4 |
| `##### text` | `{h=5 t=text}` | Heading 5 |
| `###### text` | `{h=6 t=text}` | Heading 6 |

**Examples:**

```bash
# These pairs are equivalent:
s/TITLE/# My Document/        # Markdown
s/TITLE/{h=1 t=My Document}/  # Brace

s/Introduction/## Introduction/  # Markdown
s/Introduction/{h=2}/            # Brace (preserves text)
```

### Lists (RECOMMENDED — Standard CommonMark)

| Syntax | Effect |
|--------|--------|
| `- item` | Bullet list item |
| `1. item` | Numbered list item |
| `  - nested` | Nested bullet (L1, 2 spaces) |
| `    - nested` | Nested bullet (L2, 4 spaces) |
| `  1. nested` | Nested numbered (L1, 2 spaces) |
| `    1. nested` | Nested numbered (L2, 4 spaces) |

**Examples:**

```bash
# Create a bullet list
s/ITEMS/- First item\n- Second item\n- Third item/

# Numbered list
s/STEPS/1. Step one\n2. Step two\n3. Step three/

# Nested lists
s/OUTLINE/- Top level\n  - Nested L1\n    - Nested L2/
```

### Horizontal Rules (RECOMMENDED — Standard CommonMark)

Three or more of `-`, `*`, or `_` on a line create a horizontal rule:

```bash
s/DIVIDER/---/
s/BREAK/***/
s/SEPARATOR/___/
```

Brace equivalent: `{+}`

### Blockquotes (RECOMMENDED — Standard CommonMark)

Lines prefixed with `>` create blockquotes (left-indented with grey left border):

```bash
s/QUOTE/> This is a blockquote/
s/EPIGRAPH/> To be or not to be\n> That is the question/
```

### Code Blocks (RECOMMENDED — Standard CommonMark)

Triple backtick fences create code blocks (Courier New font, grey background). Language hints after the opening fence are stripped:

```bash
s/CODE/```\nfunction hello() {\n  console.log("hi");\n}\n```/
s/SNIPPET/```python\nprint("hello")\n```/
```

### Footnotes (RECOMMENDED)

The `[^text]` syntax creates native footnotes in the target document:

```bash
s/citation/[^See Smith et al., 2024]/
s/note/[^This is a footnote with detailed explanation]/
```

> **Note**: While not part of CommonMark core, footnote syntax (`[^...]`) is widely supported in extended Markdown flavors and is RECOMMENDED.

### Combining Markdown Formats

Formats MAY be nested according to Markdown precedence:

```bash
# Bold AND italic
s/warning/***warning***/

# Bold with brace underline
s/critical/**{_ t=critical}**/
```

**Processing Order:**

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#333333', 'primaryTextColor': '#ffffff', 'primaryBorderColor': '#555555', 'lineColor': '#aaaaaa', 'secondaryColor': '#2a2a2a', 'tertiaryColor': '#222222', 'background': '#111111', 'edgeLabelBackground': '#222222' }}}%%
graph LR
    TEXT[text] --> B["**bold**"]
    B --> I["*italic*"]
    I --> FINAL["***text***"]
    
    style TEXT fill:#2a2a2a,color:#ffffff,stroke:#555555
    style FINAL fill:#1a4a1a,color:#ffffff,stroke:#2a7a2a
```

Implementations MUST process nesting from innermost to outermost.

---

## Grammar

### ABNF Notation

```abnf
sedmat-line    = sedmat-expr / comment
comment        = "#" *VCHAR

; --- Commands ---
sedmat-expr    = subst-expr / delete-expr / append-expr / insert-expr / xlat-expr

subst-expr     = "s" delimiter pattern delimiter replacement delimiter [flags]
delete-expr    = "d" delimiter pattern delimiter
append-expr    = "a" delimiter pattern delimiter text delimiter
insert-expr    = "i" delimiter pattern delimiter text delimiter
xlat-expr      = "y" delimiter source-chars delimiter dest-chars delimiter

delimiter      = "/" / "#" / "|" / %x21-2E / %x3A-40 / %x5B-60 / %x7B-7E
                 ; Any printable ASCII except alphanumeric

pattern        = regex-pattern
source-chars   = 1*CHAR
dest-chars     = 1*CHAR          ; Must be same length as source-chars

; --- Flags ---
flags          = *( "g" / DIGIT / "m" )

; --- Replacement ---
replacement    = *( plain-text / brace-expr / format-expr / back-ref / image-expr
                   / link-expr / heading-expr / list-expr
                   / block-expr / footnote-expr )

; --- Brace Syntax (v3.5) ---
brace-expr     = "{" brace-body "}"
brace-body     = reset-all / brace-flags
reset-all      = "0" *( SP brace-flag )                  ; {0} or {0 b i}

brace-flags    = brace-flag *( SP brace-flag )
brace-flag     = bool-flag / neg-flag / value-flag / break-flag
                 / comment-flag / bookmark-flag / table-flag / image-flag
                 / toc-flag / check-flag

bool-flag      = bool-key ["=" val-value]                ; {b} or {b=text} for inline scoping
bool-key       = "b" / "bold" / "i" / "italic" / "_" / "underline"
                 / "-" / "strike" / "#" / "code" / "^" / "sup"
                 / "," / "sub" / "w" / "smallcaps"

neg-flag       = "!" bool-key                            ; Negation (prefix only)

check-flag     = "check" ["=" ("y" / "n")]               ; Tri-state checkbox

toc-flag       = "toc" ["=" DIGIT+]                      ; Table of contents with optional depth

value-flag     = val-key "=" val-value
val-key        = "t" / "text" / "c" / "color" / "z" / "bg"
                 / "f" / "font" / "s" / "size" / "u" / "url"
                 / "l" / "leading" / "a" / "align" / "o" / "opacity"
                 / "n" / "indent" / "k" / "kerning" / "x" / "width"
                 / "y" / "height" / "p" / "spacing" / "h" / "heading"
                 / "e" / "effect" / "cols"
val-value      = 1*(%x21-7E)                             ; Non-space printable ASCII
spacing-value  = DIGIT+ ["," DIGIT+]                     ; Single value or above,below

break-flag     = "+" ["=" break-type]
break-type     = "p" / "c" / "s"                         ; page / column / section

comment-flag   = DQUOTE "=" comment-text
comment-text   = 1*(%x20-7E)                             ; Printable ASCII incl. space

bookmark-flag  = "@" "=" bookmark-name
bookmark-name  = 1*(%x21-7E)

; --- Table Brace Syntax (v3.1) ---
table-flag     = "T" "=" table-spec
table-spec     = table-create / table-ref-spec
table-create   = DIGIT+ "x" DIGIT+ [":header"]           ; {T=3x4}, {T=3x4:header}
table-ref-spec = table-index ["!" cell-spec]
table-index    = ["-"] DIGIT+ / "*"                      ; 1, -1, *
cell-spec      = excel-cell / row-col-cell / cell-range / row-op / col-op / wildcard
excel-cell     = ALPHA+ DIGIT+                           ; A1, B12
row-col-cell   = (DIGIT+ / "*") "," (DIGIT+ / "*")       ; 1,2 or 1,* or *,2
cell-range     = (excel-cell / row-col-cell) ":" (excel-cell / row-col-cell)
row-op         = "row" "=" ("+" DIGIT+ / "$+" / ["-"] DIGIT+)
col-op         = "col" "=" ("+" DIGIT+ / "$+" / ["-"] DIGIT+)
wildcard       = "*"

; --- Image Brace Syntax (v3.1) ---
image-flag     = "img" "=" image-spec
image-spec     = ["-"] DIGIT+ / "*" / regex-pattern      ; 1, -1, *, pattern

heading-value  = "t" / "s" / "0" / "1" / "2" / "3" / "4" / "5" / "6"
                 ; t=TITLE, s=SUBTITLE, 0=NORMAL_TEXT, 1-6=HEADING_1-6

uri-value      = standard-uri / chip-uri / bookmark-uri
standard-uri   = scheme ":" *(%x21-7E)
scheme         = "https" / "http" / "mailto" / "tel" / "geo"
                 / "sms" / "webcal" / "file"
chip-uri       = "chip://" chip-type "/" *(%x21-7E)
chip-type      = "person" / "date" / "file" / "place"
                 / "dropdown" / "chart" / "bookmark"
bookmark-uri   = "#" bookmark-name                        ; {u=#name}

color-value    = "#" 6HEXDIG / color-name                ; #RRGGBB or named
color-name     = "black" / "white" / "red" / "green" / "blue"
                 / "yellow" / "cyan" / "magenta" / "orange" / "purple"
                 / "pink" / "brown" / "gray" / "grey" / "lightgray"
                 / "darkgray" / "navy" / "teal"

; --- Markdown Formatting (convenience layer) ---
format-expr    = bold / italic / bold-italic / strike / mono

bold           = "**" content "**"
italic         = "*" content "*" / "_" content "_"
bold-italic    = "***" content "***"
strike         = "~~" content "~~"
mono           = "`" content "`"

footnote-expr  = "[^" content "]"

heading-expr   = 1*6("#") SP content    ; # through ######

list-expr      = bullet-item / numbered-item
bullet-item    = [indent] "- " content
numbered-item  = [indent] DIGIT+ ". " content
indent         = 2*SP                   ; 2 spaces per nesting level

block-expr     = horiz-rule / blockquote / code-block
horiz-rule     = "---" / "***" / "___"
blockquote     = "> " content
code-block     = "```" [lang-hint] LF content LF "```"

back-ref       = "$" DIGIT / "\" DIGIT / "&" / "$$"

link-expr      = "[" link-text "](" url [SP "\"" title "\""] ")"
               / "<" url ">"

image-expr     = image-insert / image-brace-ref
image-insert   = "![" [alt-text] "](" url ")" [dimensions]
image-brace-ref = "{" "img" "=" image-spec "}"

dimensions     = "{" *( dim-spec SP ) "}"
dim-spec       = ("width" | "w" | "x") "=" NUMBER
               / ("height" | "h" | "y") "=" NUMBER

positional-pat = "^$" / "^" / "$"
```

---

## Conformance Levels

### Level 1: Core (REQUIRED)

- Basic `s/pattern/replacement/` and `s/pattern/replacement/g`
- Flags: `g`, `n` (nth occurrence), `m` (multiline)
- Brace syntax: boolean flags (`{b}`, `{i}`, `{_}`, `{-}`, `{#}`, `{^}`, `{,}`, `{w}`), negation (`{!b}`)
- Brace syntax: inline scoping (`{b=text}`, `{^=text}`, `{,=text}`, etc.)
- Brace syntax: value flags (`{c=}`, `{z=}`, `{f=}`, `{s=}`, `{u=}`, `{t=}`, `{l=}`, `{a=}`, `{o=}`, `{n=}`, `{k=}`, `{x=}`, `{y=}`, `{p=}`, `{h=}`, `{e=}`)
- Brace syntax: reset (`{0}`)
- Brace syntax: implicit `t=$0` behavior
- Brace syntax: `{h}` defaults to HEADING_1, `{h=0}` for NORMAL_TEXT
- Brace syntax: named colors (RECOMMENDED) and hex colors (REQUIRED)
- Back-references: `$1`-`$9`, `$0`, `&`, `$$` escaping
- Links: `[text](url)`, `<url>`, `{u=url}`

### Level 2: Extended (REQUIRED)

- All Level 1 features
- Image insertion: `![alt](url)`, `![](url)` (standard markdown)
- Image dimensions: `{x=N}`, `{y=N}`, `{x=N y=M}`
- Image brace references: `{img=n}`, `{img=-n}`, `{img=*}`, `{img=regex}`
- Brace syntax: heading shorthands (`{h=t}`, `{h=s}`, `{h=1}`–`{h=6}`, `{h=0}`)
- Brace syntax: paragraph spacing (`{p=N}`, `{p=N,M}`)

### Level 3: Structural (REQUIRED)

- All Level 2 features
- Positional insert: `^$`, `^`, `$`
- Table brace syntax: `{T=RxC}`, `{T=RxC:header}` (creation)
- Table brace references: `{T=1}`, `{T=-1}`, `{T=*}` (reference/deletion)
- Table brace cell access: `{T=1!A1}`, `{T=1!R,C}`, wildcards (`{T=1!1,*}`, `{T=1!*,2}`, `{T=1!*}`)
- Table brace rectangular ranges: `{T=1!A1:C3}`
- Table brace row/column operations: `{T=1!row=+N}`, `{T=1!col=$+}`, etc.
- Table merge: `{T=1!A1:C3}/merge/`
- Brace syntax: breaks (`{+}`, `{+=p}`, `{+=c}`, `{+=s}`)
- Brace syntax: columns (`{cols=N}`)
- Brace syntax: checkboxes (`{check}`, `{check=n}`, `{check=y}`)
- Brace syntax: table of contents (`{toc}`, `{toc=N}`)
- Brace syntax: comments (`{"=text}`)
- Brace syntax: bookmarks (`{@=name}`, `{u=#name}`)

### Level 4: Commands & CLI (REQUIRED)

- All Level 3 features
- Delete command: `d/pattern/`
- Append command: `a/pattern/text/`
- Insert command: `i/pattern/text/`
- Transliterate command: `y/source/dest/`
- Dry-run mode: `--dry-run` / `-n`
- Batch processing: `-f file.sed`

### Level 5: Smart Chips & Charts (REQUIRED)

- All Level 4 features
- URI schemes: standard (`https:`, `mailto:`, `tel:`, `geo:`, `sms:`, `webcal:`, `file:`)
- URI schemes: `chip://` smart chips (person, date, file, place, dropdown, chart, bookmark)
- Charts: `chip://chart/` references

### Level 6: Markdown Convenience (RECOMMENDED)

- All Level 5 features
- Markdown shortcuts: `**bold**`, `*italic*`, `` `mono` ``, `~~strike~~`
- Markdown headings: `#` through `######`
- Markdown lists: `- bullet`, `1. numbered`, nested
- Markdown blocks: `---` (horizontal rule), `> quote`, ``` code blocks ```
- Footnotes: `[^text]`

---

## Implementation Status

| Feature | Status | Requirement Level |
|---------|--------|-------------------|
| Basic `s///` syntax | ✅ Stable | REQUIRED |
| Global flag `g` | ✅ Stable | REQUIRED |
| Nth occurrence flag `n` | ✅ Stable | REQUIRED |
| Multiline flag `m` | ✅ Stable | REQUIRED |
| Brace syntax: basic parsing | ✅ Stable | REQUIRED |
| Brace syntax: boolean flags (`b`, `i`, `_`, `-`, `#`, `^`, `,`, `w`) | ✅ Stable | REQUIRED |
| Brace syntax: inline scoping (`{b=text}`, `{^=text}`, etc.) | ✅ Stable | REQUIRED |
| Brace syntax: negation (`!b`) | ✅ Stable | REQUIRED |
| Brace syntax: value flags (`c=`, `z=`, `f=`, `s=`, etc.) | ✅ Stable | REQUIRED |
| Brace syntax: text flag (`t=`) with implicit `$0` | ✅ Stable | REQUIRED |
| Brace syntax: reset (`{0}`) | ✅ Stable | REQUIRED |
| Brace syntax: named colors | ✅ Stable | RECOMMENDED |
| Brace syntax: hex colors (`#RRGGBB`) | ✅ Stable | REQUIRED |
| Brace syntax: heading shorthands (`h=t`, `h=1`, `{h}` → HEADING_1) | ✅ Stable | REQUIRED |
| Brace syntax: `{h=0}` for NORMAL_TEXT | ✅ Stable | REQUIRED |
| Brace syntax: paragraph spacing (`p=N`, `p=N,M`) | ✅ Stable | REQUIRED |
| Brace syntax: breaks (`{+}`, `{+=p}`, etc.) | ✅ Stable | REQUIRED |
| Brace syntax: columns (`{cols=N}`) | ✅ Stable | REQUIRED |
| Brace syntax: checkboxes (`{check}`, `{check=n}`, `{check=y}`) | ✅ Stable | REQUIRED |
| Brace syntax: table of contents (`{toc}`, `{toc=N}`) | ✅ Stable | REQUIRED |
| Brace syntax: comments (`{"=text}`) | ✅ Stable | REQUIRED |
| Brace syntax: bookmarks (`{@=name}`) | ✅ Stable | REQUIRED |
| Brace syntax: URL/link (`u=`) | ✅ Stable | REQUIRED |
| Brace syntax: `chip://` smart chips | ✅ Stable | REQUIRED |
| Brace syntax: superscript/subscript (`{^}`, `{,}`) | ✅ Stable | REQUIRED |
| Brace syntax: table addressing (`{T=}`) | ✅ Stable | REQUIRED |
| Brace syntax: image references (`{img=}`) | ✅ Stable | REQUIRED |
| Back-references `$1`-`$9`, `$0`, `&` | ✅ Stable | REQUIRED |
| Dollar sign escaping `$$` | ✅ Stable | REQUIRED |
| Links `[text](url)`, `{u=url}` | ✅ Stable | REQUIRED |
| Auto-links `<url>` | ✅ Stable | REQUIRED |
| Image insert `![](url)`, `![alt](url)` | ✅ Stable | REQUIRED |
| Image dimensions `{x=N}`, `{x=N y=M}` | ✅ Stable | REQUIRED |
| Native regex mode | ✅ Stable | RECOMMENDED |
| Positional insert (`^$`, `^`, `$`) | ✅ Stable | REQUIRED |
| Delete command `d/pattern/` | ✅ Stable | REQUIRED |
| Append command `a/pattern/text/` | ✅ Stable | REQUIRED |
| Insert command `i/pattern/text/` | ✅ Stable | REQUIRED |
| Transliterate command `y/src/dst/` | ✅ Stable | REQUIRED |
| Table creation (`{T=RxC}`) | ✅ Stable | REQUIRED |
| Table deletion (`{T=N}`, `{T=*}`) | ✅ Stable | REQUIRED |
| Cell references (`{T=1!A1}`, `{T=1!R,C}`) | ✅ Stable | REQUIRED |
| Cell wildcards (`{T=1!1,*}`, `{T=1!*,2}`, `{T=1!*}`) | ✅ Stable | REQUIRED |
| Rectangular ranges (`{T=1!A1:C3}`) | ✅ Stable | REQUIRED |
| Row operations (`{T=1!row=+N}`, `{T=1!row=$+}`) | ✅ Stable | REQUIRED |
| Column operations (`{T=1!col=+N}`, `{T=1!col=$+}`) | ✅ Stable | REQUIRED |
| Table merge (`{T=1!r1,c1:r2,c2}/merge/`) | ✅ Stable | REQUIRED |
| Dry-run mode (`--dry-run`, `-n`) | ✅ Stable | REQUIRED |
| Batch processing (`-f file.sed`) | ✅ Stable | REQUIRED |
| Markdown formatting (`**`, `*`, `` ` ``, `~~`) | ✅ Stable | RECOMMENDED |
| Markdown headings (`#` through `######`) | ✅ Stable | RECOMMENDED |
| Markdown lists (`-`, `1.`, nested) | ✅ Stable | RECOMMENDED |
| Markdown blocks (rules, quotes, code) | ✅ Stable | RECOMMENDED |
| Footnotes `[^text]` | ✅ Stable | RECOMMENDED |
| Table cell styling | 🔮 Proposed | OPTIONAL |
| @ mentions | 🔮 Proposed | OPTIONAL |

---

## Complete Examples

### Template Processing

```bash
# Replace placeholders with content
s/{{NAME}}/Acme Corp/
s/{{DATE}}/2026-02-17/
s/{{LOGO}}/![](https:\/\/acme.com\/logo.png){x=200}/
s/{{SIGNATURE}}/{u=mailto:john@acme.com t=John Doe}/
```

### Document Structure with Brace Syntax

```bash
# Build a structured document
s/^$/{h=t t=Quarterly Report}/
a/Quarterly Report/{h=2 t=Executive Summary}/
a/Executive Summary/The company performed well this quarter./
a/performed well/{h=2 t=Financial Results}/
a/Financial Results/{T=3x3}/
s/{T=1!A1}/Metric/
s/{T=1!B1}/Q3/
s/{T=1!C1}/Q4/
s/{T=1!A2}/Revenue/
s/{T=1!B2}/$$10M/
s/{T=1!C2}/$$12M/
s/{T=1!A3}/Growth/
s/{T=1!B3}/15%/
s/{T=1!C3}/20%/
s/{T=1!1,*}/{b}/
s/$/\n{+}\n> Report generated automatically.\n[^Internal use only.]/
```

### Batch Cleanup

**cleanup.sed:**
```sed
# Header formatting
s/TITLE/{h=t}/
s/SUBTITLE/{h=s}/

# Clean up
s/DRAFT//g
d/TODO/

# Style important terms
s/([A-Z]{3,})/{b t=$1}/g

# Add header
s/^/{h=t t=Final Report}\n/
```

```bash
gog docs sed -f cleanup.sed <doc-id>
```

### Table Building

```bash
# Create and populate a pricing table
s/{{PRICING}}/{T=4x3}/
s/{T=1!A1}/{b t=Plan}/
s/{T=1!B1}/{b t=Monthly}/
s/{T=1!C1}/{b t=Annual}/
s/{T=1!1,*}/{b}/
s/{T=1!A2}/Basic/
s/{T=1!A3}/Pro/
s/{T=1!A4}/Enterprise/

# Merge header cells
s/{T=1!1,1:1,3}/merge/

# Add a row
s/{T=1!row=$+}//
s/{T=1!A5}/Custom/
```

### Smart Chip Integration

```bash
# Add person chips for team mentions
s/assigned to viz/{u=chip://person/viz@example.com t=@viz}/g
s/deadline/{u=chip://date/2026-03-15 t=March 15, 2026}/g

# Add project links
s/project doc/{u=chip://file/DOC_ID t=Project Documentation}/g

# Add status dropdown
s/STATUS/{u=chip://dropdown/Draft|Review|Done t=Draft}/g
```

### Scientific Notation with Inline Scoping

```bash
# Chemistry formulas
s/H2O/H{,=2}O/g
s/CO2/CO{,=2}/g
s/H2SO4/H{,=2}SO{,=4}/g
s/NaHCO3/NaHCO{,=3}/g

# Physics equations
s/E=mc2/E=mc{^=2}/g
s/F=ma/F=ma/g
s/v2=u2\+2as/v{^=2}=u{^=2}+2as/g

# Math expressions
s/x2\+y2=r2/x{^=2}+y{^=2}=r{^=2}/g
s/an=a1\+\(n-1\)d/a{,=n}=a{,=1}+(n-1)d/g
```

### Newsletter Layout with Columns

```bash
# Title in single column
s/^/{h=t t=Monthly Newsletter}\n/

# Switch to 2-column layout for articles
s/ARTICLES_START/{+=s cols=2 h=2 t=Feature Articles}/g

# Column break between major articles
s/ARTICLE_BREAK/{+=c}/g

# Sidebar in narrower 3-column layout
s/SIDEBAR_START/{+=s cols=3 h=3 t=Quick Updates}/g

# Back to single column for footer/contact
s/CONTACT_US/{+=s cols=1 h=2 t=Contact Us}/g
```

### Checklist Creation

```bash
# Convert markdown-style checkboxes
s/\[ \]/{check=n}/g
s/\[x\]/{check=y}/g

# Create task list from placeholders
s/TODO: (.+)/{check=n t=$1}/g
s/DONE: (.+)/{check=y t=$1}/g

# Styled checklist items
s/ACTION: (.+)/{check=n b c=red t=$1}/g
```

### Document with Table of Contents

```bash
# Insert TOC after title
s/^/{h=t t=User Guide}\n{toc=3}\n/

# Or replace placeholder
s/{{TABLE_OF_CONTENTS}}/{toc}/

# Depth-limited TOC
s/{{TOC}}/{toc=2}/   # Only h1 and h2
```

---

## Error Handling

Implementations MUST handle errors gracefully:

| Error | Behavior |
|-------|----------|
| Invalid regex | MUST report error, MUST NOT modify document |
| Unmatched delimiter | MUST report error |
| Invalid brace syntax | SHOULD report warning, MAY attempt recovery |
| Invalid image reference | SHOULD report warning, MAY skip |
| Invalid table reference | SHOULD report warning, MAY skip |
| Mismatched `y///` lengths | MUST report error |
| Circular reference | MUST detect and reject |
| Invalid `chip://` scheme | SHOULD report warning, MAY skip |

---

## Security Considerations

1. **URL Validation**: Implementations SHOULD validate URLs in link and image expressions
2. **Injection Prevention**: Pattern and replacement MUST be treated as data, not executable code
3. **Resource Limits**: Implementations SHOULD limit regex complexity to prevent ReDoS attacks
4. **Sanitization**: Implementations operating on shared documents SHOULD sanitize output
5. **Smart Chip Validation**: Implementations SHOULD validate `chip://` URIs before insertion

---

## Future Considerations

> **Note**: The following features are under consideration for future versions but are not part of SEDMAT 3.5.

### Document-Level Directives

Page-level settings via `!^!{key=value}` syntax:

```bash
!^!{margin=1in size=letter orientation=portrait}
!^!{header=Company Name}
!^!{footer=Page {{page}} of {{pages}}}
```

This feature is deferred pending implementation experience.

---

## References

- [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119) - Key words for use in RFCs
- [CommonMark Spec](https://spec.commonmark.org/) - Markdown formatting reference
- [POSIX Extended Regular Expressions](https://pubs.opengroup.org/onlinepubs/9699919799/basedefs/V1_chap09.html)
- [sed(1) man page](https://www.gnu.org/software/sed/manual/sed.html) - GNU sed reference

---

## Appendix A: Quick Reference Card

```
┌──────────────────────────────────────────────────────────────────┐
│                    SEDMAT 3.5 Quick Reference                     │
├──────────────────────────────────────────────────────────────────┤
│ COMMANDS                                                          │
│   s/pattern/replacement/[flags]  Substitute                       │
│   d/pattern/                     Delete matching paragraphs       │
│   a/pattern/text/                Append after matching paragraphs │
│   i/pattern/text/                Insert before matching paragraphs│
│   y/source/dest/                 Transliterate characters         │
├──────────────────────────────────────────────────────────────────┤
│ FLAGS                                                             │
│   g          Global (all matches)    m     Multiline mode         │
│   2-9        Nth occurrence only     (none) First match only      │
├──────────────────────────────────────────────────────────────────┤
│ BRACE SYNTAX {flags} — CANONICAL FORMATTING             ✅ STABLE │
│                                                                   │
│ BOOLEAN FLAGS (presence activates):                               │
│   {b}         Bold          {i}         Italic                    │
│   {_}         Underline     {-}         Strikethrough             │
│   {#}         Code/mono     {^}         Superscript               │
│   {,}         Subscript     {w}         Small caps                │
│   {!b}        Negate bold   {b i _}     Combine flags             │
│                                                                   │
│ CHECKBOXES (tri-state):                                           │
│   {check}     Unchecked ☐   {check=n}   Unchecked ☐               │
│   {check=y}   Checked ☑     {check=n b} Checkbox + bold           │
│                                                                   │
│ INLINE SCOPING (apply style to part of replacement):              │
│   {b=text}    Bold just "text"         {i=text}   Italic "text"   │
│   {^=text}    Superscript "text"       {,=text}   Subscript "text"│
│   H{,=2}O → H₂O    E=mc{^=2} → E=mc²   {b=Warning}: msg           │
│                                                                   │
│ VALUE FLAGS (key=value):                                          │
│   {t=text}    Replacement text (default: $0 = matched text)       │
│   {c=red}     Color         {z=yellow}  Background                │
│   {c=#FF0000} Color (hex)   {z=#FFFF00} Background (hex)          │
│   {f=Roboto}  Font          {s=14}      Size (pt)                 │
│   {u=URL}     Link          {u=#name}   Bookmark link             │
│   {h=1}       Heading 1     {h=t}       Title                     │
│   {h=s}       Subtitle      {h}         Heading 1 (default)       │
│   {h=0}       Normal text   {cols=N}    Section columns           │
│   {p=12}      Spacing 12pt  {p=12,6}    12pt above, 6pt below     │
│                                                                   │
│ NAMED COLORS:                                                     │
│   black white red green blue yellow cyan magenta                  │
│   orange purple pink brown gray/grey lightgray darkgray navy teal │
│                                                                   │
│ IMPLICIT t= RULE:                                                 │
│   {b} expands to {b t=$0} — matched text is preserved             │
│                                                                   │
│ RESET:                                                            │
│   {0}         Reset ALL     {0 b}       Reset + bold              │
│                                                                   │
│ BREAKS:                                                           │
│   {+}         Horiz rule    {+=p}       Page break                │
│   {+=c}       Column break  {+=s}       Section break             │
│                                                                   │
│ COLUMNS (structural — targets containing section):                │
│   {cols=2}    2-column section          {cols=1}   Single column  │
│   {+=s cols=2} Section break + 2 cols   {+=c}      Column break   │
│                                                                   │
│ TABLE OF CONTENTS:                                                │
│   {toc}       Full TOC      {toc=2}     TOC depth 2 (h1-h2 only)  │
│                                                                   │
│ SUPERSCRIPT/SUBSCRIPT:                                            │
│   {^}         Whole sup     {,}         Whole sub                 │
│   {^=text}    Sup inline    {,=text}    Sub inline                │
│   H{,=2}O    x{^=2}+y{^=2}  CO{,=2}    E=mc{^=2}                  │
│                                                                   │
│ COMMENTS & BOOKMARKS:                                             │
│   {"=text}    Comment       {@=name}    Bookmark anchor           │
│   {u=#name}   Link to bookmark                                    │
│                                                                   │
│ SMART CHIPS:                                                      │
│   {u=chip://person/email}   Person chip                           │
│   {u=chip://date/YYYY-MM-DD} Date chip                            │
│   {u=chip://file/ID}        File chip                             │
│   {u=chip://dropdown/a|b|c} Dropdown chip                         │
│   {u=chip://chart/ID/0}     Chart embed                           │
├──────────────────────────────────────────────────────────────────┤
│ TABLES — BRACE SYNTAX {T=}                              ✅ STABLE │
│   {T=3x4}                   Create 3-row, 4-col table             │
│   {T=3x4:header}            Create with header                    │
│   {T=1}, {T=-1}, {T=*}      Table reference / delete              │
│   {T=1!A1}  {T=1!1,2}       Cell reference                        │
│   {T=1!1,*} {T=1!*,2}       Row/column wildcard                   │
│   {T=1!*}                   Entire table                          │
│   {T=1!A1:C3}               Rectangular range                     │
│   {T=1!row=+2} {T=1!row=$+} Insert/append row                     │
│   {T=1!col=+2} {T=1!col=$+} Insert/append column                  │
│   {T=1!A1:C3}/merge/        Merge cell range                      │
├──────────────────────────────────────────────────────────────────┤
│ IMAGES                                                  ✅ STABLE │
│   ![alt](url)               Insert image (standard markdown)      │
│   ![](url)                  Insert (no alt text)                  │
│   {x=N} or {y=N}            Set dimensions                        │
│   {img=1}, {img=-1}         Reference by position                 │
│   {img=*}                   All images                            │
│   {img=regex}               Match by alt text                     │
├──────────────────────────────────────────────────────────────────┤
│ MARKDOWN SHORTCUTS (convenience layer)                            │
│   **text**    Bold          *text*      Italic                    │
│   ~~text~~    Strike        `text`      Monospace                 │
│   ***text***  Bold+Italic                                         │
│   # text      Heading 1     ## text     Heading 2, etc.           │
│   - item      Bullet list   1. item     Numbered list             │
│   ---         Horiz rule    > text      Blockquote                │
│   ```code```  Code block    [^text]     Footnote                  │
├──────────────────────────────────────────────────────────────────┤
│ BACK-REFERENCES                                                   │
│   $0 or &     Entire match              $1-$9       Capture groups│
│   $$          Literal $                 \&          Literal &     │
├──────────────────────────────────────────────────────────────────┤
│ LINKS                                                             │
│   [text](url)               Hyperlink                             │
│   {u=url}                   Brace syntax link                     │
├──────────────────────────────────────────────────────────────────┤
│ POSITIONAL INSERT                                                 │
│   s/^$/text/                Insert into empty document            │
│   s/^/text/                 Prepend to document                   │
│   s/$/text/                 Append to document                    │
├──────────────────────────────────────────────────────────────────┤
│ CLI OPTIONS                                                       │
│   --dry-run / -n            Preview changes (no modify)           │
│   -f file.sed               Batch expressions from file           │
└──────────────────────────────────────────────────────────────────┘
```

---

## Changelog

### v3.5 (2026-02-18)

**New Features:**

- **Checkboxes (`{check}`)**: Native checklist support with tri-state semantics
  - `{check}` or `{check=n}` — unchecked checkbox
  - `{check=y}` — checked checkbox
  - Can combine with other flags: `{check=n b}`
- **Table of Contents (`{toc}`)**: Auto-generated TOC from document headings
  - `{toc}` — full table of contents
  - `{toc=N}` — TOC with max depth N
  - Implementations SHOULD auto-link entries to headings
- **Named Colors**: Standard 18-color palette for `c=` and `z=` flags
  - black, white, red, green, blue, yellow, cyan, magenta
  - orange, purple, pink, brown, gray/grey, lightgray, darkgray, navy, teal
  - Hex (`#RRGGBB`) REQUIRED; named colors RECOMMENDED
- **Paragraph Spacing (`{p=}`)**: Clarified spacing syntax
  - `{p=12}` — 12pt above and below
  - `{p=12,6}` — 12pt above, 6pt below
  - `{p}` bare — reset to default

**Documentation:**

- Added sections: Checkboxes, Table of Contents, Named Colors, Paragraph Spacing
- Trimmed Columns section from ~112 to ~60 lines
- Consolidated changelog (v3.0–v3.4 merged into v3.5)
- Updated ABNF with `check-flag`, `toc-flag`, `color-value`, `spacing-value`
- Updated Conformance Levels, Implementation Status, Quick Reference Card

**Includes all features from v3.0–v3.4:**

- Brace Syntax as canonical formatting system (v3.0)
- Unified `{T=}` table addressing and `{img=}` image references (v3.1)
- Inline scoping for boolean flags (`{b=text}`, `{^=text}`, `{,=text}`) (v3.3)
- Column support (`{cols=N}`) as structural flag (v3.4)
- Structural vs text flag distinction
- `chip://` smart chips for Google Docs
- All extended commands (`d/`, `a/`, `i/`, `y/`)

### v2.0 (2026-02-17)

- **New commands**: `d/pattern/` (delete), `a/pattern/text/` (append), `i/pattern/text/` (insert), `y/source/dest/` (transliterate)
- **New flags**: `n` (nth occurrence, e.g. `s/foo/bar/2`), `m` (multiline mode)
- **Heading styles**: `#` through `######` → native Heading 1–6
- **Paragraph styles**: `- bullet`, `1. numbered`, nested lists with indentation
- **Horizontal rules**: `---`, `***`, `___` → grey bottom border
- **Blockquotes**: `> text` → left-indented with grey left border
- **Code blocks**: Triple backtick fences → Courier New + grey background
- **Superscript/subscript**: `^{text}`, `~{text}`
- **Footnotes**: `[^text]` → native document footnotes
- **Pipe tables**: Markdown pipe syntax parsed into native tables
- **Table merge**: `s/|N|[r1,c1:r2,c2]/merge/` for cell merging
- **Dollar sign escaping**: `$$` → literal `$` in replacements
- **Whole-match backreference**: `&` for entire match, `\&` for literal ampersand
- **Dry-run mode**: `--dry-run` / `-n` flag for previewing changes
- **Batch processing**: `-f file.sed` for reading expressions from files
- Reference implementation: [gogcli](https://github.com/steipete/gogcli) `gog docs sed`

### v1.1 (2026-02-09)

- **Positional insert**: Added `^$`, `^`, `$` patterns for document-level insertion
- **Table creation**: Promoted to STABLE — `|RxC|` and `|RxC:header|` syntax
- **Table deletion**: Added `s/|N|//`, `s/|-N|//`, `s/|*|//`
- **Cell wildcards**: Added `[1,*]`, `[*,2]`, `[*,*]`
- **Row operations**: Added `[row:N]`, `[row:+N]`, `[row:$+]`
- **Column operations**: Added `[col:N]`, `[col:+N]`, `[col:$+]`
- Reference implementation: [gogcli](https://github.com/steipete/gogcli) `gog docs sed`

### v1.0 (2026-02-07)

- Initial specification
- Core syntax defined
- Text formatting standardized
- Image operations specified
- Table operations proposed
- Conformance levels established
