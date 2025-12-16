# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is **"From Junior to Senior: A Hands-On React Developer Journey"** - an educational book teaching React developers how to advance from junior to senior level. The book consists of 28 markdown chapters covering React fundamentals, architecture, testing, DevOps, and professional skills.

**Key characteristics:**
- Self-contained markdown-based learning resource
- No code to build/run - purely educational content
- Designed for read-only consumption with interactive exercises
- Uses collapsible markdown sections (`<details>/<summary>`) for progressive disclosure

## Content Architecture

### Chapter Structure (Standardized Format)

**Current State:** Only Chapter 1 (`01-rethinking-fundamentals.md`) has been fully converted to the new 10/10 format. All other chapters use older formats and need conversion.

**Chapter 1 Format (Template for all chapters):**
1. **Introduction & Learning Objectives** - Sets context and goals
2. **Main Sections (e.g., 1.1, 1.2, 1.3)** - Core content with Junior vs Senior perspectives
3. **Visual Diagrams** - ASCII diagrams showing concepts (5+ per chapter)
4. **Interactive Exercises** - 5 exercises using collapsible solutions:
   - Code Review Challenges
   - Design Exercises
   - Debugging Scenarios
   - Discussion Questions
5. **Knowledge Checks** - Quiz sections (2-3 per chapter, 3 questions each) with `<details>` answers
6. **Real War Stories** - 1+ production incidents with financial impact and post-mortems
7. **Senior Think-Aloud** - 1+ sections showing senior decision-making process
8. **Common Mistakes Gallery** - 6 patterns from real code reviews (added before Review Checklist)
9. **Chapter Project** - Comprehensive build-something project with evaluation checklist
10. **Review Checklist** - Self-assessment before moving to next chapter

### File Organization

```
junior-to-senior/
├── README.md                      # Main entry point, book overview
├── HOW-TO-USE-THIS-BOOK.md       # Learning paths, time commitments, strategies
├── EXERCISE-FORMAT-GUIDE.md      # Template for creating/converting chapters
├── 01-rethinking-fundamentals.md # ✅ GOLD STANDARD (10/10 format)
├── 02-advanced-patterns.md       # ⚠️ OLD FORMAT - needs conversion
├── 03-mastering-hooks.md         # ⚠️ OLD FORMAT - needs conversion
├── [04-28...].md                 # ⚠️ OLD FORMAT - needs conversion
├── BOOK-STATUS.md                # Completion tracking
└── *.sh                          # Helper scripts (not used)
```

### Chapter Naming Convention

- Chapters: `[01-28]-kebab-case-name.md`
- Always use 2-digit numbers with leading zero
- File name reflects chapter title but shortened

### Book Structure (9 Parts, 28 Chapters)

1. **Part 1: Solidifying Foundations** (Ch 1-3)
2. **Part 2: Performance & Optimization** (Ch 4-6)
3. **Part 3: Architecture & Design** (Ch 7-10)
4. **Part 4: TypeScript & Type Safety** (Ch 11-12)
5. **Part 5: Testing Mastery** (Ch 13-15)
6. **Part 6: DevOps & Tooling** (Ch 16-18)
7. **Part 7: Real-World Systems** (Ch 19-22)
8. **Part 8: Leadership & Professional Growth** (Ch 23-26)
9. **Part 9: Capstone Projects** (Ch 27-28)

## Working with Content

### Converting Chapters to 10/10 Format

**Reference:** Use `EXERCISE-FORMAT-GUIDE.md` as your template and `01-rethinking-fundamentals.md` as the gold standard example.

**Process:**
1. Read the existing chapter to understand current content
2. Identify sections that need enhancement (exercises, explanations)
3. Add required elements:
   - ASCII diagrams for complex concepts
   - Convert TODO exercises to progressive disclosure format
   - Add knowledge checks after major sections
   - Create/enhance war stories with real impact
   - Add senior think-aloud sections
   - Create common mistakes gallery
   - Enhance chapter project with evaluation criteria
4. Validate markdown syntax (all `<details>` paired, code blocks closed)
5. Ensure emoji usage follows conventions

**Key Principles:**
- **Progressive disclosure:** Always use `<details>/<summary>` for hints/solutions
- **Real-world context:** Include actual incidents, costs, company scales
- **Visual learning:** Add ASCII diagrams for every complex concept
- **Active learning:** Questions before answers
- **Senior mindset:** Show thinking process, not just solutions

### Markdown Conventions

**Collapsible Sections:**
```markdown
<details>
<summary>💡 Hint: How to approach this</summary>

Hint content here...

</details>

<details>
<summary>✅ Solution</summary>

Solution with explanation...

</details>
```

**Emoji Usage (Consistent Icons):**
- ✅ Checkmark: Solutions, correct answers, completed items
- 💡 Lightbulb: Hints, insights
- 🔍 Magnifying glass: Analysis, deep dives
- 🧠 Brain: Senior thinking process
- 💥 Explosion: War stories, incidents
- 🚫 Prohibited: Common mistakes
- 🎯 Target: Key points, challenges
- ⚠️ Warning: Cautions, red flags
- 📚 Books: Deep dives, references

**ASCII Diagrams:**
- Use box-drawing characters for visual flow
- Keep diagrams under 80 characters wide
- Add annotations with arrows (→, ↓, ←, ↑)
- Use `┌─┐│└┘` for boxes

### Content Quality Standards

**For 10/10 Rating, Each Chapter Must Have:**
- [ ] 5+ interactive exercises with progressive disclosure
- [ ] 2-3 knowledge check sections (9+ total questions)
- [ ] 1+ real war story with actual costs/impact
- [ ] 5+ ASCII diagrams
- [ ] 1+ senior think-aloud walkthrough
- [ ] 6 common mistake patterns
- [ ] Comprehensive chapter project
- [ ] Copy-paste ready code examples
- [ ] All markdown syntax valid (paired tags, closed code blocks)

### Validation Commands

**Check markdown syntax:**
```bash
# Count collapsible sections (should be equal)
grep -c "<details>" [chapter].md
grep -c "</details>" [chapter].md

# Verify code blocks are paired
python3 -c "print('Paired' if open('[chapter].md').read().count('\`\`\`') % 2 == 0 else 'UNPAIRED')"

# Check for broken links
grep -o '\[.*\]()' [chapter].md
```

**Content statistics:**
```bash
# Get file size and line count
wc -l [chapter].md

# Count exercises
grep -c "### Exercise" [chapter].md

# Count knowledge checks
grep -c "Quick Knowledge Check" [chapter].md
```

## Important Notes for AI Assistants

### When Converting Chapters

1. **Never remove content** - Only enhance and restructure
2. **Preserve all code examples** - They're carefully chosen
3. **Add, don't replace** - Old exercises can be enhanced, not deleted
4. **Use Chapter 1 as reference** - Copy patterns, not content
5. **Validate before finishing** - Run markdown checks
6. **Keep reading level consistent** - Junior-to-senior audience (1-2 years React experience)

### When Editing Existing Content

1. **Chapter 1 is gold standard** - Don't change its structure
2. **Preserve collapsible sections** - They're pedagogically important
3. **Maintain emoji consistency** - Use established conventions
4. **Keep ASCII diagrams** - Don't convert to image references
5. **Respect progressive disclosure** - Always hint before solution

### Common Pitfalls to Avoid

- ❌ Adding external dependencies (CodeSandbox links, image URLs)
- ❌ Breaking markdown syntax (unclosed tags, unpaired code blocks)
- ❌ Generic war stories (need specific costs, companies, dates)
- ❌ TODO-style exercises (use progressive disclosure instead)
- ❌ Walls of text (break up with diagrams, collapsibles)
- ❌ Junior-level content (audience has 1-2 years experience)

### Educational Principles (Research-Backed)

This book's format is validated by learning science research:
- **Progressive disclosure** - Prevents cognitive overload (neuroscience)
- **Testing effect** - 1,215+ studies prove retrieval practice works
- **Worked examples** - Reduces cognitive load (Cognitive Load Theory)
- **Expert modeling** - Think-aloud shows decision processes
- **Case-based learning** - Real war stories improve retention
- **Visual learning** - ASCII diagrams support dual coding

## Meta Files

- **README.md** - Keep updated with chapter status
- **EXERCISE-FORMAT-GUIDE.md** - Reference for creating exercises, don't modify unless improving format
- **HOW-TO-USE-THIS-BOOK.md** - Learning paths for users, update if adding new learning modes
- **BOOK-STATUS.md** - Track completion status, update when chapters are converted

## Quick Reference

**To convert a chapter to 10/10:**
1. Read `EXERCISE-FORMAT-GUIDE.md`
2. Study `01-rethinking-fundamentals.md` structure
3. Apply format to target chapter
4. Add: diagrams, knowledge checks, war story, think-aloud, mistakes gallery
5. Validate markdown syntax
6. Update BOOK-STATUS.md

**To validate chapter quality:**
- Check against "Content Quality Standards" checklist above
- Compare to Chapter 1's structure and depth
- Run validation commands
- Ensure 27+ collapsible sections, 9+ quiz questions, 5+ diagrams

**Target audience:**
- 1-2 years React experience
- Can build features, wants to design systems
- Preparing for senior roles or interviews
- Learning by doing, not passive reading
