# Extract Book Chapter to Markdown Study Notes

Extract a chapter from a PDF book and create comprehensive markdown study notes.

## Usage
`/extract-chapter <book_name> <chapter_number> <output_filename>`

Examples:
- `/extract-chapter "Geometry for Programmers" 7 Geometry_07.md`
- `/extract-chapter "Computer Graphics from Scratch" 3 CGScratch_03.md`
- `/extract-chapter Geometry 8 Geometry_08.md`

### Arguments
1. **book_name** (required): The name (or partial name) of the PDF book to search for in the current working directory. Matches against PDF filenames.
2. **chapter_number** (required): The chapter number to extract.
3. **output_filename** (required): The name of the output markdown file (e.g., `Geometry_07.md`).

If any arguments are missing, ask the user to provide them.

## Instructions

You are creating comprehensive study notes from a book PDF. Follow these steps precisely:

### Step 1: Identify the PDF and Chapter

- Search the current working directory for a PDF file matching the given `book_name`. Use Glob with `**/*.pdf` and match against the `book_name` argument (case-insensitive partial match).
- If multiple PDFs match, ask the user to clarify which one.
- If no PDF matches, inform the user and list available PDFs.
- The output file will be saved in the current working directory with the given `output_filename`.

### Step 2: Determine Page Offset

- Read the first ~10 pages of the PDF to find the table of contents (TOC).
- From the TOC, identify the page range for the requested chapter (start page = chapter start, end page = next chapter start - 1).
- Determine the **page offset** by comparing a known book page number visible in the PDF content with the actual PDF page number. For example, if book page 157 appears on PDF page 181, the offset is +24.
- If existing markdown files for this book are available (e.g., `Geometry_01.md`), check their source line for page references to help calibrate the offset.

### Step 3: Read All Chapter Pages

- Calculate the PDF page range using the offset: `pdf_page = book_page + offset`
- Read the chapter pages from the PDF in chunks of 20 pages (the maximum per read).
- Make sure to read ALL pages of the chapter, including exercises and solutions.

### Step 4: Create the Markdown File

Check if existing markdown files for this book exist in the directory (e.g., `Geometry_01.md`). If so, read the first 50 lines to match the established format. If no prior files exist, use the default format below.

#### Format Rules:

1. **Header**: `# Chapter N: [Chapter Title]`
2. **Source line**: `> **Source**: *[Book Title]* ([Author], [Publisher], [Year]), pp. [start]–[end]`
3. **Prerequisites**: `> **Prerequisites**: Chapter [N-1] ([title])` (if applicable)
4. **"This Chapter Covers"** section with bullet points
5. **Overview** section summarizing the chapter intro
6. **All sections and subsections** using `##` for main sections (e.g., `## 6.1`), `###` for subsections (e.g., `### 6.1.1`)
7. **Section summaries** preserved as written in the book
8. **Mathematical formulas** in LaTeX: inline `$...$`, display `$$...$$`
9. **Code examples** in fenced code blocks with language identifier
10. **Listing references** preserved (e.g., `> **Listing 6.1** ...`)
11. **Figure descriptions** as blockquotes: `> **Figure N.N** [description]`
12. **Notes and sidebars** as blockquotes with bold labels
13. **SEE ALSO** references preserved
14. **Tables** in markdown table format
15. **Exercises** section with all exercises
16. **Solutions to exercises** section with all solutions
17. **Summary** section with all bullet points
18. **References and Further Reading** section with all URLs and citations

#### Content Rules:

- Paraphrase explanatory text in your own words while preserving technical accuracy
- **All mathematical formulas must be reproduced exactly** (math is not copyrightable). This includes ALL of the following — missing any of these is a common failure mode:
  - **Core formulas and derivations** (the obvious ones)
  - **Numerical value assignments** displayed as equations (e.g., `$z_{11} = 0.3;\; z_{12} = 0.5$`). These are NOT expendable example data — they are displayed equations that readers need to reproduce figures and verify results
  - **Intermediate/transitional formulas** that bridge between a general form and its decomposition (e.g., if a book shows $f: (x,y) \to (x_t, y_t, z_t)$ then decomposes into $f_x$, $f_y$, $f_z$, include BOTH the combined and decomposed forms)
  - **Simplified/specialized versions** of previously shown formulas (e.g., the $n=1$ special case of a general formula with power $n$). These are NOT redundant — they are the forms actually used in practice
  - **Mathematical mapping tables** that define coordinate-to-variable correspondences (e.g., a grid showing which $(x,y)$ maps to which $z_{ij}$)
- **All code examples must be included** (functional code for educational use)
- **All figure descriptions** should describe what the figure shows
- Preserve the pedagogical flow and logical structure
- Keep definitions, theorems, and key insights clearly marked
- Include all cross-references to other chapters

### Step 5: Verify

After creating the file, read the first and last 20 lines to verify the format matches existing files (if any).

## Example Output Structure

```markdown
# Chapter N: [Title]

> **Source**: *[Book Title]* ([Author], [Publisher], [Year]), pp. X–Y
> **Prerequisites**: Chapter N-1 ([title])

---

## This Chapter Covers
- ...

---

## Overview
...

---

## N.1 [Section Title]
...

### N.1.1 [Subsection Title]
...

---

## Summary
- ...

---

## References and Further Reading
...
```

## Known Books and Their Offsets

When working with these books, use the pre-calculated offsets to save time:

| Book | PDF filename | Page offset |
|---|---|---|
| Geometry for Programmers | `(2023) Geometry for Programmers (Oleksandr Kaleniuk).pdf` | +24 |

If the book is not in this table, calculate the offset in Step 2.
