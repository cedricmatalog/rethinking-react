# Chapter 26: System Design Interviews

## Introduction

Junior developers struggle with system design interviews because they focus on implementation details. Senior developers excel by thinking about architecture holistically, discussing trade-offs, and demonstrating deep understanding of how systems scale and interact.

## Learning Objectives

- Approach system design interviews systematically
- Understand common frontend architecture patterns
- Discuss trade-offs effectively
- Scale frontend applications
- Design real-time systems
- Handle performance at scale
- Navigate ambiguity confidently
- Communicate technical decisions clearly

## 26.1 The System Design Interview Framework

### The RADIO Framework

```typescript
const RADIOFramework = {
  R: 'Requirements',
  A: 'Architecture',
  D: 'Data Model',
  I: 'Interface Definitions',
  O: 'Optimizations & Deep Dives',
};

// Step-by-step breakdown

const interviewProcess = {
  // Step 1: Requirements (5-10 minutes)
  requirements: {
    clarify: [
      'Who are the users?',
      'What are the core features?',
      'What scale are we talking about?',
      'What devices/browsers to support?',
      'Any performance requirements?',
      'What about offline support?',
    ],

    functional: [
      'User authentication',
      'Create/read/update/delete data',
      'Real-time updates',
      'File uploads',
      'Search functionality',
    ],

    nonFunctional: [
      'Performance: < 3s load time',
      'Scale: 1M daily active users',
      'Availability: 99.9% uptime',
      'Security: OAuth 2.0',
      'Accessibility: WCAG AA',
    ],

    outOfScope: [
      'Mobile apps (web only)',
      'Payment processing',
      'Admin dashboard',
    ],
  },

  // Step 2: Architecture (15-20 minutes)
  architecture: {
    components: [
      'Client application',
      'API Gateway',
      'Backend services',
      'Database',
      'Cache layer',
      'CDN',
      'File storage',
    ],

    diagram: `
      [Browser] <--> [CDN]
          |
          v
      [Load Balancer]
          |
          v
      [API Gateway]
          |
          +-> [Auth Service]
          +-> [User Service]
          +-> [Post Service]
          |
          v
      [Cache] <--> [Database]
    `,

    discuss: [
      'Why this architecture?',
      'How components communicate',
      'Where state lives',
      'How to scale each piece',
    ],
  },

  // Step 3: Data Model (5-10 minutes)
  dataModel: {
    entities: {
      User: {
        id: 'string',
        email: 'string',
        profile: 'Profile',
        createdAt: 'timestamp',
      },
      Post: {
        id: 'string',
        authorId: 'string',
        content: 'string',
        likes: 'number',
        comments: 'Comment[]',
      },
    },

    relationships: [
      'User has many Posts',
      'Post belongs to User',
      'Post has many Comments',
    ],

    storage: {
      client: 'IndexedDB for offline',
      cache: 'Redis for sessions',
      database: 'PostgreSQL for core data',
      files: 'S3 for uploads',
    },
  },

  // Step 4: Interface Definitions (5-10 minutes)
  interfaces: {
    api: {
      'POST /api/auth/login': {
        body: { email: 'string', password: 'string' },
        response: { token: 'string', user: 'User' },
      },
      'GET /api/posts': {
        query: { page: 'number', limit: 'number' },
        response: { posts: 'Post[]', total: 'number' },
      },
      'POST /api/posts': {
        body: { content: 'string' },
        response: { post: 'Post' },
      },
    },

    events: {
      'post:created': { postId: 'string' },
      'post:liked': { postId: 'string', userId: 'string' },
      'user:online': { userId: 'string' },
    },
  },

  // Step 5: Optimizations & Deep Dives (10-15 minutes)
  optimizations: {
    performance: [
      'Code splitting by route',
      'Image lazy loading',
      'Virtual scrolling for lists',
      'Debounce search input',
      'Prefetch on hover',
    ],

    caching: [
      'Browser cache for static assets',
      'Service worker for offline',
      'React Query for server state',
      'CDN for global distribution',
    ],

    scalability: [
      'Horizontal scaling with load balancer',
      'Database sharding by user_id',
      'Read replicas for queries',
      'Message queue for async tasks',
    ],
  },
};
```

## 26.2 Common Frontend System Designs

### Design: News Feed (Twitter/Facebook Clone)

```typescript
const newsFeedDesign = {
  requirements: {
    functional: [
      'Users can post updates',
      'Feed shows posts from followed users',
      'Real-time updates',
      'Like/comment on posts',
      'Infinite scroll',
      'Image/video uploads',
    ],

    scale: {
      users: '10M daily active',
      posts: '100M posts/day',
      reads: '10B feed views/day',
    },
  },

  architecture: {
    frontend: {
      framework: 'React with TypeScript',
      stateManagement: 'React Query + Zustand',
      routing: 'React Router',
      realTime: 'WebSocket connection',
    },

    backend: {
      api: 'GraphQL (flexible queries)',
      database: 'PostgreSQL + Redis',
      storage: 'S3 for media',
      cdn: 'CloudFront for delivery',
      websocket: 'Socket.io server',
    },
  },

  dataModel: {
    Post: {
      id: 'uuid',
      userId: 'uuid',
      content: 'text',
      mediaUrls: 'string[]',
      likes: 'number',
      createdAt: 'timestamp',
      updatedAt: 'timestamp',
    },

    Feed: {
      userId: 'uuid',
      postIds: 'uuid[]', // Pre-computed feed
      lastUpdated: 'timestamp',
    },
  },

  challenges: {
    feedGeneration: {
      problem: 'Computing feed for 10M users is expensive',
      solution: {
        approach: 'Fan-out on write',
        implementation: `
          When user posts:
          1. Store post in database
          2. Queue job to update followers' feeds
          3. Write post_id to each follower's feed cache (Redis)
          4. Notify online followers via WebSocket
        `,
        tradeoff: 'More writes, but faster reads',
      },
    },

    scalability: {
      problem: 'Feed queries need to be fast',
      solutions: [
        'Cache feeds in Redis (pre-computed)',
        'Paginate with cursor-based pagination',
        'Use read replicas for queries',
        'CDN for media files',
      ],
    },

    realTime: {
      problem: 'Show new posts without refresh',
      solutions: [
        'WebSocket for instant updates',
        'SSE for one-way notifications',
        'Long polling as fallback',
        'Optimistic UI updates',
      ],
    },
  },

  optimizations: {
    performance: [
      'Virtual scrolling for long feeds',
      'Image lazy loading',
      'Prefetch next page on scroll',
      'Compress images before upload',
      'Use WebP format',
    ],

    caching: [
      'Feed cache in Redis (30min TTL)',
      'Browser cache for static assets',
      'Service worker for offline viewing',
      'Prefetch followed users\' latest posts',
    ],
  },

  monitoring: [
    'Feed load time < 2s',
    'Post creation < 500ms',
    'WebSocket connection status',
    'Error rate < 0.1%',
    'Cache hit rate > 90%',
  ],
};
```

### Design: Real-Time Collaborative Editor (Google Docs)

```typescript
const collaborativeEditorDesign = {
  requirements: {
    functional: [
      'Multiple users edit simultaneously',
      'See others\' cursors in real-time',
      'Conflict resolution',
      'Undo/redo',
      'Auto-save',
      'Version history',
    ],

    scale: {
      users: '1M documents',
      concurrent: '100 users per document',
      latency: '< 100ms for updates',
    },
  },

  architecture: {
    frontend: {
      editor: 'ProseMirror / Slate.js',
      realTime: 'WebSocket',
      conflictResolution: 'Operational Transformation',
      storage: 'IndexedDB for offline',
    },

    backend: {
      sync: 'WebSocket server',
      storage: 'PostgreSQL for documents',
      cache: 'Redis for active sessions',
      queue: 'RabbitMQ for async operations',
    },
  },

  keyTechnologies: {
    operationalTransformation: {
      purpose: 'Resolve conflicting edits',
      example: `
        User A: Insert "hello" at position 0
        User B: Insert "world" at position 0

        OT transforms operations so both succeed:
        User A sees: "hello world"
        User B sees: "hello world"
        (not "world hello" or "hwelloorld")
      `,
      implementation: 'Use ShareDB or Yjs library',
    },

    presence: {
      purpose: 'Show who is online and where',
      implementation: {
        cursorTracking: `
          1. User moves cursor
          2. Throttle updates (every 50ms)
          3. Broadcast position via WebSocket
          4. Render others' cursors with color coding
        `,
        activeUsers: `
          1. Maintain presence map in Redis
          2. Heartbeat every 30s
          3. Remove after 60s of inactivity
          4. Broadcast user join/leave events
        `,
      },
    },

    autoSave: {
      strategy: 'Debounced save on changes',
      implementation: `
        1. User types
        2. Debounce 2 seconds
        3. Generate delta (diff from last save)
        4. Send delta to server
        5. Server applies + stores
        6. Confirm to client
      `,
      offline: `
        1. Queue changes in IndexedDB
        2. Sync when reconnected
        3. Resolve conflicts with OT
      `,
    },
  },

  challenges: {
    conflictResolution: {
      problem: 'Two users edit same position',
      solution: 'Operational Transformation',
      alternative: 'CRDT (Conflict-free Replicated Data Type)',
      tradeoff: 'OT: Complex but precise | CRDT: Simpler but eventual consistency',
    },

    scalability: {
      problem: '100 concurrent users per document',
      solutions: [
        'Use dedicated rooms per document',
        'Load balance WebSocket connections',
        'Compress messages',
        'Batch operations when possible',
      ],
    },

    offline: {
      problem: 'Continue editing when disconnected',
      solutions: [
        'Store document in IndexedDB',
        'Queue operations locally',
        'Sync on reconnect',
        'Show offline indicator',
        'Handle conflicts on sync',
      ],
    },
  },

  optimizations: [
    'Compress WebSocket messages',
    'Batch cursor updates',
    'Use binary protocol for operations',
    'Lazy load document history',
    'Prune old operations (keep snapshots)',
  ],
};
```

### Design: Image Sharing Platform (Instagram-like)

```typescript
const imageSharingDesign = {
  requirements: {
    functional: [
      'Upload images/videos',
      'Image filters and editing',
      'Feed of posts',
      'Like/comment',
      'User profiles',
      'Search and discovery',
    ],

    scale: {
      uploads: '10M photos/day',
      storage: '100TB total',
      traffic: '1B requests/day',
    },
  },

  architecture: {
    frontend: {
      imageHandling: 'Client-side resize before upload',
      filtering: 'Canvas API / WebGL',
      optimization: 'Lazy loading + progressive images',
    },

    backend: {
      upload: 'Direct to S3 with presigned URLs',
      processing: 'Lambda functions for thumbnails',
      cdn: 'CloudFront for delivery',
      database: 'DynamoDB for metadata',
    },
  },

  imageProcessing: {
    clientSide: {
      resize: `
        1. User selects image
        2. Create canvas element
        3. Resize to max 2048px
        4. Compress to ~200KB
        5. Generate preview
        6. Upload to S3
      `,

      code: `
        async function resizeImage(file: File): Promise<Blob> {
          const img = await createImageBitmap(file);
          const canvas = document.createElement('canvas');

          const maxSize = 2048;
          const ratio = Math.min(maxSize / img.width, maxSize / img.height);

          canvas.width = img.width * ratio;
          canvas.height = img.height * ratio;

          const ctx = canvas.getContext('2d')!;
          ctx.drawImage(img, 0, 0, canvas.width, canvas.height);

          return new Promise((resolve) => {
            canvas.toBlob(resolve, 'image/jpeg', 0.85);
          });
        }
      `,
    },

    serverSide: {
      thumbnails: 'Generate multiple sizes (150px, 300px, 600px)',
      format: 'Convert to WebP for smaller size',
      metadata: 'Extract EXIF data, geolocation',
      storage: {
        original: 'S3 Standard',
        thumbnails: 'S3 with CloudFront',
      },
    },
  },

  imageDelivery: {
    responsiveImages: `
      <picture>
        <source
          type="image/webp"
          srcset="
            /img/photo-150.webp 150w,
            /img/photo-300.webp 300w,
            /img/photo-600.webp 600w
          "
        />
        <img
          src="/img/photo-600.jpg"
          alt="Description"
          loading="lazy"
          decoding="async"
        />
      </picture>
    `,

    progressive: {
      technique: 'LQIP (Low Quality Image Placeholder)',
      implementation: `
        1. Show blurred thumbnail (20px, base64)
        2. Load full image in background
        3. Fade in when loaded
      `,
    },

    cdn: {
      strategy: 'CloudFront with custom domain',
      caching: 'Cache-Control: max-age=31536000',
      invalidation: 'Only when image deleted',
    },
  },

  challenges: {
    uploadSpeed: {
      problem: 'Large files = slow uploads',
      solutions: [
        'Client-side compression',
        'Multipart upload for > 5MB',
        'Direct to S3 (bypass server)',
        'Upload in background',
        'Progressive upload (show preview)',
      ],
    },

    storage: {
      problem: '100TB of images',
      solutions: [
        'S3 Intelligent-Tiering',
        'Lifecycle policies (archive old images)',
        'Deduplication (hash-based)',
        'Delete unused thumbnails',
      ],
    },

    performance: {
      problem: 'Feed with many images',
      solutions: [
        'Lazy load below fold',
        'Virtual scrolling',
        'Blur placeholders',
        'Prefetch next images',
        'Use CDN',
      ],
    },
  },
};
```

## 26.3 Discussing Trade-offs

### Common Trade-off Questions

```typescript
const tradeoffDiscussions = {
  clientVsServerRendering: {
    question: 'Should we use CSR, SSR, or SSG?',

    options: {
      CSR: {
        pros: ['Simple deployment', 'Rich interactivity', 'Lower server cost'],
        cons: ['Slow initial load', 'Poor SEO', 'Large bundle size'],
        when: 'Authenticated dashboard, admin panel',
      },

      SSR: {
        pros: ['Fast initial load', 'Good SEO', 'Dynamic content'],
        cons: ['Complex infrastructure', 'Higher server cost', 'TTFB delay'],
        when: 'E-commerce, news sites, social media',
      },

      SSG: {
        pros: ['Fastest load time', 'Cheap hosting', 'Great SEO'],
        cons: ['Rebuild on changes', 'Not for dynamic content'],
        when: 'Blogs, documentation, marketing sites',
      },
    },

    recommendation: `
      Use SSG for static pages (landing, about)
      Use SSR for dynamic pages (product listings)
      Use CSR for app-like features (dashboard)

      Or use Next.js and choose per-route!
    `,
  },

  restVsGraphQL: {
    question: 'REST or GraphQL for API?',

    options: {
      REST: {
        pros: ['Simple', 'Cacheable', 'Widely understood'],
        cons: ['Over-fetching', 'Multiple requests', 'Versioning needed'],
        when: 'Simple CRUD, public API, caching important',
      },

      GraphQL: {
        pros: ['Flexible queries', 'Single request', 'Strong typing'],
        cons: ['Complex setup', 'Harder caching', 'Learning curve'],
        when: 'Complex data requirements, mobile apps',
      },
    },

    hybrid: `
      Use REST for simple endpoints (auth, upload)
      Use GraphQL for complex data fetching (feed, profile)
    `,
  },

  centralized VsDistributed: {
    question: 'Where should state live?',

    options: {
      centralized: {
        approach: 'Redux/Zustand global store',
        pros: ['Single source of truth', 'Predictable', 'Time-travel debugging'],
        cons: ['Boilerplate', 'Overkill for simple apps'],
      },

      distributed: {
        approach: 'React Query + local useState',
        pros: ['Less boilerplate', 'Automatic caching', 'Optimized re-renders'],
        cons: ['State spread across components'],
      },
    },

    recommendation: `
      Server state: React Query
      Global UI state: Zustand (theme, modals)
      Local state: useState
      URL state: React Router
    `,
  },
};
```

## 26.4 Handling Ambiguity

### Clarifying Questions Template

```typescript
const clarificationQuestions = {
  scope: [
    'Is this web-only or do we need mobile apps?',
    'What browsers do we need to support?',
    'Are there any compliance requirements (GDPR, HIPAA)?',
    'What\'s the timeline for MVP vs full launch?',
  ],

  scale: [
    'How many users do we expect in year 1? Year 3?',
    'What\'s the expected traffic pattern? (steady, spiky, seasonal)',
    'What regions are we targeting?',
    'What\'s the acceptable latency?',
  ],

  features: [
    'What are the must-have features for MVP?',
    'What features can we defer to v2?',
    'Are there any similar products we should reference?',
    'What makes this product unique?',
  ],

  technical: [
    'Do we have any existing infrastructure to use?',
    'Are there any technology constraints?',
    'What\'s the team\'s expertise?',
    'What\'s the budget for infrastructure?',
  ],

  quality: [
    'What\'s acceptable downtime? (99%, 99.9%, 99.99%)',
    'What performance targets? (Time to Interactive, etc.)',
    'What browsers/devices must we support?',
    'What accessibility level? (WCAG A, AA, AAA)',
  ],
};

// How to handle "I don't know"
const handleUncertainty = {
  makeAssumptions: `
    "I'll assume we're targeting modern browsers (last 2 versions)
     and expecting about 100K daily active users. If that's not correct,
     let me know and I'll adjust."
  `,

  stateConstraints: `
    "Given we're a startup, I'll optimize for speed of development
     and assume we can refactor later as we scale. We can use
     managed services to reduce operational complexity."
  `,

  askForPriorities: `
    "What's more important for this use case:
     - Time to market?
     - Operational cost?
     - Developer experience?
     - User experience?
    This will help me make better trade-offs."
  `,
};
```

## 26.5 Communication Tips

### Thinking Out Loud

```typescript
const communicationTechniques = {
  narrateThinking: {
    bad: *silence while thinking*,

    good: `
      "Let me think through the data flow here...
       User uploads image -> we need to validate size client-side...
       then upload to S3... that triggers Lambda for thumbnails...
       which updates the database... and we can show a placeholder
       while processing. Does that make sense so far?"
    `,
  },

  checkUnderstanding: {
    bad: "Does this make sense?",

    good: `
      "So far I've designed the upload flow. Before I move on to
       the feed generation, do you have any questions about the
       upload process? Is there anything I should reconsider?"
    `,
  },

  inviteInput: {
    bad: "This is how we'll do it.",

    good: `
      "I'm thinking we use Redis for caching. What do you think?
       Are there any concerns I should address?"
    `,
  },

  trackTime: {
    strategy: `
      - Glance at clock periodically
      - If 20 min in and still on requirements, move faster
      - If 40 min in and haven't discussed optimizations, speed up
      - Leave 5 minutes for questions
    `,
  },

  useWhiteboard: {
    tips: [
      'Draw as you talk',
      'Use boxes for components',
      'Arrows for data flow',
      'Label everything clearly',
      'Group related components',
      'Use colors for emphasis',
    ],
  },
};
```

## Real-World Example: Design a Video Streaming Platform

```typescript
const videoStreamingDesign = {
  // 1. Requirements (5 min)
  requirements: {
    interviewer: "Design a video streaming platform like YouTube",

    you: `
      Great! Let me clarify a few things:

      - Scale: How many users? Daily video uploads?
      - Features: Just playback or also upload? Comments? Likes?
      - Quality: 4K support? Adaptive bitrate?
      - Regions: Global or specific region?
      - Devices: Web only or mobile apps too?

      [After clarification]

      Okay, so we're building:
      - Web-only (for now)
      - 10M daily active users
      - 100K video uploads per day
      - Adaptive streaming (360p to 1080p)
      - Features: upload, playback, like, comment, subscriptions

      For MVP, we'll defer:
      - Live streaming
      - 4K support
      - Mobile apps

      Is this aligned with your thinking?
    `,
  },

  // 2. Architecture (15 min)
  architecture: `
    [Your explanation while drawing]

    "Let me start with the high-level architecture...

    CLIENT SIDE:
    - React app (Next.js for SSR on video pages)
    - Video player (Video.js for adaptive streaming)
    - Upload component with chunking
    - React Query for data fetching

    UPLOAD FLOW:
    - User uploads video → multipart upload to S3
    - Triggers video processing pipeline
    - FFmpeg in Lambda transcodes to multiple qualities
    - Generates thumbnails
    - Updates video metadata in database
    - Notifies user when processing complete

    PLAYBACK FLOW:
    - Request video → fetch metadata from DB
    - Serve video chunks from CDN
    - Adaptive bitrate based on network
    - Track analytics (view count, watch time)

    INFRASTRUCTURE:
    - CloudFront CDN for video delivery
    - S3 for video storage
    - RDS for metadata
    - ElastiCache for caching
    - SQS for processing queue

    Does this make sense? Any concerns so far?"
  `,

  // 3. Deep Dive: Video Upload (10 min)
  uploadDeepDive: `
    "Let's dive into the upload flow since it's critical...

    CLIENT-SIDE:
    1. User selects video file
    2. Validate: size < 2GB, format supported
    3. Generate thumbnail preview
    4. Chunk file into 5MB pieces
    5. Upload chunks in parallel (4 at a time)
    6. Show progress bar
    7. Retry failed chunks

    CODE SKETCH:

    async function uploadVideo(file: File) {
      // 1. Get presigned URLs for multipart upload
      const { uploadId, urls } = await api.initUpload({
        filename: file.name,
        size: file.size
      });

      // 2. Chunk and upload
      const chunkSize = 5 * 1024 * 1024; // 5MB
      const chunks = Math.ceil(file.size / chunkSize);

      const uploads = [];
      for (let i = 0; i < chunks; i++) {
        const chunk = file.slice(
          i * chunkSize,
          (i + 1) * chunkSize
        );

        uploads.push(
          uploadChunk(chunk, urls[i], (progress) => {
            updateProgress(i, progress);
          })
        );

        // Limit concurrency
        if (uploads.length >= 4) {
          await Promise.race(uploads);
        }
      }

      await Promise.all(uploads);

      // 3. Complete upload
      await api.completeUpload(uploadId);
    }

    PROCESSING:
    - Upload complete → SQS message
    - Worker picks up job
    - Transcode to 360p, 720p, 1080p
    - Generate 10 thumbnails
    - Create HLS manifest
    - Update database
    - Send notification

    OPTIMIZATIONS:
    - Resume failed uploads
    - Background upload (Service Worker)
    - Client-side compression
    - Parallel processing

    This ensures reliable uploads even for large files and poor connections."
  `,

  // 4. Optimizations (5 min)
  optimizations: `
    "For optimizations, I'd focus on:

    PERFORMANCE:
    - CDN for global video delivery
    - Preload video metadata
    - Progressive video loading
    - Lazy load recommendations

    COST:
    - S3 Intelligent-Tiering
    - Compress thumbnails to WebP
    - Archive old videos
    - Cache popular videos closer to users

    SCALABILITY:
    - Separate upload and playback services
    - Use message queues for async processing
    - Database read replicas
    - Shard by user_id for writes

    Would you like me to dive deeper into any of these?"
  `,
};
```

## Chapter Exercise: Practice System Design

Practice designing these systems:

**Level 1 (Beginner):**
1. URL Shortener
2. Todo List App
3. Chat Application

**Level 2 (Intermediate):**
4. E-commerce Product Page
5. Social Media Feed
6. Real-time Collaborative Whiteboard

**Level 3 (Advanced):**
7. Video Streaming Platform
8. Ride-sharing App (Uber-like)
9. Google Docs Clone

**Practice Tips:**
- Time yourself (45 minutes)
- Draw diagrams
- Think out loud
- Consider trade-offs
- Discuss monitoring and scaling

## Review Checklist

- [ ] RADIO framework understood
- [ ] Common patterns practiced
- [ ] Trade-offs clearly articulated
- [ ] Clarifying questions prepared
- [ ] Whiteboarding skills sharp
- [ ] Communication clear and structured
- [ ] Time management practiced
- [ ] Deep dives prepared for common topics

## Key Takeaways

1. **Clarify before designing** - Understand requirements first
2. **Think holistically** - Consider full system, not just code
3. **Discuss trade-offs** - No perfect solution exists
4. **Communicate clearly** - Think out loud, draw diagrams
5. **Start broad, go deep** - High-level first, details later
6. **Know common patterns** - Caching, CDN, queues, etc.
7. **Practice regularly** - System design is a learnable skill

## Further Reading

- Designing Data-Intensive Applications (Martin Kleppmann)
- System Design Interview (Alex Xu)
- Grokking the System Design Interview
- Frontend System Design Guide (greatfrontend.com)
- Web Scalability for Startup Engineers

## Next Steps

You've completed all technical chapters! Continue to:
- [Chapter 27: Capstone Project](./27-capstone-project.md) - Already complete!
- [Chapter 28: Your Senior Developer Roadmap](./28-roadmap.md) - Already complete!

Congratulations on finishing the technical deep dives!
