# EduAgent - Product Requirements Document

> **Document Status:** Approved for Development  
> **Version:** 1.0  
> **Last Updated:** 2024-12-10

---

## Executive Summary

**EduAgent** is an AI-powered personalized tutoring app that provides adaptive, conversational learning experiences. Unlike static courses or video content, EduAgent teaches through dialogue—explaining concepts, answering questions, testing understanding, and building upon previous knowledge.

### Vision

*"Everyone deserves a personal tutor who adapts to their learning style, remembers what they've learned, and is available 24/7."*

### Core Value Proposition

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                  │
│  TRADITIONAL LEARNING          vs.       EDUAGENT                │
│  ─────────────────────                   ────────                │
│  Fixed curriculum              →         Personalized path       │
│  One-size-fits-all             →         Adapts to your level    │
│  Static content                →         Interactive dialogue    │
│  No memory of prior lessons    →         Builds on what you know │
│  Expensive tutors ($50+/hr)    →         $9.99/month unlimited   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Table of Contents

1. [Product Overview](#product-overview)
2. [Target Users](#target-users)
3. [User Stories](#user-stories)
4. [Feature Requirements](#feature-requirements)
5. [Non-Functional Requirements](#non-functional-requirements)
6. [Technical Architecture](#technical-architecture)
7. [Data Model](#data-model)
8. [API Overview](#api-overview)
9. [AI System](#ai-system)
10. [Business Model](#business-model)
11. [Risks & Mitigations](#risks--mitigations)
12. [Success Metrics](#success-metrics)
13. [Roadmap](#roadmap)
14. [Related Documents](#related-documents)

---

## Product Overview

### What is EduAgent?

EduAgent is a mobile-first learning application (iOS, Android, Web) where users learn through conversation with an AI tutor. The AI:

- **Teaches** concepts through dialogue, not videos
- **Remembers** what users have learned previously
- **Adapts** explanations to user's demonstrated level
- **Tests** understanding with personalized assessments
- **Tracks** progress and encourages daily learning habits

### Key Differentiators

| Competitor | Their Approach | Our Approach |
|------------|----------------|--------------|
| Duolingo | Gamified drills | Conversational understanding |
| Coursera/Udemy | Video lectures | Interactive dialogue |
| ChatGPT | General chat | Structured curriculum + memory |
| Khan Academy | Videos + exercises | Personalized AI tutoring |

### Why Now?

1. **LLM capability** - Claude/GPT-4 can now teach effectively
2. **Mobile-first learning** - Users prefer apps over desktop
3. **Cost democratization** - AI tutoring at scale is now affordable
4. **Learning continuity** - We can now maintain context across sessions

---

## Target Users

### Primary Personas

#### 1. The Self-Learner (60% of users)

```
Name: Alex, 28
Occupation: Marketing professional
Goal: Learn Python for data analysis
Pain points:
- Can't afford coding bootcamp
- Online courses feel impersonal
- Forgets concepts between sessions
- Needs flexible schedule (learns at night)

What they want:
✓ Learn at own pace
✓ Ask questions without judgment
✓ Pick up exactly where left off
✓ Feel progress, not frustration
```

#### 2. The Student Supplement (30% of users)

```
Name: Maya, 16
Occupation: High school student
Goal: Better understand calculus
Pain points:
- School moves too fast
- Embarrassed to ask "dumb" questions
- Khan Academy videos aren't interactive
- Needs explanations in different ways

What they want:
✓ Explain things multiple ways
✓ Practice problems that match skill level
✓ Available when studying late at night
✓ Actually understand, not just memorize
```

#### 3. The Career Changer (10% of users)

```
Name: Sam, 35
Occupation: Accountant → aspiring data scientist
Goal: Learn machine learning fundamentals
Pain points:
- Already knows math, doesn't need basics
- Courses assume too much or too little
- Limited time, needs efficient learning
- Needs structured path to job-readiness

What they want:
✓ Skip what they know, focus on gaps
✓ Realistic time estimates
✓ Industry-relevant skills
✓ Portfolio-worthy projects (future)
```

### Age Range

- **MVP:** 13+ (teenagers and adults)
- **Future:** 6+ (with simplified UX and parental controls)

### Geographic Focus

- **MVP:** English-speaking markets (US, UK, Canada, Australia)
- **Future:** Global with LLM-powered translation

---

## User Stories

### Epic 1: Onboarding & Interview

| ID | As a... | I want to... | So that... | Priority |
|----|---------|--------------|------------|----------|
| U1.1 | New user | Sign up with email or Google | I can create an account quickly | P0 |
| U1.2 | New user | Tell AI what I want to learn (any subject) | I get personalized content for MY goal | P0 |
| U1.3 | New user | Have a short interview conversation | AI understands my goals, background, and level | P0 |
| U1.4 | New user | Opt for a thorough assessment if I want | I can invest more time for better personalization | P1 |
| U1.5 | New user | See my AI-generated learning path | I understand the journey ahead | P0 |
| U1.6 | New user | See why topics are in a specific order | I trust the AI's curriculum design | P1 |
| U1.7 | New user | Skip topics I already know | I don't waste time on basics | P0 |
| U1.8 | New user | Challenge the curriculum order | I feel in control of my learning | P1 |
| U1.9 | New user | Start my first lesson immediately | I experience value in first session | P0 |

### Epic 1b: Curriculum Management

| ID | As a... | I want to... | So that... | Priority |
|----|---------|--------------|------------|----------|
| U1b.1 | Learner | View my full curriculum at any time | I can plan my learning | P0 |
| U1b.2 | Learner | Ask "why this order?" for any topic | I understand the pedagogy | P1 |
| U1b.3 | Learner | Challenge and regenerate my curriculum | I can correct AI mistakes | P1 |
| U1b.4 | Learner | See confidence badges (Core/Contemporary/Emerging) | I know what's established vs. cutting-edge | P2 |
| U1b.5 | Learner | Have my curriculum adapt after each module | It stays relevant to my progress | P0 |
| U1b.6 | Learner | Flag if curriculum seems wrong | Errors get logged and reviewed | P1 |

### Epic 2: Learning Experience

| ID | As a... | I want to... | So that... | Priority |
|----|---------|--------------|------------|----------|
| U2.1 | Learner | Chat with AI about my current topic | I learn through conversation | P0 |
| U2.2 | Learner | Ask follow-up questions | I clarify my understanding | P0 |
| U2.3 | Learner | Have AI remember what I learned before | I don't repeat basics | P0 |
| U2.4 | Learner | See AI reference my prior knowledge | I feel learning continuity | P0 |
| U2.5 | Learner | Request simpler/complex explanations | Content matches my level | P1 |
| U2.6 | Learner | Flag if something seems wrong | Errors get corrected | P2 |

### Epic 3: Assessment & Retention

| ID | As a... | I want to... | So that... | Priority |
|----|---------|--------------|------------|----------|
| U3.1 | Learner | Be quizzed during lessons | I confirm understanding | P0 |
| U3.2 | Learner | Explain my reasoning, not just give answers | AI catches my first error in thinking | P0 |
| U3.3 | Learner | Get feedback on WHERE I went wrong (not just "wrong") | I know exactly what to fix | P0 |
| U3.4 | Learner | See my mastery level per topic | I know what I've mastered | P0 |
| U3.5 | Learner | Review summaries of past lessons | I refresh my memory | P0 |
| U3.6 | Learner | Request a re-test on old topics | I verify retention | P1 |
| U3.7 | Learner | Get spaced repetition reminders | I don't forget over time | P1 |

### Epic 4: Progress & Motivation

| ID | As a... | I want to... | So that... | Priority |
|----|---------|--------------|------------|----------|
| U4.1 | Learner | See my progress through the path | I feel accomplishment | P0 |
| U4.2 | Learner | See my knowledge decay over time | I know what needs review | P0 |
| U4.3 | Learner | Earn verified XP from recall tests | XP reflects real knowledge | P0 |
| U4.4 | Learner | Maintain an honest streak | I stay motivated to review | P1 |
| U4.5 | Learner | See "completed vs verified" topics | I understand true mastery | P1 |
| U4.6 | Learner | Get reminded to review fading topics | I maintain my knowledge | P1 |

### Epic 5: Subscription

| ID | As a... | I want to... | So that... | Priority |
|----|---------|--------------|------------|----------|
| U5.1 | Free user | See when I hit session limit | I understand the paywall | P1 |
| U5.2 | Free user | Upgrade to premium | I get unlimited learning | P2 |
| U5.3 | Premium user | Cancel my subscription | I control my payment | P2 |
| U5.4 | Any user | Delete my account | I exercise my privacy rights | P1 |

---

## Feature Requirements

### Core Features (MVP)

#### F1: User Authentication
- Email + password registration
- Google OAuth
- Email verification
- Password reset
- Session persistence

#### F2: Interview & Dynamic Curriculum

**Interview (Onboarding):**
- Conversational interview with AI (not a quiz)
- 4 phases: Goal → Background → Constraints → Spot Check
- Quick mode (~3 min) or Thorough mode (~5-7 min, opt-in)
- Interview summary stored for curriculum generation
- Correction mode for regeneration requests

**Curriculum Generation:**
- AI-generated curriculum for ANY subject
- Source constraints: textbooks, university syllabi, peer-reviewed only
- Progressive generation: Module 1 in detail, future modules as preview
- Confidence tagging: Core / Recommended / Contemporary / Emerging
- Prerequisite relationships based on verified pedagogical sources
- Estimated time per topic (realistic, not optimistic)

**Curriculum Visibility:**
- User sees full curriculum with learning outcomes
- "Why this order?" explanation on demand
- Skip topics user already knows
- Challenge and regenerate based on user feedback
- Curriculum adapts after each module based on performance

**Curriculum Feedback (Cross-User Learning):**
- Aggregate signals: completion rates, prerequisite failures, skips
- Logged for future prompt improvements (MVP: log only, don't act)

#### F3: Conversational Learning
- Real-time chat with AI tutor
- Streaming responses
- Prior knowledge injection
- Adaptive explanations
- Example generation
- **Cognitive load management (chunking)**:
  - 1-2 concepts per message maximum
  - Understanding checks after each concept
  - Confusion detection → immediate simplification
  - Multi-turn progressive disclosure for complex topics
- **Worked examples with fading (d=0.57)**:
  - Novices: Full step-by-step worked examples
  - Developing: Partial examples (fading)
  - Competent: Problem-first (examples only if struggling)
  - Automatic mode transition based on performance

#### F4: Learning Continuity
- Topic summaries auto-generated
- Summaries accessible for review
- AI references prior learning
- Context passed between sessions
- Retention scoring

#### F4.1: Learning Book (My Notes)

**The user's personal learning journal — browse, review, and re-test past learning.**

```
┌─────────────────────────────────────────────────────────────────┐
│  📚 MY LEARNING BOOK                                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  📖 Python Fundamentals                         12/24 topics    │
│     Progress: ████████████░░░░░░░░░  50%                        │
│     Last studied: 2 days ago                                    │
│                                                                  │
│  ├── ✓ Variables & Data Types    ████████░░ Strong   [Review]  │
│  ├── ✓ Control Flow              ██████░░░░ Fading   [Review]  │
│  ├── ✓ Functions                 ███░░░░░░░ Weak ⚠️  [Review]  │
│  ├── ✓ Lists & Dictionaries      ████████░░ Strong   [Review]  │
│  ├── 🔒 Classes & Objects        ░░░░░░░░░░ Locked             │
│  └── 🔒 Error Handling           ░░░░░░░░░░ Locked             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Topic Review View:**

```
┌─────────────────────────────────────────────────────────────────┐
│  📖 REVIEW: Functions                                            │
│  Learned: December 5, 2024 (5 days ago)                          │
│  Retention: ████░░░░░░ 40% (needs review)                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  📝 KEY CONCEPTS                                                 │
│  • Functions are defined with `def function_name():`            │
│  • Parameters go inside parentheses                             │
│  • Return sends a value back to caller                          │
│  • Default parameters: `def greet(name="World"):`               │
│                                                                  │
│  💡 EXAMPLES WE USED                                             │
│  • Calculator functions (add, subtract)                         │
│  • Recipe analogy: function = recipe, parameters = ingredients  │
│                                                                  │
│  📋 YOUR SUMMARY                                                 │
│  "You learned how functions encapsulate reusable logic.         │
│   You initially confused return with print but clarified        │
│   that return sends data back while print just displays it."    │
│                                                                  │
│  [Quiz Me Now]     [Re-learn This Topic]     [Next Topic →]     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Actions from Learning Book:**
- **Review:** Read your summary and key concepts
- **Quiz Me:** Take a quick retention test (updates verified XP)
- **Re-learn:** Start a fresh lesson with AI remembering your past attempts

#### F5: Assessments
- In-lesson quick checks
- Topic completion assessments
- Re-testing from summaries
- Multiple question types
- Instant feedback

#### F6: Outcome-Based Gamification

**Philosophy:** Reward what matters — knowledge retention, not app opens.

| Traditional Gamification | EduAgent Gamification |
|--------------------------|------------------------|
| Complete topic → +50 XP | Complete topic → 0 XP (pending) |
| Daily streak = opened app | Honest streak = passed recall |
| XP never decays | XP decays without review |
| Badge for "first lesson!" | Badge for "90%+ recall at 6 weeks" |

**Core Mechanics:**

1. **Retention XP (not completion XP)**
   - Complete topic → XP is *pending verification*
   - Pass 2-week recall → +30 verified XP
   - Pass 6-week recall → +50 verified XP
   - Fail recall → XP decays
   - User sees: "20 topics completed, 12 verified"

2. **Knowledge Half-Life (visual decay)**
   - Progress bars that fade over time
   - "Strong" (tested recently) → "Almost gone" (untested)
   - Creates real urgency — knowledge IS decaying
   - [Review Weakest] button drives engagement

3. **Honest Streak**
   - Only counts days where user passed a recall challenge
   - NOT days the app was opened
   - Streak "pauses" for 3 days (not instant break)

4. **Interleaved Retrieval Sessions** (d=1.21 effect size)
   - Multiple topics due → combined into one mixed session
   - Questions interleaved (never same topic twice in a row)
   - Topic HIDDEN until after answer (forces discrimination)
   - "Was this testing Functions or Loops?" → deeper processing
   - SM-2 scheduling determines what's due each day

5. **Stability Milestones**
   - Topic becomes "Stable" after 5+ consecutive successful retrievals
   - Stable topics: 60+ day intervals, less frequent review
   - Progress from "Learning" → "Stable" is visible and celebrated

**Learning Mode Selection (Onboarding):**

```
┌─────────────────────────────────────────────────────────────────┐
│  CHOOSE YOUR LEARNING STYLE                                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  🎓 SERIOUS LEARNER (Recommended)                                │
│     "I want to actually retain what I learn"                     │
│     → Mastery gates, verified XP, honest streak                  │
│     → Harder but more effective (research-backed)                │
│                                                                  │
│  🌱 CASUAL EXPLORER                                              │
│     "I want to learn at my own pace, no pressure"               │
│     → No gates, completion XP, activity streak                   │
│     → Easier but less structured                                 │
│                                                                  │
│  [Choose Serious] [Choose Casual]                               │
│                                                                  │
│  "You can change this anytime in Settings"                       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

This lets casual learners opt out of anxiety-inducing features while serious learners get the full research-backed experience.

**Why This Matters:**
- Creates real switching cost (your learning history means something)
- Differentiates from Duolingo-style vanity metrics
- Appeals to "serious learner" target persona

#### F7: Free/Premium Tiers
- 3 sessions/day free limit
- Unlimited premium tier
- Web-based Stripe checkout

---

## Non-Functional Requirements

### Performance

| Requirement | Target | Notes |
|-------------|--------|-------|
| API response time | <200ms (p95) | Excluding LLM calls |
| LLM first token | <2s | Streaming response start |
| App cold start | <3s | On modern devices |
| WebSocket latency | <100ms | Real-time feel |

### Reliability

| Requirement | Target | Notes |
|-------------|--------|-------|
| Uptime | 99.5% | Excluding planned maintenance |
| Data durability | 99.99% | Supabase managed |
| LLM fallback | Yes | OpenAI backup |

### Security

| Requirement | Implementation |
|-------------|---------------|
| Authentication | JWT via Supabase Auth |
| Data encryption | TLS in transit, AES at rest |
| API security | Rate limiting, input validation |
| Privacy | GDPR compliance, account deletion |
| Age restriction | 13+ ToS, no PII from minors |

### Scalability

| Phase | Users | Infrastructure |
|-------|-------|----------------|
| MVP | 1-1,000 | Railway, Supabase free/pro |
| Growth | 1,000-50,000 | Railway scaled, Supabase pro |
| Scale | 50,000+ | AWS/GCP migration |

---

## Technical Architecture

### Stack Summary

```
┌─────────────────────────────────────────────────────────────────┐
│  FRONTEND                                                        │
│  └── Expo (React Native + Web) / TypeScript                      │
├─────────────────────────────────────────────────────────────────┤
│  BACKEND                                                         │
│  └── Python + FastAPI + LangGraph                                │
├─────────────────────────────────────────────────────────────────┤
│  DATABASE                                                        │
│  └── Supabase (PostgreSQL + Auth + Storage + Realtime)           │
├─────────────────────────────────────────────────────────────────┤
│  CACHE                                                           │
│  └── Upstash Redis                                               │
├─────────────────────────────────────────────────────────────────┤
│  AI/LLM                                                          │
│  └── Anthropic Claude (primary) + OpenAI (fallback)              │
├─────────────────────────────────────────────────────────────────┤
│  HOSTING                                                         │
│  └── Railway (backend) + Expo EAS (apps) + CloudFlare (CDN)      │
└─────────────────────────────────────────────────────────────────┘
```

### Detailed Documentation

→ See [ARCHITECTURE_DECISIONS.md](./ARCHITECTURE_DECISIONS.md)

---

## Data Model

### Core Entities

```
┌─────────────┐    ┌─────────────────┐    ┌─────────────┐
│   users     │───<│  learning_paths │>───│   modules   │
└─────────────┘    └─────────────────┘    └─────────────┘
       │                   │                     │
       │                   │                     │
       ▼                   ▼                     ▼
┌─────────────┐    ┌─────────────────┐    ┌─────────────┐
│  sessions   │    │  conversations  │    │   topics    │
└─────────────┘    └─────────────────┘    └─────────────┘
       │                   │                     │
       │                   │                     │
       ▼                   ▼                     ▼
┌─────────────┐    ┌─────────────────┐    ┌─────────────┐
│ assessments │    │topic_summaries  │    │topic_progress│
└─────────────┘    └─────────────────┘    └─────────────┘
```

### Key Tables

| Table | Purpose |
|-------|---------|
| `users` | User accounts and preferences |
| `learning_paths` | User's enrolled curricula |
| `modules` | Grouped topics within paths |
| `topics` | Individual learning units |
| `topic_progress` | User progress per topic |
| `conversations` | Chat sessions with AI |
| `topic_summaries` | Generated lesson summaries |
| `assessments` | Quizzes and scores |
| `sessions` | LLM usage tracking (cost) |
| `subscriptions` | Payment status |

### Detailed Documentation

→ See [DATA_MODEL.md](./DATA_MODEL.md)

---

## API Overview

### Endpoints (Summary)

| Category | Key Endpoints |
|----------|---------------|
| **Auth** | `/me`, `PATCH /me`, `DELETE /me` |
| **Learning Paths** | `/learning-paths`, `/learning-paths/{id}/enroll` |
| **Topics** | `/topics/{id}`, `/topics/{id}/start`, `/topics/{id}/complete` |
| **Conversations** | `WSS /ws/learn`, `/conversations/{id}/messages` |
| **Assessments** | `/topics/{id}/assess`, `/assessments/{id}/answers` |
| **Progress** | `/me/progress`, `/me/topic-summaries` |
| **Gamification** | `/me/achievements`, `/me/streak` |
| **Subscriptions** | `/me/subscription`, `/subscriptions/checkout` |

### Detailed Documentation

→ See [API_SPECIFICATION.md](./API_SPECIFICATION.md)

---

## AI System

### Multi-Agent Architecture (12 Agents)

```
┌────────────────────────────────────────────────────────────────────────────┐
│                        LANGGRAPH AGENTS (12 Total)                          │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ONBOARDING AGENTS                                                          │
│  ┌───────────────┐  ┌───────────────┐  ┌───────────────────┐              │
│  │   Interview   │  │     Path      │  │    Curriculum     │              │
│  │    Agent      │→ │    Agent      │→ │    Explainer      │              │
│  │   (Sonnet)    │  │   (Sonnet)    │  │     (Haiku)       │              │
│  │ Onboarding    │  │ Generates     │  │ Explains order,   │              │
│  │ conversation  │  │ curriculum    │  │ handles challenges│              │
│  └───────────────┘  └───────────────┘  └───────────────────┘              │
│                                                                             │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                             │
│  SESSION AGENTS                                                             │
│  ┌─────────────────┐                                                        │
│  │  Orchestrator   │  Routes user input to correct agent                    │
│  │  (Claude Haiku) │                                                        │
│  └────────┬────────┘                                                        │
│           │                                                                 │
│  ┌────────┴────────┬─────────────┬─────────────┬─────────────┐            │
│  ▼                 ▼             ▼             ▼             ▼            │
│ ┌───────────┐ ┌───────────┐ ┌───────────┐ ┌───────────┐ ┌───────────┐    │
│ │ Teaching  │ │Assessment │ │Diagnostic │ │Reflection │ │Verification│    │
│ │  Agent    │ │  Agent    │ │  Agent    │ │  Agent    │ │  Agent     │    │
│ │ (Sonnet)  │ │ (Sonnet)  │ │ (Sonnet)  │ │ (Gemini)  │ │ (Sonnet)   │    │
│ │ Explains  │ │ Tests     │ │ Analyzes  │ │ Has user  │ │ Fact-checks│    │
│ │ concepts  │ │ knowledge │ │ errors    │ │ explain   │ │ on request │    │
│ └───────────┘ └───────────┘ └───────────┘ └───────────┘ └───────────┘    │
│                                                                             │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                             │
│  END-OF-SESSION AGENTS (run when session ends)                              │
│  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐                  │
│  │    Summary    │  │ Learner Model │  │  Preferences  │                  │
│  │    Agent      │  │    Agent      │  │    Agent      │                  │
│  │   (Haiku)     │  │   (Haiku)     │  │   (Haiku)     │                  │
│  │ Topic summary │  │ Updates model │  │ Adjusts style │                  │
│  │ for review    │  │ JSON (±1500ch)│  │ prefs (±0.1)  │                  │
│  └───────────────┘  └───────────────┘  └───────────────┘                  │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

### Agent Summary

| Agent | Model | Purpose |
|-------|-------|---------|
| **Interview** | Sonnet | Onboarding conversation, goal/background/spot check |
| **Path** | Sonnet | Generates personalized curriculum from interview |
| **Curriculum Explainer** | Haiku | Explains "why this order?", handles challenges |
| **Orchestrator** | Haiku | Routes user input to correct agent |
| **Teaching** | Sonnet | Explains concepts, answers questions |
| **Assessment** | Sonnet | Tests knowledge, generates questions |
| **Diagnostic** | Sonnet | Analyzes misconceptions, error patterns |
| **Reflection** | Gemini Flash | Has user explain back, reveals gaps |
| **Verification** | Sonnet | Fact-checks on user request |
| **Summary** | Haiku | Generates topic summaries on completion |
| **Learner Model** | Haiku | Updates subject_learner_models.model JSON |
| **Preferences** | Haiku | Adjusts learning_preferences (±0.1) |

### Learning Continuity

The AI tutor maintains learning continuity by:

1. **Onboarding:** Interview captures goals, background, verified level
2. **Before teaching:** Fetching prior topic summaries + learner model
3. **During teaching:** Referencing prior knowledge, adapting to preferences
4. **After teaching:** Generating topic summary, updating learner model

### Detailed Documentation

→ See [AGENT_PROMPTS.md](./AGENT_PROMPTS.md) for full prompts and trigger conditions

---

## Business Model

### Pricing

| Tier | Price | Features |
|------|-------|----------|
| **Free** | $0 | 3 sessions/day, basic progress |
| **Premium** | $9.99/mo | Unlimited, streak freeze, priority |
| **Annual** | $79.99/yr | Premium + 33% discount |

### Revenue Projections (Conservative)

| Metric | Month 3 | Month 6 | Month 12 |
|--------|---------|---------|----------|
| Total Users | 1,000 | 5,000 | 20,000 |
| Premium (2%) | 20 | 100 | 400 |
| MRR | $200 | $1,000 | $4,000 |

### Unit Economics

> **⚠️ CRITICAL:** LLM costs are 4× higher than original estimates. See ADR-011 for details.

```
REALISTIC COSTS (Multi-agent architecture):
├── Teaching call (Sonnet):      $0.18
├── 3× Assessment calls:         $0.12
├── 10× Orchestrator (Haiku):    $0.008
├── Summary + Learning Loop:     $0.05
────────────────────────────────────────
Per session:                     ~$0.36

Per Premium User (moderate: 30 sessions/month):
├── Revenue:        $9.99/month
├── LLM costs:      ~$10.80/month (30 × $0.36)
├── Infrastructure: ~$0.50/month
├── Payment fees:   ~$0.60/month
└── Gross margin:   -$1.91 (LOSS) ❌

REQUIRED MITIGATIONS:
├── 40% cache hit rate:           $0.36 → $0.22/session
├── 50% Haiku for simple queries: $0.22 → $0.14/session
├── Optimized:                    $4.20/month LLM costs
└── Gross margin with opts:       $4.69 (47%) ✓

ALTERNATIVE: Raise price to $24.99/month for serious learners.
```

---

## Risks & Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| **LLM costs exceed revenue** | HIGH | CRITICAL | Caching (40%), model tiering (50% Haiku), session limits, or raise price to $24.99 |
| **Claude outage** | Low | High | OpenAI fallback, graceful degradation |
| **Poor learning outcomes** | Medium | High | Iterate prompts, user testing, feedback loops |
| **Low conversion** | High | Medium | A/B test pricing, improve free experience |
| **App store rejection** | Low | Medium | Follow guidelines strictly |
| **GDPR complaint** | Low | Medium | Account deletion, data export ready |

---

## Success Metrics

### Primary Metrics

| Metric | Definition | Target |
|--------|------------|--------|
| **WAL** | Weekly Active Learners | Growing 20%/month |
| **Day 7 Retention** | Return after 1 week | >30% |
| **Session Completion** | Finish topics started | >70% |
| **Free→Paid Conversion** | Hit limit, subscribe | >2% |

### Secondary Metrics

| Metric | Definition | Target |
|--------|------------|--------|
| Learning efficacy | Re-test pass rate | >50% |
| NPS | User satisfaction | >40 |
| App rating | Store reviews | >4.0 |
| LLM cost/session | API costs (with optimizations) | <$0.15 |

---

## Roadmap

```
┌─────────────────────────────────────────────────────────────────┐
│  PROTOTYPE (Validation)                            4-6 weeks    │
├─────────────────────────────────────────────────────────────────┤
│  Goal: Validate core learning experience                         │
│                                                                  │
│  ✓ Interview → Curriculum → Chat → Assessment                   │
│  ✓ Spaced retrieval + interleaving                              │
│  ✓ Step-level feedback + worked examples                        │
│  ✓ Mode switching (Teach/Quiz/Quick Answer)                     │
│  ✓ "X% of learners struggled here" signals                      │
│  ✓ Confidence badges on curriculum                              │
│  ✓ Basic gamification (XP, streak)                              │
│                                                                  │
│  Validation: Do people learn? Do they return? Tolerate gates?   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  v1.0 (MVP — First Paying Customers)               Q1 2025      │
├─────────────────────────────────────────────────────────────────┤
│  Core (from Prototype):                                          │
│  ✓ Any subject (dynamic curriculum)                             │
│  ✓ Learning continuity + Learning Book                          │
│  ✓ Outcome-based gamification (Retention XP, Decay, Streak)     │
│  ✓ Code execution (Pyodide)                                     │
│                                                                  │
│  Growth & Positioning:                                           │
│  • Shareable Proof of Knowledge link                             │
│  • "Prep for [AWS/Python Cert]" positioning                      │
│  • Public commitment share (opt-in Twitter/LinkedIn)             │
│  • Weekly AI check-in email (re-engagement)                      │
│  • Discord community link                                        │
│                                                                  │
│  UX Polish:                                                      │
│  • Speed runs (compressed review for returning users)            │
│  • "Open in Replit" fallback for code                            │
│                                                                  │
│  Business:                                                       │
│  • Free/Premium tiers ($9.99/month)                              │
│  • iOS + Android + Web (Expo)                                    │
│                                                                  │
│  Validation: Will people pay? Organic growth signals?            │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  v1.5 (Retention & Accountability)                 Q2 2025      │
├─────────────────────────────────────────────────────────────────┤
│  Human Accountability (based on retention data):                 │
│  • Cohort starts ("Join January Python cohort")                  │
│  • Buddy matching (pair users learning similar subjects)         │
│  • Study groups (async, topic-based)                             │
│                                                                  │
│  Trust & Quality:                                                │
│  • Expert-audited curricula (top 10 subjects)                    │
│  • Cross-user curriculum learning (aggregate signals → improve)  │
│                                                                  │
│  Engagement:                                                     │
│  • Struggle badges (Hard Won, Comeback, Deep Roots)              │
│  • Self-competition dashboard (compare to past self)             │
│                                                                  │
│  Validation: Which retention lever works? What differentiates?   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  v2.0 (Scale & Moat)                               Q3-Q4 2025   │
├─────────────────────────────────────────────────────────────────┤
│  Advanced Features:                                              │
│  • Paid human coaching add-on ($50/month)                        │
│  • Portfolio projects (guided capstones → GitHub)                │
│  • Employer partnerships                                         │
│                                                                  │
│  Platform:                                                       │
│  • Multi-language UI (i18n)                                      │
│  • Creator marketplace (vetted user-generated courses)           │
│  • B2B/Team licensing                                            │
│  • API for third-party integrations                              │
│                                                                  │
│  Expansion:                                                      │
│  • Age 6-12 support (COPPA compliant)                            │
│  • Offline mode                                                  │
│  • Dark mode                                                     │
└─────────────────────────────────────────────────────────────────┘
```

### Feature Prioritization Logic

| When | Add If... |
|------|-----------|
| **v1.0** | Feature helps validate payment or organic growth |
| **v1.5** | Retention is the bottleneck (users learn but don't return) |
| **v2.0** | Core works, now scaling and building moat |

### What We Learned to Skip

| Feature | Why Deferred |
|---------|--------------|
| Discussion threads | Moderation cost, only if community demand |
| Full video content | Not differentiated, ChatGPT does this |
| Native code IDE | Pyodide + Replit link is sufficient |
| Credentials (our own) | "Prep for [existing cert]" is smarter positioning |

---

## Related Documents

| Document | Description |
|----------|-------------|
| [ARCHITECTURE_DECISIONS.md](./ARCHITECTURE_DECISIONS.md) | All technical decisions with rationale |
| [DATA_MODEL.md](./DATA_MODEL.md) | Complete database schema |
| [API_SPECIFICATION.md](./API_SPECIFICATION.md) | REST + WebSocket API reference |
| [AGENT_PROMPTS.md](./AGENT_PROMPTS.md) | LangGraph agent prompts |
| [MVP_DEFINITION.md](./MVP_DEFINITION.md) | What's in v1.0 |

---

## Appendix: Glossary

| Term | Definition |
|------|------------|
| **Learning Path** | A structured curriculum to achieve a learning goal |
| **Module** | A group of related topics within a path |
| **Topic** | A single learning unit (15-30 min) |
| **Topic Summary** | AI-generated summary for review/continuity |
| **Mastery Level** | 0-1 score indicating understanding |
| **Session** | One conversation interaction with AI |
| **Honest Streak** | Consecutive days of passing a recall challenge (not just app opens) |
| **Verified XP** | Experience points earned by passing delayed recall tests |
| **Pending XP** | XP awaiting verification through recall |
| **Knowledge Decay** | Visual representation of knowledge fading over time |
| **WAL** | Weekly Active Learners |

---

## Document History

| Date | Version | Change | Author |
|------|---------|--------|--------|
| 2024-12-10 | 1.0 | Initial consolidated PRD | Claude + User |

