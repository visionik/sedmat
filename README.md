# SEDMAT: Sed-Expression-Driven Markdown Annotation & Transformation

**Version**: 1.0  
**Last Updated**: 2026-02-07  
**Status**: Draft

## Abstract

SEDMAT (Sed-Expression-Driven Markdown Annotation & Transformation) is a specification for document manipulation using sed-inspired syntax combined with Markdown-style formatting. It provides a portable, human-readable DSL for batch text transformations, formatting operations, and structural modifications.

## Introduction

SEDMAT enables:

- **Familiar Syntax**: sed-style `s/pattern/replacement/flags` expressions
- **Rich Formatting**: Markdown-compatible styling (`**bold**`, `*italic*`, `[links](url)`)
- **Structural Operations**: Images, tables, and object references
- **Regex Power**: Full Extended Regular Expression (ERE) support with back-references
- **Batch Processing**: Transform multiple elements in a single expression

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

### With Formatting

```bash
# Make "warning" bold
s/warning/**warning**/

# Wrap in link
s/Google/[Google](https://google.com)/

# Insert image at placeholder
s/{{LOGO}}/![](https://example.com/logo.png)/
```

## Core Concepts

### Expression Anatomy

```mermaid
graph LR
    subgraph "SEDMAT Expression"
        CMD[s] --> SEP1["/"]
        SEP1 --> PAT["pattern"]
        PAT --> SEP2["/"]
        SEP2 --> REP["replacement"]
        REP --> SEP3["/"]
        SEP3 --> FLAGS["flags"]
    end
    
    style CMD fill:#e1f5e1
    style PAT fill:#fff4e1
    style REP fill:#e1e8f5
    style FLAGS fill:#f5e1e8
```

### Processing Model

```mermaid
flowchart TB
    INPUT[Input Document] --> PARSE[Parse Expression]
    PARSE --> HASFORMAT{Contains<br/>Formatting?}
    HASFORMAT -->|No| NATIVE[Native Regex API<br/>Fast Path]
    HASFORMAT -->|Yes| WALK[Document Walk<br/>Full Processing]
    NATIVE --> OUTPUT[Output Document]
    WALK --> OUTPUT
    
    style NATIVE fill:#90ee90
    style WALK fill:#fff4e1
```

SEDMAT processors SHOULD detect plain-text replacements and use native regex APIs for optimal performance.

## Syntax Reference

### Flags

Flag | Keyword | Meaning
--- | --- | ---
`g` | Global | Replace ALL matches
*(none)* | - | Replace first match only

**Conformance**: Implementations MUST support both global and single-match modes.

### Delimiters

The delimiter `/` MAY be replaced with any consistent character not appearing in the pattern or replacement:

```bash
# Standard
s/path/to/file/replacement/

# Alternate delimiter (when pattern contains /)
s#path/to/file#replacement#
s|path/to/file|replacement|
```

Implementations MUST support at minimum `/`, `#`, and `|` as delimiters.

## Text Formatting

### Inline Styles

Syntax | Effect | Status
--- | --- | ---
`**text**` | Bold | REQUIRED
`*text*` | Italic | REQUIRED
`_text_` | Italic | REQUIRED
`***text***` | Bold + Italic | REQUIRED
`~~text~~` | Strikethrough | REQUIRED
`` `text` `` | Monospace | REQUIRED
`__text__` | Underline | RECOMMENDED

**Examples:**

```bash
# Bold a keyword
s/CRITICAL/**CRITICAL**/g

# Italicize quoted text
s/"([^"]+)"/*$1*/g

# Strikethrough deprecated terms
s/deprecated/~~deprecated~~/g

# Code formatting for inline code
s/console\.log/`console.log`/g
```

### Combining Formats

Formats MAY be nested according to Markdown precedence:

```bash
# Bold AND italic
s/warning/***warning***/

# Bold and underline
s/critical/**__critical__**/

# All three
s/urgent/***__urgent__***/
```

**Processing Order:**

```mermaid
graph LR
    TEXT[text] --> U["__underline__"]
    U --> B["**bold**"]
    B --> I["*italic*"]
    I --> FINAL["***__text__***"]
    
    style TEXT fill:#f0f0f0
    style FINAL fill:#e1f5e1
```

Implementations MUST process nesting from innermost to outermost.

### Back-References

SEDMAT MUST support capture groups using `(...)` with references via `$1`-`$9` or `\1`-`\9`:

```bash
# Capture and format
s/(important)/**$1**/g

# Bold all-caps words (2+ chars)
s/([A-Z]{2,})/**$1**/g

# Wrap names in links
s/(@\w+)/[$1](https://twitter.com/$1)/g
```

Reference | Alternative | Meaning
--- | --- | ---
`$0` | `&` | Entire match
`$1` | `\1` | First capture group
`$2` | `\2` | Second capture group
`$n` | `\n` | Nth capture group (1-9)

**Conformance**: Implementations MUST support `$1`-`$9` notation. Support for `\1`-`\9` is RECOMMENDED for sed compatibility.

## Links

### Link Syntax

Syntax | Effect | Status
--- | --- | ---
`[text](url)` | Hyperlink with text | REQUIRED
`<url>` | Auto-link (bare URL) | REQUIRED
`[text](url "title")` | Link with title attribute | RECOMMENDED

**Examples:**

```bash
# Add hyperlink to text
s/Google/[Google](https:\/\/google.com)/

# Link email addresses
s/(\S+@\S+)/[$1](mailto:$1)/g

# Auto-link bare URLs
s/(https:\/\/\S+)/<$1>/g

# Link with title
s/RFC 2119/[RFC 2119](https:\/\/datatracker.ietf.org\/doc\/html\/rfc2119 "IETF RFC 2119")/
```

### Link Processing Flow

```mermaid
flowchart LR
    MATCH[Match Text] --> HASURL{URL in<br/>Replacement?}
    HASURL -->|"[text](url)"| HYPER[Create Hyperlink]
    HASURL -->|"<url>"| AUTO[Create Auto-link]
    HASURL -->|No| PLAIN[Plain Text Replace]
    
    HYPER --> OUT[Output]
    AUTO --> OUT
    PLAIN --> OUT
```

## Images

### Image Insertion

Syntax | Effect | Status
--- | --- | ---
`![](url)` | Image, no alt text | REQUIRED
`!(url)` | Image, shorthand | REQUIRED
`![alt](url)` | Image with alt text | REQUIRED
`![alt](url "caption")` | Image with caption | RECOMMENDED
`![](url){width=N}` | Width in pixels | REQUIRED
`![](url){height=N}` | Height in pixels | REQUIRED
`![](url){w=N h=M}` | Both dimensions | REQUIRED

**Examples:**

```bash
# Insert at placeholder
s/{{LOGO}}/![](https:\/\/example.com\/logo.png)/

# With alt text
s/{{LOGO}}/![Company Logo](https:\/\/example.com\/logo.png)/

# With dimensions
s/{{HERO}}/![](https:\/\/example.com\/hero.jpg){width=600}/

# With all attributes
s/{{BANNER}}/![Welcome Banner](https:\/\/example.com\/banner.png){w=800 h=200}/
```

### Image References

Existing images MAY be referenced by position or alt text:

Pattern | Meaning | Status
--- | --- | ---
`!(1)` | First image | REQUIRED
`!(2)` | Second image | REQUIRED
`!(-1)` | Last image | REQUIRED
`!(-2)` | Second to last | REQUIRED
`!(*)` | All images | REQUIRED
`![regex]` | Images matching alt text | REQUIRED

**Reference Resolution:**

```mermaid
graph TB
    REF["Image Reference"] --> TYPE{Reference<br/>Type?}
    TYPE -->|"!(n)"| POS[Position-based<br/>1-indexed]
    TYPE -->|"!(-n)"| NEG[Negative index<br/>From end]
    TYPE -->|"!(*)"| ALL[All images]
    TYPE -->|"![regex]"| ALT[Alt-text<br/>regex match]
    
    POS --> RESOLVE[Resolve to Image]
    NEG --> RESOLVE
    ALL --> MULTI[Multi-image<br/>Operation]
    ALT --> RESOLVE
```

**Examples:**

```bash
# Replace first image
s/!(1)/!(https:\/\/new-image.png)/

# Replace last image
s/!(-1)/![New Logo](https:\/\/new.png)/

# Replace all images (global required)
s/!(*)/!(https:\/\/placeholder.png)/g

# Delete first image
s/!(1)//

# Delete all images
s/(*)//g

# Replace by alt text (exact)
s/![old-logo]/![new-logo](https:\/\/new.png)/

# Replace by alt text (regex - starts with "fig-")
s/![fig-.*]/!(https:\/\/placeholder.png)/g

# Case-insensitive alt match
s/![(?i)logo]/!(https:\/\/new-logo.png)/
```

### Image Dimensions Grammar

```
dimensions     = "{" dimension-list "}"
dimension-list = dimension ( " " dimension )*
dimension      = width | height
width          = ("width" | "w") "=" number
height         = ("height" | "h") "=" number
number         = DIGIT+
```

## Tables

**Status**: PROPOSED (not yet standardized)

### Table References

Pattern | Meaning
--- | ---
&#124;1&#124; | First table
&#124;2&#124; | Second table
&#124;-1&#124; | Last table
&#124;*&#124; | All tables

### Table Creation

```bash
# Create 3×4 table (3 rows, 4 columns)
s/{{TABLE}}/|3x4|/

# With header row pinning
s/{{TABLE}}/|3x4:header|/
```

### Cell References

```mermaid
graph TB
    subgraph "Cell Reference Syntax"
        A["[A1]<br/>Excel-style"] 
        B["[1,1]<br/>Row,Col (1-indexed)"]
        C["[0,0]<br/>Row,Col (0-indexed)"]
        D["[A1:C3]<br/>Range"]
        E["[1,*]<br/>Entire row"]
        F["[*,2]<br/>Entire column"]
        G["[*,*]<br/>All cells"]
    end
```

Pattern | Meaning
--- | ---
&#124;1&#124;[A1] | Cell A1 (Excel-style)
&#124;1&#124;[1,1] | Row 1, Col 1 (1-indexed)
&#124;1&#124;[A1:C3] | Range A1 to C3
&#124;1&#124;[1,*] | Entire row 1
&#124;1&#124;[*,2] | Entire column 2
&#124;1&#124;[*,*] | All cells

**Examples:**

```bash
# Set cell content
s/|1|[A1]/Name/
s/|1|[B1]/Value/

# Bold existing cell content (& = matched content)
s/|1|[A1]/**&**/

# Format entire header row
s/|1|[1,*]/**&**/g

# Find/replace within table
s/|1|[*,*]:TODO/DONE/g
```

### Row and Column Operations

```bash
# Delete row 2
s/|1|[row:2]//

# Insert row before row 2
s/|1|[row:+2]//

# Append row at end
s/|1|[row:$+]//

# Delete column B
s/|1|[col:B]//

# Set column width (points)
s/|1|[col:A]/width=100/
```

### Cell Styling

```bash
# Background color
s/|1|[A1]/bg=#ff0000/

# Header row background
s/|1|[1,*]/bg=#eeeeee/

# Alternating rows
s/|1|[odd,*]/bg=#f5f5f5/

# Combined: set content + style
s/|1|[A1]/{bg=#ffff00}Important/
```

## People / Mentions

**Status**: PROPOSED

Syntax | Effect
--- | ---
`@email` | @ mention person

```bash
# Insert mention
s/TODO/@john@example.com/

# Replace name with mention
s/John Smith/@john.smith@company.com/
```

## Processing Modes

### Native Mode (Fast Path)

When replacement contains **no formatting markers**, implementations SHOULD use native regex APIs:

```bash
# Native mode (no formatting):
s/colour/color/g          # Plain text
s/\bfoo\b/bar/g           # Word boundary
s/2023/2024/g             # Numeric
```

### Document Walk Mode

Required when formatting is present:

```bash
# Document walk mode (formatting needed):
s/foo/**foo**/g                        # Bold
s/bar/[bar](https:\/\/bar.com)/g       # Link
s/{{IMG}}/!(https:\/\/img.png)/        # Image
```

```mermaid
flowchart TB
    EXPR[Expression] --> ANALYZE{Analyze<br/>Replacement}
    ANALYZE -->|Plain text only| NATIVE["Native Regex<br/>(Fast)"]
    ANALYZE -->|Contains **,*,~~,etc| WALK["Document Walk<br/>(Full features)"]
    ANALYZE -->|Contains [](url)| WALK
    ANALYZE -->|Contains !()| WALK
    ANALYZE -->|Contains \|table\|| WALK
    
    NATIVE --> APPLY[Apply Changes]
    WALK --> APPLY
    
    style NATIVE fill:#90ee90
    style WALK fill:#fff4e1
```

## Escaping

### Special Characters

Characters with special meaning MUST be escaped with backslash:

Character | Escape | Context
--- | --- | ---
`/` | `\/` | When used as delimiter
`*` | `\*` | Literal asterisk
`[` | `\[` | Literal bracket
`]` | `\]` | Literal bracket
`(` | `\(` | Literal paren (in replacement)
`)` | `\)` | Literal paren (in replacement)
`$` | `\$` | Literal dollar sign
`\` | `\\` | Literal backslash

**Example:**

```bash
# Match literal asterisks
s/\*\*bold\*\*/**bold**/

# URL with slashes (use alternate delimiter)
s#http://old.com#https://new.com#g
```

## Grammar

### ABNF Notation

```abnf
sedmat-expr    = "s" delimiter pattern delimiter replacement delimiter [flags]

delimiter      = "/" / "#" / "|" / %x21-2E / %x3A-40 / %x5B-60 / %x7B-7E
                 ; Any printable ASCII except alphanumeric

pattern        = regex-pattern

replacement    = *( plain-text / format-expr / back-ref / image-expr / link-expr / table-expr )

flags          = *( "g" )

format-expr    = bold / italic / bold-italic / strike / mono / underline

bold           = "**" content "**"
italic         = "*" content "*" / "_" content "_"
bold-italic    = "***" content "***"
strike         = "~~" content "~~"
mono           = "`" content "`"
underline      = "__" content "__"

back-ref       = "$" DIGIT / "\" DIGIT / "&"

link-expr      = "[" link-text "](" url ["\"" title "\""] ")"
               / "<" url ">"

image-expr     = image-insert / image-ref
image-insert   = "![" [alt-text] "](" url ")" [dimensions]
               / "!(" url ")" [dimensions]
image-ref      = "!(" index ")" / "![" regex "]"

dimensions     = "{" *( dim-spec SP ) "}"
dim-spec       = ("width" / "w") "=" NUMBER
               / ("height" / "h") "=" NUMBER

table-ref      = "|" index "|"
cell-ref       = table-ref "[" cell-spec "]"
```

## Implementation Status

Feature | Status | Requirement Level
--- | --- | ---
Basic `s///` syntax | ✅ Stable | REQUIRED
Global flag `g` | ✅ Stable | REQUIRED
Text formatting | ✅ Stable | REQUIRED
Back-references | ✅ Stable | REQUIRED
Links `[text](url)` | ✅ Stable | REQUIRED
Auto-links `<url>` | ✅ Stable | REQUIRED
Image insert `![](url)` | ✅ Stable | REQUIRED
Image shorthand `!(url)` | ✅ Stable | REQUIRED
Image dimensions | ✅ Stable | REQUIRED
Image reference `!(n)` | ✅ Stable | REQUIRED
Image alt regex `![regex]` | ✅ Stable | REQUIRED
Native regex mode | ✅ Stable | RECOMMENDED
Table creation | 🔮 Proposed | OPTIONAL
Table cell operations | 🔮 Proposed | OPTIONAL
Table row/column ops | 🔮 Proposed | OPTIONAL
Table styling | 🔮 Proposed | OPTIONAL
@ mentions | 🔮 Proposed | OPTIONAL

## Complete Examples

### Template Processing

```bash
# Replace placeholders with content
s/{{NAME}}/Acme Corp/
s/{{DATE}}/2026-02-07/
s/{{LOGO}}/![](https:\/\/acme.com\/logo.png){width=200}/
s/{{SIGNATURE}}/[John Doe](mailto:john@acme.com)/
```

### Formatting Cleanup

```bash
# Remove double spaces
s/  +/ /g

# Smart quotes
s/"([^"]+)"/"$1"/g

# Bold section headers (lines starting with #)
s/^# (.+)$/**$1**/g
```

### Link Management

```bash
# Convert plain URLs to links
s/(https:\/\/\S+)/[$1]($1)/g

# Update domain in all links
s/\[([^\]]+)\]\(https:\/\/old\.com/[$1](https:\/\/new.com/g

# Add tracking parameter to links
s/\]\(https:\/\/([^)]+)\)/](https:\/\/$1?utm_source=doc)/g
```

### Image Management

```bash
# List then replace specific images
s/!(1)/!(https:\/\/new-hero.jpg)/
s/![header.*]/!(https:\/\/new-header.png)/g

# Resize all images to max 600px
s/!(*){width=\d+}/!(*){width=600}/g
```

### Table Building

```bash
# Create and populate a pricing table
s/{{PRICING}}/|4x3|/
s/|1|[A1]/**Plan**/
s/|1|[B1]/**Monthly**/
s/|1|[C1]/**Annual**/
s/|1|[1,*]/bg=#4a86e8/
s/|1|[A2]/Basic/
s/|1|[A3]/Pro/
s/|1|[A4]/Enterprise/
```

### Batch Operations Pipeline

```mermaid
flowchart LR
    DOC[Original<br/>Document] --> E1["s/{{NAME}}/Acme/"]
    E1 --> E2["s/DRAFT/**DRAFT**/g"]
    E2 --> E3["s/!(1)/!(logo.png)/"]
    E3 --> E4["s/(https://\\S+)/<$1>/g"]
    E4 --> OUT[Transformed<br/>Document]
    
    style DOC fill:#f0f0f0
    style OUT fill:#e1f5e1
```

## Error Handling

Implementations MUST handle errors gracefully:

Error | Behavior
--- | ---
Invalid regex | MUST report error, MUST NOT modify document
Unmatched delimiter | MUST report error
Invalid image reference | SHOULD report warning, MAY skip
Invalid table reference | SHOULD report warning, MAY skip
Circular reference | MUST detect and reject

## Security Considerations

1. **URL Validation**: Implementations SHOULD validate URLs in link and image expressions
2. **Injection Prevention**: Pattern and replacement MUST be treated as data, not executable code
3. **Resource Limits**: Implementations SHOULD limit regex complexity to prevent ReDoS attacks
4. **Sanitization**: Implementations operating on shared documents SHOULD sanitize output

## Conformance Levels

### Level 1: Core (REQUIRED)

- Basic `s/pattern/replacement/` and `s/pattern/replacement/g`
- Text formatting: `**bold**`, `*italic*`, `` `mono` ``, `~~strike~~`
- Back-references: `$1`-`$9`
- Links: `[text](url)`, `<url>`

### Level 2: Extended (REQUIRED)

- All Level 1 features
- Image insertion: `![alt](url)`, `!(url)`
- Image dimensions: `{width=N}`, `{height=N}`
- Image references: `!(n)`, `!(-n)`, `!(*)`
- Alt-text matching: `![regex]`

### Level 3: Advanced (OPTIONAL)

- All Level 2 features
- Table operations
- Cell references and styling
- @ mentions
- Extended flags

## References

- [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119) - Key words for use in RFCs
- [CommonMark Spec](https://spec.commonmark.org/) - Markdown formatting reference
- [POSIX Extended Regular Expressions](https://pubs.opengroup.org/onlinepubs/9699919799/basedefs/V1_chap09.html)
- [sed(1) man page](https://www.gnu.org/software/sed/manual/sed.html) - GNU sed reference

## Appendix A: Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────┐
│                     SEDMAT Quick Reference                       │
├─────────────────────────────────────────────────────────────────┤
│ BASIC SYNTAX                                                     │
│   s/pattern/replacement/     Replace first match                │
│   s/pattern/replacement/g    Replace all matches                │
├─────────────────────────────────────────────────────────────────┤
│ FORMATTING                                                       │
│   **text**    Bold          *text*     Italic                   │
│   ~~text~~    Strike        `text`     Monospace                │
│   __text__    Underline     ***text*** Bold+Italic              │
├─────────────────────────────────────────────────────────────────┤
│ BACK-REFERENCES                                                  │
│   $1-$9       Capture groups    &       Entire match            │
├─────────────────────────────────────────────────────────────────┤
│ LINKS                                                            │
│   [text](url)               Hyperlink                           │
│   <url>                     Auto-link                           │
├─────────────────────────────────────────────────────────────────┤
│ IMAGES                                                           │
│   ![alt](url)               Insert image                        │
│   !(url)                    Insert (shorthand)                  │
│   {width=N}                 Set width                           │
│   !(1), !(-1), !(*)         Reference by position               │
│   ![regex]                  Reference by alt text               │
├─────────────────────────────────────────────────────────────────┤
│ TABLES (Proposed)                                                │
│   |3x4|                     Create 3-row, 4-col table           │
│   |1|[A1]                   Cell reference                      │
│   |1|[1,*]                  Row reference                       │
│   |1|[*,1]                  Column reference                    │
└─────────────────────────────────────────────────────────────────┘
```

## Changelog

### v1.0 (2026-02-07)
- Initial specification
- Core syntax defined
- Text formatting standardized
- Image operations specified
- Table operations proposed
- Conformance levels established
