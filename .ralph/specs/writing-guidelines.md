# Writing Guidelines

## Voice and Tone

### Target Voice
- **Authoritative but accessible:** Expert explaining to a peer, not lecturing a student
- **Direct:** State positions clearly, avoid hedging language
- **Precise:** Technical accuracy matters; don't oversimplify
- **Engaging:** Dense doesn't mean dry; use analogies and concrete examples

### What to Avoid
- Marketing language or hype
- Unnecessary qualifiers ("it could be argued that...")
- Filler phrases ("It's important to note that...")
- Excessive hedging ("might," "perhaps," "possibly" when you know the answer)
- Condescension ("simply," "just," "obviously")

## Chapter Structure

Each chapter should follow this general structure:

### 1. Opening Hook (1-2 paragraphs)
- Start with a problem, question, or scenario
- Make the reader care about what follows
- Don't start with definitions

### 2. Historical Context
- What problem necessitated this technology/pattern?
- What existed before and why was it insufficient?
- Key decision points in the evolution

### 3. Core Concepts
- Build from first principles where possible
- Use diagrams for architecture and data flow
- Include mental models and frameworks
- Compare/contrast with concepts reader already knows

### 4. Practical Patterns
- Common use patterns
- Anti-patterns to avoid
- Configuration considerations

### 5. Decision Framework
- When to use this vs alternatives
- Key factors in the decision
- Trade-off matrices where appropriate

### 6. Case Study Application
- Concrete example applying the concepts
- Preferably connected to trading/financial data where relevant

### 7. Summary and Cross-References
- Key takeaways (bulleted)
- Related chapters for deeper exploration
- Further reading (papers, docs)

## Code Examples

### Format
```
# Pseudocode style - focus on logic, not syntax details
# Python-like for familiarity
# SQL where appropriate

def transform_events(events: Stream[Event]) -> Stream[Enriched]:
    """
    Enrich raw events with dimension lookups.
    Key concept: late-arriving dimension handling.
    """
    return (
        events
        .window(TumblingWindow(duration=5.minutes))
        .join(dimensions, on="dimension_key")
        .handle_late_data(strategy=EMIT_PARTIAL)
    )
```

### Guidelines
- Include comments explaining the "why"
- Focus on patterns, not syntax correctness
- Use meaningful names that convey intent
- Keep examples short (10-30 lines typically)
- No boilerplate or imports unless relevant to the concept

## Diagrams

### When to Include
- Architecture overviews
- Data flow through systems
- Component interactions
- Comparison matrices
- Decision trees

### Style Guidelines
- Simple, clean layouts
- Consistent visual language throughout book
- Label everything clearly
- Include brief caption explaining what to notice

### Diagram Types to Use
- Box-and-arrow for architectures
- Sequence diagrams for interactions
- Tables for comparisons
- Flowcharts for decision frameworks

## Cross-References

### Internal References
- Reference related concepts by chapter: "See Chapter 5: Table Formats for details on how Iceberg handles this"
- Use forward references sparingly: "We'll explore this further in Chapter 9"
- Prefer backward references to build on established concepts

### External References
- Official documentation (preferred)
- Foundational papers for theoretical concepts
- Avoid blog posts unless authoritative (engineering blogs from companies like Netflix, Uber, etc. are acceptable)

## Terminology

### Consistency
- Define terms on first use
- Use consistent terminology throughout
- Include all terms in glossary

### Acronyms
- Spell out on first use: "Apache Iceberg (Iceberg)"
- Use short form thereafter
- Common industry acronyms (SQL, API, ETL) don't need expansion

## Length Guidelines

### Per Chapter
- Target: 25-60 pages depending on topic depth
- Core concept chapters (Iceberg, Spark): 50-60 pages
- Survey chapters (OLAP, Orchestration Tools): 30-45 pages
- Foundation chapters: 25-40 pages

### Per Section
- Avoid sections longer than 8-10 pages without subheadings
- Break up dense content with examples, diagrams, or callouts

### Per Paragraph
- Average 4-6 sentences
- Maximum 8-10 sentences for complex explanations
- One idea per paragraph

## Special Elements

### Callout Boxes

**Key Insight:** For important mental model shifts
> This box highlights a fundamental insight that the reader should internalize.

**Historical Note:** For evolution/context
> Provides background on why things are the way they are.

**Decision Point:** For framework guidance
> Summarizes the key factors when making a choice.

**Warning:** For common pitfalls
> Alerts reader to a common mistake or misconception.

### Summary Boxes
At end of major sections, bulleted lists of:
- Key concepts introduced
- Decisions/trade-offs discussed
- Connections to other chapters

## Quality Checklist

Before considering a chapter complete:

- [ ] Opens with engaging hook, not definition
- [ ] Historical context explains the "why"
- [ ] Core concepts build from first principles
- [ ] Mental models/frameworks are explicitly stated
- [ ] Trade-offs are clearly articulated
- [ ] Decision framework included
- [ ] Code examples are conceptual, not tutorial
- [ ] Diagrams support (not replace) text explanation
- [ ] Cross-references link to related chapters
- [ ] Summary captures key takeaways
- [ ] No marketing language or hype
- [ ] Terminology is consistent with glossary
- [ ] Length is appropriate for topic depth
