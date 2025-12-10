# EduAgent Data Model

> **Document Status:** Living document  
> **Last Updated:** 2024-12-10  
> **Database:** Supabase (PostgreSQL)  
> **Cache:** Upstash Redis

---

## Table of Contents

1. [Overview](#overview)
2. [Entity Relationship Diagram](#entity-relationship-diagram)
3. [Core Tables](#core-tables)
4. [Supporting Tables](#supporting-tables)
5. [Redis Cache Schema](#redis-cache-schema)
6. [Indexes](#indexes)
7. [Row-Level Security (RLS)](#row-level-security-rls)
8. [Migrations Strategy](#migrations-strategy)

---

## Overview

### Design Principles

1. **Users own their data** - RLS ensures users only see their own data
2. **AI-generated content is ephemeral** - Don't store what can be regenerated
3. **Cache aggressively** - Redis for hot data, PostgreSQL for persistence
4. **Track costs** - LLM usage logged for cost management
5. **Soft deletes** - Never hard delete user data (GDPR compliance)

### What Goes Where

| Data Type | Storage | Reason |
|-----------|---------|--------|
| User profiles | PostgreSQL | Persistent, relational |
| Learning paths | PostgreSQL | User's history, progress |
| Progress/scores | PostgreSQL | Must persist |
| Conversations | PostgreSQL (JSONB) | Context for AI, history |
| AI explanations | Redis (cache) | Regeneratable, expensive to store |
| UI translations | Redis (cache) | LLM-generated, long TTL |
| Session tokens | Redis | Ephemeral, fast access |
| Rate limits | Redis | Ephemeral counters |

---

## Entity Relationship Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           ENTITY RELATIONSHIPS                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌──────────────┐         ┌──────────────────┐                             │
│   │    users     │────────<│  learning_paths  │                             │
│   │              │    1:N  │                  │                             │
│   │  - profile   │         │  - user's goals  │                             │
│   │  - prefs     │         │  - AI curriculum │                             │
│   └──────┬───────┘         └────────┬─────────┘                             │
│          │                          │                                        │
│          │                          │ 1:N                                    │
│          │                          ▼                                        │
│          │                 ┌──────────────────┐                             │
│          │                 │   path_modules   │                             │
│          │                 │                  │                             │
│          │                 │  - ordered units │                             │
│          │                 │  - in a path     │                             │
│          │                 └────────┬─────────┘                             │
│          │                          │                                        │
│          │                          │ 1:N                                    │
│          │                          ▼                                        │
│          │                 ┌──────────────────┐                             │
│          │                 │   module_topics  │                             │
│          │                 │                  │                             │
│          │                 │  - ordered topics│                             │
│          │                 │  - in a module   │                             │
│          │                 └────────┬─────────┘                             │
│          │                          │                                        │
│          │ 1:N                      │ 1:1                                    │
│          ▼                          ▼                                        │
│   ┌──────────────┐         ┌──────────────────┐      ┌──────────────────┐   │
│   │   sessions   │         │  topic_progress  │      │ topic_summaries  │   │
│   │              │         │                  │      │                  │   │
│   │  - LLM calls │         │  - completion    │      │ - key_concepts   │   │
│   │  - costs     │         │  - mastery score │      │ - for continuity │   │
│   └──────────────┘         └──────────────────┘      │ - for re-testing │   │
│                                                      └──────────────────┘   │
│          │                                                                   │
│          │ 1:N                                                               │
│          ▼                                                                   │
│   ┌──────────────┐         ┌──────────────────┐        ┌─────────────────┐  │
│   │conversations │         │   assessments    │───────>│assessment_results│  │
│   │              │         │                  │   1:N  │                 │  │
│   │  - chat hist │         │  - quiz questions│        │  - user answers │  │
│   └──────────────┘         └──────────────────┘        └─────────────────┘  │
│                                                                              │
│   ┌──────────────┐         ┌──────────────────┐        ┌─────────────────┐  │
│   │   streaks    │         │  achievements    │───────>│ user_achievements│  │
│   │              │         │                  │   N:M  │                 │  │
│   │  - daily use │         │  - badge defs    │        │  - earned badges│  │
│   └──────────────┘         └──────────────────┘        └─────────────────┘  │
│                                                                              │
│   ┌──────────────┐         ┌──────────────────┐                             │
│   │verifications │         │  subscriptions   │                             │
│   │              │         │                  │                             │
│   │  - flagged   │         │  - plan tier     │                             │
│   │  - escalated │         │  - billing       │                             │
│   └──────────────┘         └──────────────────┘                             │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Core Tables

### 1. users

Extends Supabase `auth.users`. Store app-specific profile data.

```sql
CREATE TABLE public.users (
    -- Primary key matches Supabase auth.users
    id UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
    
    -- Profile
    display_name TEXT,
    avatar_url TEXT,
    
    -- Preferences (ADR-010: i18n)
    language VARCHAR(10) DEFAULT 'en',  -- UI + learning language
    timezone VARCHAR(50) DEFAULT 'UTC',
    
    -- Learning preferences
    learning_style VARCHAR(20) DEFAULT 'balanced',  -- visual, reading, interactive, balanced
    daily_goal_minutes INTEGER DEFAULT 15,
    
    -- Onboarding
    onboarding_completed BOOLEAN DEFAULT FALSE,
    
    -- Outcome-Based Gamification (ADR-016)
    verified_xp INTEGER DEFAULT 0,           -- XP from passed recall tests
    pending_xp INTEGER DEFAULT 0,            -- XP awaiting verification
    honest_streak INTEGER DEFAULT 0,         -- Days where recall was passed
    last_recall_pass_date DATE,              -- For honest streak calculation
    longest_honest_streak INTEGER DEFAULT 0, -- Historical best
    
    -- Metadata
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    deleted_at TIMESTAMPTZ  -- Soft delete (GDPR)
);

-- Trigger for updated_at
CREATE TRIGGER update_users_updated_at
    BEFORE UPDATE ON users
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();

COMMENT ON TABLE users IS 'User profiles extending Supabase auth';
COMMENT ON COLUMN users.language IS 'ISO 639-1 code for UI and learning language';
COMMENT ON COLUMN users.learning_style IS 'Preferred learning modality';
```

---

### 2. learning_paths

A user's learning goal with AI-generated curriculum.

```sql
CREATE TABLE public.learning_paths (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    
    -- Goal definition
    title TEXT NOT NULL,  -- "Learn Machine Learning"
    goal TEXT,            -- User's stated goal in their words
    subject_area TEXT,    -- "Computer Science", "Mathematics", etc.
    
    -- Interview & Curriculum Generation (NEW)
    interview_summary JSONB,              -- Output from Interview Agent
    curriculum_json JSONB,                -- Full generated curriculum structure
    generation_prompt_version TEXT,       -- Version of curriculum generation prompt used
    generated_at TIMESTAMPTZ,             -- When curriculum was generated
    
    -- AI-generated curriculum metadata
    estimated_hours INTEGER,
    difficulty_level INTEGER DEFAULT 1 CHECK (difficulty_level BETWEEN 1 AND 5),
    
    -- Status
    status VARCHAR(20) DEFAULT 'active' CHECK (status IN ('active', 'paused', 'completed', 'abandoned')),
    
    -- Progress summary (denormalized for fast reads)
    total_modules INTEGER DEFAULT 0,
    completed_modules INTEGER DEFAULT 0,
    progress_percent NUMERIC(5,2) DEFAULT 0,
    
    -- Timestamps
    started_at TIMESTAMPTZ DEFAULT NOW(),
    completed_at TIMESTAMPTZ,
    last_activity_at TIMESTAMPTZ DEFAULT NOW(),
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_learning_paths_user_id ON learning_paths(user_id);
CREATE INDEX idx_learning_paths_status ON learning_paths(user_id, status);
CREATE INDEX idx_learning_paths_subject ON learning_paths(user_id, subject_area);

COMMENT ON TABLE learning_paths IS 'User learning goals with AI-generated curricula';
COMMENT ON COLUMN learning_paths.interview_summary IS 'Structured output from Interview Agent (goal, background, verified level)';
COMMENT ON COLUMN learning_paths.curriculum_json IS 'Full curriculum structure generated by Path Agent';
```

---

### 3. path_modules

Ordered modules within a learning path (e.g., chapters).

```sql
CREATE TABLE public.path_modules (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    learning_path_id UUID NOT NULL REFERENCES learning_paths(id) ON DELETE CASCADE,
    
    -- Module info
    title TEXT NOT NULL,           -- "Introduction to Neural Networks"
    description TEXT,
    order_index INTEGER NOT NULL,  -- 1, 2, 3...
    
    -- Status
    status VARCHAR(20) DEFAULT 'locked' CHECK (status IN ('locked', 'active', 'completed')),
    
    -- Progress (denormalized)
    total_topics INTEGER DEFAULT 0,
    completed_topics INTEGER DEFAULT 0,
    
    -- Timestamps
    unlocked_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_path_modules_path ON path_modules(learning_path_id, order_index);

COMMENT ON TABLE path_modules IS 'Ordered modules (chapters) within a learning path';
```

---

### 4. module_topics

Individual topics within a module (the actual learning units).

```sql
CREATE TABLE public.module_topics (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    module_id UUID NOT NULL REFERENCES path_modules(id) ON DELETE CASCADE,
    
    -- Topic info
    title TEXT NOT NULL,           -- "Backpropagation"
    description TEXT,
    order_index INTEGER NOT NULL,  -- Order within module
    
    -- Learning metadata
    estimated_minutes INTEGER DEFAULT 10,
    difficulty INTEGER DEFAULT 1 CHECK (difficulty BETWEEN 1 AND 5),
    
    -- Prerequisites (optional, for adaptive learning)
    prerequisite_topic_ids UUID[] DEFAULT '{}',
    
    -- Status
    status VARCHAR(20) DEFAULT 'locked' CHECK (status IN ('locked', 'available', 'in_progress', 'completed')),
    
    -- Aggregate stats (for "X% struggled here" signals)
    -- Updated periodically via cron or on-demand
    total_attempts INTEGER DEFAULT 0,
    struggle_count INTEGER DEFAULT 0,  -- Users who needed 2+ attempts
    avg_mastery_score NUMERIC(5,2),
    
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_module_topics_module ON module_topics(module_id, order_index);

COMMENT ON TABLE module_topics IS 'Individual learning topics within a module';
COMMENT ON COLUMN module_topics.prerequisite_topic_ids IS 'Topics that should be completed first';
COMMENT ON COLUMN module_topics.struggle_count IS 'Count of users who needed extra practice (for social proof signals)';
```

---

### 5. topic_progress

User's progress on each topic.

```sql
CREATE TABLE public.topic_progress (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    topic_id UUID NOT NULL REFERENCES module_topics(id) ON DELETE CASCADE,
    
    -- Progress
    status VARCHAR(20) DEFAULT 'not_started' 
        CHECK (status IN ('not_started', 'in_progress', 'completed', 'needs_review')),
    
    -- Mastery
    mastery_score NUMERIC(5,2) DEFAULT 0,  -- 0-100
    attempts INTEGER DEFAULT 0,
    
    -- Retention XP (ADR-016: Outcome-Based Gamification)
    pending_xp INTEGER DEFAULT 0,            -- Awaiting recall verification
    verified_xp INTEGER DEFAULT 0,           -- Earned from passed recalls
    
    -- Recall Tracking (for Knowledge Decay)
    last_recall_at TIMESTAMPTZ,              -- Last time topic was tested
    recall_2wk_score NUMERIC(5,2),           -- 2-week delayed recall score
    recall_2wk_at TIMESTAMPTZ,               -- When 2-week recall was taken
    recall_6wk_score NUMERIC(5,2),           -- 6-week delayed recall score
    recall_6wk_at TIMESTAMPTZ,               -- When 6-week recall was taken
    
    -- Worked Example Tracking (d=0.57 effect size)
    example_mode VARCHAR(20) DEFAULT 'full'
        CHECK (example_mode IN ('full', 'fading', 'problem_first')),
    examples_shown INTEGER DEFAULT 0,        -- Count of worked examples provided
    problems_solved_independently INTEGER DEFAULT 0,  -- Count of problems solved without example
    
    -- Time tracking
    time_spent_seconds INTEGER DEFAULT 0,
    
    -- Timestamps
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    last_activity_at TIMESTAMPTZ,
    
    UNIQUE(user_id, topic_id)
);

CREATE INDEX idx_topic_progress_user ON topic_progress(user_id);
CREATE INDEX idx_topic_progress_user_topic ON topic_progress(user_id, topic_id);
CREATE INDEX idx_topic_progress_needs_recall ON topic_progress(user_id, last_recall_at) 
    WHERE status = 'completed';  -- For finding topics needing review

COMMENT ON TABLE topic_progress IS 'User progress on individual topics';
COMMENT ON COLUMN topic_progress.pending_xp IS 'XP awaiting verification through recall tests';
COMMENT ON COLUMN topic_progress.verified_xp IS 'XP earned by passing delayed recall tests';
COMMENT ON COLUMN topic_progress.last_recall_at IS 'Last recall test date, used for knowledge decay calculation';
```

---

### 5b. topic_retrievals (Spaced Repetition Scheduling)

**SM-2 algorithm tracking for each topic.** Determines when topics are due for retrieval practice.

```sql
CREATE TABLE public.topic_retrievals (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    topic_id UUID NOT NULL REFERENCES module_topics(id) ON DELETE CASCADE,
    
    -- SM-2 Spaced Repetition State
    retrieval_count INTEGER DEFAULT 0,          -- Number of successful retrievals
    interval_days INTEGER DEFAULT 1,            -- Current interval (1, 3, 7, 14, 30, 60...)
    ease_factor NUMERIC(3,2) DEFAULT 2.5,       -- SM-2 ease factor (adjusted per performance)
    next_retrieval_due DATE,                    -- When next retrieval is due
    
    -- Stability tracking
    is_stable BOOLEAN DEFAULT FALSE,            -- True after 5+ successful retrievals
    consecutive_successes INTEGER DEFAULT 0,    -- Resets on failure
    
    -- Last retrieval result
    last_retrieval_at TIMESTAMPTZ,
    last_retrieval_score NUMERIC(3,2),          -- 0.0-1.0
    last_retrieval_quality INTEGER,             -- SM-2 quality: 0-5
    
    -- History for analytics
    total_attempts INTEGER DEFAULT 0,
    total_successes INTEGER DEFAULT 0,
    
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    
    UNIQUE(user_id, topic_id)
);

CREATE INDEX idx_topic_retrievals_user ON topic_retrievals(user_id);
CREATE INDEX idx_topic_retrievals_due ON topic_retrievals(user_id, next_retrieval_due) 
    WHERE NOT is_stable;  -- Focus on topics still being learned
CREATE INDEX idx_topic_retrievals_stable ON topic_retrievals(user_id, is_stable);

COMMENT ON TABLE topic_retrievals IS 'SM-2 spaced repetition scheduling per topic';
COMMENT ON COLUMN topic_retrievals.ease_factor IS 'SM-2 ease factor: 2.5 default, adjusted by performance';
COMMENT ON COLUMN topic_retrievals.is_stable IS 'True after 5+ consecutive successful retrievals';
```

**SM-2 Algorithm Implementation:**

```python
def update_retrieval_schedule(
    retrieval: TopicRetrieval,
    quality: int  # 0-5 (0-2 = fail, 3 = hard, 4 = good, 5 = easy)
) -> TopicRetrieval:
    """
    Update SM-2 schedule based on retrieval quality.
    
    Quality scale:
    0 = Complete failure, no recall
    1 = Wrong answer, recognized correct when shown
    2 = Wrong answer, but felt familiar
    3 = Correct, but with difficulty (hard)
    4 = Correct, with some hesitation (good)
    5 = Correct, instant recall (easy)
    """
    retrieval.total_attempts += 1
    retrieval.last_retrieval_at = now()
    retrieval.last_retrieval_quality = quality
    
    if quality >= 3:  # Success
        retrieval.total_successes += 1
        retrieval.consecutive_successes += 1
        retrieval.retrieval_count += 1
        
        # SM-2 interval calculation
        if retrieval.retrieval_count == 1:
            retrieval.interval_days = 1
        elif retrieval.retrieval_count == 2:
            retrieval.interval_days = 3
        else:
            retrieval.interval_days = int(retrieval.interval_days * retrieval.ease_factor)
        
        # Update ease factor
        retrieval.ease_factor = max(1.3, retrieval.ease_factor + (0.1 - (5 - quality) * (0.08 + (5 - quality) * 0.02)))
        
        # Check stability
        if retrieval.consecutive_successes >= 5:
            retrieval.is_stable = True
            
    else:  # Failure
        retrieval.consecutive_successes = 0
        retrieval.interval_days = 1  # Reset to 1 day
        retrieval.ease_factor = max(1.3, retrieval.ease_factor - 0.2)
        retrieval.is_stable = False
    
    retrieval.next_retrieval_due = today() + timedelta(days=retrieval.interval_days)
    return retrieval
```

**Interleaved Session Builder:**

```python
def build_interleaved_session(due_topics: list[Topic], questions_per_topic: int = 2) -> list[Question]:
    """
    Generate interleaved questions from due topics.
    Never same topic twice in a row. Hide topic labels.
    """
    questions_by_topic = {
        topic.id: generate_questions(topic, n=questions_per_topic) 
        for topic in due_topics
    }
    
    interleaved = []
    last_topic_id = None
    
    while any(questions_by_topic.values()):
        # Pick from topics that have questions left AND aren't last_topic
        available = [
            tid for tid, qs in questions_by_topic.items() 
            if qs and tid != last_topic_id
        ]
        
        if not available:
            available = [tid for tid, qs in questions_by_topic.items() if qs]
        
        topic_id = random.choice(available)
        question = questions_by_topic[topic_id].pop(0)
        question.topic_hidden = True  # Don't reveal topic until after answer
        interleaved.append(question)
        last_topic_id = topic_id
    
    return interleaved
```

---

### 6. conversations

Chat history with the AI (stored as JSONB for flexibility).

```sql
CREATE TABLE public.conversations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    
    -- Context
    learning_path_id UUID REFERENCES learning_paths(id) ON DELETE SET NULL,
    topic_id UUID REFERENCES module_topics(id) ON DELETE SET NULL,
    conversation_type VARCHAR(20) DEFAULT 'learning' 
        CHECK (conversation_type IN ('learning', 'assessment', 'general', 'verification')),
    
    -- Messages (flexible JSONB structure)
    messages JSONB NOT NULL DEFAULT '[]',
    /*
    Format:
    [
        {
            "role": "user" | "assistant" | "system",
            "content": "...",
            "timestamp": "2024-01-01T00:00:00Z",
            "agent": "teaching" | "assessment" | "orchestrator",  -- optional
            "tokens_used": 150  -- optional, for cost tracking
        }
    ]
    */
    
    -- Summary (for long conversations, AI-summarized)
    summary TEXT,
    
    -- Metadata
    message_count INTEGER DEFAULT 0,
    total_tokens INTEGER DEFAULT 0,  -- For cost tracking
    
    -- Status
    is_active BOOLEAN DEFAULT TRUE,
    
    -- Timestamps
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    last_message_at TIMESTAMPTZ DEFAULT NOW(),
    
    -- Cognitive Load Tracking (ADR: Chunking)
    chunk_metrics JSONB DEFAULT '{}'
    /*
    Format:
    {
      "messages_sent": 12,
      "avg_concepts_per_message": 1.4,
      "max_concepts_in_message": 3,
      "overload_regenerations": 1,
      "user_confusion_signals": 2,
      "confusion_phrases": ["wait, what?", "slow down"]
    }
    */
);

CREATE INDEX idx_conversations_user ON conversations(user_id, is_active);
CREATE INDEX idx_conversations_context ON conversations(user_id, topic_id);

COMMENT ON TABLE conversations IS 'Chat history with AI agents';
COMMENT ON COLUMN conversations.messages IS 'JSONB array of message objects';
COMMENT ON COLUMN conversations.chunk_metrics IS 'Cognitive load metrics for teaching quality';
```

**Confusion Signal Detection:**

```python
CONFUSION_SIGNALS = [
    "wait", "confused", "lost", "slow down", "what do you mean",
    "too much", "back up", "start over", "huh", "??", "I don't get",
    "can you explain again", "I'm not following", "one thing at a time"
]

def detect_confusion(user_message: str) -> bool:
    """Detect if user is signaling cognitive overload."""
    return any(signal in user_message.lower() for signal in CONFUSION_SIGNALS)
```

If confusion detected → log it, immediately simplify, teach ONE thing.

---

### 7. assessments

Quiz/assessment questions (can be AI-generated and cached).

```sql
CREATE TABLE public.assessments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    topic_id UUID NOT NULL REFERENCES module_topics(id) ON DELETE CASCADE,
    
    -- Question
    question TEXT NOT NULL,
    question_type VARCHAR(20) DEFAULT 'multiple_choice'
        CHECK (question_type IN ('multiple_choice', 'true_false', 'short_answer', 'explanation')),
    
    -- Answer options (for multiple choice)
    options JSONB,  -- ["Option A", "Option B", "Option C", "Option D"]
    correct_answer TEXT NOT NULL,
    
    -- Metadata
    difficulty INTEGER DEFAULT 1 CHECK (difficulty BETWEEN 1 AND 5),
    explanation TEXT,  -- Why the answer is correct
    
    -- Source
    is_ai_generated BOOLEAN DEFAULT TRUE,
    
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_assessments_topic ON assessments(topic_id);
CREATE INDEX idx_assessments_topic_difficulty ON assessments(topic_id, difficulty);

COMMENT ON TABLE assessments IS 'Quiz questions for topics';
```

---

### 8. assessment_results

User's answers to assessments with **step-level evaluation**.

```sql
CREATE TABLE public.assessment_results (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    assessment_id UUID NOT NULL REFERENCES assessments(id) ON DELETE CASCADE,
    topic_id UUID NOT NULL REFERENCES module_topics(id) ON DELETE CASCADE,
    
    -- Answer
    user_answer TEXT NOT NULL,
    user_reasoning TEXT,  -- User's explanation of their thinking
    
    -- Step-level evaluation (the core of learning feedback)
    reasoning_provided BOOLEAN DEFAULT FALSE,
    steps_evaluated JSONB,
    /*
    Format:
    [
        {"step": "Identified power rule", "correct": true},
        {"step": "Applied exponent multiplication", "correct": true},
        {"step": "Reduced exponent by 1", "correct": false, "issue": "Kept original exponent"}
    ]
    */
    first_error_at INTEGER,  -- null if all correct, 0-indexed step number
    
    -- Overall result
    is_correct BOOLEAN NOT NULL,
    score NUMERIC(3,2),  -- 0.00 to 1.00 (e.g., 2/3 steps = 0.66)
    
    -- AI feedback (targets first error only)
    feedback TEXT,
    follow_up_question TEXT,  -- Probing question for the error
    misconception_identified TEXT,  -- Detected pattern
    
    -- Timestamps
    answered_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_assessment_results_user ON assessment_results(user_id);
CREATE INDEX idx_assessment_results_user_topic ON assessment_results(user_id, topic_id);
CREATE INDEX idx_assessment_results_misconceptions ON assessment_results(misconception_identified) 
    WHERE misconception_identified IS NOT NULL;

COMMENT ON TABLE assessment_results IS 'User answers with step-level evaluation';
COMMENT ON COLUMN assessment_results.steps_evaluated IS 'JSON array of reasoning steps with correctness';
COMMENT ON COLUMN assessment_results.first_error_at IS 'Index of first incorrect step (null if all correct)';
COMMENT ON COLUMN assessment_results.follow_up_question IS 'AI question targeting the specific error';
```

---

### 9. topic_summaries

**Stored summaries of what user learned** - for review and re-testing.

```sql
CREATE TABLE public.topic_summaries (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    topic_id UUID NOT NULL REFERENCES module_topics(id) ON DELETE CASCADE,
    learning_path_id UUID NOT NULL REFERENCES learning_paths(id) ON DELETE CASCADE,
    
    -- Summary content (AI-generated at lesson completion)
    title TEXT NOT NULL,                    -- "Backpropagation Basics"
    key_concepts JSONB NOT NULL DEFAULT '[]',  
    /*
    Format:
    [
        "Backpropagation calculates gradients by chain rule",
        "Error flows backward from output to input layers",
        "Learning rate controls step size in gradient descent",
        "Vanishing gradients can occur in deep networks"
    ]
    */
    
    summary_text TEXT,                      -- 2-3 paragraph summary
    
    -- What was covered (for re-testing context)
    questions_discussed JSONB DEFAULT '[]', -- Questions user asked
    examples_used JSONB DEFAULT '[]',       -- Examples AI provided
    
    -- Difficulty and depth
    depth_level VARCHAR(20) DEFAULT 'foundational'
        CHECK (depth_level IN ('introductory', 'foundational', 'intermediate', 'advanced', 'expert')),
    
    -- For spaced repetition
    times_reviewed INTEGER DEFAULT 0,
    last_reviewed_at TIMESTAMPTZ,
    next_review_suggested TIMESTAMPTZ,      -- Spaced repetition schedule
    retention_score NUMERIC(5,2),           -- How well user remembers (from re-tests)
    
    -- Timestamps
    learned_at TIMESTAMPTZ DEFAULT NOW(),   -- When first completed
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    
    UNIQUE(user_id, topic_id)
);

CREATE INDEX idx_topic_summaries_user ON topic_summaries(user_id);
CREATE INDEX idx_topic_summaries_user_path ON topic_summaries(user_id, learning_path_id);
CREATE INDEX idx_topic_summaries_review ON topic_summaries(user_id, next_review_suggested);

COMMENT ON TABLE topic_summaries IS 'Stored summaries of completed lessons for review and re-testing';
COMMENT ON COLUMN topic_summaries.key_concepts IS 'Array of key points learned';
COMMENT ON COLUMN topic_summaries.next_review_suggested IS 'For spaced repetition scheduling';
```

#### How It Works

```
┌─────────────────────────────────────────────────────────────────┐
│                    LESSON SUMMARY FLOW                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  STEP 1: User Completes Topic                                    │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  User finishes learning "Backpropagation"                │    │
│  │  AI generates summary from conversation                  │    │
│  │  Extracts: key concepts, examples used, depth level      │    │
│  └─────────────────────────────────────────────────────────┘    │
│                            │                                     │
│                            ▼                                     │
│  STEP 2: Summary Stored                                          │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  topic_summaries record created:                         │    │
│  │  • key_concepts: ["Chain rule", "Gradient flow", ...]    │    │
│  │  • summary_text: "You learned that..."                   │    │
│  │  • next_review_suggested: NOW() + 3 days                 │    │
│  └─────────────────────────────────────────────────────────┘    │
│                            │                                     │
│                            ▼                                     │
│  STEP 3: User Can Review                                         │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  "Show me what I learned about Neural Networks"          │    │
│  │                                                          │    │
│  │  App fetches topic_summaries for that path               │    │
│  │  Shows: key concepts, when learned, retention score      │    │
│  └─────────────────────────────────────────────────────────┘    │
│                            │                                     │
│                            ▼                                     │
│  STEP 4: User Can Re-Test                                        │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  "Test me on Backpropagation"                            │    │
│  │                                                          │    │
│  │  System sends to LLM:                                    │    │
│  │  "Create a quiz based on these key concepts:             │    │
│  │   [key_concepts from topic_summaries]                    │    │
│  │   User learned this {days_ago} days ago.                 │    │
│  │   Depth level: {depth_level}"                            │    │
│  │                                                          │    │
│  │  LLM generates personalized quiz                         │    │
│  │  Results update retention_score                          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                            │                                     │
│                            ▼                                     │
│  STEP 5: Spaced Repetition                                       │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  If retention_score high → next_review in 7 days         │    │
│  │  If retention_score low → next_review in 1 day           │    │
│  │                                                          │    │
│  │  App can suggest: "Time to review Backpropagation!"      │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

#### Re-Testing Prompt Example

```
When user says "Test me on Backpropagation":

System retrieves topic_summaries and sends to LLM:

┌─────────────────────────────────────────────────────────────┐
│  "Create a quiz to test retention of these concepts:        │
│                                                             │
│   Topic: Backpropagation                                    │
│   Learned: 14 days ago                                      │
│   Depth: Foundational                                       │
│                                                             │
│   Key concepts to test:                                     │
│   1. Backpropagation calculates gradients by chain rule     │
│   2. Error flows backward from output to input layers       │
│   3. Learning rate controls step size                       │
│   4. Vanishing gradients can occur in deep networks         │
│                                                             │
│   Examples previously discussed:                            │
│   - Simple 2-layer network example                          │
│   - XOR problem demonstration                               │
│                                                             │
│   Generate 4 questions (mix of recall and application).     │
│   Difficulty should match 'foundational' level."            │
└─────────────────────────────────────────────────────────────┘
```

#### Learning Continuity: Building on Previous Knowledge

**Critical:** AI must reference and build upon what user already learned.

```
┌─────────────────────────────────────────────────────────────────┐
│                    CUMULATIVE LEARNING FLOW                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  When teaching a NEW topic, system provides LLM with:            │
│                                                                  │
│  1. CURRENT TOPIC                                                │
│     └── What we're about to teach                                │
│                                                                  │
│  2. PRIOR KNOWLEDGE (from topic_summaries)                       │
│     └── All completed topics in this learning path               │
│     └── Key concepts already mastered                            │
│     └── Examples user has seen before                            │
│                                                                  │
│  3. INSTRUCTION TO BUILD UPON                                    │
│     └── "Reference previous concepts"                            │
│     └── "Don't re-explain basics"                                │
│     └── "Connect new concepts to what user knows"                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘

EXAMPLE: Teaching "Convolutional Neural Networks"

┌─────────────────────────────────────────────────────────────────┐
│  TEACHING PROMPT:                                                │
│                                                                  │
│  "You are teaching 'Convolutional Neural Networks' to a user.   │
│                                                                  │
│   THE USER HAS ALREADY LEARNED:                                  │
│                                                                  │
│   ✓ Topic 1: What is a Neural Network (5 days ago)              │
│     - Neurons, weights, biases                                   │
│     - Activation functions (ReLU, sigmoid)                       │
│     - Forward propagation                                        │
│                                                                  │
│   ✓ Topic 2: Backpropagation (3 days ago)                       │
│     - Chain rule for gradients                                   │
│     - Error flows backward                                       │
│     - Learning rate                                              │
│                                                                  │
│   ✓ Topic 3: Training Neural Networks (1 day ago)               │
│     - Loss functions                                             │
│     - Gradient descent                                           │
│     - Epochs and batches                                         │
│                                                                  │
│   INSTRUCTIONS:                                                  │
│   - DO NOT re-explain what neurons, weights, or backprop are    │
│   - DO reference these concepts: 'Remember how we discussed...' │
│   - DO build upon their knowledge: 'CNNs use the same backprop' │
│   - DO connect new concepts: 'The convolution is like a filter' │
│   - Assume they understand: activation, gradients, training     │
│                                                                  │
│   User's current depth level: Foundational                       │
│   User's learning style: Visual (use diagrams/examples)"         │
└─────────────────────────────────────────────────────────────────┘
```

#### Database View for Learning Context

```sql
-- View to get user's prior knowledge for a learning path
CREATE VIEW user_learning_context AS
SELECT 
    ts.user_id,
    ts.learning_path_id,
    lp.title AS path_title,
    jsonb_agg(
        jsonb_build_object(
            'topic_id', ts.topic_id,
            'title', ts.title,
            'key_concepts', ts.key_concepts,
            'depth_level', ts.depth_level,
            'learned_at', ts.learned_at,
            'days_ago', EXTRACT(DAY FROM NOW() - ts.learned_at)::INTEGER,
            'retention_score', ts.retention_score
        )
        ORDER BY ts.learned_at
    ) AS prior_knowledge
FROM topic_summaries ts
JOIN learning_paths lp ON ts.learning_path_id = lp.id
GROUP BY ts.user_id, ts.learning_path_id, lp.title;

-- Usage: Get all prior knowledge for a user's learning path
-- SELECT prior_knowledge FROM user_learning_context 
-- WHERE user_id = ? AND learning_path_id = ?;
```

#### How Teaching Agent Uses This

```python
async def teach_topic(user_id: str, topic_id: str, path_id: str):
    # 1. Get the new topic to teach
    new_topic = await get_topic(topic_id)
    
    # 2. Get ALL prior knowledge from this learning path
    prior_knowledge = await db.query("""
        SELECT title, key_concepts, depth_level, 
               learned_at, retention_score
        FROM topic_summaries
        WHERE user_id = $1 AND learning_path_id = $2
        ORDER BY learned_at ASC
    """, user_id, path_id)
    
    # 3. Build context for LLM
    context = {
        "new_topic": new_topic.title,
        "prior_topics": [
            {
                "title": t.title,
                "concepts": t.key_concepts,
                "days_ago": (now - t.learned_at).days,
                "retention": t.retention_score
            }
            for t in prior_knowledge
        ],
        "user_depth": get_current_depth(prior_knowledge),
        "learning_style": user.learning_style
    }
    
    # 4. Teaching prompt includes prior knowledge
    prompt = build_teaching_prompt(context)
    
    return await teaching_agent.teach(prompt)
```

#### What Gets Referenced

| Prior Knowledge | How It's Used |
|-----------------|---------------|
| **Completed topic titles** | "Remember when we covered X..." |
| **Key concepts mastered** | Don't re-explain; reference instead |
| **Examples user has seen** | Build on familiar examples |
| **Depth level achieved** | Match complexity to user's level |
| **Days since learned** | Recent = fresh; old = brief refresh |
| **Retention scores** | Low retention = quick review first |

#### Edge Cases

```
CASE 1: First Topic in Path
└── No prior knowledge
└── Start from basics
└── Establish foundational vocabulary

CASE 2: Low Retention on Prerequisite
└── Prior topic retention < 50%
└── AI does quick 1-paragraph refresh
└── "Before we continue, let's quickly recall..."

CASE 3: User Skipped Topics
└── Gap in knowledge detected
└── AI flags: "You might want to review X first"
└── Offer to explain missing concept briefly

CASE 4: Long Gap Between Sessions
└── Last activity > 7 days ago
└── AI starts with brief recap
└── "Last time we covered... shall I quickly review?"
```

---

## Cognitive Learner Model (Simplified)

> **THE MOAT:** These structures capture how each user learns, not just what they've learned.
> This accumulated understanding is what makes the app irreplaceable.
>
> **Design Philosophy:** The LLM writes FOR itself. Keep it simple, let the LLM summarize.

### Design: 2 Structures Instead of 5 Tables

**Previous design:** 5 normalized tables with granular event logging.

**New design:** 2 simple structures that the LLM can read and write directly.

```
┌─────────────────────────────────────────────────────────────────┐
│  WHY THIS IS BETTER                                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  OLD: LLM outputs → Parse → INSERT 5 tables → JOIN → Format → LLM│
│  NEW: LLM outputs → JSONB column → LLM reads                     │
│                                                                  │
│  The LLM writes what IT needs to remember.                       │
│  User can see and correct the model in Settings.                 │
│  1500 char limit forces prioritization.                          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

### 10. users.learning_preferences (Global)

Add to existing users table — persists across all subjects.

```sql
-- Add to users table (global preferences)
ALTER TABLE users ADD COLUMN learning_preferences JSONB DEFAULT '{
    "detail_level": 0.5,
    "formality": 0.5,
    "pace": 0.5,
    "example_preference": "auto"
}';

/*
Preference scales (0.0 to 1.0):
- detail_level:  0.0 = brief/concise       → 1.0 = detailed/thorough
- formality:     0.0 = casual/friendly     → 1.0 = formal/academic
- pace:          0.0 = fast/skip basics    → 1.0 = slow/thorough
- example_preference: "code_first" | "concept_first" | "auto"

Starting at 0.5 = neutral, system learns over time.
Adjustments are gradual (±0.1 per session) based on user responses.
*/

COMMENT ON COLUMN users.learning_preferences IS 'Global learning style preferences, updated automatically by LLM';
```

---

### 11. subject_learner_models (Per Subject)

One row per user per subject — the LLM writes a summary after each session.

```sql
CREATE TABLE public.subject_learner_models (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    subject_area TEXT NOT NULL,  -- "python", "calculus", "spanish"
    
    -- LLM-written model (the core of personalization)
    model JSONB DEFAULT '{
        "misconceptions": [],
        "struggles": [],
        "strengths": [],
        "effective_explanations": [],
        "example_mode": "full",
        "example_concepts": [],
        "notes": ""
    }',
    /*
    Worked Example Tracking (d=0.57):
    - example_mode: "full" | "fading" | "problem_first"
      - full: Novice — always show complete worked examples
      - fading: Developing — show partial examples, leave later steps
      - problem_first: Competent — let them try, only example if struggle
    - example_concepts: List of concepts where user has graduated to problem_first
      - e.g., ["power_rule", "chain_rule"] means no examples needed for these
      - Empty = needs examples for all concepts in this subject
    Schema enforced by Summary Agent prompt:
    {
        "misconceptions": [
            {"topic": "recursion", "belief": "thinks it's just a loop", "confidence": 0.8}
        ],
        "struggles": ["understanding return vs print", "off-by-one errors"],
        "strengths": ["list comprehensions", "string formatting"],
        "effective_explanations": ["code_first", "real_world_analogy"],
        "notes": "Responds well to cooking analogies. Gets frustrated with math notation."
    }
    
    MUST BE <1500 CHARACTERS (enforced by Summary Agent prompt)
    */
    
    -- Last session context (for continuity)
    last_session_summary TEXT,  -- "Last time we covered list comprehensions..."
    
    -- Metadata
    session_count INTEGER DEFAULT 0,
    model_version INTEGER DEFAULT 1,  -- Bump when schema changes
    
    -- Timestamps
    created_at TIMESTAMPTZ DEFAULT NOW(),
    last_updated_at TIMESTAMPTZ DEFAULT NOW(),
    
    UNIQUE(user_id, subject_area)
);

CREATE INDEX idx_subject_learner_models_user ON subject_learner_models(user_id);
CREATE INDEX idx_subject_learner_models_subject ON subject_learner_models(user_id, subject_area);

COMMENT ON TABLE subject_learner_models IS 'Per-subject learning model, written by LLM after each session';
COMMENT ON COLUMN subject_learner_models.model IS 'LLM-written summary of misconceptions, struggles, strengths (<1500 chars)';
COMMENT ON COLUMN subject_learner_models.last_session_summary IS 'Context for next session opening';
```

---

### Learner Model Flow

```
┌──────────────────────────────────────────────────────────────────┐
│  ONBOARDING                                                       │
├──────────────────────────────────────────────────────────────────┤
│  Assessment Agent:                                                │
│  - Tests baseline knowledge                                       │
│  - Detects initial misconceptions                                 │
│  - Infers tone/detail preference from how user responds           │
│                                                                   │
│  Writes:                                                          │
│  → users.learning_preferences (global)                            │
│  → subject_learner_models.model (first entry)                     │
└──────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│  DURING SESSION                                                   │
├──────────────────────────────────────────────────────────────────┤
│  Teaching Agent prompt includes:                                  │
│  - users.learning_preferences (translated to natural language)    │
│  - subject_learner_models.model                                   │
│                                                                   │
│  LLM adapts tone, detail, examples accordingly.                   │
│  Conversation stored in memory (context window).                  │
└──────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│  END OF SESSION (two parallel agent calls)                        │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  1. Summary Agent:                                                │
│     Input: full conversation + current model                      │
│     Output: updated model JSON (<1500 chars)                      │
│     Writes: subject_learner_models.model                          │
│                                                                   │
│  2. Preferences Agent:                                            │
│     Input: conversation + current preferences                     │
│     Output: adjustment scores (±0.1)                              │
│     Writes: users.learning_preferences (gradual drift)            │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

---

### User Adjustment UI

Users can see and correct their model in Settings:

```
┌─────────────────────────────────────────────────────────────────┐
│  SETTINGS → "How I Learn"                                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Explanation style:   ○ Brief    ● Detailed    ○ Auto           │
│  Tone:                ○ Formal   ● Friendly    ○ Auto           │
│  Examples:            ● Code-first  ○ Concept-first  ○ Auto     │
│                                                                  │
│  ─────────────────────────────────────────────────────────────  │
│                                                                  │
│  "Claude thinks you struggle with:"                              │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ • Variable scope vs function scope        [✓] [✗]       │    │
│  │ • Recursion base cases                    [✓] [✗]       │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  [✓] = "Yes, still working on this"                             │
│  [✗] = "I've got this now, remove it"                           │
│                                                                  │
│  ─────────────────────────────────────────────────────────────  │
│                                                                  │
│  "Claude thinks you're strong at:"                               │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ • List comprehensions ✓                                  │    │
│  │ • String formatting ✓                                    │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**User correction query:**

```sql
-- When user clicks [✗] on a misconception
UPDATE subject_learner_models 
SET model = model - 'misconceptions->0'  -- Remove first misconception
WHERE user_id = $1 AND subject_area = $2;

-- More robust: filter out specific topic
UPDATE subject_learner_models 
SET model = jsonb_set(
    model,
    '{misconceptions}',
    (
        SELECT jsonb_agg(m)
        FROM jsonb_array_elements(model->'misconceptions') m
        WHERE m->>'topic' != 'variable_scope'
    )
)
WHERE user_id = $1 AND subject_area = $2;
```

---

### What We Lost (And Why It's OK)

| Lost Data | Why It's OK |
|-----------|-------------|
| Per-event timestamps | Session recordings have this for debugging |
| Occurrence counts | LLM can note "recurring issue" in notes |
| Intervention history | Session recordings log all exchanges |
| Struggle signal patterns | Can add back in v2 if needed |
| Retention curve params | Using standard decay for now |

**For analytics later (v2):** Add append-only `learning_events` table if needed.

---

## Supporting Tables

### 15. sessions (LLM Usage Tracking)

**Critical for cost management (ADR-011).**

```sql
CREATE TABLE public.sessions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    
    -- Session info
    session_type VARCHAR(20) DEFAULT 'learning'
        CHECK (session_type IN ('learning', 'assessment', 'path_generation', 'verification')),
    
    -- LLM usage (for cost tracking)
    llm_calls INTEGER DEFAULT 0,
    input_tokens INTEGER DEFAULT 0,
    output_tokens INTEGER DEFAULT 0,
    estimated_cost_usd NUMERIC(10,6) DEFAULT 0,
    
    -- Provider tracking
    primary_provider VARCHAR(20),   -- 'claude', 'openai'
    fallback_used BOOLEAN DEFAULT FALSE,
    cache_hits INTEGER DEFAULT 0,
    
    -- Teaching metrics (ADR-019: Testing is Teaching enforcement)
    teaching_metrics JSONB DEFAULT '{}',
    /*
    Format:
    {
      "total_ai_turns": 15,
      "explain_turns": 4,
      "test_turns": 11,
      "explain_ratio": 0.27,
      "questions_asked": 12,
      "regenerations_for_ratio": 1,
      "responses_without_question": 0
    }
    */
    
    -- Timestamps
    started_at TIMESTAMPTZ DEFAULT NOW(),
    ended_at TIMESTAMPTZ,
    
    -- Associated conversation
    conversation_id UUID REFERENCES conversations(id) ON DELETE SET NULL
);

CREATE INDEX idx_sessions_user ON sessions(user_id);
CREATE INDEX idx_sessions_user_date ON sessions(user_id, started_at);

COMMENT ON TABLE sessions IS 'Learning sessions with LLM usage tracking for cost management';
COMMENT ON COLUMN sessions.teaching_metrics IS 'Tracks explain vs test ratio for quality enforcement';
```

---

### 16. session_recordings (Debugging & Analysis)

**Full session recordings for debugging, quality analysis, and support.**

```sql
CREATE TABLE public.session_recordings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id UUID NOT NULL REFERENCES sessions(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    
    -- Full conversation for replay/debugging
    messages JSONB NOT NULL DEFAULT '[]',
    /*
    Format:
    [
        {
            "role": "user" | "assistant",
            "content": "...",
            "timestamp": "ISO 8601",
            "agent": "teaching|assessment|orchestrator",
            "tokens": 150
        }
    ]
    */
    
    -- All LLM calls with prompts/responses (for debugging)
    agent_calls JSONB DEFAULT '[]',
    /*
    Format:
    [
        {
            "agent": "teaching",
            "model": "claude-3-5-sonnet",
            "system_prompt_hash": "abc123",  -- Hash to avoid storing full prompt
            "input_tokens": 4000,
            "output_tokens": 1200,
            "latency_ms": 1500,
            "timestamp": "ISO 8601"
        }
    ]
    */
    
    -- Metadata
    total_tokens INTEGER,
    total_cost_cents INTEGER,
    duration_seconds INTEGER,
    
    -- Retention policy (auto-delete after N days)
    expires_at TIMESTAMPTZ DEFAULT (NOW() + INTERVAL '90 days'),
    
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_session_recordings_session ON session_recordings(session_id);
CREATE INDEX idx_session_recordings_user ON session_recordings(user_id);
CREATE INDEX idx_session_recordings_expires ON session_recordings(expires_at);

COMMENT ON TABLE session_recordings IS 'Full session recordings for debugging and quality analysis';
COMMENT ON COLUMN session_recordings.expires_at IS 'Auto-delete after 90 days to manage storage';
```

---

### 17. assessment_flags (Quality Control)

**User-flagged unfair questions for review.**

```sql
CREATE TABLE public.assessment_flags (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    
    -- Assessment context
    assessment_result_id UUID REFERENCES assessment_results(id) ON DELETE SET NULL,
    topic_id UUID REFERENCES module_topics(id) ON DELETE SET NULL,
    
    -- The flagged question
    question_text TEXT NOT NULL,
    user_answer TEXT,
    expected_answer TEXT,
    
    -- User feedback
    flag_reason TEXT,  -- Optional: "Question was unclear", "Not taught in lesson"
    
    -- Admin review
    reviewed BOOLEAN DEFAULT FALSE,
    reviewed_at TIMESTAMPTZ,
    reviewed_by UUID,  -- Admin user ID
    review_outcome VARCHAR(20),  -- 'valid_flag', 'invalid_flag', 'question_fixed'
    review_notes TEXT,
    
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_assessment_flags_user ON assessment_flags(user_id);
CREATE INDEX idx_assessment_flags_unreviewed ON assessment_flags(reviewed) WHERE reviewed = FALSE;
CREATE INDEX idx_assessment_flags_topic ON assessment_flags(topic_id);

COMMENT ON TABLE assessment_flags IS 'User-flagged unfair assessment questions for admin review';
```

---

### 18. curriculum_feedback (Cross-User Learning)

**Aggregate signals about curriculum effectiveness for future improvements.**

```sql
CREATE TABLE public.curriculum_feedback (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    
    -- What curriculum element
    subject_area TEXT NOT NULL,
    topic_title TEXT NOT NULL,
    module_title TEXT,
    
    -- Signal type
    signal_type VARCHAR(30) NOT NULL CHECK (signal_type IN (
        'low_completion',      -- < 50% complete this topic
        'high_completion',     -- > 90% complete this topic
        'prerequisite_fail',   -- Failed topic Y after passing X (X might not be prereq)
        'user_flag',           -- User clicked "this doesn't seem right"
        'skip_request',        -- User skipped topic
        'time_overrun',        -- Took > 2x estimated time
        'high_struggle',       -- Multiple failed assessments
        'easy_pass',           -- Passed with > 90% on first try
        'subject_requested',   -- Subject was requested (for popularity tracking)
        'subject_abandoned',   -- User abandoned early (possible poor curriculum)
        'curriculum_challenged' -- User challenged curriculum structure
    )),
    
    -- Aggregate data
    signal_value FLOAT,        -- Percentage or count depending on type
    sample_size INTEGER,       -- Number of users in this sample
    
    -- Time window
    period_start DATE,
    period_end DATE,
    
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_curriculum_feedback_subject ON curriculum_feedback(subject_area);
CREATE INDEX idx_curriculum_feedback_signal ON curriculum_feedback(signal_type);
CREATE INDEX idx_curriculum_feedback_topic ON curriculum_feedback(subject_area, topic_title);

COMMENT ON TABLE curriculum_feedback IS 'Aggregated curriculum effectiveness signals for cross-user learning';
COMMENT ON COLUMN curriculum_feedback.signal_value IS 'Interpretation depends on signal_type: percentage for rates, count for occurrences';
```

---

### 19. curriculum_changes (Admin Reports)

**Log of all curriculum modifications for admin review.**

```sql
CREATE TABLE public.curriculum_changes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    learning_path_id UUID REFERENCES learning_paths(id) ON DELETE SET NULL,
    
    -- What subject/context
    subject_area TEXT NOT NULL,
    
    -- Change type
    change_type VARCHAR(20) NOT NULL CHECK (change_type IN (
        'regenerate',   -- Full curriculum regeneration
        'skip',         -- User skipped a topic
        'reorder',      -- User requested reorder
        'flag',         -- User flagged topic as incorrect
        'challenge'     -- User challenged order with reason
    )),
    
    -- Before/after
    original_curriculum JSONB,
    new_curriculum JSONB,
    
    -- User input
    user_reason TEXT,          -- What user said was wrong
    
    -- LLM response
    llm_explanation TEXT,      -- Why LLM made the change (or defended original)
    llm_agreed BOOLEAN,        -- Did LLM agree with user or push back?
    
    -- Admin review (for prototype: manual review)
    reviewed BOOLEAN DEFAULT FALSE,
    reviewed_at TIMESTAMPTZ,
    reviewed_by UUID,
    review_notes TEXT,
    
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_curriculum_changes_user ON curriculum_changes(user_id);
CREATE INDEX idx_curriculum_changes_subject ON curriculum_changes(subject_area);
CREATE INDEX idx_curriculum_changes_unreviewed ON curriculum_changes(reviewed) WHERE reviewed = FALSE;
CREATE INDEX idx_curriculum_changes_type ON curriculum_changes(change_type);

COMMENT ON TABLE curriculum_changes IS 'Log of curriculum modifications for admin review and analysis';
COMMENT ON COLUMN curriculum_changes.original_curriculum IS 'Curriculum state before change (can be partial for reorder)';
COMMENT ON COLUMN curriculum_changes.llm_agreed IS 'Whether LLM agreed with user challenge or defended original order';
```

---

### 11. streaks

Daily usage tracking for gamification.

```sql
CREATE TABLE public.streaks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    
    -- Honest Streak (ADR-016: counts days with passed recall, not app opens)
    honest_streak INTEGER DEFAULT 0,           -- Days where recall was passed
    longest_honest_streak INTEGER DEFAULT 0,   -- Historical best
    last_recall_pass_date DATE,                -- For streak calculation
    honest_streak_start_date DATE,             -- When current streak started
    
    -- Legacy activity streak (for comparison/analytics)
    activity_streak INTEGER DEFAULT 0,         -- Days app was opened
    last_activity_date DATE,
    
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    
    UNIQUE(user_id)
);

COMMENT ON TABLE streaks IS 'User streaks - honest streak (recall-based) is primary';
COMMENT ON COLUMN streaks.honest_streak IS 'Consecutive days where user passed at least one recall challenge';
COMMENT ON COLUMN streaks.activity_streak IS 'Legacy: consecutive days app was opened (not displayed to user)';
```

---

### 13. achievements

Badge/achievement definitions.

```sql
CREATE TABLE public.achievements (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    
    -- Achievement info
    code VARCHAR(50) NOT NULL UNIQUE,  -- 'first_lesson', 'week_streak', etc.
    name TEXT NOT NULL,
    description TEXT,
    icon_name VARCHAR(50),  -- Icon identifier
    
    -- Criteria (for auto-awarding)
    criteria_type VARCHAR(30),  -- 'streak', 'lessons_completed', 'paths_completed', etc.
    criteria_value INTEGER,     -- e.g., 7 for 7-day streak
    
    -- Rarity
    rarity VARCHAR(20) DEFAULT 'common'
        CHECK (rarity IN ('common', 'rare', 'epic', 'legendary')),
    
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Seed some achievements
INSERT INTO achievements (code, name, description, criteria_type, criteria_value, rarity) VALUES
    ('first_lesson', 'First Steps', 'Complete your first lesson', 'lessons_completed', 1, 'common'),
    ('streak_7', 'Week Warrior', 'Maintain a 7-day streak', 'streak', 7, 'common'),
    ('streak_30', 'Monthly Master', 'Maintain a 30-day streak', 'streak', 30, 'rare'),
    ('path_complete', 'Path Finder', 'Complete a learning path', 'paths_completed', 1, 'common'),
    ('perfect_quiz', 'Perfect Score', 'Get 100% on an assessment', 'perfect_assessment', 1, 'rare');

COMMENT ON TABLE achievements IS 'Achievement/badge definitions';
```

---

### 14. user_achievements

Achievements earned by users.

```sql
CREATE TABLE public.user_achievements (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    achievement_id UUID NOT NULL REFERENCES achievements(id) ON DELETE CASCADE,
    
    earned_at TIMESTAMPTZ DEFAULT NOW(),
    
    UNIQUE(user_id, achievement_id)
);

CREATE INDEX idx_user_achievements_user ON user_achievements(user_id);

COMMENT ON TABLE user_achievements IS 'Achievements earned by users';
```

---

### 15. verification_requests

Content verification flow (ADR: Private Verification).

```sql
CREATE TABLE public.verification_requests (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    
    -- Context
    topic_id UUID REFERENCES module_topics(id) ON DELETE SET NULL,
    conversation_id UUID REFERENCES conversations(id) ON DELETE SET NULL,
    
    -- Flagged content
    original_content TEXT NOT NULL,      -- What AI said
    user_concern TEXT,                   -- User's reason for flagging
    
    -- AI verification attempt
    ai_verification_response TEXT,       -- AI's double-check response
    ai_verification_at TIMESTAMPTZ,
    
    -- Status
    status VARCHAR(20) DEFAULT 'pending'
        CHECK (status IN ('pending', 'ai_resolved', 'escalated', 'admin_resolved', 'dismissed')),
    
    -- Escalation
    escalated_at TIMESTAMPTZ,
    escalated_to_email TEXT,            -- Admin email
    user_notified BOOLEAN DEFAULT FALSE,
    
    -- Resolution
    admin_response TEXT,
    resolved_at TIMESTAMPTZ,
    resolved_by UUID REFERENCES users(id),
    
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_verification_requests_user ON verification_requests(user_id);
CREATE INDEX idx_verification_requests_status ON verification_requests(status);

COMMENT ON TABLE verification_requests IS 'Content verification requests from users';
```

---

### 16. subscriptions

User subscription/payment info.

```sql
CREATE TABLE public.subscriptions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    
    -- Plan
    plan_type VARCHAR(20) DEFAULT 'free'
        CHECK (plan_type IN ('free', 'premium', 'enterprise')),
    
    -- Billing
    stripe_customer_id TEXT,
    stripe_subscription_id TEXT,
    
    -- Status
    status VARCHAR(20) DEFAULT 'active'
        CHECK (status IN ('active', 'cancelled', 'past_due', 'trialing')),
    
    -- Dates
    current_period_start TIMESTAMPTZ,
    current_period_end TIMESTAMPTZ,
    cancelled_at TIMESTAMPTZ,
    
    -- Usage limits
    daily_sessions_limit INTEGER,  -- NULL = unlimited
    daily_sessions_used INTEGER DEFAULT 0,
    
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    
    UNIQUE(user_id)
);

CREATE INDEX idx_subscriptions_user ON subscriptions(user_id);
CREATE INDEX idx_subscriptions_stripe ON subscriptions(stripe_customer_id);

COMMENT ON TABLE subscriptions IS 'User subscription and billing info';
```

---

## Redis Cache Schema

Not SQL, but important to document:

```
REDIS KEY PATTERNS:

# UI Translations (ADR-010: i18n)
translations:{language_code}
  → JSON object of translated UI strings
  → TTL: 30 days
  → Example: translations:es → {"Start Learning": "Comenzar a Aprender", ...}

# Explanation Cache (cost optimization)
explanation:{topic_hash}:{language}
  → Cached AI explanation for common topics
  → TTL: 24 hours
  → Example: explanation:abc123:en → "Neural networks are..."

# Assessment Cache
assessments:{topic_id}:{difficulty}
  → Cached quiz questions
  → TTL: 1 hour
  → Example: assessments:uuid:3 → [{question: "...", ...}]

# Rate Limiting
ratelimit:{user_id}:{endpoint}
  → Counter for rate limiting
  → TTL: 1 minute
  → Example: ratelimit:uuid:teach → 5

# Session State
session:{session_id}
  → Current session state (conversation context)
  → TTL: 30 minutes
  → Example: session:uuid → {topic_id: "...", context: {...}}

# User Daily Stats (for streaks, limits)
daily:{user_id}:{date}
  → Daily usage counters
  → TTL: 48 hours
  → Example: daily:uuid:2024-01-01 → {sessions: 3, tokens: 5000}

# LLM Fallback Status
llm:status:{provider}
  → Health status of LLM providers
  → TTL: 5 minutes
  → Example: llm:status:claude → {healthy: true, error_rate: 0.01}
```

---

## Indexes

Summary of all indexes (already defined inline above):

```sql
-- Users
-- (Primary key index automatic)
CREATE INDEX idx_users_created_at ON users(created_at);

-- Learning Paths
CREATE INDEX idx_learning_paths_user_id ON learning_paths(user_id);
CREATE INDEX idx_learning_paths_status ON learning_paths(user_id, status);
CREATE INDEX idx_learning_paths_user_active ON learning_paths(user_id, status, last_activity_at DESC)
    WHERE status = 'active';  -- Partial index for active paths
CREATE INDEX idx_learning_paths_created_at ON learning_paths(created_at);

-- Path Modules
CREATE INDEX idx_path_modules_path ON path_modules(learning_path_id, order_index);

-- Module Topics
CREATE INDEX idx_module_topics_module ON module_topics(module_id, order_index);

-- Topic Progress
CREATE INDEX idx_topic_progress_user ON topic_progress(user_id);
CREATE INDEX idx_topic_progress_user_topic ON topic_progress(user_id, topic_id);
CREATE INDEX idx_topic_progress_user_status ON topic_progress(user_id, status);

-- Conversations
CREATE INDEX idx_conversations_user ON conversations(user_id, is_active);
CREATE INDEX idx_conversations_context ON conversations(user_id, topic_id);
CREATE INDEX idx_conversations_created_at ON conversations(created_at);

-- Assessments
CREATE INDEX idx_assessments_topic ON assessments(topic_id);
CREATE INDEX idx_assessments_topic_difficulty ON assessments(topic_id, difficulty);

-- Assessment Results
CREATE INDEX idx_assessment_results_user ON assessment_results(user_id);
CREATE INDEX idx_assessment_results_user_topic ON assessment_results(user_id, topic_id);
CREATE INDEX idx_assessment_results_answered_at ON assessment_results(answered_at);

-- Topic Summaries (for learning continuity)
CREATE INDEX idx_topic_summaries_user ON topic_summaries(user_id);
CREATE INDEX idx_topic_summaries_user_path ON topic_summaries(user_id, learning_path_id);
CREATE INDEX idx_topic_summaries_review ON topic_summaries(user_id, next_review_suggested);
CREATE INDEX idx_topic_summaries_user_topic ON topic_summaries(user_id, topic_id);  -- For FK lookups

-- Sessions
CREATE INDEX idx_sessions_user ON sessions(user_id);
CREATE INDEX idx_sessions_user_date ON sessions(user_id, started_at);
CREATE INDEX idx_sessions_started_at ON sessions(started_at);  -- For time-range queries

-- User Achievements
CREATE INDEX idx_user_achievements_user ON user_achievements(user_id);

-- Verification Requests
CREATE INDEX idx_verification_requests_user ON verification_requests(user_id);
CREATE INDEX idx_verification_requests_status ON verification_requests(status);
CREATE INDEX idx_verification_requests_created_at ON verification_requests(created_at);

-- Subscriptions
CREATE INDEX idx_subscriptions_user ON subscriptions(user_id);
CREATE INDEX idx_subscriptions_stripe ON subscriptions(stripe_customer_id);

-- Streaks
CREATE INDEX idx_streaks_user ON streaks(user_id);
CREATE INDEX idx_streaks_last_activity ON streaks(last_activity_date);

-- Composite indexes for common joins
CREATE INDEX idx_topic_progress_user_path ON topic_progress(user_id, topic_id)
    INCLUDE (status, mastery_score);  -- Covering index for progress queries

CREATE INDEX idx_module_topics_path_lookup ON module_topics(module_id)
    INCLUDE (title, order_index, status);  -- For path detail queries
```

---

## Row-Level Security (RLS)

Supabase RLS policies - users only see their own data.

### Performance Considerations

```
┌─────────────────────────────────────────────────────────────────┐
│  RLS PERFORMANCE STRATEGY                                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  PROBLEM: Nested subqueries in RLS policies are slow at scale   │
│                                                                  │
│  SOLUTION: Denormalize user_id onto child tables                 │
│                                                                  │
│  path_modules     → has user_id (denormalized from learning_path)│
│  module_topics    → has user_id (denormalized from path_modules) │
│                                                                  │
│  This avoids JOIN in RLS = O(1) instead of O(n)                 │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Schema Updates for RLS Performance

```sql
-- Add denormalized user_id to path_modules
ALTER TABLE path_modules ADD COLUMN user_id UUID REFERENCES users(id);

-- Populate from existing data
UPDATE path_modules pm
SET user_id = lp.user_id
FROM learning_paths lp
WHERE pm.learning_path_id = lp.id;

-- Make NOT NULL after population
ALTER TABLE path_modules ALTER COLUMN user_id SET NOT NULL;

-- Add denormalized user_id to module_topics  
ALTER TABLE module_topics ADD COLUMN user_id UUID REFERENCES users(id);

-- Populate from existing data
UPDATE module_topics mt
SET user_id = pm.user_id
FROM path_modules pm
WHERE mt.module_id = pm.id;

-- Make NOT NULL after population
ALTER TABLE module_topics ALTER COLUMN user_id SET NOT NULL;

-- Add indexes for the new columns
CREATE INDEX idx_path_modules_user ON path_modules(user_id);
CREATE INDEX idx_module_topics_user ON module_topics(user_id);
```

### RLS Policies

```sql
-- Enable RLS on all tables
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
ALTER TABLE learning_paths ENABLE ROW LEVEL SECURITY;
ALTER TABLE path_modules ENABLE ROW LEVEL SECURITY;
ALTER TABLE module_topics ENABLE ROW LEVEL SECURITY;
ALTER TABLE topic_progress ENABLE ROW LEVEL SECURITY;
ALTER TABLE conversations ENABLE ROW LEVEL SECURITY;
ALTER TABLE assessments ENABLE ROW LEVEL SECURITY;
ALTER TABLE assessment_results ENABLE ROW LEVEL SECURITY;
ALTER TABLE topic_summaries ENABLE ROW LEVEL SECURITY;
ALTER TABLE sessions ENABLE ROW LEVEL SECURITY;
ALTER TABLE streaks ENABLE ROW LEVEL SECURITY;
ALTER TABLE user_achievements ENABLE ROW LEVEL SECURITY;
ALTER TABLE verification_requests ENABLE ROW LEVEL SECURITY;
ALTER TABLE subscriptions ENABLE ROW LEVEL SECURITY;

-- ============================================================
-- USERS
-- ============================================================

-- Users: can only read/update own profile, exclude soft-deleted
CREATE POLICY "Users can view own profile"
    ON users FOR SELECT
    USING (auth.uid() = id AND deleted_at IS NULL);

CREATE POLICY "Users can update own profile"
    ON users FOR UPDATE
    USING (auth.uid() = id AND deleted_at IS NULL);

-- ============================================================
-- LEARNING PATHS (direct user_id check)
-- ============================================================

CREATE POLICY "Users can CRUD own learning paths"
    ON learning_paths FOR ALL
    USING (auth.uid() = user_id);

-- ============================================================
-- PATH MODULES (now has denormalized user_id - fast!)
-- ============================================================

CREATE POLICY "Users can CRUD own path modules"
    ON path_modules FOR ALL
    USING (auth.uid() = user_id);

-- ============================================================
-- MODULE TOPICS (now has denormalized user_id - fast!)
-- ============================================================

CREATE POLICY "Users can CRUD own module topics"
    ON module_topics FOR ALL
    USING (auth.uid() = user_id);

-- ============================================================
-- DIRECT USER OWNERSHIP TABLES
-- ============================================================

-- Topic Progress: users see only their own
CREATE POLICY "Users can CRUD own progress"
    ON topic_progress FOR ALL
    USING (auth.uid() = user_id);

-- Conversations: users see only their own
CREATE POLICY "Users can CRUD own conversations"
    ON conversations FOR ALL
    USING (auth.uid() = user_id);

-- Assessments: public read (shared questions)
CREATE POLICY "Anyone can view assessments"
    ON assessments FOR SELECT
    USING (true);

-- Assessment Results: users see only their own
CREATE POLICY "Users can CRUD own assessment results"
    ON assessment_results FOR ALL
    USING (auth.uid() = user_id);

-- Topic Summaries: users see only their own
CREATE POLICY "Users can CRUD own topic summaries"
    ON topic_summaries FOR ALL
    USING (auth.uid() = user_id);

-- Sessions: users see only their own
CREATE POLICY "Users can view own sessions"
    ON sessions FOR SELECT
    USING (auth.uid() = user_id);

-- Streaks: users see only their own
CREATE POLICY "Users can CRUD own streaks"
    ON streaks FOR ALL
    USING (auth.uid() = user_id);

-- Achievements: public read (definitions)
CREATE POLICY "Anyone can view achievements"
    ON achievements FOR SELECT
    USING (true);

-- User Achievements: users see only their own
CREATE POLICY "Users can view own achievements"
    ON user_achievements FOR SELECT
    USING (auth.uid() = user_id);

-- Verification Requests: users see only their own
CREATE POLICY "Users can CRUD own verification requests"
    ON verification_requests FOR ALL
    USING (auth.uid() = user_id);

-- Subscriptions: users see only their own
CREATE POLICY "Users can view own subscription"
    ON subscriptions FOR SELECT
    USING (auth.uid() = user_id);
```

### Triggers for Denormalization

```sql
-- Automatically set user_id on path_modules from learning_paths
CREATE OR REPLACE FUNCTION set_path_module_user_id()
RETURNS TRIGGER AS $$
BEGIN
    NEW.user_id := (SELECT user_id FROM learning_paths WHERE id = NEW.learning_path_id);
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_set_path_module_user_id
    BEFORE INSERT ON path_modules
    FOR EACH ROW
    EXECUTE FUNCTION set_path_module_user_id();

-- Automatically set user_id on module_topics from path_modules
CREATE OR REPLACE FUNCTION set_module_topic_user_id()
RETURNS TRIGGER AS $$
BEGIN
    NEW.user_id := (SELECT user_id FROM path_modules WHERE id = NEW.module_id);
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_set_module_topic_user_id
    BEFORE INSERT ON module_topics
    FOR EACH ROW
    EXECUTE FUNCTION set_module_topic_user_id();
```

---

## Migrations Strategy

### Approach

1. Use Supabase migrations (SQL files)
2. Version controlled in `/supabase/migrations/`
3. Run via `supabase db push` or CI/CD

### Migration File Naming

```
/supabase/migrations/
  20241208000001_create_users.sql
  20241208000002_create_learning_paths.sql
  20241208000003_create_modules_topics.sql
  20241208000004_create_progress.sql
  20241208000005_create_conversations.sql
  20241208000006_create_assessments.sql
  20241208000007_create_gamification.sql
  20241208000008_create_verification.sql
  20241208000009_create_subscriptions.sql
  20241208000010_create_sessions.sql
  20241208000011_add_rls_policies.sql
  20241208000012_add_indexes.sql
  20241208000013_seed_achievements.sql
```

### Rollback Strategy

- Each migration should have a corresponding down migration
- Test rollbacks in staging before production
- Never modify deployed migrations; create new ones

---

## Mastery Calculation Formulas

### Core Mastery Score

```python
def calculate_mastery(assessment_results: list[dict]) -> float:
    """
    Calculate mastery score from assessment results.
    
    Args:
        assessment_results: List of {"question_id": str, "correct": bool, "weight": float}
    
    Returns:
        Mastery score 0.0 to 1.0
    
    Example:
        results = [
            {"question_id": "q1", "correct": True, "weight": 1.0},   # recall
            {"question_id": "q2", "correct": True, "weight": 1.5},   # application
            {"question_id": "q3", "correct": False, "weight": 2.0},  # analysis
        ]
        # weighted_correct = 1.0 + 1.5 = 2.5
        # total_weight = 1.0 + 1.5 + 2.0 = 4.5
        # mastery = 2.5 / 4.5 = 0.556
    """
    if not assessment_results:
        return 0.0
    
    total_weight = sum(r["weight"] for r in assessment_results)
    weighted_correct = sum(
        r["weight"] for r in assessment_results 
        if r["correct"] and not r.get("flagged_unfair", False)
    )
    
    return weighted_correct / total_weight
```

### Question Weight by Type

```python
QUESTION_WEIGHTS = {
    "recall": 1.0,        # "What is a function?"
    "comprehension": 1.2, # "Explain why X happens"
    "application": 1.5,   # "Write a function that..."
    "analysis": 2.0,      # "What's the difference between..."
    "synthesis": 2.5,     # "Design a solution for..."
}
```

### Mastery Thresholds

```python
MASTERY_THRESHOLDS = {
    "fail": 0.00,         # Below minimum
    "learning": 0.40,     # Still learning, needs more practice
    "pass": 0.70,         # Can proceed to next topic (soft gate)
    "verified": 0.80,     # Earns "verified" badge, XP becomes verified
    "mastery": 0.90,      # Full mastery, rare
}

def get_mastery_status(score: float) -> str:
    """Convert score to human-readable status"""
    if score >= 0.90:
        return "mastery"
    elif score >= 0.80:
        return "verified"
    elif score >= 0.70:
        return "pass"
    elif score >= 0.40:
        return "learning"
    else:
        return "fail"
```

### Mastery Update on Re-test (Spaced Repetition)

```python
def update_mastery_on_retest(
    current_mastery: float,
    retest_score: float,
    days_since_learned: int
) -> tuple[float, int]:
    """
    Update mastery score after a retention re-test.
    
    Args:
        current_mastery: Current mastery score (0.0-1.0)
        retest_score: Score from re-test (0.0-1.0)
        days_since_learned: Days since topic was first learned
    
    Returns:
        (new_mastery, xp_earned)
    
    Logic:
    - If retest_score >= current: weighted average favoring new score
    - If retest_score < current: decay based on time passed
    - XP earned depends on interval (2-week = 30 XP, 6-week = 50 XP)
    """
    if retest_score >= current_mastery:
        # Improvement: weight new score higher (reinforcement)
        new_mastery = (current_mastery * 0.3) + (retest_score * 0.7)
        
        # XP based on interval
        if days_since_learned >= 42:  # 6 weeks
            xp = 50 if retest_score >= 0.70 else 0
        elif days_since_learned >= 14:  # 2 weeks
            xp = 30 if retest_score >= 0.70 else 0
        else:
            xp = 0  # Too soon, no XP
            
        return (new_mastery, xp)
    
    else:
        # Decline: the longer since learning, the more weight to new score
        # After 30 days, new score matters 80%. Before, less impact.
        decay_weight = min(0.8, 0.3 + (days_since_learned / 30) * 0.5)
        new_mastery = (current_mastery * (1 - decay_weight)) + (retest_score * decay_weight)
        
        return (new_mastery, 0)  # No XP for failed recall
```

### Knowledge Decay (Visual Display)

```python
def calculate_decay_level(
    mastery_score: float,
    last_tested_at: datetime,
    now: datetime = None
) -> dict:
    """
    Calculate decay level for visual display.
    
    Returns:
        {
            "level": "strong" | "fading" | "weak" | "critical",
            "percentage": 0-100 (visual bar),
            "days_since_test": int,
            "needs_review": bool
        }
    """
    now = now or datetime.utcnow()
    days = (now - last_tested_at).days
    
    # Base percentage from mastery
    base_pct = int(mastery_score * 100)
    
    # Decay factor: knowledge fades over time if not reinforced
    # After 14 days: start fading
    # After 30 days: significant fade
    # After 60 days: critical
    if days <= 7:
        decay_factor = 0
    elif days <= 14:
        decay_factor = (days - 7) * 2  # Up to 14% decay
    elif days <= 30:
        decay_factor = 14 + (days - 14) * 2  # Up to 46% total
    else:
        decay_factor = min(80, 46 + (days - 30))  # Up to 80% decay
    
    visual_pct = max(0, base_pct - decay_factor)
    
    # Determine level
    if visual_pct >= 70:
        level = "strong"
    elif visual_pct >= 50:
        level = "fading"
    elif visual_pct >= 25:
        level = "weak"
    else:
        level = "critical"
    
    return {
        "level": level,
        "percentage": visual_pct,
        "days_since_test": days,
        "needs_review": days >= 14 and visual_pct < 70
    }
```

### Minimum Questions for Assessment

```python
ASSESSMENT_CONFIG = {
    "min_questions": 5,        # Minimum questions per assessment
    "max_questions": 10,       # Maximum questions per assessment
    "question_distribution": {
        "recall": 2,           # 2 easy questions
        "comprehension": 1,    # 1 understanding question
        "application": 2,      # 2 practice questions
    },
    "mastery_assessment_questions": 8,  # More questions for mastery test
}

def calculate_confidence_interval(
    correct: int,
    total: int
) -> tuple[float, float]:
    """
    Calculate 95% confidence interval for mastery estimate.
    Used to determine if we need more questions.
    
    Returns:
        (lower_bound, upper_bound)
    """
    if total == 0:
        return (0.0, 1.0)
    
    p = correct / total
    z = 1.96  # 95% confidence
    
    # Wilson score interval
    denominator = 1 + z**2 / total
    center = (p + z**2 / (2 * total)) / denominator
    spread = z * ((p * (1 - p) + z**2 / (4 * total)) / total) ** 0.5 / denominator
    
    return (max(0, center - spread), min(1, center + spread))
```

---

## Document History

| Date | Change | Author |
|------|--------|--------|
| 2024-12-10 | Added mastery calculation formulas | Claude + User |
| 2024-12-08 | Initial data model | Claude + User |

---

## Next Steps

- [ ] Review and approve schema
- [ ] Create migration files
- [ ] Set up Supabase project
- [ ] Implement backend models (Pydantic/SQLAlchemy)

