# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Online Course Marketplace · Created: 2026-05-19

## Philosophy

The event-sourced model stores every state change as an immutable event in an append-only event store. The current state of any entity is derived by replaying its event stream. Read-optimized projections (materialized views) are built asynchronously from the event stream to serve queries (CQRS -- Command Query Responsibility Segregation). This architecture is used by financial trading platforms, healthcare record systems, and any domain where a complete audit trail is not just a compliance checkbox but a core product feature.

For an online course marketplace, event sourcing is compelling because the domain has strong temporal requirements: "What was the course curriculum when this student enrolled?", "What price was this course at the time of purchase?", "When exactly did the learner complete each lesson?", "What was the instructor's revenue share rate when this payout was calculated?" These questions are trivial with event sourcing and painful with traditional CRUD.

The event store also becomes the foundation for AI analytics: learning pattern recognition, churn prediction, and recommendation engines all benefit from having the full behavioral history of every learner, instructor, and course as a stream of typed events rather than point-in-time snapshots.

**Best for:** Platforms prioritizing complete audit trails, temporal queries, AI-powered analytics on behavioral data, and regulatory environments (FERPA, GDPR) where proving what happened and when is critical.

**Trade-offs:**
- (+) Complete, immutable audit trail -- every state change is recorded with timestamp and actor
- (+) Temporal queries are natural: "state as of date X" is just replaying events to that point
- (+) Event streams are ideal input for AI/ML pipelines, recommendation engines, and analytics
- (+) Schema evolution is additive: new event types are added without modifying existing data
- (+) Event replay enables rebuilding read models without data loss
- (-) Higher storage requirements: events accumulate indefinitely; snapshots mitigate but add complexity
- (-) Eventual consistency between event store and read projections introduces latency
- (-) More complex infrastructure: event store + projection engine + read database
- (-) Harder to query ad-hoc across entities without well-designed read projections
- (-) Development team must understand event sourcing patterns, which have a steeper learning curve
- (-) GDPR right-to-erasure conflicts with immutability; requires crypto-shredding or event redaction patterns

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| IEEE 9274.1.1 (xAPI) | xAPI statements are events by nature; the LRS is essentially an event store for learning activities. Event sourcing aligns perfectly with xAPI's actor-verb-object statement model |
| SCORM 1.2 / 2004 | SCORM tracking interactions stored as events; CMI model state derived from event replay |
| Open Badges 3.0 / W3C VC | Credential issuance and revocation are events; the credential's current validity is derived from its event history |
| GDPR | Right-to-erasure handled via crypto-shredding: PII encrypted with per-user key; key deletion makes events unreadable without altering the event stream |
| FERPA | Complete event history provides auditable proof of data access, modification, and consent |
| ISO/IEC 40180 | Quality metrics (completion rates, engagement patterns) computed from event stream aggregation |
| PCI DSS v4.0.1 | Payment events reference Stripe IDs only; no card data enters the event store |

---

## Event Store Core

```sql
-- ============================================================
-- EVENT STORE -- the single source of truth
-- ============================================================

CREATE TABLE event_store (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_type     VARCHAR(100) NOT NULL,                 -- 'Course','Enrollment','Order','User','Assessment'
    stream_id       UUID NOT NULL,                         -- aggregate root ID
    event_type      VARCHAR(200) NOT NULL,                 -- 'CourseCreated','LessonCompleted','PaymentProcessed'
    event_version   INT NOT NULL,                          -- monotonically increasing per stream
    payload         JSONB NOT NULL,                        -- event-specific data
    metadata        JSONB NOT NULL DEFAULT '{}',           -- actor_id, ip_address, correlation_id, causation_id
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_id, event_version)
);

-- Partitioned by month for storage management
-- In production, use declarative partitioning:
-- CREATE TABLE event_store (...) PARTITION BY RANGE (created_at);

CREATE INDEX idx_event_stream ON event_store (stream_id, event_version);
CREATE INDEX idx_event_type ON event_store (event_type);
CREATE INDEX idx_event_created ON event_store (created_at);
CREATE INDEX idx_event_stream_type ON event_store (stream_type, stream_id);

-- Snapshot store for performance: avoids replaying entire streams
CREATE TABLE event_snapshot (
    stream_type     VARCHAR(100) NOT NULL,
    stream_id       UUID NOT NULL,
    snapshot_version INT NOT NULL,
    state           JSONB NOT NULL,                        -- serialized aggregate state
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_id, snapshot_version)
);

-- Outbox for reliable event publishing to projections / external consumers
CREATE TABLE event_outbox (
    id              BIGSERIAL PRIMARY KEY,
    event_id        UUID NOT NULL REFERENCES event_store (event_id),
    destination     VARCHAR(100) NOT NULL,                 -- 'projections', 'analytics', 'notifications'
    published       BOOLEAN NOT NULL DEFAULT FALSE,
    published_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_outbox_unpublished ON event_outbox (destination) WHERE NOT published;
```

### Example Event Payloads

```sql
-- CourseCreated event
-- payload: {
--   "title": "Introduction to Machine Learning",
--   "slug": "intro-ml",
--   "instructor_id": "550e8400-e29b-41d4-a716-446655440001",
--   "organisation_id": "550e8400-e29b-41d4-a716-446655440002",
--   "language_code": "en",
--   "level": "beginner",
--   "description": "Learn the fundamentals of ML..."
-- }
-- metadata: {
--   "actor_id": "550e8400-e29b-41d4-a716-446655440001",
--   "correlation_id": "req-abc-123",
--   "ip_address": "203.0.113.42"
-- }

-- LessonCompleted event
-- payload: {
--   "enrollment_id": "...",
--   "lesson_id": "...",
--   "time_spent_secs": 1847,
--   "video_watched_pct": 95.5,
--   "completed_at": "2026-05-19T14:32:00Z"
-- }

-- PaymentProcessed event
-- payload: {
--   "order_id": "...",
--   "user_id": "...",
--   "course_id": "...",
--   "amount": 49.99,
--   "currency": "USD",
--   "stripe_payment_intent_id": "pi_3abc...",
--   "coupon_code": "LEARN20",
--   "discount_amount": 10.00
-- }

-- CredentialIssued event
-- payload: {
--   "credential_id": "...",
--   "user_id": "...",
--   "course_id": "...",
--   "credential_type": "completion",
--   "open_badge_json": { ... },          -- Open Badges 3.0 JSON-LD
--   "verification_url": "https://..."
-- }
```

### Event Type Registry

```sql
-- Registry of all known event types with their schemas
CREATE TABLE event_type_registry (
    event_type      VARCHAR(200) PRIMARY KEY,
    stream_type     VARCHAR(100) NOT NULL,
    schema_version  INT NOT NULL DEFAULT 1,
    json_schema     JSONB NOT NULL,                        -- JSON Schema for payload validation
    description     TEXT,
    deprecated      BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Example event types:
-- User domain:       UserRegistered, UserProfileUpdated, UserDeactivated, UserRoleGranted, UserPIIRedacted
-- Course domain:     CourseCreated, CourseTitleUpdated, CoursePublished, CourseArchived, SectionAdded,
--                    LessonAdded, LessonVideoUploaded, LessonReordered, CoursePriceChanged
-- Enrollment domain: EnrollmentCreated, LessonStarted, LessonCompleted, AssessmentAttempted,
--                    AssessmentPassed, CourseCompleted, EnrollmentExpired, EnrollmentRefunded
-- Payment domain:    OrderCreated, PaymentProcessed, PaymentFailed, RefundIssued, PayoutCalculated, PayoutTransferred
-- Review domain:     ReviewSubmitted, ReviewUpdated, ReviewFlagged, ReviewRemoved
-- Credential domain: CredentialIssued, CredentialRevoked, CredentialVerified
-- SCORM domain:      SCORMPackageUploaded, SCORMSessionStarted, SCORMDataUpdated, SCORMSessionCompleted
-- LTI domain:        LTIToolRegistered, LTILaunchInitiated, LTIGradePassedBack
```

---

## Read Projections (Materialized Read Models)

These tables are **derived** from the event store and can be rebuilt at any time by replaying events. They serve as the query layer.

```sql
-- ============================================================
-- PROJECTION: Course Catalog (optimized for search & browse)
-- ============================================================

CREATE TABLE proj_course (
    id                  UUID PRIMARY KEY,
    organisation_id     UUID NOT NULL,
    instructor_id       UUID NOT NULL,
    instructor_name     VARCHAR(255),                      -- denormalized for display
    title               VARCHAR(500) NOT NULL,
    slug                VARCHAR(500) NOT NULL,
    subtitle            VARCHAR(500),
    description         TEXT,
    language_code       VARCHAR(10),
    level               VARCHAR(20),
    status              VARCHAR(30),
    thumbnail_url       TEXT,
    estimated_duration  INTERVAL,
    is_free             BOOLEAN DEFAULT FALSE,
    published_at        TIMESTAMPTZ,
    -- denormalized aggregates
    avg_rating          NUMERIC(3,2) DEFAULT 0.00,
    rating_count        INT DEFAULT 0,
    enrollment_count    INT DEFAULT 0,
    lesson_count        INT DEFAULT 0,
    total_video_secs    INT DEFAULT 0,
    -- pricing snapshot
    base_price          NUMERIC(10,2),
    currency_code       CHAR(3),
    -- categories and skills as arrays for search
    category_ids        UUID[],
    category_names      TEXT[],
    skill_tags          TEXT[],
    -- projection metadata
    last_event_id       UUID,
    last_event_version  INT,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_course_status ON proj_course (status);
CREATE INDEX idx_proj_course_instructor ON proj_course (instructor_id);
CREATE INDEX idx_proj_course_categories ON proj_course USING GIN (category_ids);
CREATE INDEX idx_proj_course_skills ON proj_course USING GIN (skill_tags);
CREATE INDEX idx_proj_course_search ON proj_course USING GIN (
    to_tsvector('english', coalesce(title, '') || ' ' || coalesce(subtitle, '') || ' ' || coalesce(description, ''))
);

-- ============================================================
-- PROJECTION: Course Curriculum (for course player)
-- ============================================================

CREATE TABLE proj_curriculum (
    course_id       UUID NOT NULL,
    section_id      UUID NOT NULL,
    section_title   VARCHAR(500),
    section_order   INT,
    lesson_id       UUID NOT NULL,
    lesson_title    VARCHAR(500),
    lesson_type     VARCHAR(30),
    lesson_order    INT,
    is_preview      BOOLEAN DEFAULT FALSE,
    duration_secs   INT,
    PRIMARY KEY (course_id, section_id, lesson_id)
);

-- ============================================================
-- PROJECTION: Learner Dashboard
-- ============================================================

CREATE TABLE proj_enrollment (
    id              UUID PRIMARY KEY,
    user_id         UUID NOT NULL,
    course_id       UUID NOT NULL,
    course_title    VARCHAR(500),                          -- denormalized
    course_thumbnail TEXT,
    instructor_name VARCHAR(255),
    status          VARCHAR(30),
    enrolled_at     TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    progress_pct    NUMERIC(5,2) DEFAULT 0.00,
    lessons_completed INT DEFAULT 0,
    lessons_total   INT DEFAULT 0,
    last_lesson_id  UUID,
    last_accessed   TIMESTAMPTZ,
    certificate_url TEXT,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_enrollment_user ON proj_enrollment (user_id, status);
CREATE INDEX idx_proj_enrollment_course ON proj_enrollment (course_id);

-- ============================================================
-- PROJECTION: Instructor Analytics
-- ============================================================

CREATE TABLE proj_instructor_stats (
    instructor_id       UUID NOT NULL,
    period              DATE NOT NULL,                     -- first day of month
    course_id           UUID NOT NULL,
    enrollments         INT DEFAULT 0,
    completions         INT DEFAULT 0,
    revenue_gross       NUMERIC(10,2) DEFAULT 0.00,
    revenue_net         NUMERIC(10,2) DEFAULT 0.00,
    refunds             INT DEFAULT 0,
    avg_rating          NUMERIC(3,2),
    new_reviews         INT DEFAULT 0,
    avg_completion_pct  NUMERIC(5,2),
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (instructor_id, period, course_id)
);

-- ============================================================
-- PROJECTION: Revenue & Payments
-- ============================================================

CREATE TABLE proj_order (
    id              UUID PRIMARY KEY,
    user_id         UUID NOT NULL,
    user_email      VARCHAR(320),
    status          VARCHAR(20),
    currency_code   CHAR(3),
    total           NUMERIC(10,2),
    discount        NUMERIC(10,2) DEFAULT 0.00,
    coupon_code     VARCHAR(50),
    stripe_id       VARCHAR(255),
    items           JSONB,                                 -- [{course_id, title, amount}]
    completed_at    TIMESTAMPTZ,
    refunded_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_order_user ON proj_order (user_id);
CREATE INDEX idx_proj_order_created ON proj_order (created_at);

-- ============================================================
-- PROJECTION: xAPI Learning Record Store (LRS)
-- ============================================================

CREATE TABLE proj_xapi_statement (
    statement_id    UUID PRIMARY KEY,
    actor_id        UUID NOT NULL,
    actor_email     VARCHAR(320),
    verb_id         VARCHAR(500) NOT NULL,
    verb_display    VARCHAR(100),
    object_id       VARCHAR(500) NOT NULL,
    object_type     VARCHAR(50),
    result_score    JSONB,                                 -- {scaled, raw, min, max}
    result_success  BOOLEAN,
    result_complete BOOLEAN,
    result_duration VARCHAR(50),                           -- ISO 8601 duration
    context         JSONB,
    stored          TIMESTAMPTZ NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_xapi_actor ON proj_xapi_statement (actor_id);
CREATE INDEX idx_proj_xapi_verb ON proj_xapi_statement (verb_id);
CREATE INDEX idx_proj_xapi_stored ON proj_xapi_statement (stored);

-- ============================================================
-- PROJECTION: User Profile (consolidated view)
-- ============================================================

CREATE TABLE proj_user (
    id              UUID PRIMARY KEY,
    email           VARCHAR(320) NOT NULL,
    display_name    VARCHAR(255),
    avatar_url      TEXT,
    roles           TEXT[],                                -- ['learner', 'instructor']
    organisation_ids UUID[],
    is_active       BOOLEAN DEFAULT TRUE,
    enrolled_courses INT DEFAULT 0,
    completed_courses INT DEFAULT 0,
    created_at      TIMESTAMPTZ,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## GDPR Crypto-Shredding Support

```sql
-- Per-user encryption keys for PII fields in events
CREATE TABLE user_encryption_key (
    user_id         UUID PRIMARY KEY,
    encryption_key  BYTEA NOT NULL,                        -- AES-256 key, itself encrypted with a master key
    key_version     INT NOT NULL DEFAULT 1,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- When GDPR erasure is requested:
-- 1. Delete the row from user_encryption_key
-- 2. All events containing that user's PII become unreadable
-- 3. Projections are rebuilt without the deleted user's PII
-- 4. The event stream itself remains intact for audit purposes

-- Erasure request tracking
CREATE TABLE gdpr_erasure_request (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL,
    requested_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ,
    status          VARCHAR(20) NOT NULL DEFAULT 'pending', -- 'pending','processing','completed','failed'
    affected_events INT
);
```

---

## Projection Rebuild Infrastructure

```sql
-- Tracks the position of each projection consumer in the event stream
CREATE TABLE projection_checkpoint (
    projection_name VARCHAR(100) PRIMARY KEY,
    last_event_id   UUID,
    last_position   BIGINT NOT NULL DEFAULT 0,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Example query: rebuild a projection from scratch
-- 1. DELETE FROM proj_enrollment;
-- 2. UPDATE projection_checkpoint SET last_position = 0 WHERE projection_name = 'enrollment';
-- 3. Run projection engine which replays all Enrollment events
```

### Example: Temporal Query (state at a point in time)

```sql
-- "What was the course price on January 1, 2026?"
SELECT payload->>'amount' AS price,
       payload->>'currency' AS currency
FROM event_store
WHERE stream_type = 'Course'
  AND stream_id = '550e8400-...'
  AND event_type IN ('CourseCreated', 'CoursePriceChanged')
  AND created_at <= '2026-01-01T00:00:00Z'
ORDER BY event_version DESC
LIMIT 1;

-- "What lessons had the student completed by March 15?"
SELECT payload->>'lesson_id' AS lesson_id,
       payload->>'completed_at' AS completed_at
FROM event_store
WHERE stream_type = 'Enrollment'
  AND stream_id = '<enrollment_id>'
  AND event_type = 'LessonCompleted'
  AND created_at <= '2026-03-15T00:00:00Z'
ORDER BY event_version;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store Core | 4 | event_store, event_snapshot, event_outbox, event_type_registry |
| Projection: Catalog | 2 | proj_course, proj_curriculum |
| Projection: Learner | 1 | proj_enrollment |
| Projection: Instructor | 1 | proj_instructor_stats |
| Projection: Payments | 1 | proj_order |
| Projection: xAPI/LRS | 1 | proj_xapi_statement |
| Projection: Users | 1 | proj_user |
| GDPR Support | 2 | user_encryption_key, gdpr_erasure_request |
| Infrastructure | 1 | projection_checkpoint |
| **Total** | **~14** | Plus projections added as needed |

---

## Key Design Decisions

1. **Single event_store table** rather than stream-per-table. PostgreSQL partitioning by `created_at` handles storage growth. The `stream_type` + `stream_id` pair identifies the aggregate, and `event_version` ensures ordering within a stream.

2. **JSONB payloads** for event data rather than typed columns. Event schemas evolve over time (new fields are added, old events retain their original structure), and JSONB handles this naturally. The `event_type_registry` table stores JSON Schema definitions for validation.

3. **Outbox pattern** for reliable event publishing. Rather than publishing events directly to a message broker, events are written transactionally to `event_outbox` and a poller/CDC process publishes them. This guarantees at-least-once delivery to projection engines.

4. **Projections are disposable read models.** The `projection_checkpoint` table tracks how far each projection has consumed. Any projection can be dropped and rebuilt by replaying from position 0. This means the read schema can evolve independently of the event schema.

5. **Crypto-shredding for GDPR compliance.** PII in events is encrypted with per-user keys. Deleting the key makes the PII unrecoverable without altering the event stream. This resolves the fundamental tension between event immutability and right-to-erasure.

6. **xAPI statements as events.** The xAPI actor-verb-object model maps directly to the event sourcing pattern. Instead of a separate LRS, xAPI statements are stored as events in the event store and projected into a queryable `proj_xapi_statement` table that implements the xAPI query API.

7. **Denormalized projections** (proj_course includes instructor_name, category_names as arrays). Projections are optimized for specific read patterns, so denormalization is expected and desirable. Updates flow from events, not from direct writes.

8. **Event snapshots** for performance. Long-lived aggregates (a course with thousands of edits) use periodic snapshots to avoid replaying the entire event stream on every command.

9. **Temporal queries by design.** Because every state change is an event with a timestamp, questions like "what was true on date X?" require only filtering the event stream by `created_at`. No bi-temporal columns or SCD patterns needed.

10. **AI/ML pipeline integration.** The event store is the ideal source for analytics pipelines: export events to a data lake, train models on learner behavior sequences, and generate recommendations from engagement patterns -- all without impacting the operational database.
