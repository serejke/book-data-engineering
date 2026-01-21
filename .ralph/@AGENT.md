# Agent Build Instructions: Data Engineering Book

## Project Overview
This is a technical book project, not a software project. The "build" process produces Markdown chapters that will be compiled into a final book.

## Project Structure
```
data-engineering/
├── src/
│   ├── chapters/           # Main chapter content
│   │   ├── 01-landscape/
│   │   │   └── chapter.md
│   │   ├── 02-first-principles/
│   │   │   └── chapter.md
│   │   └── ...
│   ├── appendices/         # Reference material
│   │   ├── a-comparison-tables.md
│   │   ├── b-decision-flowcharts.md
│   │   ├── c-glossary.md
│   │   └── d-further-reading.md
│   └── front-matter/       # Book introduction
│       ├── preface.md
│       └── introduction.md
├── .ralph/
│   ├── specs/              # Book specifications
│   │   ├── requirements.md
│   │   ├── chapter-outline.md
│   │   └── writing-guidelines.md
│   ├── @fix_plan.md        # Chapter progress tracking
│   ├── @AGENT.md           # This file
│   └── PROMPT.md           # Ralph instructions
└── README.md
```

## Quality Validation

### Word Count Check
```bash
# Count words in a chapter
wc -w src/chapters/*/chapter.md

# Total word count across all chapters
find src/chapters -name "chapter.md" -exec cat {} + | wc -w
```

### Markdown Linting (Optional)
```bash
# If markdownlint is installed
npx markdownlint src/chapters/**/*.md

# Or use mdl
mdl src/chapters/
```

### Link Checking (Optional)
```bash
# Check for broken internal references
grep -rn "See Chapter" src/chapters/ | grep -v "Chapter [0-9]"
```

## Chapter Writing Process

### 1. Start a New Chapter
```bash
# Create chapter directory
mkdir -p src/chapters/XX-chapter-name

# Create chapter file with frontmatter template
cat > src/chapters/XX-chapter-name/chapter.md << 'EOF'
---
chapter: X
title: "Chapter Title Here"
estimated_pages: XX-XX
status: draft
last_updated: YYYY-MM-DD
---

# Chapter Title Here

[Chapter content begins here]
EOF
```

### 2. Research Phase
- Use WebSearch to verify current state of technologies
- Check official documentation for latest features
- Review authoritative engineering blogs (Netflix, Uber, etc.)
- Note version numbers and dates for time-sensitive information

### 3. Writing Phase
- Follow structure in writing-guidelines.md
- Write section by section
- Add diagrams as Mermaid code blocks
- Update glossary with new terms

### 4. Review Phase
Run through writing-guidelines.md checklist:
- [ ] Opens with engaging hook
- [ ] Historical context explains "why"
- [ ] Core concepts from first principles
- [ ] Mental models explicit
- [ ] Trade-offs articulated
- [ ] Decision framework included
- [ ] Cross-references accurate
- [ ] Summary captures key points

## Git Workflow for Book

### Commit Conventions
```bash
# Chapter work
git commit -m "chapter(01): Complete Data Engineering Landscape"
git commit -m "chapter(05): Add Iceberg deep dive section"

# Appendix work
git commit -m "appendix(c): Add table format comparison"

# Research/planning
git commit -m "docs: Update chapter outline with case studies"

# Fixes/edits
git commit -m "edit(03): Fix consistency model explanation"
```

### Branch Strategy (if desired)
```bash
# Work on one chapter at a time
git checkout -b chapter/05-table-formats
# ... write chapter ...
git add .
git commit -m "chapter(05): Complete Table Formats chapter"
git checkout main
git merge chapter/05-table-formats
```

## Key Learnings

### Content Quality
- First principles before implementation details
- Always explain WHY before HOW
- Historical context makes technologies memorable
- Decision frameworks > prescriptive recommendations

### Writing Efficiency
- Research phase should be time-boxed (don't go down rabbit holes)
- Write section-by-section, don't try to write entire chapter at once
- Keep glossary updated as you write (don't defer)
- Cross-references can be placeholder `[See Chapter X]` until chapter exists

### Technical Accuracy
- Technologies evolve quickly - note versions and dates
- Mark what's fundamental vs what's current-state
- When in doubt, reference official documentation
- Engineering blogs provide real-world validation

## Chapter Completion Checklist

Before marking any chapter complete in @fix_plan.md:

- [ ] All sections from chapter-outline.md are written
- [ ] Word count appropriate for estimated pages (~300 words/page)
- [ ] Writing guidelines checklist passed
- [ ] Diagrams included where specified
- [ ] New terms added to glossary (Appendix C)
- [ ] Cross-references to existing chapters are accurate
- [ ] Chapter metadata (frontmatter) is complete
- [ ] Committed to git with descriptive message
- [ ] @fix_plan.md updated with completion

## Estimated Book Metrics

| Metric | Target |
|--------|--------|
| Total Pages | 560-620 |
| Total Words | ~170,000-185,000 |
| Chapters | 17 |
| Appendices | 4 |
| Diagrams | 40-60 |

## Progress Tracking

Update these as chapters complete:

```
Chapters Complete: 0/17
Appendices Complete: 0/4
Estimated Words Written: 0
Current Phase: 1 (Foundations)
```
