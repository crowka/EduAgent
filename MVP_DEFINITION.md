# EduAgent MVP Definition (v1.0)

> **Document Status:** Living document  
> **Last Updated:** 2024-12-10  
> **Target Launch:** TBD

---

## Table of Contents

1. [MVP Philosophy](#mvp-philosophy)
2. [Core User Journey](#core-user-journey)
3. [In Scope (v1.0)](#in-scope-v10)
4. [Explicitly Out of Scope](#explicitly-out-of-scope)
5. [Feature Breakdown](#feature-breakdown)
6. [Success Metrics](#success-metrics)
7. [Launch Checklist](#launch-checklist)

---

## MVP Philosophy

```
┌─────────────────────────────────────────────────────────────────┐
│  MVP MANTRA                                                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  "A user can learn ANY subject through AI-powered tutoring      │
│   with personalized curriculum and retention verification."     │
│                                                                  │
│  That's it. If we nail this, everything else follows.           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### What MVP Must Prove

1. **AI Teaching Works** - Users actually learn from AI tutoring
2. **Personalization Matters** - Adaptive learning beats static content
3. **Continuity Helps** - Building on prior knowledge improves outcomes
4. **Users Pay** - Some segment willing to pay for premium

### What MVP Doesn't Need to Prove

- Multi-language support (English-only is fine)
- Social features (solo learning works)
- Offline mode (connected users only)
- Pre-curated content (AI generates curriculum dynamically)

---

## Core User Journey

### MVP Happy Path

```
┌────────────────────────────────────────────────────────────────────┐
│                      MVP USER JOURNEY                               │
├────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. DISCOVERY                                                       │
│     ┌─────────────────────────────────────────────────────────┐    │
│     │  User downloads app from App Store / Play Store          │    │
│     │  OR visits web app                                       │    │
│     └─────────────────────────────────────────────────────────┘    │
│                              │                                      │
│                              ▼                                      │
│  2. ONBOARDING (5 minutes)                                         │
│     ┌─────────────────────────────────────────────────────────┐    │
│     │  • Create account (email or Google)                      │    │
│     │  • "What do you want to learn?" → Select subject         │    │
│     │  • "How much do you already know?" → Quick assessment    │    │
│     │  • AI generates personalized learning path               │    │
│     └─────────────────────────────────────────────────────────┘    │
│                              │                                      │
│                              ▼                                      │
│  3. FIRST LESSON (10-15 minutes)                                   │
│     ┌─────────────────────────────────────────────────────────┐    │
│     │  • AI introduces first topic                             │    │
│     │  • Conversational teaching (chat interface)              │    │
│     │  • User asks questions, AI adapts                        │    │
│     │  • Quick check: "Let me see if you got that..."          │    │
│     │  • Summary generated and saved                           │    │
│     └─────────────────────────────────────────────────────────┘    │
│                              │                                      │
│                              ▼                                      │
│  4. DAILY ENGAGEMENT                                               │
│     ┌─────────────────────────────────────────────────────────┐    │
│     │  • Notification: "Ready for your next lesson?"           │    │
│     │  • Continue from last topic                              │    │
│     │  • AI references prior learning: "Remember when..."      │    │
│     │  • Track progress, earn XP                               │    │
│     │  • Review past lessons from summaries                    │    │
│     └─────────────────────────────────────────────────────────┘    │
│                              │                                      │
│                              ▼                                      │
│  5. RETENTION LOOP                                                 │
│     ┌─────────────────────────────────────────────────────────┐    │
│     │  • "It's been a week - want a quick review?"             │    │
│     │  • Re-test from saved summaries                          │    │
│     │  • Spaced repetition keeps knowledge fresh               │    │
│     │  • Honest streak motivates real review                   │    │
│     └─────────────────────────────────────────────────────────┘    │
│                                                                     │
└────────────────────────────────────────────────────────────────────┘
```

---

## In Scope (v1.0)

### ✅ Core Features

| Feature | Priority | Effort | Notes |
|---------|----------|--------|-------|
| **User Authentication** | P0 | Low | Supabase Auth (email + Google) |
| **Onboarding Flow** | P0 | Medium | Subject selection, initial assessment |
| **AI Learning Path Generation** | P0 | Medium | Path Agent creates curriculum |
| **Conversational Learning** | P0 | High | Teaching Agent, WebSocket chat |
| **Prior Knowledge Context** | P0 | Medium | Learning continuity system |
| **Topic Summaries** | P0 | Medium | Auto-generated, stored for review |
| **Progress Tracking** | P0 | Medium | Topics completed, mastery levels |
| **Quick Assessments** | P0 | Medium | In-lesson knowledge checks |
| **Topic Re-Testing** | P1 | Medium | Test from saved summaries |
| **Outcome-Based Gamification** | P1 | Low | Honest streak, Retention XP, decay bars |
| **Push Notifications** | P1 | Low | Daily reminders |
| **Free Tier Limits** | P1 | Low | 3 sessions/day cap |
| **Premium Subscription** | P2 | Medium | Stripe integration |

### ✅ Any Subject (Dynamic Curriculum)

**Core value proposition: Learn anything with personalized AI tutoring.**

> See **ADR-018: Dynamic Curriculum Generation** for full architecture.

How it works:
- User types any subject they want to learn
- **Subject viability check** (quick LLM assessment: HIGH/MEDIUM/LOW confidence)
- If MEDIUM/LOW: warn user that curriculum may need adjustment
- Interview Agent assesses goal, background, level (~3 min)
- Path Agent generates personalized curriculum on demand
- Source constraints ensure quality (textbooks, university syllabi only)
- User can challenge/adjust curriculum

Subject viability handling:
| Confidence | Example | UX Treatment |
|------------|---------|--------------|
| HIGH | Python, Spanish, Calculus | Proceed silently |
| MEDIUM | Quantum Computing, Jazz Theory | Show warning, proceed |
| LOW | [Emerging Framework], Niche Tool | Strong warning, simplified mode |

Why this matters for MVP:
- Tests the core differentiator (not just another Python course)
- No content creation bottleneck
- Validates AI's ability to teach any domain
- Attracts broader user base for testing

Code execution support (programming subjects):
- **Pyodide** (browser-based Python via WebAssembly) — no server needed
- Inline code editor in chat interface
- Real-time execution with output display
- AI validates code logic, not just syntax
- Test cases (visible + hidden) for exercises
- Step-level feedback on code errors
- Other subjects work via conversation + examples

### ✅ Technical Scope

| Component | MVP Scope |
|-----------|-----------|
| **Frontend** | Expo (iOS + Android + Web) |
| **Backend** | FastAPI on Railway |
| **Database** | Supabase PostgreSQL |
| **Auth** | Supabase Auth |
| **LLM** | Anthropic Claude (Sonnet + Haiku) |
| **Agent Framework** | LangGraph |
| **Cache** | Upstash Redis |
| **Payments** | Stripe (web checkout) |
| **Analytics** | PostHog (free tier) |

### ✅ Platforms

| Platform | MVP | Notes |
|----------|-----|-------|
| iOS | ✅ | Expo EAS Build |
| Android | ✅ | Expo EAS Build |
| Web | ✅ | Expo Web |

---

## Explicitly Out of Scope

### ❌ Not in v1.0

| Feature | Reason | Target Version |
|---------|--------|----------------|
| **Cohorts** | Needs user volume for matching | v1.5 |
| **Buddy Matching** | Needs user volume for matching | v1.5 |
| **Study Groups** | Social complexity, validate learning first | v1.5 |
| **Human Coaching** | New business model | v2.0 |
| **Portfolio Projects** | Complex, needs curriculum stability | v2.0 |
| **Employer Partnerships** | Business dev, needs credibility | v2.0 |
| **Multi-language UI** | English-only MVP | v2.0 |
| **Offline Mode** | Requires significant caching | v2.0 |
| **Creator Marketplace** | Verification complexity | v2.0 |
| **Age 6-12 Mode** | Different UX needs, COPPA | v2.0 |
| **B2B Dashboard** | Focus on B2C first | v2.0 |
| **Dark Mode** | Polish feature | v2.0 |
| **Discussion Threads** | Moderation cost, only if demand | v2.0+ |
| **Leaderboards** | Research says less effective | Never |

### ❌ Technical Debt Allowed

| Shortcut | Acceptable Because | Fix In |
|----------|-------------------|--------|
| No CDN for media | Minimal static assets in MVP | v1.1 |
| Basic error pages | Focus on happy path | v1.1 |
| Manual deployments | Low traffic, easy rollback | v1.2 |
| No A/B testing | Need users first | v1.2 |
| Simple analytics | PostHog free tier enough | v1.1 |

---

## Feature Breakdown

### 1. Authentication (P0)

```
MVP Scope:
├── Email + password signup
├── Google OAuth
├── Email verification
├── Password reset
└── Session management (JWT)

NOT in MVP:
├── Apple Sign-In (requires paid dev account)
├── Magic links
└── 2FA
```

### 2. Onboarding (P0)

```
MVP Scope:
├── Welcome screen
├── Subject input (user types any subject)
├── Interview conversation (~3 min, goal/background/spot check)
├── AI generates personalized curriculum
├── Curriculum review (skip/challenge)
├── First lesson teaser
└── Push notification permission

NOT in MVP:
├── Learning style quiz
├── Goal setting (timeline)
└── Skip assessment option
```

### 3. Learning Experience (P0)

```
MVP Scope:
├── Chat interface (message input + AI responses)
├── Streaming responses (real-time typing effect)
├── Prior knowledge injection
├── In-lesson quick checks (2-3 questions)
├── "I don't understand" button → AI rephrases
├── End of topic summary generation
└── Progress bar within topic

NOT in MVP:
├── Voice input/output
├── Rich media (videos, images)
└── Drawing/annotation tools

Code Execution (Included):
├── Pyodide (browser-based Python via WebAssembly)
├── Code validation API (POST /code/validate)
├── Exercise generation API (POST /code/exercise)
└── Test case runner API (POST /code/test)
```

### 4. Progress & Review (P0/P1)

```
MVP Scope:
├── Learning path overview (modules → topics)
├── Topic completion status
├── Mastery level per topic
├── Saved topic summaries
├── Re-test from summary
├── "Continue where I left off"
└── Weekly progress summary

NOT in MVP:
├── Detailed analytics dashboard
├── Learning time tracking
└── Comparison with average
```

### 5. Outcome-Based Gamification (P1)

> **See ADR-016** for full rationale. We reward knowledge retention, not app opens.

```
MVP Scope:
├── Honest Streak (consecutive days passing recall, NOT app opens)
├── Retention XP (pending → verified after 2-week/6-week recall)
├── Knowledge decay bars (visual: Strong → Fading → Weak)
├── Review reminders for decaying topics
└── Verified vs pending XP display

NOT in MVP:
├── Struggle badges (Hard Won, Comeback, Deep Roots)
├── Self-competition dashboards
├── Proof of Knowledge certificates
├── Leaderboards
└── Streak freezes (premium)
```

**Key difference from traditional gamification:**
| Traditional | EduAgent (Honest) |
|-------------|-------------------|
| Daily streak = opened app | Honest streak = passed recall |
| XP on completion | XP pending until verified by recall |
| Static progress bar | Decay bar shows knowledge fading |

### 6. Subscription (P2)

```
MVP Scope:
├── Free tier: 3 sessions/day
├── Premium tier: Unlimited
├── Stripe Checkout (web redirect)
├── Cancel subscription
└── Subscription status display

NOT in MVP:
├── In-app purchase (requires app store setup)
├── Multiple plan tiers
├── Team/family plans
└── Promo codes
```

---

## Success Metrics

### North Star Metric

**Weekly Active Learners (WAL)**: Users who complete at least one learning session per week.

### MVP Success Criteria

| Metric | Target | Measurement |
|--------|--------|-------------|
| **Onboarding Completion** | >60% | Users who finish onboarding |
| **First Week Retention** | >30% | Return after day 7 |
| **Session Completion** | >70% | Sessions not abandoned mid-topic |
| **Learning Efficacy** | >50% | Pass rate on re-tests (2 weeks later) |
| **Free → Paid Conversion** | >2% | After hitting daily limit |
| **App Store Rating** | >4.0 | User satisfaction |

### Tracking Implementation

```
PostHog Events (MVP):
├── onboarding_started
├── onboarding_completed
├── session_started
├── session_completed
├── topic_completed
├── summary_viewed
├── retest_requested
├── honest_streak_achieved
├── paywall_shown
├── subscription_started
└── subscription_cancelled
```

---

## Launch Checklist

### Pre-Launch

- [ ] **Legal**
  - [ ] Privacy Policy
  - [ ] Terms of Service
  - [ ] GDPR compliance (delete account flow)
  - [ ] Cookie consent (web)

- [ ] **App Store**
  - [ ] iOS App Store listing
  - [ ] Google Play listing
  - [ ] Screenshots (6 screens)
  - [ ] App description
  - [ ] Age rating (13+)

- [ ] **Infrastructure**
  - [ ] Production environment deployed
  - [ ] SSL certificates
  - [ ] Error monitoring (Sentry)
  - [ ] Backup strategy tested
  - [ ] Rate limiting configured

- [ ] **AI & Prompts**
  - [ ] Interview Agent prompts tested
  - [ ] Path Agent curriculum generation tested
  - [ ] Teaching Agent prompts refined
  - [ ] Source constraint validation working

- [ ] **Testing**
  - [ ] Full user journey tested on iOS
  - [ ] Full user journey tested on Android
  - [ ] Full user journey tested on Web
  - [ ] Payment flow tested
  - [ ] Edge cases (no network, session expiry)

### Launch Day

- [ ] Deploy production build
- [ ] Submit to app stores
- [ ] Monitor error rates
- [ ] Watch for LLM cost spikes
- [ ] Respond to first user feedback

### Post-Launch (Week 1)

- [ ] Daily metrics review
- [ ] Address critical bugs
- [ ] Collect user feedback
- [ ] Identify drop-off points
- [ ] Plan v1.1 priorities

---

## Version Roadmap

```
PROTOTYPE (Pre-MVP Validation) — 4-6 weeks
├── Core learning experience (Interview → Chat → Assessment)
├── Mode switching (Teach/Quiz/Quick Answer)
├── "X% of learners struggled here" signals
├── Confidence badges visible
└── Transparency disclaimer on AI-generated curriculum

v1.0 (MVP) — This Document
├── Any subject, dynamic curriculum
├── Core learning with step-level feedback
├── Outcome-based gamification (Retention XP, Decay, Streak)
├── Code execution (Pyodide)
│
├── GROWTH & POSITIONING:
│   ├── Shareable Proof of Knowledge link
│   ├── "Prep for [AWS/Python Cert]" positioning
│   ├── Public commitment share (opt-in social)
│   ├── Weekly AI check-in email
│   └── Discord community link
│
├── UX POLISH:
│   ├── Speed runs (compressed review)
│   └── "Open in Replit" fallback
│
└── BUSINESS:
    ├── Free/Premium tiers ($9.99/month)
    └── iOS + Android + Web (Expo)

v1.5 (Retention & Accountability)
├── Cohorts ("Join January Python cohort")
├── Buddy matching (pair similar learners)
├── Study groups (async, topic-based)
├── Expert-audited curricula (top 10 subjects)
├── Struggle badges (Hard Won, Comeback, Deep Roots)
├── Self-competition dashboard
└── Cross-user curriculum learning (aggregate signals)

v2.0 (Scale & Moat)
├── Paid human coaching add-on ($50/month)
├── Portfolio projects (guided capstones)
├── Employer partnerships
├── Multi-language UI (i18n)
├── Creator marketplace
├── B2B/Team licensing
├── Age 6-12 support
├── Offline mode
└── Dark mode
```

### Feature Timing Logic

| Add When | Criteria |
|----------|----------|
| **Prototype** | Validates core learning hypothesis |
| **v1.0** | Helps validate payment or organic growth |
| **v1.5** | Retention is the bottleneck |
| **v2.0** | Core works, now scaling |

---

## Risk Mitigation

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| LLM costs exceed budget | Medium | High | Hard limit on sessions, caching |
| Claude outage | Low | High | OpenAI fallback, graceful degradation |
| Poor learning outcomes | Medium | High | Iterate on prompts, user testing |
| App store rejection | Low | Medium | Follow guidelines, no web links to payment |
| Low conversion rate | High | Medium | A/B test paywalls post-MVP |

---

## Document History

| Date | Change | Author |
|------|--------|--------|
| 2024-12-10 | Initial MVP definition | Claude + User |

