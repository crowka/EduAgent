# EduAgent API Specification

> **Document Status:** Living document  
> **Last Updated:** 2024-12-10 (Rev 2)  
> **Base URL:** `https://api.eduagent.app/v1`

---

## Table of Contents

1. [Overview](#overview)
2. [Authentication](#authentication)
3. [Core Endpoints](#core-endpoints)
4. [Learning Paths](#learning-paths)
5. [Topics & Modules](#topics--modules)
6. [Conversations (AI Learning)](#conversations-ai-learning)
7. [Assessments](#assessments)
8. [Progress & Summaries](#progress--summaries)
9. [Gamification](#gamification)
10. [User Settings](#user-settings)
11. [Subscriptions](#subscriptions)
12. [Error Handling](#error-handling)
13. [Rate Limiting](#rate-limiting)

---

## Overview

### Design Principles

```
┌─────────────────────────────────────────────────────────────────┐
│  API DESIGN PRINCIPLES                                          │
├─────────────────────────────────────────────────────────────────┤
│  • RESTful for CRUD, WebSocket for real-time AI chat            │
│  • JSON request/response bodies                                  │
│  • snake_case for all field names                                │
│  • UUID for all resource IDs                                     │
│  • ISO 8601 for all timestamps                                   │
│  • Pagination via cursor (not offset)                            │
│  • Consistent error format                                       │
└─────────────────────────────────────────────────────────────────┘
```

### API Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                         CLIENT (Expo App)                             │
└──────────────────────────────────┬───────────────────────────────────┘
                                   │
                    ┌──────────────┴──────────────┐
                    │                             │
                    ▼                             ▼
        ┌───────────────────┐         ┌───────────────────┐
        │   REST Endpoints  │         │    WebSocket      │
        │   (FastAPI)       │         │    /ws/learn      │
        │                   │         │                   │
        │ • Auth/Profile    │         │ • Real-time chat  │
        │ • Learning Paths  │         │ • Streaming AI    │
        │ • Progress        │         │ • Live updates    │
        │ • Assessments     │         │                   │
        │ • Gamification    │         │                   │
        └─────────┬─────────┘         └─────────┬─────────┘
                  │                             │
                  └──────────────┬──────────────┘
                                 │
                                 ▼
                    ┌───────────────────────┐
                    │   Supabase Backend    │
                    │   (PostgreSQL + Auth) │
                    └───────────────────────┘
```

---

## Authentication

### Auth Flow (Supabase)

All endpoints (except public) require `Authorization: Bearer <jwt>`.

JWT is obtained from Supabase Auth:
- Email/Password login
- OAuth (Google, Apple)
- Magic link (future)

```
Headers:
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Content-Type: application/json
```

### Token Refresh

Supabase handles token refresh automatically. Client uses Supabase SDK.

---

## Core Endpoints

### Health Check

```http
GET /health
```

**Response:**
```json
{
  "status": "healthy",
  "version": "1.0.0",
  "timestamp": "2024-12-10T12:00:00Z"
}
```

### Current User Profile

```http
GET /me
```

**Response:**
```json
{
  "id": "uuid",
  "email": "user@example.com",
  "display_name": "John Doe",
  "avatar_url": "https://...",
  "language": "en",
  "age_group": "adult",
  "onboarding_completed": true,
  "subscription_tier": "free",
  "streak_days": 7,
  "total_xp": 1250,
  "created_at": "2024-01-01T00:00:00Z"
}
```

### Update Profile

```http
PATCH /me
```

**Request:**
```json
{
  "display_name": "John D.",
  "native_language": "en",
  "learning_language": "fr",
  "notification_preferences": {
    "daily_reminder": true,
    "reminder_time": "09:00"
  }
}
```

---

## Interview & Curriculum (Dynamic Onboarding)

The interview system allows users to learn ANY subject with a personalized, AI-generated curriculum.

### Check Subject Viability (MVP)

Quick assessment before starting interview. Warns user if subject may have curriculum quality issues.

```http
POST /subjects/viability
```

**Request:**
```json
{
  "subject": "quantum field theory"
}
```

**Response:**
```json
{
  "confidence": "MEDIUM",
  "reasoning": "Specialized physics topic with established curricula, but requires strong prerequisites.",
  "concerns": ["Requires calculus, linear algebra, quantum mechanics prerequisites"],
  "recommendation": "warn_user",
  "user_message": "This is a specialized topic. The curriculum may need adjustment based on your background. You can edit it after generation.",
  "proceed": true
}
```

**Confidence Levels:**

| Level | Example Subjects | UX Treatment |
|-------|------------------|--------------|
| **HIGH** | Python, Calculus, Spanish | Proceed silently |
| **MEDIUM** | Quantum Computing, Ancient Greek | Show warning, proceed |
| **LOW** | [Emerging Framework], Fringe Theory | Strong warning, simplified mode |

**Note:** This endpoint is optional. If skipped, interview proceeds without viability check.

---

### Start Interview

Begin the onboarding interview for a new subject.

```http
POST /interview/start
```

**Request:**
```json
{
  "subject": "machine learning",
  "mode": "quick"  // "quick" (default, ~3 min) or "thorough" (~5-7 min)
}
```

**Response:**
```json
{
  "interview_id": "uuid",
  "first_message": "Great choice! Let's personalize your machine learning journey. What's your main goal — is this for a job, a project, or personal interest?",
  "phase": "goal_understanding",
  "estimated_questions_remaining": 4
}
```

---

### Send Interview Response

Continue the interview conversation.

```http
POST /interview/message
```

**Request:**
```json
{
  "interview_id": "uuid",
  "message": "I want to build recommendation systems for my startup"
}
```

**Response:**
```json
{
  "response": "That's a great practical goal! Tell me about your background — what's your experience with math and programming?",
  "phase": "background_probe",
  "estimated_questions_remaining": 3,
  "is_complete": false
}
```

---

### Complete Interview

Finalize interview and trigger curriculum generation.

```http
POST /interview/complete
```

**Request:**
```json
{
  "interview_id": "uuid"
}
```

**Response:**
```json
{
  "learning_path_id": "uuid",
  "interview_summary": {
    "subject_area": "machine learning",
    "goal": {
      "primary": "Build recommendation systems",
      "motivation": "professional"
    },
    "verified_level": {
      "actual_level": "beginner-intermediate",
      "gaps_detected": ["probability", "linear algebra"]
    }
  },
  "curriculum_generating": true,
  "estimated_wait_seconds": 5
}
```

---

### Get Curriculum

Retrieve the generated curriculum for a learning path.

```http
GET /curriculum/{learning_path_id}
```

**Response:**
```json
{
  "learning_path_id": "uuid",
  "subject": "machine learning",
  "goal_alignment": "Building recommendation systems",
  "estimated_total_hours": 40,
  "modules": [
    {
      "id": "uuid",
      "title": "Module 1: Mathematical Foundations",
      "status": "in_progress",
      "topics": [
        {
          "id": "uuid",
          "title": "Probability Fundamentals",
          "estimated_minutes": 45,
          "learning_outcome": "After this, you can calculate conditional probabilities",
          "confidence": "core",
          "status": "not_started",
          "can_skip": true
        }
      ]
    },
    {
      "id": "uuid",
      "title": "Module 2: Supervised Learning",
      "status": "preview",
      "preview_topics": ["Linear Regression", "Logistic Regression", "Model Evaluation"]
    }
  ]
}
```

---

### Explain Curriculum Order

Ask the AI to explain why topics are in a specific order.

```http
POST /curriculum/explain
```

**Request:**
```json
{
  "learning_path_id": "uuid",
  "topic_id": "uuid",
  "question": "Why do I need probability before machine learning?"
}
```

**Response:**
```json
{
  "explanation": "Great question! Machine learning is fundamentally probabilistic...",
  "related_topics": ["uuid1", "uuid2"],
  "can_be_skipped": false,
  "skip_warning": "Skipping this may cause confusion in later topics"
}
```

---

### Challenge Curriculum

User disagrees with curriculum order or content.

```http
POST /curriculum/challenge
```

**Request:**
```json
{
  "learning_path_id": "uuid",
  "challenge_type": "reorder",  // "reorder", "skip", "remove", "add"
  "user_reason": "I already know Python well, I'd rather jump to ML concepts",
  "affected_topics": ["uuid1", "uuid2"]
}
```

**Response:**
```json
{
  "acknowledged": true,
  "llm_response": "I understand — you want to get to the interesting stuff! I can adjust, but...",
  "proposed_changes": [
    {
      "type": "reorder",
      "original_position": 4,
      "new_position": 1,
      "topic_id": "uuid"
    }
  ],
  "requires_confirmation": true,
  "change_id": "uuid"
}
```

---

### Confirm Curriculum Change

Accept the proposed curriculum changes.

```http
POST /curriculum/changes/{change_id}/confirm
```

**Response:**
```json
{
  "success": true,
  "curriculum_updated": true,
  "changes_logged": true  // Logged to curriculum_changes table for admin review
}
```

---

### Skip Topic

Mark a topic as skipped (user claims to know it).

```http
POST /curriculum/skip
```

**Request:**
```json
{
  "learning_path_id": "uuid",
  "topic_id": "uuid",
  "reason": "I use this daily at work"  // Optional
}
```

**Response:**
```json
{
  "success": true,
  "topic_status": "skipped",
  "signal_logged": true,  // Logged to curriculum_feedback
  "warning": null  // or "This is a prerequisite for 3 later topics"
}
```

---

### Regenerate Curriculum

Request full curriculum regeneration (triggers correction interview).

```http
POST /curriculum/regenerate
```

**Request:**
```json
{
  "learning_path_id": "uuid",
  "reason": "The entire approach seems too theoretical for my practical goals"
}
```

**Response:**
```json
{
  "regeneration_started": true,
  "interview_id": "uuid",  // Starts a "correction" mode interview
  "first_message": "I hear you — let's adjust. What specifically feels too theoretical?"
}
```

---

### Admin: Curriculum Changes Report

Get all curriculum changes for admin review.

```http
GET /admin/curriculum-changes?reviewed=false&limit=50
```

**Response:**
```json
{
  "changes": [
    {
      "id": "uuid",
      "user_id": "uuid",
      "subject_area": "machine learning",
      "change_type": "challenge",
      "user_reason": "I already know Python",
      "llm_explanation": "Acknowledged and reordered",
      "llm_agreed": true,
      "original_curriculum": {...},
      "new_curriculum": {...},
      "created_at": "2024-12-10T..."
    }
  ],
  "total_unreviewed": 15
}
```

---

## Learning Paths

### List Available Subjects

```http
GET /subjects
```

**Response:**
```json
{
  "subjects": [
    {
      "id": "uuid",
      "name": "Mathematics",
      "description": "From basics to advanced calculus",
      "icon": "calculator",
      "category": "stem",
      "path_count": 12
    }
  ]
}
```

### Discover Learning Paths

```http
GET /learning-paths?subject_id={uuid}&difficulty={beginner|intermediate|advanced}
```

**Response:**
```json
{
  "paths": [
    {
      "id": "uuid",
      "title": "Introduction to Machine Learning",
      "description": "Learn ML fundamentals from scratch",
      "subject": "Computer Science",
      "difficulty_level": "intermediate",
      "estimated_hours": 20,
      "module_count": 5,
      "topic_count": 24,
      "enrolled_count": 1500,
      "preview_topics": ["Neural Networks", "Backpropagation", "CNNs"]
    }
  ],
  "cursor": "abc123"
}
```

### Get Learning Path Details

```http
GET /learning-paths/{path_id}
```

**Response:**
```json
{
  "id": "uuid",
  "title": "Introduction to Machine Learning",
  "description": "...",
  "modules": [
    {
      "id": "uuid",
      "title": "Foundations",
      "order": 1,
      "topics": [
        {
          "id": "uuid",
          "title": "What is Machine Learning?",
          "order": 1,
          "estimated_minutes": 15,
          "prerequisites": []
        },
        {
          "id": "uuid",
          "title": "Types of ML",
          "order": 2,
          "estimated_minutes": 20,
          "prerequisites": ["uuid-of-previous-topic"]
        }
      ]
    }
  ],
  "user_progress": {
    "enrolled": true,
    "completed_topics": 5,
    "total_topics": 24,
    "current_module_id": "uuid",
    "current_topic_id": "uuid",
    "percent_complete": 21
  }
}
```

### Enroll in Learning Path

```http
POST /learning-paths/{path_id}/enroll
```

**Response:**
```json
{
  "enrolled": true,
  "learning_path_id": "uuid",
  "started_at": "2024-12-10T12:00:00Z",
  "first_topic": {
    "id": "uuid",
    "title": "What is Machine Learning?"
  }
}
```

### Get My Learning Paths

```http
GET /me/learning-paths?status={active|completed|all}
```

**Response:**
```json
{
  "paths": [
    {
      "id": "uuid",
      "title": "Introduction to Machine Learning",
      "percent_complete": 45,
      "last_activity": "2024-12-09T18:30:00Z",
      "current_topic": {
        "id": "uuid",
        "title": "Backpropagation"
      }
    }
  ]
}
```

### AI-Generated Path (On Demand)

```http
POST /learning-paths/generate
```

**Request:**
```json
{
  "goal": "I want to learn Python for data science",
  "current_knowledge": "I know basic programming concepts",
  "time_commitment_hours_per_week": 5,
  "target_duration_weeks": 8
}
```

**Response:**
```json
{
  "path_id": "uuid",
  "title": "Python for Data Science",
  "generated": true,
  "modules": [...],
  "estimated_hours": 40,
  "message": "I've created a personalized learning path for you!"
}
```

---

## Topics & Modules

### Get Topic Details

```http
GET /topics/{topic_id}
```

**Response:**
```json
{
  "id": "uuid",
  "title": "Backpropagation",
  "module_id": "uuid",
  "learning_path_id": "uuid",
  "estimated_minutes": 25,
  "difficulty": "intermediate",
  "prerequisites": [
    {
      "id": "uuid",
      "title": "Neural Networks",
      "completed": true
    }
  ],
  "user_progress": {
    "status": "in_progress",
    "mastery_level": 0.4,
    "time_spent_minutes": 12,
    "last_learned_at": "2024-12-09T15:00:00Z"
  },
  "community_stats": {
    "struggle_percentage": 43,
    "avg_attempts": 1.7,
    "total_learners": 156
  }
}
```

**Note:** `struggle_percentage` is shown to users as "43% of learners needed extra practice here" — reduces shame, increases persistence.

---

### Set Learning Mode

Update user's preferred learning mode for the current session.

```http
POST /me/learning-mode
```

**Request:**
```json
{
  "mode": "quiz"  // "teach" | "quiz" | "quick"
}
```

**Response:**
```json
{
  "mode": "quiz",
  "description": "Jump straight to testing. Explanations only when you get it wrong."
}
```

**Modes:**
| Mode | Behavior |
|------|----------|
| `teach` | Full explanations + worked examples (default for new topics) |
| `quiz` | Skip explanations, go straight to questions |
| `quick` | Just answer questions, no pedagogy (like ChatGPT) |

---

### Start Topic Learning

```http
POST /topics/{topic_id}/start
```

**Response:**
```json
{
  "session_id": "uuid",
  "topic": {...},
  "prior_knowledge": {
    "completed_topics": ["Neural Networks", "Activation Functions"],
    "key_concepts": ["neurons", "weights", "bias", "forward propagation"]
  },
  "conversation_id": "uuid",
  "websocket_url": "wss://api.eduagent.app/v1/ws/learn?session={session_id}"
}
```

### Mark Topic Complete

```http
POST /topics/{topic_id}/complete
```

**Request:**
```json
{
  "mastery_level": 0.8,
  "notes": "Optional user notes"
}
```

**Response:**
```json
{
  "completed": true,
  "xp_earned": 50,
  "achievements_unlocked": [
    {
      "id": "uuid",
      "name": "First Step",
      "description": "Complete your first topic"
    }
  ],
  "next_topic": {
    "id": "uuid",
    "title": "Training Neural Networks"
  },
  "summary_generated": true
}
```

---

## Conversations (AI Learning)

### WebSocket Connection (Primary)

```
wss://api.eduagent.app/v1/ws/learn?session={session_id}
```

#### Connection Lifecycle

```
┌─────────────────────────────────────────────────────────────────┐
│  WEBSOCKET LIFECYCLE                                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. CLIENT CONNECTS                                              │
│     GET /topics/{id}/start → returns session_id                  │
│     CONNECT wss://api.../ws/learn?session={session_id}           │
│                                                                  │
│  2. SERVER ACKNOWLEDGES                                          │
│     Server sends: { type: "connected", session_id, topic }       │
│                                                                  │
│  3. HEARTBEAT (every 30s)                                        │
│     Client sends: { type: "ping" }                               │
│     Server sends: { type: "pong" }                               │
│     If no pong in 10s → client reconnects                        │
│                                                                  │
│  4. RECONNECTION                                                 │
│     On disconnect, client attempts reconnect with same session   │
│     Server resumes from last message_id                          │
│     Max 3 reconnect attempts, then fall back to REST             │
│                                                                  │
│  5. GRACEFUL CLOSE                                               │
│     Client sends: { type: "close" }                              │
│     Server persists conversation, closes connection              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

#### Message Protocol

All messages are JSON with required `type` field:

```typescript
interface WebSocketMessage {
  type: string;
  message_id?: string;  // For tracking/deduplication
  timestamp?: string;   // ISO 8601
  [key: string]: any;
}
```

#### Client → Server Messages

| Type | Purpose | Fields |
|------|---------|--------|
| `ping` | Heartbeat | - |
| `message` | User chat message | `content`, `context?` |
| `request_assessment` | Start quiz | `question_count?` |
| `submit_answer` | Answer question | `question_id`, `answer` |
| `flag_content` | Report issue | `message_id`, `reason` |
| `close` | End session | - |

```json
// User message
{
  "type": "message",
  "message_id": "client-uuid-1",
  "content": "Can you explain gradients?",
  "context": "teaching"
}

// Heartbeat
{
  "type": "ping"
}

// Request assessment
{
  "type": "request_assessment",
  "question_count": 3
}

// Submit answer (during assessment)
{
  "type": "submit_answer",
  "question_id": "uuid",
  "answer": "b"
}

// Flag content
{
  "type": "flag_content",
  "message_id": "uuid",
  "reason": "This doesn't seem correct"
}

// Close session
{
  "type": "close"
}
```

#### Server → Client Messages

| Type | Purpose | Fields |
|------|---------|--------|
| `connected` | Connection ack | `session_id`, `topic` |
| `pong` | Heartbeat response | - |
| `stream_start` | AI response starting | `message_id` |
| `stream_chunk` | AI response chunk | `message_id`, `content` |
| `stream_end` | AI response complete | `message_id`, `agent` |
| `assessment` | Quiz question | `question`, `reasoning_prompt` |
| `answer_result` | Step-level feedback | `steps`, `first_error_at`, `feedback`, `follow_up` |
| `progress_update` | Mastery/XP update | `mastery_level`, `xp_earned` |
| `summary_ready` | Topic summary done | `topic_summary_id` |
| `error` | Error occurred | `code`, `message`, `retry?` |

```json
// Connection acknowledged
{
  "type": "connected",
  "session_id": "uuid",
  "topic": {
    "id": "uuid",
    "title": "Backpropagation"
  },
  "prior_knowledge_loaded": true
}

// Heartbeat response
{
  "type": "pong"
}

// Streaming response start
{
  "type": "stream_start",
  "message_id": "uuid"
}

// Streaming chunk
{
  "type": "stream_chunk",
  "message_id": "uuid",
  "content": "Gradients represent..."
}

// Streaming complete
{
  "type": "stream_end",
  "message_id": "uuid",
  "agent": "teaching",
  "tokens_used": 450
}

// Assessment question (always asks for reasoning)
{
  "type": "assessment",
  "question": {
    "id": "uuid",
    "text": "What is the purpose of backpropagation?",
    "reasoning_prompt": "Walk me through why you chose that answer.",
    "type": "multiple_choice",
    "options": [
      {"id": "a", "text": "Forward pass"},
      {"id": "b", "text": "Calculate gradients"},
      {"id": "c", "text": "Initialize weights"},
      {"id": "d", "text": "Normalize data"}
    ],
    "expected_steps": ["Identify what backprop does", "Connect to gradient calculation"]
  }
}

// Answer result (step-level feedback)
{
  "type": "answer_result",
  "question_id": "uuid",
  "reasoning_provided": true,
  "steps_evaluated": [
    {"step": "Identified backprop role in training", "correct": true},
    {"step": "Connected to gradient calculation", "correct": true}
  ],
  "first_error_at": null,
  "overall_correct": true,
  "score": 1.0,
  "feedback": "Exactly right! You correctly identified that backpropagation calculates gradients.",
  "follow_up_question": null,
  "xp_earned": 10
}

// Answer result (with error - step-level)
{
  "type": "answer_result",
  "question_id": "uuid",
  "reasoning_provided": true,
  "steps_evaluated": [
    {"step": "Identified it's part of training", "correct": true},
    {"step": "Thought it initializes weights", "correct": false, "issue": "Confused with initialization phase"}
  ],
  "first_error_at": 1,
  "overall_correct": false,
  "score": 0.5,
  "feedback": "You're right that it's part of training. But think about WHEN backprop runs - is it at the start, or after we see how wrong we were?",
  "follow_up_question": "What information do we need before we can adjust weights?",
  "misconception_identified": "Confuses backprop with weight initialization",
  "xp_earned": 0
}

// Progress update
{
  "type": "progress_update",
  "mastery_level": 0.6,
  "xp_earned": 10
}

// Topic summary generated
{
  "type": "summary_ready",
  "topic_summary_id": "uuid"
}

// Error
{
  "type": "error",
  "code": "LLM_TIMEOUT",
  "message": "AI response timed out",
  "retry": true
}
```

#### Error Handling

| Code | Meaning | Client Action |
|------|---------|---------------|
| `AUTH_EXPIRED` | JWT expired | Refresh token, reconnect |
| `SESSION_INVALID` | Session not found | Start new session |
| `RATE_LIMITED` | Too many messages | Wait `retry_after_seconds` |
| `LLM_TIMEOUT` | AI slow | Retry or wait |
| `LLM_ERROR` | AI failed | Show cached/fallback |
| `INTERNAL_ERROR` | Server error | Fall back to REST |

#### Reconnection Strategy

```javascript
// Client-side reconnection (pseudocode)
const MAX_RECONNECT_ATTEMPTS = 3;
const RECONNECT_DELAYS = [1000, 3000, 5000]; // Exponential backoff

async function reconnect(sessionId, attempt = 0) {
  if (attempt >= MAX_RECONNECT_ATTEMPTS) {
    return fallbackToREST(sessionId);
  }
  
  await delay(RECONNECT_DELAYS[attempt]);
  
  try {
    const ws = new WebSocket(`wss://api.../ws/learn?session=${sessionId}`);
    ws.onopen = () => {
      // Send last received message_id for resume
      ws.send(JSON.stringify({ 
        type: "resume", 
        last_message_id: lastReceivedMessageId 
      }));
    };
  } catch (e) {
    reconnect(sessionId, attempt + 1);
  }
}
```

### REST Fallback (If WebSocket unavailable)

```http
POST /conversations/{conversation_id}/messages
```

**Request:**
```json
{
  "content": "Explain backpropagation",
  "stream": false
}
```

**Response:**
```json
{
  "id": "uuid",
  "role": "assistant",
  "content": "Backpropagation is...",
  "agent": "teaching",
  "created_at": "2024-12-10T12:00:00Z"
}
```

### Get Conversation History

```http
GET /conversations/{conversation_id}/messages?limit=50&cursor={cursor}
```

**Response:**
```json
{
  "messages": [
    {
      "id": "uuid",
      "role": "user",
      "content": "What is a gradient?",
      "created_at": "2024-12-10T11:55:00Z"
    },
    {
      "id": "uuid",
      "role": "assistant",
      "content": "A gradient is...",
      "agent": "teaching",
      "created_at": "2024-12-10T11:55:30Z"
    }
  ],
  "cursor": "abc123"
}
```

---

## Code Execution (Programming Subjects)

Pyodide runs Python in the browser (client-side). The backend handles code validation and exercise generation.

### Validate Code Output

Validates user's code output against expected results. Called after Pyodide executes code client-side.

```http
POST /code/validate
```

**Request:**
```json
{
  "conversation_id": "uuid",
  "question_id": "uuid",
  "code": "def add(a, b):\n    return a + b\n\nprint(add(2, 3))",
  "output": "5",
  "execution_time_ms": 45,
  "error": null
}
```

**Response:**
```json
{
  "correct": true,
  "score": 1.0,
  "feedback": "Perfect! Your function correctly adds the two numbers.",
  "steps_evaluated": [
    {"step": "Function definition", "correct": true},
    {"step": "Return statement", "correct": true},
    {"step": "Function call", "correct": true},
    {"step": "Output matches expected", "correct": true}
  ],
  "test_cases_passed": 3,
  "test_cases_total": 3,
  "follow_up": null
}
```

**Error Response (code has issues):**
```json
{
  "correct": false,
  "score": 0.5,
  "feedback": "Your function works for simple cases, but there's an edge case issue.",
  "steps_evaluated": [
    {"step": "Function definition", "correct": true},
    {"step": "Return statement", "correct": true},
    {"step": "Edge case: negative numbers", "correct": false, "issue": "Returns incorrect result for negative inputs"}
  ],
  "test_cases_passed": 2,
  "test_cases_total": 3,
  "follow_up": "What happens if both a and b are negative? Try add(-2, -3).",
  "hint": "Check your logic for handling negative numbers."
}
```

---

### Generate Code Exercise

Generate a coding exercise for the current topic.

```http
POST /code/exercise
```

**Request:**
```json
{
  "topic_id": "uuid",
  "difficulty": "medium",
  "exercise_type": "write_function"  // "write_function", "fix_bug", "predict_output", "fill_blank"
}
```

**Response:**
```json
{
  "exercise_id": "uuid",
  "type": "write_function",
  "prompt": "Write a function called `is_palindrome` that takes a string and returns True if it's a palindrome, False otherwise.",
  "starter_code": "def is_palindrome(s):\n    # Your code here\n    pass",
  "test_cases": [
    {"input": "\"racecar\"", "expected_output": "True", "visible": true},
    {"input": "\"hello\"", "expected_output": "False", "visible": true},
    {"input": "\"A man a plan a canal Panama\"", "expected_output": "True", "visible": false}
  ],
  "hints": [
    "Think about comparing characters from both ends",
    "You might want to clean the string first (lowercase, remove spaces)"
  ],
  "concepts_tested": ["string manipulation", "boolean logic", "functions"]
}
```

---

### Run Test Cases (Batch Validation)

Run multiple test cases against user's code. Uses Pyodide client-side, then validates results.

```http
POST /code/test
```

**Request:**
```json
{
  "exercise_id": "uuid",
  "code": "def is_palindrome(s):\n    s = s.lower().replace(' ', '')\n    return s == s[::-1]",
  "results": [
    {"test_case_id": 1, "output": "True", "error": null},
    {"test_case_id": 2, "output": "False", "error": null},
    {"test_case_id": 3, "output": "True", "error": null}
  ]
}
```

**Response:**
```json
{
  "all_passed": true,
  "score": 1.0,
  "results": [
    {"test_case_id": 1, "passed": true, "expected": "True", "actual": "True"},
    {"test_case_id": 2, "passed": true, "expected": "False", "actual": "False"},
    {"test_case_id": 3, "passed": true, "expected": "True", "actual": "True"}
  ],
  "feedback": "Excellent! Your solution handles all cases correctly, including the tricky one with spaces and mixed case.",
  "code_quality": {
    "readability": "good",
    "efficiency": "good",
    "suggestions": []
  },
  "xp_earned": 15
}
```

---

## Assessments

### Get Topic Assessment

```http
POST /topics/{topic_id}/assess
```

**Request:**
```json
{
  "question_count": 5,
  "difficulty": "adaptive"
}
```

**Response:**
```json
{
  "assessment_id": "uuid",
  "questions": [
    {
      "id": "uuid",
      "order": 1,
      "type": "multiple_choice",
      "text": "What is the main purpose of backpropagation?",
      "options": [
        {"id": "a", "text": "Forward pass"},
        {"id": "b", "text": "Calculate gradients"},
        {"id": "c", "text": "Initialize weights"},
        {"id": "d", "text": "Normalize data"}
      ]
    },
    {
      "id": "uuid",
      "order": 2,
      "type": "free_response",
      "text": "Explain the chain rule in your own words."
    }
  ]
}
```

### Submit Assessment Answer

```http
POST /assessments/{assessment_id}/answers
```

**Request:**
```json
{
  "question_id": "uuid",
  "answer": "b"
}
```

**Response:**
```json
{
  "correct": true,
  "explanation": "Correct! Backpropagation calculates gradients...",
  "xp_earned": 10,
  "next_question_id": "uuid"
}
```

### Complete Assessment

```http
POST /assessments/{assessment_id}/complete
```

**Response:**
```json
{
  "score": 0.8,
  "correct_count": 4,
  "total_count": 5,
  "xp_earned": 50,
  "mastery_updated": true,
  "new_mastery_level": 0.75,
  "feedback": "Great job! You've demonstrated solid understanding...",
  "weak_areas": ["chain rule application"],
  "recommendations": [
    {
      "type": "review",
      "topic_id": "uuid",
      "reason": "Strengthen chain rule understanding"
    }
  ]
}
```

### Request Re-Test (From Summary)

```http
POST /topic-summaries/{summary_id}/retest
```

**Request:**
```json
{
  "question_count": 4
}
```

**Response:**
```json
{
  "assessment_id": "uuid",
  "based_on_summary": true,
  "days_since_learned": 14,
  "questions": [...]
}
```

---

## Retrieval Sessions (Spaced + Interleaved)

**Evidence-based learning:** Combines spaced repetition (d=0.79) with interleaving (d=1.21).

### Get Session Plan

Returns topics due for retrieval practice today, with interleaved questions.

```http
GET /session/plan
```

**Response:**
```json
{
  "due_topics": [
    {
      "topic_id": "uuid",
      "title": "Functions",
      "retrieval_count": 3,
      "interval_days": 7,
      "days_overdue": 1
    },
    {
      "topic_id": "uuid",
      "title": "Loops",
      "retrieval_count": 2,
      "interval_days": 3,
      "days_overdue": 0
    }
  ],
  "questions": [
    {
      "id": "uuid",
      "order": 1,
      "topic_hidden": true,
      "text": "What does this code print?...",
      "reasoning_prompt": "Walk me through your thinking.",
      "type": "free_response"
    },
    {
      "id": "uuid",
      "order": 2,
      "topic_hidden": true,
      "text": "Write a function that...",
      "reasoning_prompt": "Explain your approach.",
      "type": "code"
    }
  ],
  "interleaved": true,
  "total_questions": 4,
  "estimated_minutes": 8
}
```

**Note:** `topic_hidden: true` means the topic is NOT revealed until after the user answers.

---

### Submit Retrieval Answer

Submit answer for a retrieval question. Topic revealed after submission.

```http
POST /session/answer
```

**Request:**
```json
{
  "question_id": "uuid",
  "answer": "The function returns the filtered list...",
  "reasoning": "I used a list comprehension because..."
}
```

**Response:**
```json
{
  "topic_revealed": {
    "id": "uuid",
    "title": "Functions",
    "concepts_tested": ["list comprehension", "filtering"]
  },
  "evaluation": {
    "steps_evaluated": [
      {"step": "Identified filtering pattern", "correct": true},
      {"step": "Used list comprehension", "correct": true},
      {"step": "Return statement correct", "correct": true}
    ],
    "first_error_at": null,
    "overall_correct": true,
    "score": 1.0,
    "feedback": "Perfect! You correctly identified this tests list filtering with comprehensions."
  },
  "progress": {
    "questions_remaining": 3,
    "current_score": "1/1"
  }
}
```

---

### Complete Retrieval Session

Finalize session and update all retrieval schedules.

```http
POST /session/complete
```

**Response:**
```json
{
  "session_summary": {
    "total_questions": 4,
    "correct": 3,
    "score": 0.75
  },
  "topic_updates": [
    {
      "topic_id": "uuid",
      "title": "Functions",
      "quality": 4,
      "previous_interval": 7,
      "new_interval": 14,
      "next_due": "2024-12-24",
      "consecutive_successes": 4,
      "is_stable": false
    },
    {
      "topic_id": "uuid",
      "title": "Loops",
      "quality": 2,
      "previous_interval": 3,
      "new_interval": 1,
      "next_due": "2024-12-11",
      "consecutive_successes": 0,
      "is_stable": false,
      "note": "Reset due to failed retrieval"
    }
  ],
  "xp_earned": {
    "verified": 30,
    "pending_converted": 0
  },
  "honest_streak": {
    "maintained": true,
    "current": 6
  },
  "recommendation": "Good session! Loops needs more practice - we'll review it tomorrow."
}
```

---

### Get Retrieval History

View retrieval schedule and history for all topics.

```http
GET /me/retrievals?status=due|upcoming|stable
```

**Response:**
```json
{
  "retrievals": [
    {
      "topic_id": "uuid",
      "title": "Functions",
      "status": "due",
      "retrieval_count": 3,
      "consecutive_successes": 3,
      "interval_days": 7,
      "next_due": "2024-12-10",
      "is_stable": false,
      "last_score": 0.85
    },
    {
      "topic_id": "uuid",
      "title": "Variables",
      "status": "stable",
      "retrieval_count": 6,
      "consecutive_successes": 6,
      "interval_days": 60,
      "next_due": "2025-02-01",
      "is_stable": true,
      "last_score": 1.0
    }
  ],
  "summary": {
    "total_topics": 8,
    "due_today": 2,
    "stable": 3,
    "learning": 5
  }
}
```

---

## Progress & Summaries

### Get Topic Summary

```http
GET /topic-summaries/{summary_id}
```

**Response:**
```json
{
  "id": "uuid",
  "topic_id": "uuid",
  "topic_title": "Backpropagation",
  "key_concepts": [
    "Gradient calculation",
    "Chain rule",
    "Weight updates",
    "Learning rate"
  ],
  "summary_text": "You learned how backpropagation works by calculating gradients...",
  "examples_used": [
    "Simple 2-layer network example",
    "Gradient flow visualization"
  ],
  "depth_level": "foundational",
  "retention_score": 0.85,
  "learned_at": "2024-12-05T14:00:00Z",
  "next_review_suggested": "2024-12-12T00:00:00Z"
}
```

### List My Topic Summaries

```http
GET /me/topic-summaries?learning_path_id={uuid}&needs_review={true|false}
```

**Response:**
```json
{
  "summaries": [
    {
      "id": "uuid",
      "topic_title": "Backpropagation",
      "retention_score": 0.6,
      "learned_at": "2024-11-20T00:00:00Z",
      "needs_review": true,
      "days_since_learned": 20
    }
  ]
}
```

### Re-learn Topic

Start a new learning session for a previously completed topic. The AI will remember what you learned before and build upon it.

```http
POST /topics/{topic_id}/relearn
```

**Request Body:**
```json
{
  "reason": "forgot|deeper|different_approach",
  "specific_focus": "optional: what aspect to focus on"
}
```

**Response:**
```json
{
  "conversation_id": "uuid",
  "mode": "relearn",
  "context": {
    "original_learned_at": "2024-11-20T00:00:00Z",
    "previous_mastery": 0.75,
    "current_retention": 0.4,
    "key_concepts_covered": ["concept1", "concept2"],
    "previous_struggles": ["return vs print confusion"]
  },
  "opening_message": "Welcome back to Functions! Last time we covered how to define functions and use parameters. I see you found the difference between return and print a bit tricky. Let's start there..."
}
```

**What happens:**
1. System fetches `topic_summaries` for this topic
2. System fetches `subject_learner_models` for misconceptions
3. Teaching Agent gets full context of prior learning
4. Agent opens with personalized re-introduction
5. Session is marked as `type: relearn` for analytics

### Quiz on Topic

Generate a retention quiz for a previously learned topic.

```http
POST /topics/{topic_id}/quiz
```

**Request Body:**
```json
{
  "question_count": 3,
  "difficulty": "match_mastery"
}
```

**Response:**
```json
{
  "quiz_id": "uuid",
  "topic_title": "Functions",
  "days_since_learned": 15,
  "questions": [
    {
      "id": "uuid",
      "type": "free_recall",
      "question": "Write a function that takes two numbers and returns their average.",
      "concepts_tested": ["function definition", "parameters", "return"]
    }
  ]
}
```

**Notes:**
- Questions are generated by AI based on `topic_summaries.key_concepts`
- Different examples than original lesson (tests transfer, not memorization)
- Passing quiz updates `verified_xp` and `honest_streak`

### Get Overall Progress

```http
GET /me/progress
```

**Response:**
```json
{
  "total_topics_completed": 45,
  "total_time_spent_hours": 32,
  "current_streak_days": 7,
  "longest_streak_days": 14,
  "total_xp": 2500,
  "level": 5,
  "xp_to_next_level": 200,
  "paths_completed": 2,
  "paths_in_progress": 1,
  "weekly_goal_progress": {
    "target_minutes": 120,
    "completed_minutes": 85,
    "percent": 71
  }
}
```

### Get Learning Path Progress

```http
GET /me/learning-paths/{path_id}/progress
```

**Response:**
```json
{
  "learning_path_id": "uuid",
  "title": "Introduction to Machine Learning",
  "modules": [
    {
      "id": "uuid",
      "title": "Foundations",
      "status": "completed",
      "topics": [
        {
          "id": "uuid",
          "title": "What is ML?",
          "status": "completed",
          "mastery_level": 0.9
        }
      ]
    },
    {
      "id": "uuid",
      "title": "Neural Networks",
      "status": "in_progress",
      "topics": [
        {
          "id": "uuid",
          "title": "Backpropagation",
          "status": "in_progress",
          "mastery_level": 0.4
        }
      ]
    }
  ],
  "overall_mastery": 0.65,
  "estimated_completion": "2024-12-25"
}
```

---

## Gamification

> **ADR-016: Outcome-Based Gamification**
> 
> We reward knowledge retention, not app activity. XP is earned by passing 
> delayed recall tests, not by completing topics. Streaks count days where 
> recall was passed, not days the app was opened.

### Get Knowledge Dashboard

Returns user's knowledge decay status, XP, and honest streak.

```http
GET /me/knowledge
```

**Response:**
```json
{
  "summary": {
    "topics_completed": 20,
    "topics_verified": 12,
    "verified_xp": 450,
    "pending_xp": 320,
    "honest_streak": 7,
    "longest_honest_streak": 14
  },
  "topics": [
    {
      "id": "uuid",
      "title": "Functions",
      "completed_at": "2024-11-25T10:00:00Z",
      "last_recall_at": "2024-12-07T14:00:00Z",
      "decay_level": 85,
      "strength": "strong",
      "pending_xp": 0,
      "verified_xp": 80,
      "recall_2wk_score": 0.85,
      "recall_6wk_score": null
    },
    {
      "id": "uuid",
      "title": "Loops",
      "completed_at": "2024-11-15T10:00:00Z",
      "last_recall_at": "2024-11-29T14:00:00Z",
      "decay_level": 45,
      "strength": "fading",
      "pending_xp": 50,
      "verified_xp": 30,
      "recall_2wk_score": 0.78,
      "recall_6wk_score": null
    }
  ]
}
```

**Decay Level Calculation:**
```
decay_level = 100 * exp(-days_since_last_recall / 20)
```

**Strength Labels:**
- `strong`: decay_level > 80
- `fading`: decay_level 50-80
- `weak`: decay_level 25-50
- `almost_gone`: decay_level < 25

### Get Topics Needing Review

Returns topics sorted by urgency (lowest decay first).

```http
GET /me/knowledge/review
```

**Query Parameters:**
| Param | Type | Description |
|-------|------|-------------|
| `limit` | int | Max topics to return (default: 5) |

**Response:**
```json
{
  "topics": [
    {
      "id": "uuid",
      "title": "Variables",
      "decay_level": 15,
      "strength": "almost_gone",
      "days_since_recall": 42,
      "potential_xp_loss": 50
    }
  ],
  "total_at_risk": 3
}
```

### Submit Recall Test

Submit answers for a delayed recall assessment.

```http
POST /me/knowledge/recall
```

**Request Body:**
```json
{
  "topic_id": "uuid",
  "recall_type": "2wk",
  "answers": [
    {
      "question_id": "uuid",
      "answer": "User's free-text answer"
    }
  ]
}
```

**Response:**
```json
{
  "score": 0.85,
  "passed": true,
  "xp_earned": 30,
  "new_verified_xp": 480,
  "streak_updated": true,
  "honest_streak": 8,
  "feedback": [
    {
      "question_id": "uuid",
      "correct": true,
      "explanation": null
    },
    {
      "question_id": "uuid",
      "correct": false,
      "explanation": "The return statement was missing..."
    }
  ]
}
```

### Get Honest Streak Info

```http
GET /me/streak
```

**Response:**
```json
{
  "honest_streak": 7,
  "longest_honest_streak": 14,
  "last_recall_pass_date": "2024-12-10",
  "streak_start_date": "2024-12-04",
  "activity_streak": 45,
  "note": "Honest streak counts days where you passed a recall challenge"
}
```

### Get XP History

```http
GET /me/xp/history
```

**Query Parameters:**
| Param | Type | Description |
|-------|------|-------------|
| `days` | int | Days of history (default: 30) |

**Response:**
```json
{
  "current_verified_xp": 450,
  "current_pending_xp": 320,
  "history": [
    {
      "date": "2024-12-10",
      "event": "recall_passed",
      "topic_title": "Functions",
      "xp_change": 30,
      "type": "earned"
    },
    {
      "date": "2024-12-05",
      "event": "topic_completed",
      "topic_title": "Classes",
      "xp_change": 80,
      "type": "pending"
    }
  ]
}
```

---

## User Settings

### Get Settings

```http
GET /me/settings
```

**Response:**
```json
{
  "notifications": {
    "daily_reminder": true,
    "reminder_time": "09:00",
    "achievement_alerts": true,
    "streak_warnings": true
  },
  "learning": {
    "session_duration_minutes": 15,
    "difficulty_preference": "adaptive",
    "voice_enabled": false
  },
  "privacy": {
    "profile_visible": false,
    "activity_visible": false
  },
  "accessibility": {
    "font_size": "medium",
    "high_contrast": false,
    "reduce_motion": false
  }
}
```

### Update Settings

```http
PATCH /me/settings
```

**Request:**
```json
{
  "notifications": {
    "daily_reminder": false
  }
}
```

### Delete Account (GDPR)

```http
DELETE /me
```

**Request:**
```json
{
  "confirmation": "DELETE_MY_ACCOUNT",
  "reason": "optional feedback"
}
```

**Response:**
```json
{
  "scheduled_deletion": "2024-12-17T00:00:00Z",
  "message": "Your account will be deleted in 7 days. You can cancel this by logging in."
}
```

### Export User Data (GDPR Article 20)

```http
GET /me/data-export?format={json|csv}
```

**Response:**
```json
{
  "profile": {...},
  "learning_paths": [...],
  "progress": [...],
  "topic_summaries": [...],
  "conversations": [...],
  "exported_at": "2024-12-10T12:00:00Z"
}
```

---

## Subscriptions

### Get Subscription Status

```http
GET /me/subscription
```

**Response:**
```json
{
  "tier": "free",
  "features": {
    "daily_sessions": 3,
    "streak_freeze": false,
    "advanced_analytics": false,
    "offline_mode": false
  },
  "usage_today": {
    "sessions_used": 2,
    "sessions_remaining": 1
  }
}
```

### Premium Response

```json
{
  "tier": "premium",
  "billing_period": "monthly",
  "current_period_end": "2025-01-10T00:00:00Z",
  "cancel_at_period_end": false,
  "features": {
    "daily_sessions": "unlimited",
    "streak_freeze": true,
    "advanced_analytics": true,
    "offline_mode": true,
    "priority_support": true
  }
}
```

### Create Checkout Session

```http
POST /subscriptions/checkout
```

**Request:**
```json
{
  "plan": "premium_monthly",
  "success_url": "eduagent://subscription/success",
  "cancel_url": "eduagent://subscription/cancel"
}
```

**Response:**
```json
{
  "checkout_url": "https://checkout.stripe.com/...",
  "session_id": "cs_..."
}
```

### Cancel Subscription

```http
POST /subscriptions/cancel
```

**Response:**
```json
{
  "cancelled": true,
  "active_until": "2025-01-10T00:00:00Z",
  "message": "Your subscription will remain active until the end of the billing period."
}
```

### Stripe Webhooks

```http
POST /webhooks/stripe
```

Handles Stripe events. Webhook secret configured in environment.

**Handled Events:**

| Event | Action |
|-------|--------|
| `checkout.session.completed` | Activate subscription |
| `customer.subscription.updated` | Sync status changes |
| `customer.subscription.deleted` | Mark subscription cancelled |
| `invoice.payment_failed` | Mark as past_due, notify user |
| `invoice.paid` | Reset daily limits, extend period |

**Response:**
```json
{
  "received": true
}
```

---

## Account Management

### Cancel Account Deletion

Users can cancel a pending account deletion within 7 days.

```http
POST /me/cancel-deletion
```

**Response:**
```json
{
  "deletion_cancelled": true,
  "message": "Your account deletion has been cancelled."
}
```

**Error (if not pending deletion):**
```json
{
  "error": {
    "code": "NO_PENDING_DELETION",
    "message": "No pending account deletion found"
  }
}
```

---

## Pagination

### Standard Pagination

All list endpoints support cursor-based pagination:

```http
GET /me/topic-summaries?limit=20&cursor={cursor}
```

**Parameters:**

| Param | Type | Default | Max | Description |
|-------|------|---------|-----|-------------|
| `limit` | int | 20 | 100 | Items per page |
| `cursor` | string | - | - | Opaque cursor from previous response |

**Response includes:**
```json
{
  "items": [...],
  "cursor": "eyJpZCI6IjEyMzQ1In0=",
  "has_more": true
}
```

### Paginated Endpoints

| Endpoint | Default Limit |
|----------|---------------|
| `GET /me/topic-summaries` | 20 |
| `GET /me/achievements` | 50 |
| `GET /me/learning-paths` | 10 |
| `GET /conversations/{id}/messages` | 50 |
| `GET /learning-paths` | 20 |

---

## Error Handling

### Error Response Format

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid request parameters",
    "details": [
      {
        "field": "email",
        "message": "Invalid email format"
      }
    ],
    "request_id": "req_abc123"
  }
}
```

### Error Codes

| Code | HTTP Status | Description |
|------|-------------|-------------|
| `UNAUTHORIZED` | 401 | Missing or invalid auth token |
| `FORBIDDEN` | 403 | Insufficient permissions |
| `NOT_FOUND` | 404 | Resource not found |
| `VALIDATION_ERROR` | 400 | Invalid request data |
| `RATE_LIMITED` | 429 | Too many requests |
| `SUBSCRIPTION_REQUIRED` | 402 | Premium feature |
| `SERVICE_UNAVAILABLE` | 503 | LLM temporarily unavailable |
| `INTERNAL_ERROR` | 500 | Server error |

### LLM Fallback Errors

```json
{
  "error": {
    "code": "LLM_DEGRADED",
    "message": "AI service is experiencing delays",
    "fallback_active": true,
    "estimated_wait_seconds": 30
  }
}
```

---

## Recovery Flows

### Curriculum Generation Failure

**Scenario:** Path Agent fails to generate valid curriculum (timeout, invalid output, or error).

```
┌─────────────────────────────────────────────────────────────────┐
│  CURRICULUM GENERATION FAILURE FLOW                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. First attempt fails                                          │
│     └── Automatic retry with same prompt (1× retry)              │
│                                                                  │
│  2. Second attempt fails                                         │
│     └── Switch to fallback model (OpenAI GPT-4o)                 │
│     └── Retry with simplified prompt                             │
│                                                                  │
│  3. Fallback fails                                               │
│     └── Return partial curriculum if any valid modules           │
│     └── Or return generic starter curriculum for subject         │
│                                                                  │
│  4. Complete failure                                             │
│     └── Show user: "We're having trouble creating your path"    │
│     └── Offer: [Try Again] [Choose Different Subject]            │
│     └── Log to curriculum_changes with error details             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**API Response (Partial Success):**
```json
{
  "curriculum": {
    "modules": [...],  // Whatever was successfully generated
    "generation_status": "partial",
    "warning": "Some modules could not be generated. We'll complete them as you progress."
  }
}
```

**API Response (Complete Failure):**
```json
{
  "error": {
    "code": "CURRICULUM_GENERATION_FAILED",
    "message": "Unable to generate curriculum. Please try again.",
    "retry_available": true,
    "alternatives": ["try_different_subject", "contact_support"]
  }
}
```

---

### Invalid JSON from LLM

**Scenario:** LLM returns malformed JSON that cannot be parsed.

```
┌─────────────────────────────────────────────────────────────────┐
│  INVALID JSON RECOVERY FLOW                                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. Parse attempt fails                                          │
│     └── Try JSON repair (fix common issues: trailing commas,     │
│         unclosed brackets, unquoted keys)                        │
│                                                                  │
│  2. Repair fails                                                 │
│     └── Extract content via regex (look for key fields)          │
│     └── Reconstruct minimal valid response                       │
│                                                                  │
│  3. Extraction fails                                             │
│     └── Retry LLM call with stricter prompt:                     │
│         "Output ONLY valid JSON. No markdown, no explanation."   │
│                                                                  │
│  4. Retry fails                                                  │
│     └── Log raw response for debugging                           │
│     └── Return graceful fallback (see per-agent fallbacks)       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Per-Agent Fallbacks:**

| Agent | Fallback if JSON Invalid |
|-------|--------------------------|
| **Interview** | Repeat last question, simplify |
| **Path** | Return generic curriculum template |
| **Teaching** | Return raw text response (no structured data) |
| **Assessment** | Return simple true/false question |
| **Summary** | Skip summary, continue without it |

---

### Interview Failure

**Scenario:** Interview Agent fails mid-conversation or produces unusable output.

```
Recovery Steps:
1. Save partial interview_summary with what we have
2. If goal captured: proceed with minimal curriculum
3. If goal missing: show simplified onboarding form
   └── Dropdowns: Subject, Level (Beginner/Intermediate/Advanced), Goal type
4. Generate curriculum from form data instead of interview
```

**API Response:**
```json
{
  "interview_status": "incomplete",
  "fallback_mode": "form",
  "form_fields": ["subject", "level", "goal_type"],
  "message": "Let's try a simpler approach. Please select:"
}
```

---

### Teaching Session Failure

**Scenario:** Teaching Agent fails to respond during a learning session.

```
┌─────────────────────────────────────────────────────────────────┐
│  TEACHING FAILURE RECOVERY                                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. Primary LLM (Claude) fails                                   │
│     └── Automatic switch to fallback (OpenAI)                    │
│     └── User sees: brief loading indicator (no error shown)      │
│                                                                  │
│  2. Fallback also fails                                          │
│     └── Show cached response if similar question asked before    │
│     └── Or: "I'm having trouble thinking. Can you rephrase?"     │
│                                                                  │
│  3. Multiple failures in session                                 │
│     └── After 3 failures: suggest taking a break                 │
│     └── "Our AI is having issues. Save your progress and return  │
│          later? We'll pick up where you left off."               │
│     └── Save conversation state to DB                            │
│                                                                  │
│  4. Complete outage                                              │
│     └── Banner: "AI tutoring temporarily unavailable"            │
│     └── Offer: Review past summaries, take practice quizzes      │
│          (pre-generated questions cached)                        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

### WebSocket Reconnection

**Scenario:** WebSocket connection drops during session.

```
Client-Side Recovery:
1. Detect disconnect (onclose event)
2. Show: "Reconnecting..." (3 second delay)
3. Attempt reconnect with same session_id
4. If reconnect succeeds:
   └── Request conversation sync: GET /conversations/{id}/messages?since={last_id}
   └── Resume seamlessly
5. If reconnect fails (3 attempts):
   └── Fall back to REST API mode
   └── Show: "Connection unstable. Switching to slower mode."
6. If REST also fails:
   └── Show: "Connection lost. Your progress is saved."
   └── [Retry] button
```

**Reconnection API:**
```http
POST /conversations/{conversation_id}/reconnect

Response:
{
  "session_valid": true,
  "messages_since_disconnect": 2,
  "last_agent_response": "...",
  "resume_from_message_id": "uuid"
}
```

---

### Error Logging for Admin

All errors are logged to `session_recordings.agent_calls` with:

```json
{
  "error_type": "json_parse_failure",
  "agent": "path",
  "raw_response": "...",  // First 5000 chars
  "recovery_action": "used_fallback_template",
  "user_id": "uuid",
  "timestamp": "ISO 8601"
}
```

Admin can review via:
```http
GET /admin/errors?type=llm_failure&since=2024-12-01
```

---

## Rate Limiting

### Limits

| Tier | Requests/min | AI Sessions/day | WebSocket Connections |
|------|--------------|-----------------|----------------------|
| Free | 60 | 3 | 1 |
| Premium | 120 | Unlimited | 3 |

### Rate Limit Headers

```
X-RateLimit-Limit: 60
X-RateLimit-Remaining: 45
X-RateLimit-Reset: 1702200000
```

### Rate Limited Response

```json
{
  "error": {
    "code": "RATE_LIMITED",
    "message": "Too many requests",
    "retry_after_seconds": 30
  }
}
```

---

## Versioning

API version is included in URL path: `/v1/`

Breaking changes will increment version. Old versions supported for 6 months after deprecation notice.

```
Deprecation Header:
Sunset: Sat, 01 Jun 2025 00:00:00 GMT
Deprecation: true
Link: <https://api.eduagent.app/v2>; rel="successor-version"
```

---

## Document History

| Date | Change | Author |
|------|--------|--------|
| 2024-12-10 | Initial API specification | Claude + User |
| 2024-12-10 | Rev 2: Added WebSocket protocol, Stripe webhooks, pagination, cancel deletion, data export | Claude + User |

