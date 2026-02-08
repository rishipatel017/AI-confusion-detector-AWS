# Design Document: AI Confusion Detector for Learners

## Overview

The AI Confusion Detector for Learners is a client-server system that monitors learner behavior during interaction with educational content, detects confusion using behavioral heuristics, and delivers adaptive AI-generated explanations. The system operates as a lightweight overlay on existing learning platforms without requiring platform migration.

### Key Design Principles

1. **Non-intrusive monitoring**: Capture behavioral signals without disrupting learning flow
2. **Real-time detection**: Process signals and detect confusion within 500ms
3. **Privacy-first**: Encrypt data, anonymize analytics, require explicit consent
4. **Graceful degradation**: Core functionality continues even when AI services are unavailable
5. **Adaptive learning**: Improve detection accuracy through feedback loops

### Technology Stack

- **Frontend**: JavaScript SDK (vanilla JS, framework-agnostic)
- **Backend**: Node.js/TypeScript with Express
- **Database**: PostgreSQL (behavioral data, analytics) + Redis (caching, session state)
- **AI Service**: LLM API (OpenAI GPT-4, Anthropic Claude, or similar)
- **Message Queue**: RabbitMQ (async processing, event streaming)
- **Monitoring**: Prometheus + Grafana

## Architecture

### System Components

```
┌─────────────────────────────────────────────────────────────┐
│                     Learning Platform                        │
│  ┌────────────────────────────────────────────────────┐    │
│  │         Content (Video/Text/PDF)                    │    │
│  └────────────────────────────────────────────────────┘    │
│  ┌────────────────────────────────────────────────────┐    │
│  │      Behavior Tracker SDK (JavaScript)             │    │
│  │  - Event Capture                                    │    │
│  │  - Local Buffering                                  │    │
│  │  - Batch Transmission                               │    │
│  └──────────────────┬─────────────────────────────────┘    │
└─────────────────────┼──────────────────────────────────────┘
                      │ HTTPS/WebSocket
                      ▼
┌─────────────────────────────────────────────────────────────┐
│                    API Gateway                               │
│  - Authentication                                            │
│  - Rate Limiting                                             │
│  - Request Routing                                           │
└──────────────┬──────────────────────────────────────────────┘
               │
       ┌───────┴────────┐
       ▼                ▼
┌─────────────┐  ┌─────────────────────────────────┐
│   Event     │  │   Confusion Detection Service   │
│  Processor  │──▶   - Heuristic Engine            │
│             │  │   - Score Calculation           │
│             │  │   - Threshold Management        │
└─────────────┘  └──────────┬──────────────────────┘
                            │
                            ▼
                 ┌──────────────────────┐
                 │  Explanation Service │
                 │  - LLM Integration   │
                 │  - Prompt Builder    │
                 │  - Cache Manager     │
                 └──────────┬───────────┘
                            │
                            ▼
                 ┌──────────────────────┐
                 │   Delivery Service   │
                 │  - UI Rendering      │
                 │  - Feedback Handler  │
                 └──────────────────────┘
```

### Data Flow

1. **Behavior Capture**: SDK captures user interactions (pause, rewind, scroll, dwell)
2. **Event Transmission**: Batched events sent to API Gateway every 2 seconds or on buffer full
3. **Event Processing**: Events parsed, validated, and enriched with session context
4. **Confusion Detection**: Heuristic engine calculates confusion scores per content segment
5. **Explanation Generation**: High/Medium severity triggers LLM explanation request
6. **Explanation Delivery**: Formatted explanation sent to client via WebSocket or polling
7. **Feedback Collection**: User feedback updates threshold weights and analytics

## Components and Interfaces

### 1. Behavior Tracker SDK

**Purpose**: Lightweight JavaScript library embedded in learning platforms to capture behavioral signals.

**Key Classes**:

```typescript
class BehaviorTracker {
  constructor(config: TrackerConfig)
  initialize(contentId: string, learnerId: string): void
  trackVideoEvent(event: VideoEvent): void
  trackScrollEvent(event: ScrollEvent): void
  trackDwellTime(segmentId: string, duration: number): void
  flush(): Promise<void>
  destroy(): void
}

interface TrackerConfig {
  apiEndpoint: string
  batchSize: number
  flushInterval: number
  enableOfflineQueue: boolean
}

interface VideoEvent {
  type: 'play' | 'pause' | 'seek' | 'rewind'
  timestamp: number
  segmentId: string
  position: number
}

interface ScrollEvent {
  type: 'scroll' | 'revisit'
  timestamp: number
  segmentId: string
  velocity: number
  direction: 'up' | 'down'
}
```

**Responsibilities**:
- Attach event listeners to video players and text content
- Buffer events locally (max 100 events or 2 seconds)
- Batch transmit to backend
- Handle offline scenarios with local queue
- Respect user privacy settings

**Performance Considerations**:
- Use passive event listeners to avoid blocking
- Debounce scroll events (100ms)
- Limit CPU usage to <3% through efficient buffering

### 2. Event Processor Service

**Purpose**: Receive, validate, and enrich behavioral events before confusion detection.

**Key Functions**:

```typescript
interface EventProcessor {
  processEvent(event: BehaviorEvent): Promise<EnrichedEvent>
  validateEvent(event: BehaviorEvent): ValidationResult
  enrichWithContext(event: BehaviorEvent): EnrichedEvent
  publishToQueue(event: EnrichedEvent): Promise<void>
}

interface BehaviorEvent {
  eventId: string
  learnerId: string
  contentId: string
  segmentId: string
  eventType: string
  timestamp: number
  metadata: Record<string, any>
}

interface EnrichedEvent extends BehaviorEvent {
  sessionId: string
  cohortId: string
  previousEvents: BehaviorEvent[]
  cohortBaseline: CohortBaseline
}
```

**Responsibilities**:
- Validate event schema and data types
- Enrich with session context (previous events in window)
- Attach cohort baseline data
- Publish to message queue for async processing
- Handle duplicate events (idempotency)

### 3. Confusion Detection Service

**Purpose**: Apply behavioral heuristics to calculate confusion scores and classify severity.

**Key Classes**:

```typescript
class ConfusionDetector {
  calculateScore(events: EnrichedEvent[], segment: ContentSegment): ConfusionScore
  classifySeverity(score: number): ConfusionSeverity
  applyHeuristics(events: EnrichedEvent[]): HeuristicResult[]
  updateThresholds(feedback: FeedbackSignal): void
}

interface ConfusionScore {
  segmentId: string
  score: number  // 0.0 to 1.0
  severity: ConfusionSeverity
  triggeringHeuristics: string[]
  timestamp: number
}

type ConfusionSeverity = 'low' | 'medium' | 'high'

interface HeuristicResult {
  heuristicName: string
  weight: number
  contribution: number
  triggered: boolean
}
```

**Heuristic Rules**:

1. **Repeated Rewind Heuristic**:
   - Condition: ≥2 rewinds within 30 seconds on same segment
   - Weight: 0.4 (adjustable via feedback)
   - Contribution: weight × (rewind_count / 2)

2. **Excessive Dwell Time Heuristic**:
   - Condition: dwell_time > cohort_average × 1.5
   - Weight: 0.3
   - Contribution: weight × (dwell_time / (cohort_average × 1.5))

3. **Rapid Scroll-Back Heuristic**:
   - Condition: rapid scroll followed by backward scroll within 10s
   - Weight: 0.2
   - Contribution: weight × (1 if triggered else 0)

4. **Extended Pause Heuristic**:
   - Condition: video pause >5 seconds
   - Weight: 0.1
   - Contribution: weight × (pause_duration / 5)

**Score Calculation**:
```
confusion_score = min(1.0, sum(heuristic_contributions))
```

**Severity Classification**:
- Low: 0.0 ≤ score < 0.3
- Medium: 0.3 ≤ score < 0.6
- High: 0.6 ≤ score ≤ 1.0

**Responsibilities**:
- Maintain sliding window of recent events (30 second window)
- Calculate confusion scores in <500ms
- Persist scores to database for analytics
- Emit confusion events to explanation service

### 4. Cohort Baseline Service

**Purpose**: Calculate and maintain statistical baselines for behavioral metrics across learner cohorts.

**Key Functions**:

```typescript
interface CohortBaselineService {
  calculateBaseline(contentId: string, segmentId: string): CohortBaseline
  updateBaseline(contentId: string, segmentId: string, newData: BehaviorData): void
  getBaseline(contentId: string, segmentId: string): CohortBaseline
}

interface CohortBaseline {
  segmentId: string
  avgDwellTime: number
  stdDevDwellTime: number
  avgRewindCount: number
  sampleSize: number
  lastUpdated: Date
}
```

**Calculation Logic**:
- Require minimum 10 learners before calculating baseline
- Use rolling average (exponential moving average with α=0.1)
- Update every 24 hours via scheduled job
- Use default values when sample size insufficient:
  - Default dwell time: 30 seconds for text, 10 seconds for video
  - Default rewind count: 0.5

**Responsibilities**:
- Aggregate behavioral data across learners
- Anonymize data before aggregation
- Provide baseline data to confusion detector
- Handle cold-start problem with sensible defaults

### 5. Explanation Service

**Purpose**: Generate adaptive explanations using LLM based on detected confusion points.

**Key Classes**:

```typescript
class ExplanationGenerator {
  generateExplanation(confusionPoint: ConfusionScore, content: ContentSegment): Promise<Explanation>
  buildPrompt(confusionPoint: ConfusionScore, content: ContentSegment, style: ExplanationStyle): string
  formatExplanation(rawResponse: string, style: ExplanationStyle): Explanation
  cacheExplanation(key: string, explanation: Explanation): void
  getCachedExplanation(key: string): Explanation | null
}

interface Explanation {
  explanationId: string
  segmentId: string
  content: string
  style: ExplanationStyle
  generatedAt: Date
  cached: boolean
}

type ExplanationStyle = 'simplified' | 'example' | 'stepByStep' | 'diagram'

interface ContentSegment {
  segmentId: string
  text: string
  startTime?: number  // for video
  endTime?: number
  context: string  // surrounding content for context
}
```

**LLM Prompt Template**:

```
You are an educational assistant helping a learner who is confused about a specific concept.

CONTENT SEGMENT:
{segment_text}

CONTEXT:
{surrounding_context}

TASK:
Provide a {style} explanation that:
1. Addresses ONLY the content segment above
2. Uses simple, clear language appropriate for {content_level}
3. Does NOT introduce new concepts
4. Stays under 300 words
5. {style_specific_instructions}

EXPLANATION:
```

**Style-Specific Instructions**:
- **Simplified**: "Use plain language and define any technical terms"
- **Example**: "Provide exactly one real-world example that relates to everyday experience"
- **Step-by-step**: "Break down the concept into 3-5 numbered steps"
- **Diagram**: "Create an ASCII or simple text-based diagram with labels"

**Caching Strategy**:
- Cache key: hash(contentId + segmentId + style)
- TTL: 7 days
- Cache hit rate target: >60%
- Invalidate on content updates

**Responsibilities**:
- Build context-aware prompts
- Call LLM API with timeout (2 seconds)
- Parse and format LLM responses
- Handle LLM failures gracefully
- Cache frequently requested explanations
- Rate limit LLM calls (max 100/minute per learner)

### 6. Delivery Service

**Purpose**: Render explanations in the UI and collect learner feedback.

**Key Functions**:

```typescript
interface DeliveryService {
  renderExplanation(explanation: Explanation, severity: ConfusionSeverity): UIComponent
  sendToClient(learnerId: string, component: UIComponent): Promise<void>
  collectFeedback(explanationId: string, feedback: FeedbackSignal): void
  applyRateLimiting(learnerId: string): boolean
}

interface UIComponent {
  type: 'tooltip' | 'sidePanel' | 'modal'
  content: string
  position: Position
  dismissible: boolean
  feedbackEnabled: boolean
}

interface FeedbackSignal {
  explanationId: string
  learnerId: string
  helpful: boolean | null  // null = dismissed without feedback
  textFeedback?: string
  timestamp: Date
}
```

**Delivery Rules**:
- High severity → side panel (non-blocking)
- Medium severity → inline tooltip (click to expand)
- Low severity → no automatic display
- Max 1 explanation per 60 seconds per learner
- Auto-dismiss after 5 minutes of inactivity

**Responsibilities**:
- Choose appropriate UI component based on severity
- Send via WebSocket (preferred) or HTTP polling fallback
- Collect and persist feedback signals
- Apply rate limiting to prevent overwhelming learners
- Track delivery metrics (shown, dismissed, engaged)

### 7. Analytics Service

**Purpose**: Aggregate confusion data and provide instructor dashboard.

**Key Functions**:

```typescript
interface AnalyticsService {
  aggregateConfusionPoints(contentId: string, dateRange: DateRange): ConfusionHeatmap
  calculateMetrics(contentId: string): AnalyticsMetrics
  exportData(contentId: string, format: 'csv' | 'json'): Promise<string>
}

interface ConfusionHeatmap {
  contentId: string
  segments: SegmentAnalytics[]
  overallScore: number
}

interface SegmentAnalytics {
  segmentId: string
  avgConfusionScore: number
  learnerCount: number
  positiveFeedback: number
  negativeFeedback: number
  topHeuristics: string[]
}

interface AnalyticsMetrics {
  totalLearners: number
  avgCompletionRate: number
  topConfusionPoints: SegmentAnalytics[]
  feedbackSentiment: number  // -1 to 1
}
```

**Responsibilities**:
- Aggregate confusion scores across learners
- Generate heatmap visualizations
- Calculate engagement metrics
- Provide data export functionality
- Support filtering by date range and cohort

## Data Models

### Database Schema (PostgreSQL)

```sql
-- Learners table
CREATE TABLE learners (
  learner_id UUID PRIMARY KEY,
  external_id VARCHAR(255) UNIQUE,
  consent_given BOOLEAN DEFAULT FALSE,
  consent_date TIMESTAMP,
  preferences JSONB,
  created_at TIMESTAMP DEFAULT NOW()
);

-- Content table
CREATE TABLE content (
  content_id UUID PRIMARY KEY,
  external_id VARCHAR(255) UNIQUE,
  content_type VARCHAR(50),  -- 'video', 'text', 'pdf'
  metadata JSONB,
  created_at TIMESTAMP DEFAULT NOW()
);

-- Content segments table
CREATE TABLE content_segments (
  segment_id UUID PRIMARY KEY,
  content_id UUID REFERENCES content(content_id),
  segment_index INTEGER,
  start_position FLOAT,  -- timestamp for video, offset for text
  end_position FLOAT,
  text_content TEXT,
  created_at TIMESTAMP DEFAULT NOW()
);

-- Behavioral events table (time-series optimized)
CREATE TABLE behavioral_events (
  event_id UUID PRIMARY KEY,
  learner_id UUID REFERENCES learners(learner_id),
  session_id UUID,
  content_id UUID REFERENCES content(content_id),
  segment_id UUID REFERENCES content_segments(segment_id),
  event_type VARCHAR(50),
  timestamp TIMESTAMP,
  metadata JSONB,
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_events_learner_time ON behavioral_events(learner_id, timestamp DESC);
CREATE INDEX idx_events_segment ON behavioral_events(segment_id, timestamp DESC);

-- Confusion scores table
CREATE TABLE confusion_scores (
  score_id UUID PRIMARY KEY,
  learner_id UUID REFERENCES learners(learner_id),
  segment_id UUID REFERENCES content_segments(segment_id),
  score FLOAT CHECK (score >= 0 AND score <= 1),
  severity VARCHAR(20),
  triggering_heuristics JSONB,
  timestamp TIMESTAMP,
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_scores_segment ON confusion_scores(segment_id, timestamp DESC);

-- Explanations table
CREATE TABLE explanations (
  explanation_id UUID PRIMARY KEY,
  segment_id UUID REFERENCES content_segments(segment_id),
  content TEXT,
  style VARCHAR(50),
  generated_at TIMESTAMP,
  cache_key VARCHAR(255) UNIQUE,
  created_at TIMESTAMP DEFAULT NOW()
);

-- Feedback signals table
CREATE TABLE feedback_signals (
  feedback_id UUID PRIMARY KEY,
  explanation_id UUID REFERENCES explanations(explanation_id),
  learner_id UUID REFERENCES learners(learner_id),
  helpful BOOLEAN,
  text_feedback TEXT,
  timestamp TIMESTAMP,
  created_at TIMESTAMP DEFAULT NOW()
);

-- Cohort baselines table
CREATE TABLE cohort_baselines (
  baseline_id UUID PRIMARY KEY,
  content_id UUID REFERENCES content(content_id),
  segment_id UUID REFERENCES content_segments(segment_id),
  avg_dwell_time FLOAT,
  std_dev_dwell_time FLOAT,
  avg_rewind_count FLOAT,
  sample_size INTEGER,
  last_updated TIMESTAMP,
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE UNIQUE INDEX idx_baseline_segment ON cohort_baselines(content_id, segment_id);

-- Heuristic weights table (for adaptive thresholds)
CREATE TABLE heuristic_weights (
  weight_id UUID PRIMARY KEY,
  heuristic_name VARCHAR(100),
  content_type VARCHAR(50),
  weight FLOAT CHECK (weight >= 0 AND weight <= 1),
  positive_feedback_count INTEGER DEFAULT 0,
  negative_feedback_count INTEGER DEFAULT 0,
  last_updated TIMESTAMP,
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE UNIQUE INDEX idx_heuristic_type ON heuristic_weights(heuristic_name, content_type);
```

### Redis Cache Schema

```
# Session state
session:{session_id} -> {
  learner_id: string,
  content_id: string,
  start_time: timestamp,
  last_activity: timestamp,
  event_buffer: Event[]
}
TTL: 1 hour

# Explanation cache
explanation:{cache_key} -> {
  content: string,
  style: string,
  generated_at: timestamp
}
TTL: 7 days

# Rate limiting
rate_limit:explanation:{learner_id} -> count
TTL: 60 seconds

rate_limit:delivery:{learner_id} -> last_delivery_time
TTL: 60 seconds

# Cohort baseline cache
baseline:{content_id}:{segment_id} -> CohortBaseline
TTL: 24 hours
```

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*


### Property 1: Event Recording Completeness

*For any* behavioral event (pause, rewind, scroll, dwell, selection), when recorded by the system, all required fields (timestamp, segment ID, event type, and type-specific metadata) should be present and correctly populated.

**Validates: Requirements 1.1, 1.2, 2.1, 2.3, 2.4**

### Property 2: Timestamp Precision

*For any* behavioral signal recorded by the system, the timestamp should have millisecond precision (no truncation to seconds).

**Validates: Requirements 1.5**

### Property 3: Pause Duration Calculation

*For any* pause-resume event pair, the calculated pause duration should equal the difference between the resume timestamp and the pause timestamp.

**Validates: Requirements 1.3**

### Property 4: Rewind Counter Accuracy

*For any* sequence of rewind events on the same content segment, if N rewinds occur within a 30-second window, the rewind counter for that segment should equal N.

**Validates: Requirements 1.4**

### Property 5: Dwell Time Calculation

*For any* content segment where a learner dwells, the calculated dwell time should equal the duration between entering and leaving the segment.

**Validates: Requirements 2.2**

### Property 6: Text Segmentation at Boundaries

*For any* text content with paragraph or section boundaries, the system should create content segments that align exactly with those boundaries (no mid-paragraph splits).

**Validates: Requirements 2.5**

### Property 7: Repeated Rewind Heuristic

*For any* content segment with 2 or more rewind events within 30 seconds, the confusion score contribution from the repeated rewind heuristic should be at least 0.4 × (rewind_count / 2).

**Validates: Requirements 3.1**

### Property 8: Excessive Dwell Time Heuristic

*For any* content segment where dwell time exceeds 1.5 times the cohort average, the confusion score contribution from the dwell time heuristic should be at least 0.3.

**Validates: Requirements 3.2**

### Property 9: Rapid Scroll-Back Heuristic

*For any* event sequence with rapid forward scrolling followed by backward scrolling within 10 seconds, the confusion score for the target segment should increase by at least 0.2.

**Validates: Requirements 3.3**

### Property 10: Extended Pause Heuristic

*For any* video pause event lasting more than 5 seconds, the confusion score for that segment should increase by at least 0.1.

**Validates: Requirements 3.4**

### Property 11: Confusion Severity Classification

*For any* confusion score value between 0.0 and 1.0, the severity classification should be: Low if score < 0.3, Medium if 0.3 ≤ score < 0.6, and High if score ≥ 0.6.

**Validates: Requirements 3.6, 3.7, 3.8**

### Property 12: Cohort Statistics Calculation

*For any* content segment with data from at least 10 learners, the calculated cohort average (for dwell time or rewind frequency) should equal the arithmetic mean of the individual learner values for that segment.

**Validates: Requirements 4.1, 4.2**

### Property 13: Default Baseline Fallback

*For any* content segment with fewer than 10 learner interactions, the system should use default baseline values rather than calculated cohort averages.

**Validates: Requirements 4.4**

### Property 14: Baseline Storage and Retrieval

*For any* content segment, if a cohort baseline is stored with a specific segment identifier, retrieving by that identifier should return the same baseline values.

**Validates: Requirements 4.5**

### Property 15: LLM Context Transmission

*For any* confusion point with Medium or High severity, the data sent to the LLM should include the content segment text and surrounding context.

**Validates: Requirements 5.1**

### Property 16: Prompt Constraint Inclusion

*For any* explanation generation request, the LLM prompt should contain explicit instructions to: (1) address only the flagged segment, (2) use simplified language, and (3) avoid introducing new concepts.

**Validates: Requirements 5.2, 5.3, 5.4**

### Property 17: Explanation Style Preference

*For any* learner with a preferred explanation style setting, explanations generated for that learner should use the specified style (simplified, example, step-by-step, or diagram).

**Validates: Requirements 5.6**

### Property 18: Explanation Length Limit

*For any* generated explanation, the word count should not exceed 300 words.

**Validates: Requirements 6.6**

### Property 19: Severity-Based Delivery Method

*For any* confusion point, the delivery method should be: side panel for High severity, inline tooltip for Medium severity, and no automatic display for Low severity.

**Validates: Requirements 7.1, 7.2, 7.3**

### Property 20: Auto-Pause Behavior

*For any* learner with auto-pause mode enabled, when a High severity confusion point is detected during video playback, the video should pause.

**Validates: Requirements 7.6**

### Property 21: Explanation Rate Limiting

*For any* 60-second time window, at most one explanation should be displayed to a learner, regardless of how many confusion points are detected.

**Validates: Requirements 7.7**

### Property 22: Feedback Recording Correctness

*For any* feedback action (helpful click, not helpful click, or dismissal), a feedback signal should be recorded with the correct type (positive, negative, or neutral) and associated with the correct explanation and confusion point.

**Validates: Requirements 8.2, 8.3, 8.5, 8.6**

### Property 23: Adaptive Weight Adjustment

*For any* heuristic, when it accumulates 5 negative feedback signals, its weight should decrease by 0.05, and when it accumulates 10 positive feedback signals, its weight should increase by 0.05.

**Validates: Requirements 9.1, 9.2**

### Property 24: Weight Bounds Invariant

*For any* heuristic weight adjustment operation, the resulting weight should remain within the bounds [0.0, 1.0].

**Validates: Requirements 9.3**

### Property 25: Content-Type-Specific Thresholds

*For any* threshold adjustment applied to a video content heuristic, it should not affect the thresholds for text content heuristics, and vice versa.

**Validates: Requirements 9.4**

### Property 26: Threshold Persistence

*For any* threshold adjustment made during a user session, if the session ends and a new session begins, the adjusted threshold values should be retained (not reset to defaults).

**Validates: Requirements 9.5**

### Property 27: Threshold Reset on Pattern Reversal

*For any* heuristic, if the feedback pattern reverses by more than 50% (e.g., from 80% positive to 80% negative), the threshold adjustments should be reset to default values.

**Validates: Requirements 9.6**

### Property 28: Consent Enforcement

*For any* learner who has not provided explicit opt-in consent, no behavioral signals should be collected or stored for that learner.

**Validates: Requirements 10.1**

### Property 29: Data Encryption Round-Trip

*For any* behavioral signal stored in the database, encrypting then decrypting should produce the original signal data (round-trip property).

**Validates: Requirements 10.2**

### Property 30: Anonymization in Aggregation

*For any* cohort baseline calculation, the aggregated data should not contain learner identifiers (learner IDs should be stripped before aggregation).

**Validates: Requirements 10.4**

### Property 31: Graceful LLM Failure Handling

*For any* confusion point detected when the LLM service is unavailable, behavioral signal tracking should continue normally and explanation requests should be queued for later processing.

**Validates: Requirements 11.6, 15.1**

### Property 32: Cached Explanation Fallback

*For any* confusion point that matches a previously seen confusion point (same segment, similar score), if the LLM service is unavailable, the system should serve the cached explanation from the previous encounter.

**Validates: Requirements 11.7, 15.3**

### Property 33: Webhook Delivery Completeness

*For any* detected confusion point, if webhook notifications are configured, a webhook should be sent containing the segment ID, confusion score, severity, and timestamp.

**Validates: Requirements 12.5**

### Property 34: Analytics Aggregation Correctness

*For any* content with multiple learners, the aggregated analytics (average confusion score, learner count, feedback percentages) should correctly reflect the underlying individual learner data.

**Validates: Requirements 13.3, 13.4, 13.5**

### Property 35: Analytics Filtering

*For any* analytics query with date range or cohort filters applied, the returned data should include only records that match all specified filter criteria.

**Validates: Requirements 13.6**

### Property 36: CSV Export Validity

*For any* analytics data export in CSV format, the output should be valid CSV (parseable by standard CSV parsers) and contain all the data visible in the dashboard.

**Validates: Requirements 13.7**

### Property 37: Diagram Alternative Text

*For any* explanation containing a diagram, the explanation should also include alternative text describing the diagram content.

**Validates: Requirements 14.7**

### Property 38: Service Independence

*For any* external service failure (analytics service, webhook service), the core confusion detection functionality should continue operating normally.

**Validates: Requirements 15.4**

### Property 39: Offline Queue and Sync

*For any* behavioral signals generated while network connectivity is lost, those signals should be queued locally and transmitted to the server when connectivity is restored.

**Validates: Requirements 15.5**

### Property 40: Error Handling Robustness

*For any* external dependency failure (LLM, database, cache), the system should handle the error gracefully without crashing or freezing (should return error responses or fallback behavior).

**Validates: Requirements 15.6**

### Property 41: Failure Logging

*For any* service failure or external dependency error, a log entry should be created containing the failure type, timestamp, and relevant context.

**Validates: Requirements 15.7**

## Error Handling

### Error Categories

1. **Client-Side Errors**
   - Invalid event data (missing required fields)
   - Network connectivity loss
   - Browser compatibility issues
   - Local storage quota exceeded

2. **Server-Side Errors**
   - LLM API failures (timeout, rate limit, service unavailable)
   - Database connection failures
   - Cache service unavailable
   - Invalid authentication tokens

3. **Data Errors**
   - Malformed content segments
   - Missing cohort baselines
   - Corrupted cached explanations
   - Invalid feedback signals

### Error Handling Strategies

#### Client-Side Error Handling

```typescript
class ErrorHandler {
  handleEventValidationError(event: BehaviorEvent): void {
    // Log error locally
    console.warn('Invalid event data:', event)
    // Drop invalid event, continue tracking
    // Do not disrupt user experience
  }

  handleNetworkError(queuedEvents: BehaviorEvent[]): void {
    // Store events in local storage
    localStorage.setItem('pending_events', JSON.stringify(queuedEvents))
    // Set up retry with exponential backoff
    this.scheduleRetry(1000) // Start with 1 second
  }

  handleStorageQuotaExceeded(): void {
    // Clear oldest events from queue
    // Keep only most recent 100 events
    // Log warning for monitoring
  }
}
```

#### Server-Side Error Handling

```typescript
class LLMErrorHandler {
  async handleLLMTimeout(confusionPoint: ConfusionScore): Promise<Explanation | null> {
    // Check cache for similar confusion point
    const cached = await this.cache.getSimilar(confusionPoint)
    if (cached) {
      return cached
    }
    
    // Queue for retry
    await this.queue.enqueue(confusionPoint)
    
    // Return null (no explanation shown)
    return null
  }

  async handleLLMRateLimit(confusionPoint: ConfusionScore): Promise<Explanation | null> {
    // Queue with priority based on severity
    const priority = confusionPoint.severity === 'high' ? 1 : 2
    await this.queue.enqueue(confusionPoint, priority)
    
    // Check cache
    return await this.cache.getSimilar(confusionPoint)
  }

  async handleLLMServiceUnavailable(): Promise<void> {
    // Set circuit breaker to open state
    this.circuitBreaker.open()
    
    // Switch to cache-only mode
    this.mode = 'cache-only'
    
    // Alert administrators
    await this.alerting.send('LLM service unavailable')
    
    // Schedule health check in 60 seconds
    setTimeout(() => this.checkLLMHealth(), 60000)
  }
}
```

#### Database Error Handling

```typescript
class DatabaseErrorHandler {
  async handleConnectionFailure(): Promise<void> {
    // Attempt reconnection with exponential backoff
    await this.reconnect()
    
    // If reconnection fails, use in-memory buffer
    this.useInMemoryBuffer = true
    
    // Alert administrators
    await this.alerting.send('Database connection lost')
  }

  async handleQueryTimeout(query: string): Promise<void> {
    // Log slow query for optimization
    logger.warn('Query timeout:', query)
    
    // Return cached result if available
    // Otherwise return empty result set
  }
}
```

### Circuit Breaker Pattern

Implement circuit breaker for LLM API calls to prevent cascading failures:

```typescript
class CircuitBreaker {
  private state: 'closed' | 'open' | 'half-open' = 'closed'
  private failureCount: number = 0
  private failureThreshold: number = 5
  private timeout: number = 60000 // 60 seconds

  async execute<T>(operation: () => Promise<T>): Promise<T | null> {
    if (this.state === 'open') {
      // Circuit is open, fail fast
      return null
    }

    try {
      const result = await operation()
      this.onSuccess()
      return result
    } catch (error) {
      this.onFailure()
      throw error
    }
  }

  private onSuccess(): void {
    this.failureCount = 0
    if (this.state === 'half-open') {
      this.state = 'closed'
    }
  }

  private onFailure(): void {
    this.failureCount++
    if (this.failureCount >= this.failureThreshold) {
      this.state = 'open'
      setTimeout(() => {
        this.state = 'half-open'
      }, this.timeout)
    }
  }
}
```

### Retry Strategies

**Exponential Backoff for Network Errors**:
```typescript
async function retryWithBackoff<T>(
  operation: () => Promise<T>,
  maxRetries: number = 3
): Promise<T> {
  let lastError: Error
  
  for (let i = 0; i < maxRetries; i++) {
    try {
      return await operation()
    } catch (error) {
      lastError = error
      const delay = Math.min(1000 * Math.pow(2, i), 10000)
      await sleep(delay)
    }
  }
  
  throw lastError
}
```

### Graceful Degradation Modes

1. **Cache-Only Mode**: When LLM unavailable, serve only cached explanations
2. **Tracking-Only Mode**: When backend unavailable, continue tracking locally
3. **Read-Only Mode**: When database write fails, continue reads and queue writes
4. **Minimal Mode**: When all services fail, disable explanations but keep basic tracking

## Testing Strategy

### Dual Testing Approach

The system requires both unit testing and property-based testing for comprehensive coverage:

- **Unit tests**: Verify specific examples, edge cases, and error conditions
- **Property tests**: Verify universal properties across all inputs

Both approaches are complementary and necessary. Unit tests catch concrete bugs in specific scenarios, while property tests verify general correctness across a wide range of inputs.

### Property-Based Testing

**Library Selection**: Use **fast-check** for JavaScript/TypeScript property-based testing.

**Configuration**:
- Minimum 100 iterations per property test (due to randomization)
- Each property test must reference its design document property
- Tag format: `// Feature: ai-confusion-detector, Property {number}: {property_text}`

**Example Property Test**:

```typescript
import fc from 'fast-check'

// Feature: ai-confusion-detector, Property 3: Pause Duration Calculation
describe('Pause Duration Calculation', () => {
  it('should calculate pause duration as resume time minus pause time', () => {
    fc.assert(
      fc.property(
        fc.integer({ min: 0, max: 1000000 }), // pause timestamp
        fc.integer({ min: 0, max: 10000 }),   // pause duration
        (pauseTime, duration) => {
          const resumeTime = pauseTime + duration
          const event = {
            pauseTimestamp: pauseTime,
            resumeTimestamp: resumeTime
          }
          
          const calculated = calculatePauseDuration(event)
          
          expect(calculated).toBe(duration)
        }
      ),
      { numRuns: 100 }
    )
  })
})
```

### Unit Testing

**Focus Areas**:
- Specific examples demonstrating correct behavior
- Edge cases (empty inputs, boundary values, null/undefined)
- Error conditions and exception handling
- Integration points between components

**Example Unit Test**:

```typescript
// Feature: ai-confusion-detector, Requirement 5.7
describe('Default Explanation Style', () => {
  it('should use simplified text with example when no preference set', async () => {
    const learner = { id: 'learner-1', preferences: {} }
    const confusionPoint = createConfusionPoint('high')
    
    const explanation = await generateExplanation(learner, confusionPoint)
    
    expect(explanation.style).toBe('simplified')
    expect(explanation.content).toContain('example')
  })
})
```

### Test Coverage Requirements

- **Code coverage**: Minimum 80% line coverage
- **Property coverage**: Each correctness property must have at least one property test
- **Edge case coverage**: All identified edge cases must have unit tests
- **Error path coverage**: All error handling paths must be tested

### Integration Testing

**Scenarios**:
1. End-to-end flow: Event capture → Confusion detection → Explanation generation → Delivery
2. LLM integration: Verify prompt construction and response parsing
3. Database integration: Verify data persistence and retrieval
4. Cache integration: Verify cache hit/miss behavior
5. Webhook integration: Verify webhook delivery

### Performance Testing

**Metrics**:
- Confusion detection latency: <500ms (p95)
- Explanation generation latency: <2s (p95)
- Client-side CPU overhead: <3%
- API response time: <200ms (p95)
- Throughput: 100k concurrent learners

**Tools**: Apache JMeter or k6 for load testing

### Accessibility Testing

**Tools**:
- axe-core for automated accessibility testing
- Manual testing with screen readers (NVDA, JAWS, VoiceOver)
- Keyboard navigation testing

**Requirements**:
- WCAG 2.1 Level AA compliance
- All interactive elements keyboard accessible
- Proper ARIA labels and roles
- Sufficient color contrast (4.5:1 minimum)

### Security Testing

**Focus Areas**:
- Input validation and sanitization
- SQL injection prevention
- XSS prevention
- CSRF protection
- Authentication and authorization
- Data encryption verification
- Privacy compliance (GDPR, FERPA)

**Tools**: OWASP ZAP for security scanning

### Monitoring and Observability

**Metrics to Track**:
- Confusion detection accuracy (via feedback signals)
- Explanation helpfulness rate
- LLM API latency and error rate
- Cache hit rate
- System uptime
- Error rates by category

**Logging**:
- Structured logging (JSON format)
- Log levels: DEBUG, INFO, WARN, ERROR
- Include correlation IDs for request tracing
- Sensitive data redaction

**Alerting**:
- LLM service unavailable
- Database connection failures
- High error rates (>5%)
- Low explanation helpfulness (<50%)
- Performance degradation

## Implementation Notes

### Phased Rollout

**Phase 1: Core Functionality**
- Behavior tracking SDK
- Basic confusion detection (rewind and dwell time heuristics)
- Simple explanation generation
- Side panel delivery

**Phase 2: Enhanced Detection**
- Additional heuristics (scroll patterns, pause duration)
- Cohort baseline calculation
- Adaptive threshold adjustment

**Phase 3: Analytics and Optimization**
- Instructor dashboard
- Advanced analytics
- Performance optimization
- Cache optimization

**Phase 4: Advanced Features**
- Multiple explanation styles
- Diagram generation
- Webhook integrations
- Advanced accessibility features

### Technology Considerations

**Frontend SDK**:
- Keep bundle size <50KB (gzipped)
- Support IE11+ (if required) or modern browsers only
- Use passive event listeners for performance
- Implement request batching to reduce network calls

**Backend Services**:
- Use microservices architecture for scalability
- Implement service mesh for inter-service communication
- Use message queues for async processing
- Implement rate limiting and throttling

**Database**:
- Use PostgreSQL for relational data
- Consider TimescaleDB extension for time-series event data
- Implement read replicas for analytics queries
- Use connection pooling (PgBouncer)

**Caching**:
- Use Redis for session state and explanation cache
- Implement cache warming for popular content
- Use cache-aside pattern
- Set appropriate TTLs based on data volatility

**LLM Integration**:
- Support multiple LLM providers (OpenAI, Anthropic, local models)
- Implement provider fallback
- Use streaming responses for faster perceived performance
- Implement prompt versioning for A/B testing

### Security Considerations

**Data Protection**:
- Encrypt data at rest (AES-256)
- Encrypt data in transit (TLS 1.3)
- Implement key rotation
- Use secure key management (AWS KMS, HashiCorp Vault)

**Authentication**:
- Use JWT tokens for API authentication
- Implement token refresh mechanism
- Support SSO integration (SAML, OAuth)
- Implement rate limiting per user

**Privacy**:
- Implement data retention policies
- Support right to deletion (GDPR)
- Anonymize data for analytics
- Provide privacy dashboard for learners

**Compliance**:
- GDPR compliance (EU)
- FERPA compliance (US education)
- COPPA compliance (if serving minors)
- Regular security audits

### Scalability Considerations

**Horizontal Scaling**:
- Stateless services for easy scaling
- Load balancing across service instances
- Auto-scaling based on CPU/memory metrics
- Database sharding by content_id if needed

**Performance Optimization**:
- Use CDN for SDK distribution
- Implement database query optimization
- Use database indexes strategically
- Implement connection pooling
- Use async processing for non-critical paths

**Cost Optimization**:
- Cache aggressively to reduce LLM API calls
- Use spot instances for non-critical workloads
- Implement request deduplication
- Monitor and optimize LLM token usage
