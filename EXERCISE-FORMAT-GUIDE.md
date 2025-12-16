# Exercise Format Guide

## Overview

This guide explains how to write exercises for this book. All exercises use a **read-only format** with collapsible solutions.

## Exercise Types

### 1. Code Review Challenges

**Format:**
```markdown
### Exercise X.Y: [Title] - Code Review Challenge

**Scenario:** [Set up the context - e.g., "A junior developer wrote this code..."]

**The Code:**
```javascript
// Show buggy or problematic code
function BadExample() {
  // ... code with issues
}
```

**Your Task:**
1. What's wrong with this code?
2. Why does it fail?
3. How would you fix it?
4. What would you explain to the junior developer?

<details>
<summary>🔍 Analysis: What's wrong?</summary>

Explain the problem step-by-step...

</details>

<details>
<summary>✅ Solution</summary>

```javascript
// Show correct implementation
function GoodExample() {
  // ... fixed code
}
```

**Why this is better:**
- Explanation point 1
- Explanation point 2

</details>
```

**Use for:**
- Identifying anti-patterns
- Debugging exercises
- Understanding common mistakes

### 2. Discussion/Design Challenges

**Format:**
```markdown
### Exercise X.Y: [Title] - Design Challenge

**Challenge:** [Describe the design problem]

**Requirements:**
- Requirement 1
- Requirement 2
- Requirement 3

**Think About:**
- Question 1
- Question 2
- Question 3

**Try It Yourself:** [Optional link to CodeSandbox]

<details>
<summary>💡 Hint: How to approach this</summary>

- Approach suggestion 1
- Approach suggestion 2
- Key concept to remember

</details>

<details>
<summary>✅ Solution: One Approach</summary>

```javascript
// Implementation
```

**Why this works:**
- Explanation

**Alternatives:**
- Alternative approach and trade-offs

</details>

<details>
<summary>📚 Deep Dive: [Advanced Topic]</summary>

Additional context, edge cases, or advanced considerations...

</details>
```

**Use for:**
- Architecture decisions
- API design
- Component structure
- Pattern selection

### 3. Implementation Exercises

**Format:**
```markdown
### Exercise X.Y: [Title]

**Challenge:** [What to build]

**Requirements:**
1. Requirement with specific constraint
2. Must demonstrate specific pattern
3. Should achieve specific outcome

**Think About:**
- Design consideration 1
- Performance consideration 2
- Edge case 3

**Try It Yourself:** Open a [CodeSandbox](https://codesandbox.io/s/new) and implement this!

<details>
<summary>💡 Hint: Getting started</summary>

Start with:
- Step 1
- Step 2
- Step 3

</details>

<details>
<summary>✅ Solution</summary>

```javascript
// Full implementation with comments
```

**Key Observations:**
- What happens when X
- Why Y behaves this way
- The difference between A and B

**What You Learned:**
- Concept 1
- Concept 2
- Practical takeaway

</details>
```

**Use for:**
- Demonstrating specific patterns
- Testing understanding
- Hands-on practice

### 4. Debugging Scenarios

**Format:**
```markdown
### Exercise X.Y: [Title] - Debugging Exercise

**Bug Report:** "Users are experiencing [problem description]"

**The Code:**
```javascript
// Buggy code
```

**Reproduction Steps:**
1. Step 1
2. Step 2
3. Observe: What happens?

**Your Task:**
- Why does this happen?
- What's the root cause?
- How would you fix it?

<details>
<summary>🔍 Root Cause Analysis</summary>

**The Problem:**
Explain the underlying issue...

**Step-by-step breakdown:**
1. First this happens...
2. Then this happens...
3. Result: bug occurs

</details>

<details>
<summary>✅ Solution: Multiple Approaches</summary>

**Approach 1: [Name]**
```javascript
// Fix
```
**Pros/Cons**

**Approach 2: [Name]**
```javascript
// Alternative fix
```
**Pros/Cons**

</details>
```

**Use for:**
- Common React bugs
- Performance issues
- Race conditions
- Memory leaks

## Collapsible Section Guidelines

### Icons to Use

- 💡 **Hint** - Guidance without giving away the answer
- 🔍 **Analysis** - Deep dive into the problem
- ✅ **Solution** - The answer/implementation
- 📚 **Deep Dive** - Additional advanced context
- 🤔 **Discussion** - Open-ended considerations
- 🎯 **Key Points** - Summary of important takeaways
- ⚠️ **Warning** - Common pitfalls
- 📖 **Reference** - Links to documentation

### Summary Text

Make summary text actionable and descriptive:

- ✅ "Click to reveal the solution"
- ✅ "💡 Hint: How to approach this"
- ✅ "🔍 Analysis: Why this bug occurs"
- ❌ "Answer"
- ❌ "Click here"
- ❌ "More info"

## Real-World Scenarios

**Format:**
```markdown
## Real-World Scenario: [Title]

### The Situation

[Describe a realistic workplace scenario with specific details]

### Reflection Exercise

Before looking at the solution, answer these questions in your learning journal:

**Analysis:**
1. Question about understanding the problem
2. Question about measuring current state
3. Question about risk assessment

**Planning:**
1. Where would you start?
2. How would you approach incrementally?
3. How would you validate success?

<details>
<summary>💡 Senior Developer's Approach</summary>

**Step 1: [Phase Name]**
[Detailed approach]

**Step 2: [Phase Name]**
[Detailed approach]

</details>

<details>
<summary>📋 [Checklist/Template Name]</summary>

**Week 1:**
- [ ] Task 1
- [ ] Task 2

**Success Metrics:**
- ✅ Metric 1
- ✅ Metric 2

</details>
```

## Chapter Projects

**Format:**
```markdown
## Chapter Project: [Project Name]

**Goal:** [What this demonstrates]

### Project Specification

[Clear description of what to build]

**Core Features:**
1. Feature 1
2. Feature 2

**Technical Requirements:**

Must demonstrate:
- ✅ Concept from section 1
- ✅ Concept from section 2

### Architecture Guidelines

**Component Structure:**
```
[ASCII tree showing structure]
```

**State Management:**
- Guideline 1
- Guideline 2

### Self-Evaluation Checklist

Before considering this complete, verify:

**[Category 1]:**
- [ ] Specific requirement 1
- [ ] Specific requirement 2

**[Category 2]:**
- [ ] Specific requirement 3

### Implementation Approach

**Phase 1: [Name]**
```javascript
// Starting point
```

**Phase 2: [Name]**
```javascript
// Next step
```

<details>
<summary>💡 Implementation Hints</summary>

**[Sub-topic]:**
```javascript
// Helper code or pattern
```

</details>

<details>
<summary>🎯 Bonus Challenges</summary>

Once you have the basics:

1. **[Challenge Name]**
   - Detail
   - Detail

</details>

### Learning Reflection

After completing this project, write in your journal:

**What went well:**
- Reflection prompt

**What was challenging:**
- Reflection prompt

**Key insights:**
- Reflection prompt
```

## Best Practices

1. **Always include context** - Don't just show code, explain the scenario
2. **Encourage thinking first** - Put questions before solutions
3. **Multiple solutions** - Show alternatives with trade-offs
4. **Real-world relevance** - Connect to actual development scenarios
5. **Incremental complexity** - Start simple, add challenges
6. **Self-assessment** - Include checklists and reflection prompts

## What to Avoid

- ❌ "TODO: Implement X" without context or learning value
- ❌ Exercises without solutions (this is self-study material)
- ❌ Solutions without explanation
- ❌ Unrealistic or trivial examples
- ❌ Exercises that require external dependencies not explained
- ❌ Missing or vague success criteria

## Example Conversion

### Before (Old Format):
```markdown
### Exercise 1.1

Create a component that demonstrates render phases.

```javascript
// TODO: Implement PhaseDemo
function PhaseDemo() {
  // Your code here
}
```
```

### After (New Format):
```markdown
### Exercise 1.1: Understanding Render and Commit Phases

**Challenge:** Create a component that demonstrates the difference between render and commit phases.

**Requirements:**
1. Log "Rendering" during render phase with timestamp
2. Log "Committed" after commit phase with timestamp
3. Add button that triggers re-renders
4. Demonstrate batching (multiple setState in one render)

**Think About:**
- What happens if you call setState twice in one handler?
- Why do timestamps differ?

**Try It Yourself:** [CodeSandbox link]

<details>
<summary>💡 Hint: How to approach this</summary>

- Use console.log in component body for render
- Use useEffect for commit
- Use useRef to track count without re-rendering
- Try multiple setState calls to see batching

</details>

<details>
<summary>✅ Solution</summary>

```javascript
[Full working implementation with comments]
```

**Key Observations:**
- Render logs appear immediately
- Commit logs appear after DOM update
- Batching means 2 updates = 1 render
- useRef doesn't cause re-renders

**What You Learned:**
- Render = pure computation
- Commit = side effects
- React 18 auto-batches
- useRef for tracking

</details>
```

## Summary

The goal is to create **active learning experiences** that:
- Engage critical thinking
- Provide clear context
- Offer progressive disclosure (hints → solution)
- Include practical application
- Encourage reflection and practice

Every exercise should teach something specific and be valuable even when just reading (not implementing).
