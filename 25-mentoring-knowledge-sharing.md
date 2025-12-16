# Chapter 25: Mentoring & Knowledge Sharing

## Introduction

Junior developers focus solely on their own code. Senior developers multiply their impact by mentoring others, sharing knowledge widely, and building a culture of continuous learning that elevates the entire team.

## Learning Objectives

- Be an effective mentor
- Share knowledge systematically
- Create valuable documentation
- Give engaging tech talks
- Lead effective pair programming sessions
- Build learning resources
- Create a knowledge-sharing culture
- Scale your impact through others

## 25.1 Effective Mentoring

### The Mentor's Role

```typescript
// What makes a great mentor?

const greatMentor = {
  // 1. Listens First
  approach: 'Ask questions before giving answers',
  example: 'What have you tried so far? What's your thinking on this?',

  // 2. Guides, Doesn't Solve
  method: 'Help them discover the solution',
  avoid: 'Here, let me just do it for you',

  // 3. Adapts to Learning Styles
  recognize: ['Visual', 'Hands-on', 'Conceptual', 'Example-driven'],
  adjust: 'Match your teaching to their style',

  // 4. Creates Safe Space
  environment: 'No judgment for questions',
  response: 'That\'s a great question! Let\'s explore it together.',

  // 5. Celebrates Progress
  focus: 'Growth, not perfection',
  feedback: 'I noticed you handled that error case much better this time!',

  // 6. Shares Mistakes
  honesty: 'I made this mistake too when I was learning',
  value: 'Normalizes the learning process',
};
```

### Mentoring Techniques

```typescript
// Technique 1: Socratic Method
// Lead with questions to develop critical thinking

// ❌ Telling
"You should use useMemo here because it's expensive."

// ✅ Guiding
"What happens when this component re-renders?
How many times does this calculation run?
Is there a way we could avoid recalculating when the inputs haven't changed?"

// Technique 2: Think-Aloud
// Model your thought process

"When I see this bug, my first thought is to check the network tab.
Let's open DevTools... okay, I see a 404 error. Now I'm wondering
if the API endpoint changed. Let me check the backend code..."

// Technique 3: Scaffolded Learning
// Break complex topics into manageable steps

const learningPath = {
  week1: 'Basic React hooks - useState, useEffect',
  week2: 'Advanced hooks - useReducer, useContext',
  week3: 'Custom hooks - building reusable logic',
  week4: 'Performance - useMemo, useCallback',
  week5: 'Real project - apply everything learned',
};

// Technique 4: Code Reviews as Teaching
// Use reviews to explain principles

"Good start! Let's talk about why we extract this into a custom hook.
Three benefits:
1. Reusability - we can use it in other components
2. Testability - easier to test in isolation
3. Separation of concerns - component focuses on rendering

Want to try refactoring this together?"

// Technique 5: Pair Programming
// Learn by doing together

const pairProgrammingSession = {
  driver: 'Junior writes code',
  navigator: 'Senior guides and explains',
  switch: 'Every 20-30 minutes',
  debrief: 'Discuss what we learned',
};
```

### 1-on-1 Meetings Structure

```typescript
interface OneOnOneMeeting {
  frequency: 'Weekly or bi-weekly';
  duration: '30-60 minutes';

  agenda: {
    checkIn: {
      time: '5 minutes',
      questions: [
        'How are you doing?',
        'How\'s the workload?',
        'Any blockers?',
      ],
    },

    progressReview: {
      time: '10 minutes',
      discuss: [
        'Recent wins',
        'Challenges faced',
        'Lessons learned',
      ],
    },

    learningGoals: {
      time: '15 minutes',
      topics: [
        'What do you want to learn next?',
        'Progress on current learning goals',
        'Resources and support needed',
      ],
    },

    careerDevelopment: {
      time: '10 minutes',
      explore: [
        'Long-term goals',
        'Skills to develop',
        'Opportunities to pursue',
      ],
    },

    actionItems: {
      time: '5 minutes',
      capture: [
        'Tasks for mentee',
        'Support from mentor',
        'Follow-up items',
      ],
    },
  };

  notes: 'Document key points and action items';
  followUp: 'Review previous action items at next meeting';
}
```

## 25.2 Knowledge Documentation

### Writing Effective Documentation

```typescript
// Structure: README.md
const goodREADME = `
# Project Name

Brief description of what this project does (1-2 sentences)

## Quick Start

\`\`\`bash
npm install
npm run dev
\`\`\`

Visit http://localhost:3000

## Features

- Feature 1
- Feature 2
- Feature 3

## Tech Stack

- React 18
- TypeScript
- Vite
- React Query
- etc.

## Project Structure

\`\`\`
src/
├── components/     # Reusable UI components
├── pages/         # Route components
├── hooks/         # Custom React hooks
├── services/      # API calls and business logic
├── utils/         # Helper functions
└── types/         # TypeScript type definitions
\`\`\`

## Development

### Prerequisites
- Node.js 18+
- npm 9+

### Environment Variables
Create \`.env\` file:
\`\`\`
VITE_API_URL=http://localhost:3001
VITE_AUTH_TOKEN=your-token
\`\`\`

### Available Scripts
- \`npm run dev\` - Start development server
- \`npm run build\` - Build for production
- \`npm run test\` - Run tests
- \`npm run lint\` - Lint code

## Architecture

[Link to architecture docs]

## Contributing

[Link to contribution guidelines]

## License

MIT
`;

// Structure: ADR (Architecture Decision Record)
const architectureDecisionRecord = `
# ADR 001: Use React Query for Data Fetching

## Status
Accepted

## Context
We need a robust solution for server state management. Currently using
useEffect + useState leads to:
- Boilerplate code
- Inconsistent loading states
- No caching
- Duplicate requests

## Decision
We will use React Query for all server state management.

## Consequences

### Positive
- Automatic caching and deduplication
- Built-in loading/error states
- Background refetching
- Optimistic updates support
- DevTools for debugging

### Negative
- Learning curve for team
- Additional dependency (11KB gzipped)
- Migration effort for existing code

## Alternatives Considered
1. SWR - Similar features, smaller community
2. Redux + RTK Query - Too heavy for our needs
3. Apollo Client - Only needed for GraphQL

## Implementation Plan
1. Add React Query to new features (Week 1)
2. Migrate critical paths (Week 2-3)
3. Migrate remaining code (Week 4-6)
4. Team training session (Week 1)

## References
- [React Query Docs](https://tanstack.com/query)
- [Discussion: #123](link-to-discussion)
`;
```

### Code Documentation

```typescript
/**
 * Custom hook for managing paginated data fetching with React Query
 *
 * @param queryKey - Unique identifier for this query
 * @param fetchFn - Function that fetches a page of data
 * @param options - Configuration options
 *
 * @returns Object containing data, loading state, and pagination controls
 *
 * @example
 * ```tsx
 * const { data, loadMore, hasMore } = usePaginatedQuery(
 *   ['users'],
 *   ({ page }) => fetchUsers(page),
 *   { pageSize: 20 }
 * );
 * ```
 *
 * @see {@link https://tanstack.com/query/docs} for React Query docs
 */
export function usePaginatedQuery<T>(
  queryKey: string[],
  fetchFn: (params: { page: number }) => Promise<T[]>,
  options?: { pageSize?: number }
) {
  // Implementation...
}

/**
 * IMPORTANT: This function mutates the input array for performance.
 * If you need the original array, pass a copy.
 *
 * @param items - Array to sort (WILL BE MUTATED)
 * @param key - Property to sort by
 * @returns The sorted array (same reference as input)
 */
export function sortInPlace<T>(items: T[], key: keyof T): T[] {
  return items.sort((a, b) => {
    // Implementation...
  });
}

/**
 * Complex algorithm for operational transformation
 * Used in collaborative editing to resolve conflicts
 *
 * Algorithm explanation:
 * 1. Transform operation A against operation B
 * 2. Adjust indices based on operation type
 * 3. Handle edge cases (delete vs insert at same position)
 *
 * Based on: https://en.wikipedia.org/wiki/Operational_transformation
 */
function transform(opA: Operation, opB: Operation): Operation {
  // Complex logic with inline comments explaining each step
}
```

## 25.3 Tech Talks and Presentations

### Planning a Tech Talk

```typescript
interface TechTalkPlanning {
  topic: string;
  audience: 'Beginners' | 'Intermediate' | 'Advanced' | 'Mixed';
  duration: number; // in minutes

  structure: {
    hook: {
      time: '2 minutes',
      purpose: 'Grab attention',
      examples: [
        'Start with a relatable problem',
        'Show surprising demo',
        'Share personal story',
      ],
    },

    context: {
      time: '3 minutes',
      purpose: 'Set the stage',
      include: [
        'Why this topic matters',
        'What problem it solves',
        'When to use it',
      ],
    },

    content: {
      time: '15-20 minutes',
      purpose: 'Teach the core concept',
      tips: [
        'Build up complexity gradually',
        'Use live coding for engagement',
        'Show real examples',
        'Avoid too much theory',
      ],
    },

    demo: {
      time: '5 minutes',
      purpose: 'Make it concrete',
      prepare: [
        'Test demo beforehand',
        'Have backup if live coding fails',
        'Keep it simple and focused',
      ],
    },

    qa: {
      time: '5 minutes',
      purpose: 'Address questions',
      tips: [
        'Repeat questions for audience',
        'It\'s okay to say "I don\'t know"',
        'Offer to follow up after',
      ],
    },
  };

  preparation: {
    content: [
      'Create outline',
      'Build code examples',
      'Prepare slides (keep minimal)',
      'Practice timing',
      'Anticipate questions',
    ],

    logistics: [
      'Test screen sharing',
      'Check audio',
      'Have water nearby',
      'Close distracting apps',
    ],
  };
}

// Example: Planning "Introduction to React Query"
const reactQueryTalk = {
  title: 'Stop Using useEffect for Data Fetching',
  hook: 'Demo of tangled useEffect code vs clean React Query',

  outline: [
    '1. The Problem (3 min)',
    '   - Show complex useEffect example',
    '   - List pain points: loading states, caching, errors',
    '',
    '2. Enter React Query (2 min)',
    '   - What it is',
    '   - Core concepts',
    '',
    '3. Basic Usage (7 min)',
    '   - useQuery hook',
    '   - Live coding: fetch user data',
    '   - Automatic caching demo',
    '',
    '4. Advanced Features (8 min)',
    '   - Mutations',
    '   - Optimistic updates',
    '   - Infinite queries',
    '',
    '5. When to Use (3 min)',
    '   - Best use cases',
    '   - When not to use',
    '',
    '6. Getting Started (2 min)',
    '   - Resources',
    '   - Next steps',
    '',
    '7. Q&A (5 min)',
  ],

  liveDemo: {
    repo: 'github.com/you/react-query-demo',
    steps: [
      'Show app without React Query',
      'Add React Query setup',
      'Replace useEffect with useQuery',
      'Show DevTools',
      'Demo caching behavior',
    ],
    backup: 'Recorded video if live coding fails',
  },
};
```

### Presentation Tips

```typescript
const presentationTips = {
  slides: {
    design: [
      'One idea per slide',
      'Large, readable fonts (24pt+)',
      'High contrast for visibility',
      'Minimal text - speak, don\'t read',
      'Code: syntax highlighted, large font',
    ],

    content: [
      'Title slide: Topic + Your name',
      'Agenda slide: What we\'ll cover',
      'Content slides: One concept each',
      'Summary slide: Key takeaways',
      'Resources slide: Links for learning more',
    ],
  },

  delivery: {
    voice: [
      'Speak slowly and clearly',
      'Pause for emphasis',
      'Vary your tone',
      'Project confidence',
    ],

    body: [
      'Make eye contact (camera if remote)',
      'Use gestures naturally',
      'Stand/sit up straight',
      'Smile - you\'re sharing something cool!',
    ],

    engagement: [
      'Ask questions to audience',
      'Use polls for feedback',
      'Encourage questions throughout',
      'Share screen for demos',
    ],
  },

  common_mistakes: [
    'Too much content for time',
    'Reading slides verbatim',
    'Apologizing ("Sorry, this might be boring...")',
    'Going over time',
    'Not testing demos beforehand',
    'Tiny font size',
    'Complex diagrams without explanation',
  ],

  recovery: {
    'Demo fails': 'Have recorded backup, laugh it off',
    'Lose your place': 'Pause, check notes, continue',
    'Tough question': '"Great question! Let me follow up after"',
    'Technical issues': 'Stay calm, have backup plan',
  },
};
```

## 25.4 Internal Knowledge Base

### Building a Team Wiki

```typescript
// wiki-structure.md
const wikiStructure = `
# Team Wiki Structure

## 1. Getting Started
- Onboarding checklist
- Development environment setup
- First day guide
- Team contacts

## 2. Development Guides
- Code style guide
- Git workflow
- PR process
- Testing guidelines
- Deployment process

## 3. Architecture
- System overview
- Component architecture
- Data flow
- API documentation
- Database schema

## 4. Common Tasks
- How to add a new feature
- How to fix a bug
- How to add a dependency
- How to update documentation
- How to run tests

## 5. Troubleshooting
- Common errors and solutions
- Debugging guide
- Performance issues
- Build problems

## 6. Tools & Resources
- Development tools
- External services
- Learning resources
- Useful links

## 7. Decisions
- Architecture Decision Records (ADRs)
- Tech stack choices
- Design patterns used
- Migration plans

## 8. Team Processes
- Sprint planning
- Code review guidelines
- On-call rotation
- Incident response
`;

// Example: Troubleshooting Page
const troubleshootingPage = `
# Common Errors and Solutions

## "Cannot find module" Error

### Symptom
\`\`\`
Error: Cannot find module 'react-query'
\`\`\`

### Cause
Dependency not installed or node_modules corrupted

### Solution
\`\`\`bash
# Delete node_modules and reinstall
rm -rf node_modules package-lock.json
npm install
\`\`\`

### Prevention
Run \`npm ci\` instead of \`npm install\` in CI/CD

---

## Infinite Re-render Loop

### Symptom
Browser freezes, React DevTools shows thousands of renders

### Common Causes
1. Missing dependency in useEffect
2. Creating new object/function in render
3. Setting state during render

### Example & Fix
\`\`\`tsx
// ❌ Bad: Creates new function every render
<Button onClick={() => console.log('click')} />

// ✅ Good: Memoized callback
const handleClick = useCallback(() => {
  console.log('click');
}, []);
<Button onClick={handleClick} />
\`\`\`

### Debugging
1. Check React DevTools Profiler
2. Look for yellow highlights (unnecessary renders)
3. Review useEffect dependencies

---

[More common errors...]
`;
```

## 25.5 Pair Programming

### Effective Pairing Sessions

```typescript
interface PairProgrammingSession {
  roles: {
    driver: {
      responsibility: 'Write the code',
      focus: 'Tactical implementation',
      keyboard: true,
    },
    navigator: {
      responsibility: 'Guide and review',
      focus: 'Strategic thinking',
      keyboard: false,
    },
  };

  rotation: {
    frequency: 'Every 20-30 minutes',
    timer: 'Use Pomodoro timer',
    breaks: 'Take 5 min break after each rotation',
  };

  setup: {
    tools: [
      'VS Code Live Share',
      'Tuple (screen sharing)',
      'Zoom/Meet with screen share',
    ],
    environment: [
      'Quiet space',
      'Good audio setup',
      'Stable internet',
      'Close distractions',
    ],
  };

  techniques: {
    ping_pong: {
      description: 'Perfect for TDD',
      process: [
        'Navigator writes failing test',
        'Switch: Driver makes it pass',
        'Switch: Navigator refactors',
        'Repeat',
      ],
    },

    strong_style: {
      description: 'For transferring knowledge',
      process: [
        'Navigator describes what to do',
        'Driver implements',
        'Driver cannot type what they think, only what navigator says',
        'Forces explicit communication',
      ],
    },

    tutorial_style: {
      description: 'For teaching',
      process: [
        'Navigator explains concept',
        'Navigator demonstrates',
        'Driver tries with guidance',
        'Driver does independently',
      ],
    },
  };

  best_practices: [
    'Agree on goal before starting',
    'Voice your thought process',
    'Ask questions freely',
    'Take breaks regularly',
    'Switch roles frequently',
    'Be patient and kind',
    'Celebrate small wins',
  ];

  antipatterns: [
    'Grabbing keyboard from partner',
    'Not explaining your thinking',
    'Working in silence',
    'Ignoring navigator suggestions',
    'Pairing for too long without breaks',
    'Checking phone/email during session',
  ];
}
```

## 25.6 Scaling Knowledge Through Content

### Creating Learning Resources

```typescript
// Types of content to create

const contentTypes = {
  blogPosts: {
    when: 'You learned something valuable',
    topics: [
      'How to solve specific problem',
      'Comparison of approaches',
      'Deep dive into concept',
      'Lessons from production incident',
    ],
    structure: {
      title: 'Clear, specific, SEO-friendly',
      intro: 'Hook reader, explain problem',
      body: 'Teach concept with examples',
      code: 'Runnable examples',
      conclusion: 'Key takeaways, next steps',
    },
    distribution: [
      'Company blog',
      'dev.to',
      'Medium',
      'Personal blog',
      'Internal wiki',
    ],
  },

  videos: {
    when: 'Complex topic needs visual explanation',
    types: [
      'Screen recording tutorials',
      'Live coding sessions',
      'Architecture whiteboard',
      'Code review walkthrough',
    ],
    tools: [
      'Loom (quick recordings)',
      'OBS (professional recording)',
      'Zoom (live sessions)',
    ],
    tips: [
      'Keep under 15 minutes',
      'Script key points',
      'Edit out mistakes',
      'Add captions',
    ],
  },

  codeSnippets: {
    when: 'Reusable pattern discovered',
    share: [
      'Internal snippet library',
      'GitHub Gists',
      'Team Slack channel',
      'README examples',
    ],
    include: [
      'Clear comments',
      'Usage example',
      'When to use / not use',
      'Edge cases handled',
    ],
  },

  workshops: {
    when: 'Team needs to learn new skill',
    format: {
      duration: '2-4 hours',
      structure: [
        'Intro: Why this matters (10 min)',
        'Theory: Core concepts (30 min)',
        'Demo: Live example (20 min)',
        'Exercise: Hands-on practice (60 min)',
        'Review: Discuss solutions (20 min)',
        'Q&A: Open discussion (20 min)',
      ],
    },
    preparation: [
      'Create exercises with solutions',
      'Set up starter repository',
      'Test exercises yourself',
      'Prepare helper resources',
    ],
  },
};

// Example: Blog Post Outline
const blogPostExample = `
Title: "Stop Using Prop Drilling: A Guide to React Context"

## Introduction (200 words)
- Problem: Passing props through 5 levels
- Pain points: Refactoring nightmare, hard to follow
- Promise: Better solution exists

## Understanding the Problem (300 words)
- Code example showing prop drilling
- Why it happens
- When it becomes painful

## Enter React Context (400 words)
- What Context is
- When to use it
- When NOT to use it

## Implementation Guide (800 words)
- Step 1: Create context
- Step 2: Create provider
- Step 3: Consume context
- Full working example

## Advanced Patterns (600 words)
- Multiple contexts
- Context with useReducer
- Performance optimization

## Real-World Example (500 words)
- Theme switching implementation
- Auth context pattern

## Conclusion (200 words)
- When to use Context
- Common mistakes to avoid
- Further reading

Total: ~3000 words, 10-15 min read
`;
```

## Real-World Scenario: Onboarding New Developer

### The Challenge

New junior developer joining the team:
- Never used TypeScript
- Limited React experience
- First professional role
- Needs to be productive quickly

### Senior's Approach

```typescript
// Week-by-week onboarding plan

const onboardingPlan = {
  week1: {
    goals: [
      'Complete environment setup',
      'Understand codebase structure',
      'Make first small contribution',
    ],

    activities: {
      day1: [
        '9am: Welcome meeting with team',
        '10am: Pair on environment setup',
        '2pm: Codebase tour',
        '4pm: Assign first issue (documentation update)',
      ],

      day2_3: [
        'Read architecture docs',
        'Watch internal tech talks',
        'Shadow code reviews',
        'Pair on first PR',
      ],

      day4_5: [
        'Fix small bug (with guidance)',
        'Submit first PR',
        'Learn review process',
        'Start TypeScript basics',
      ],
    },

    mentor_focus: [
      'Make them feel welcome',
      'Answer ALL questions',
      'Pair program frequently',
      'Provide encouragement',
    ],
  },

  week2_4: {
    goals: [
      'Learn TypeScript fundamentals',
      'Understand React patterns we use',
      'Complete small features independently',
    ],

    learning_path: [
      'TypeScript course (online)',
      'Internal React patterns guide',
      'Pair programming 2x/week',
      'Gradually larger features',
    ],

    mentor_support: [
      'Weekly 1-on-1s',
      'Code review teaching',
      'Available for questions',
      'Share resources',
    ],
  },

  month2_3: {
    goals: [
      'Contribute independently',
      'Review others\' code',
      'Share knowledge with team',
    ],

    progression: [
      'Larger features',
      'More complex bugs',
      'Start reviewing PRs',
      'Give first tech talk',
    ],
  },

  ongoing: {
    support: [
      'Bi-weekly 1-on-1s',
      'Career development discussions',
      'Learning goal tracking',
      'Feedback sessions',
    ],
  },
};
```

## Chapter Exercise: Create Learning Resource

Create a resource to share knowledge:

**Choose one:**
1. Write tutorial blog post on recent learning
2. Record 10-minute video tutorial
3. Create troubleshooting guide
4. Plan and outline a tech talk
5. Build code example repository

**Quality criteria:**
- Clear structure
- Practical examples
- Actionable takeaways
- Appropriate for audience
- Well-documented

## Review Checklist

- [ ] Understanding of effective mentoring
- [ ] Documentation best practices
- [ ] Tech talk planning and delivery
- [ ] Pair programming techniques
- [ ] Knowledge base structure
- [ ] Content creation strategies
- [ ] Onboarding process design
- [ ] Scaling impact through teaching

## Key Takeaways

1. **Great mentors listen first** - Understand before prescribing
2. **Show, don't just tell** - Demonstrate thought process
3. **Document systematically** - Make knowledge accessible
4. **Teach to learn** - Teaching deepens your understanding
5. **Create safe spaces** - No judgment for questions
6. **Scale through content** - One explanation, many benefit
7. **Invest in others** - Compound returns for the team

## Further Reading

- The Pragmatic Programmer (Teaching section)
- Thanks for the Feedback (Sheila Heen)
- Made to Stick (Chip & Dan Heath)
- The Coaching Habit (Michael Bungay Stanier)
- Atomic Habits (James Clear) - for building learning culture

## Next Chapter

[Chapter 26: System Design Interviews](./26-system-design-interviews.md)
