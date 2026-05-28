# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Online Course Marketplace · Created: 2026-05-19

## Philosophy

The hybrid relational + JSONB model uses strongly typed relational tables for core entities with stable, queryable fields (users, courses, enrollments, orders) while leveraging PostgreSQL JSONB columns for variable, extensible, or domain-specific data (course content metadata, curriculum structures, assessment configurations, SCORM tracking data, jurisdiction-specific tax rules, and per-course custom fields). This is the architecture favored by modern SaaS platforms that need to move fast while maintaining data integrity on the fields that matter most.

This approach recognizes that a course marketplace has two kinds of data: **structural data** (who enrolled in what, how much they paid, what their progress is) that demands relational integrity, and **content data** (lesson configurations, quiz structures, video metadata, SCORM manifest fields, instructor-defined custom properties) that varies widely across courses and evolves rapidly. Forcing the second category into normalized tables produces either massive table counts or wide tables full of NULLable columns. JSONB columns provide schema-on-write flexibility with GIN indexing for performant queries.

Platforms like Teachable, Thinkific, and modern Open edX deployments use variations of this pattern -- relational core with JSON metadata columns that accommodate the diversity of course content types without requiring schema migrations for every new content feature.

**Best for:** Rapid MVP development, multi-region/multi-jurisdiction flexibility, platforms supporting diverse content types (video, SCORM, interactive, coding exercises), and teams that want relational integrity on business-critical fields with flexibility on content metadata.

**Trade-offs:**
- (+) Fewer tables than fully normalized: ~28 tables vs ~43
- (+) Adding new content types, metadata fields, or jurisdiction-specific data requires no DDL changes
- (+) JSONB columns with GIN indexes support fast containment and path queries
- (+) Curriculum structure as JSONB enables flexible nesting (sections within sections, mixed content types)
- (+) Ideal for multi-tenant platforms where different instructors need different metadata fields
- (-) JSONB fields lack database-level constraint enforcement; validation is application-side
- (-) Reporting on JSONB fields requires JSONB path operators, which are less intuitive than column queries
- (-) Schema documentation for JSONB structures must be maintained separately (JSON Schema or application code)
- (-) Migrations of data within JSONB columns are more complex than ALTER TABLE
- (-) Over-reliance on JSONB can lead to "schema-less drift" if not governed

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO 639-1 / BCP 47 | `language_code` column on courses; supported languages as a JSONB array on content |
| ISO 4217 | `currency_code` on pricing; multi-currency pricing stored as JSONB object `{"USD": 49.99, "EUR": 44.99}` |
| ISO 3166-1 alpha-2 | Jurisdiction rules in `pricing.region_pricing` JSONB and `organisation.tax_config` JSONB |
| SCORM 1.2 / 2004 | Full SCORM manifest and CMI tracking data stored as JSONB -- natural fit for the ~80-field CMI data model |
| IEEE 9274.1.1 (xAPI) | xAPI statements stored as JSONB preserving the full actor-verb-object JSON structure |
| IMS LTI 1.3 | LTI tool configuration and launch parameters in JSONB columns |
| IMS QTI 3.0 | Assessment items stored as JSONB with structure following QTI item patterns |
| Open Badges 3.0 | Credential JSON-LD stored directly as JSONB in the credential table |
| PCI DSS v4.0.1 | No card data in JSONB; Stripe reference IDs only |
| WCAG 2.2 | Accessibility metadata per content item in JSONB `accessibility` field |

---

## Users & Organizations

```sql
CREATE TABLE "user" (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(320) NOT NULL UNIQUE,
    email_verified  BOOLEAN NOT NULL DEFAULT FALSE,
    password_hash   VARCHAR(255),
    display_name    VARCHAR(255) NOT NULL,
    avatar_url      TEXT,
    roles           TEXT[] NOT NULL DEFAULT '{learner}',    -- PostgreSQL array: ['learner','instructor','admin']
    profile         JSONB NOT NULL DEFAULT '{}',
    -- profile example: {
    --   "bio": "Machine learning researcher...",
    --   "headline": "Senior Data Scientist at Acme",
    --   "website": "https://example.com",
    --   "social": {"twitter": "@handle", "linkedin": "in/name"},
    --   "locale": "en-US",
    --   "timezone": "America/New_York",
    --   "notification_prefs": {"email_digest": "weekly", "marketing": false}
    -- }
    sso_providers   JSONB NOT NULL DEFAULT '[]',
    -- sso_providers example: [
    --   {"provider": "google", "uid": "1234567890", "connected_at": "2026-01-15T10:00:00Z"},
    --   {"provider": "saml", "uid": "idp:user123", "idp_entity": "https://corp.okta.com"}
    -- ]
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_user_email ON "user" (email);
CREATE INDEX idx_user_roles ON "user" USING GIN (roles);

CREATE TABLE organisation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(255) NOT NULL UNIQUE,
    org_type        VARCHAR(50) NOT NULL,
    country_code    CHAR(2),
    settings        JSONB NOT NULL DEFAULT '{}',
    -- settings example: {
    --   "branding": {"logo_url": "...", "primary_color": "#2563eb", "custom_domain": "learn.acme.com"},
    --   "tax_config": {"tax_id": "DE123456789", "vat_registered": true, "tax_rates": {"DE": 19, "US": 0}},
    --   "payout_config": {"stripe_account_id": "acct_...", "payout_schedule": "monthly", "min_payout": 50},
    --   "lti_config": {"enabled": true, "consumer_key": "...", "shared_secret": "..."}
    -- }
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE organisation_member (
    organisation_id UUID NOT NULL REFERENCES organisation (id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES "user" (id) ON DELETE CASCADE,
    role            VARCHAR(50) NOT NULL DEFAULT 'member',
    permissions     JSONB NOT NULL DEFAULT '[]',            -- fine-grained: ["courses.edit","analytics.view"]
    joined_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (organisation_id, user_id)
);
```

## Course Catalog

```sql
CREATE TABLE category (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    parent_id   UUID REFERENCES category (id),
    name        VARCHAR(255) NOT NULL,
    slug        VARCHAR(255) NOT NULL UNIQUE,
    metadata    JSONB NOT NULL DEFAULT '{}',                -- icon_url, description, featured_courses, etc.
    sort_order  INT NOT NULL DEFAULT 0,
    is_active   BOOLEAN NOT NULL DEFAULT TRUE
);

CREATE TABLE course (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisation (id),
    instructor_id       UUID NOT NULL REFERENCES "user" (id),
    title               VARCHAR(500) NOT NULL,
    slug                VARCHAR(500) NOT NULL,
    subtitle            VARCHAR(500),
    description         TEXT,
    language_code       VARCHAR(10) NOT NULL DEFAULT 'en',
    level               VARCHAR(20) NOT NULL DEFAULT 'beginner',
    status              VARCHAR(30) NOT NULL DEFAULT 'draft',
    -- Flexible content metadata
    media               JSONB NOT NULL DEFAULT '{}',
    -- media example: {
    --   "thumbnail_url": "https://cdn.example.com/thumb.jpg",
    --   "promo_video_url": "https://cdn.example.com/promo.mp4",
    --   "promo_video_duration": 180,
    --   "og_image_url": "https://cdn.example.com/og.jpg"
    -- }
    seo                 JSONB NOT NULL DEFAULT '{}',
    -- seo example: {
    --   "meta_title": "Learn ML from Scratch",
    --   "meta_description": "...",
    --   "canonical_url": "...",
    --   "structured_data": { "@type": "Course", ... }
    -- }
    requirements        TEXT[],                             -- prerequisite descriptions
    learning_outcomes   TEXT[],                             -- what students will learn
    target_audience     TEXT[],                             -- who this course is for
    skill_tags          TEXT[],                             -- searchable skill tags
    category_ids        UUID[],                             -- denormalized for query performance
    -- Curriculum stored as structured JSONB
    curriculum          JSONB NOT NULL DEFAULT '[]',
    -- curriculum example: [
    --   {
    --     "section_id": "uuid",
    --     "title": "Getting Started",
    --     "lessons": [
    --       {
    --         "lesson_id": "uuid",
    --         "title": "Welcome & Overview",
    --         "type": "video",
    --         "duration_secs": 420,
    --         "is_preview": true,
    --         "is_mandatory": true,
    --         "content_ref": "uuid"
    --       },
    --       {
    --         "lesson_id": "uuid",
    --         "title": "Setup Your Environment",
    --         "type": "text",
    --         "is_preview": false,
    --         "content_ref": "uuid"
    --       },
    --       {
    --         "lesson_id": "uuid",
    --         "title": "First Quiz",
    --         "type": "quiz",
    --         "content_ref": "uuid"
    --       }
    --     ]
    --   }
    -- ]
    -- Aggregates
    stats               JSONB NOT NULL DEFAULT '{}',
    -- stats example: {
    --   "enrollment_count": 4523,
    --   "completion_count": 1847,
    --   "avg_rating": 4.62,
    --   "rating_count": 312,
    --   "total_video_secs": 43200,
    --   "lesson_count": 48
    -- }
    published_at        TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, slug)
);

CREATE INDEX idx_course_instructor ON course (instructor_id);
CREATE INDEX idx_course_status ON course (status) WHERE status = 'published';
CREATE INDEX idx_course_categories ON course USING GIN (category_ids);
CREATE INDEX idx_course_skills ON course USING GIN (skill_tags);
CREATE INDEX idx_course_search ON course USING GIN (
    to_tsvector('english', coalesce(title, '') || ' ' || coalesce(subtitle, '') || ' ' || coalesce(description, ''))
);
-- Query curriculum JSONB for lesson counts, types, etc.
CREATE INDEX idx_course_curriculum ON course USING GIN (curriculum jsonb_path_ops);
```

## Content Items (lesson bodies, stored separately from curriculum structure)

```sql
CREATE TABLE content_item (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    course_id       UUID NOT NULL REFERENCES course (id) ON DELETE CASCADE,
    content_type    VARCHAR(30) NOT NULL,                   -- 'video','text','pdf','quiz','coding','scorm','lti'
    title           VARCHAR(500),
    content         JSONB NOT NULL,
    -- content structure varies by type:
    --
    -- VIDEO:
    -- {
    --   "video_url": "https://cdn.example.com/videos/lesson1.mp4",
    --   "hls_url": "https://cdn.example.com/videos/lesson1/master.m3u8",
    --   "duration_secs": 847,
    --   "transcript_url": "https://cdn.example.com/transcripts/lesson1.vtt",
    --   "captions": [
    --     {"language": "en", "url": "https://cdn.example.com/captions/lesson1-en.vtt"},
    --     {"language": "es", "url": "https://cdn.example.com/captions/lesson1-es.vtt"}
    --   ],
    --   "encoding_status": "ready",
    --   "chapters": [{"title": "Introduction", "start_secs": 0}, {"title": "Demo", "start_secs": 120}]
    -- }
    --
    -- TEXT:
    -- {
    --   "body_html": "<h2>Getting Started</h2><p>In this lesson...</p>",
    --   "attachments": [
    --     {"name": "slides.pdf", "url": "...", "size_bytes": 2048000, "mime_type": "application/pdf"}
    --   ]
    -- }
    --
    -- QUIZ (QTI-aligned):
    -- {
    --   "passing_score": 70,
    --   "time_limit_secs": 1800,
    --   "max_attempts": 3,
    --   "shuffle_items": true,
    --   "items": [
    --     {
    --       "item_id": "uuid",
    --       "type": "multiple_choice",
    --       "question_html": "<p>What is supervised learning?</p>",
    --       "options": [
    --         {"id": "a", "html": "Learning with labeled data", "is_correct": true},
    --         {"id": "b", "html": "Learning without labels", "is_correct": false}
    --       ],
    --       "explanation_html": "<p>Supervised learning uses labeled training data...</p>",
    --       "points": 1
    --     }
    --   ]
    -- }
    --
    -- SCORM:
    -- {
    --   "scorm_version": "2004_4th",
    --   "package_url": "https://cdn.example.com/scorm/pkg123.zip",
    --   "launch_url": "index.html",
    --   "manifest": { ... parsed imsmanifest.xml ... }
    -- }
    --
    -- CODING:
    -- {
    --   "language": "python",
    --   "starter_code": "def solution(nums):\n    pass",
    --   "test_cases": [...],
    --   "solution_code": "def solution(nums):\n    return sorted(nums)",
    --   "sandbox_image": "python:3.11-slim"
    -- }
    accessibility   JSONB NOT NULL DEFAULT '{}',
    -- accessibility example: {
    --   "wcag_level": "AA",
    --   "has_captions": true,
    --   "has_transcript": true,
    --   "has_alt_text": true,
    --   "color_contrast_checked": true
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_content_course ON content_item (course_id);
CREATE INDEX idx_content_type ON content_item (content_type);
CREATE INDEX idx_content_json ON content_item USING GIN (content jsonb_path_ops);
```

## Enrollment & Progress

```sql
CREATE TABLE enrollment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES "user" (id) ON DELETE CASCADE,
    course_id       UUID NOT NULL REFERENCES course (id) ON DELETE CASCADE,
    status          VARCHAR(30) NOT NULL DEFAULT 'active',
    enrolled_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ,
    progress_pct    NUMERIC(5,2) NOT NULL DEFAULT 0.00,
    progress_data   JSONB NOT NULL DEFAULT '{}',
    -- progress_data example: {
    --   "lessons_completed": ["uuid1", "uuid2", "uuid3"],
    --   "lessons_total": 48,
    --   "current_lesson_id": "uuid4",
    --   "current_video_position": 245,
    --   "time_spent_secs": 18400,
    --   "last_accessed": "2026-05-18T14:32:00Z",
    --   "bookmarks": [
    --     {"lesson_id": "uuid1", "position": 120, "note": "Important concept"},
    --     {"lesson_id": "uuid2", "position": 340, "note": "Review this"}
    --   ],
    --   "notes": [
    --     {"lesson_id": "uuid1", "text": "Key insight about gradient descent", "created_at": "..."}
    --   ]
    -- }
    scorm_tracking  JSONB,
    -- scorm_tracking example (per-SCO CMI data):
    -- {
    --   "sco-id-1": {
    --     "cmi.completion_status": "completed",
    --     "cmi.success_status": "passed",
    --     "cmi.score.raw": 85,
    --     "cmi.score.max": 100,
    --     "cmi.total_time": "PT1H23M45S",
    --     "cmi.suspend_data": "base64..."
    --   }
    -- }
    UNIQUE (user_id, course_id)
);

CREATE INDEX idx_enrollment_user ON enrollment (user_id);
CREATE INDEX idx_enrollment_course ON enrollment (course_id);
CREATE INDEX idx_enrollment_status ON enrollment (status);

CREATE TABLE assessment_attempt (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    enrollment_id   UUID NOT NULL REFERENCES enrollment (id) ON DELETE CASCADE,
    content_item_id UUID NOT NULL REFERENCES content_item (id),
    attempt_number  INT NOT NULL DEFAULT 1,
    score           NUMERIC(5,2),
    passed          BOOLEAN,
    responses       JSONB NOT NULL DEFAULT '[]',
    -- responses example: [
    --   {"item_id": "uuid", "selected": ["a"], "is_correct": true, "points": 1},
    --   {"item_id": "uuid", "answer_text": "...", "points": 0, "graded_by": "instructor"}
    -- ]
    started_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    submitted_at    TIMESTAMPTZ,
    UNIQUE (enrollment_id, content_item_id, attempt_number)
);
```

## Pricing & Payments

```sql
CREATE TABLE pricing (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    course_id       UUID NOT NULL REFERENCES course (id) ON DELETE CASCADE,
    plan_type       VARCHAR(20) NOT NULL,                   -- 'one_time','subscription','free'
    base_price      NUMERIC(10,2) NOT NULL,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    interval_months INT,
    region_pricing  JSONB NOT NULL DEFAULT '{}',
    -- region_pricing example (purchasing power parity):
    -- {
    --   "IN": {"amount": 499, "currency": "INR"},
    --   "BR": {"amount": 79.90, "currency": "BRL"},
    --   "GB": {"amount": 39.99, "currency": "GBP"}
    -- }
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE coupon (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(50) NOT NULL UNIQUE,
    discount_type   VARCHAR(10) NOT NULL,
    discount_value  NUMERIC(10,2) NOT NULL,
    config          JSONB NOT NULL DEFAULT '{}',
    -- config example: {
    --   "currency_code": "USD",
    --   "max_uses": 100,
    --   "max_uses_per_user": 1,
    --   "applicable_course_ids": ["uuid1", "uuid2"],
    --   "applicable_category_ids": ["uuid3"],
    --   "min_purchase_amount": 20.00,
    --   "valid_from": "2026-05-01T00:00:00Z",
    --   "valid_until": "2026-06-01T00:00:00Z"
    -- }
    uses_count      INT NOT NULL DEFAULT 0,
    created_by      UUID NOT NULL REFERENCES "user" (id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE "order" (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id             UUID NOT NULL REFERENCES "user" (id),
    status              VARCHAR(20) NOT NULL DEFAULT 'pending',
    currency_code       CHAR(3) NOT NULL,
    subtotal            NUMERIC(10,2) NOT NULL,
    discount_amount     NUMERIC(10,2) NOT NULL DEFAULT 0.00,
    tax_amount          NUMERIC(10,2) NOT NULL DEFAULT 0.00,
    total               NUMERIC(10,2) NOT NULL,
    items               JSONB NOT NULL DEFAULT '[]',
    -- items example: [
    --   {"course_id": "uuid", "pricing_id": "uuid", "title": "Intro to ML", "amount": 49.99}
    -- ]
    payment             JSONB NOT NULL DEFAULT '{}',
    -- payment example: {
    --   "stripe_payment_intent_id": "pi_3abc...",
    --   "stripe_checkout_session_id": "cs_...",
    --   "payment_method": "card",
    --   "card_last4": "4242",
    --   "card_brand": "visa"
    -- }
    coupon_code         VARCHAR(50),
    affiliate_data      JSONB,
    -- affiliate_data example: {
    --   "affiliate_id": "uuid",
    --   "referral_code": "TEACH20",
    --   "commission_pct": 10,
    --   "commission_amount": 4.99
    -- }
    completed_at        TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_order_user ON "order" (user_id);
CREATE INDEX idx_order_status ON "order" (status);
CREATE INDEX idx_order_created ON "order" (created_at);

CREATE TABLE instructor_payout (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    instructor_id       UUID NOT NULL REFERENCES "user" (id),
    organisation_id     UUID NOT NULL REFERENCES organisation (id),
    period_start        DATE NOT NULL,
    period_end          DATE NOT NULL,
    summary             JSONB NOT NULL,
    -- summary example: {
    --   "gross_revenue": 12500.00,
    --   "platform_fee": 1875.00,
    --   "affiliate_fees": 625.00,
    --   "refunds": 249.99,
    --   "net_payout": 9750.01,
    --   "currency": "USD",
    --   "order_count": 250,
    --   "refund_count": 5,
    --   "by_course": [
    --     {"course_id": "uuid", "title": "...", "revenue": 8000, "enrollments": 160}
    --   ]
    -- }
    stripe_transfer_id  VARCHAR(255),
    status              VARCHAR(20) NOT NULL DEFAULT 'pending',
    paid_at             TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Reviews & Discussion

```sql
CREATE TABLE review (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    course_id   UUID NOT NULL REFERENCES course (id) ON DELETE CASCADE,
    user_id     UUID NOT NULL REFERENCES "user" (id),
    rating      SMALLINT NOT NULL CHECK (rating BETWEEN 1 AND 5),
    title       VARCHAR(500),
    body        TEXT,
    metadata    JSONB NOT NULL DEFAULT '{}',
    -- metadata example: {
    --   "is_verified_purchase": true,
    --   "completion_pct_at_review": 85.5,
    --   "helpful_votes": 12,
    --   "reported": false,
    --   "instructor_response": "Thank you for your feedback..."
    -- }
    is_visible  BOOLEAN NOT NULL DEFAULT TRUE,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (course_id, user_id)
);

CREATE INDEX idx_review_course ON review (course_id);

CREATE TABLE discussion (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    course_id   UUID NOT NULL REFERENCES course (id) ON DELETE CASCADE,
    parent_id   UUID REFERENCES discussion (id),           -- NULL for top-level threads, parent for replies
    user_id     UUID NOT NULL REFERENCES "user" (id),
    lesson_id   UUID,                                      -- optional: scoped to a lesson in curriculum JSONB
    title       VARCHAR(500),                              -- only for top-level threads
    body        TEXT NOT NULL,
    metadata    JSONB NOT NULL DEFAULT '{}',
    -- metadata example: {
    --   "is_pinned": false,
    --   "is_resolved": true,
    --   "is_answer": false,
    --   "upvotes": 5,
    --   "edited_at": "2026-05-18T10:00:00Z"
    -- }
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_discussion_course ON discussion (course_id);
CREATE INDEX idx_discussion_parent ON discussion (parent_id);
CREATE INDEX idx_discussion_lesson ON discussion (lesson_id) WHERE lesson_id IS NOT NULL;
```

## Credentials & xAPI

```sql
CREATE TABLE credential (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    course_id           UUID NOT NULL REFERENCES course (id),
    user_id             UUID NOT NULL REFERENCES "user" (id),
    enrollment_id       UUID NOT NULL REFERENCES enrollment (id),
    credential_type     VARCHAR(30) NOT NULL,               -- 'completion','competency','badge'
    verification_url    TEXT NOT NULL UNIQUE,
    credential_data     JSONB NOT NULL,
    -- credential_data example (Open Badges 3.0 / W3C VC):
    -- {
    --   "@context": ["https://www.w3.org/ns/credentials/v2", "https://purl.imsglobal.org/spec/ob/v3p0/context-3.0.3.json"],
    --   "type": ["VerifiableCredential", "OpenBadgeCredential"],
    --   "issuer": {"id": "https://learn.example.com", "name": "Example Academy"},
    --   "validFrom": "2026-05-19T00:00:00Z",
    --   "credentialSubject": {
    --     "type": "AchievementSubject",
    --     "achievement": {
    --       "name": "Introduction to Machine Learning",
    --       "criteria": {"narrative": "Completed all lessons and passed final assessment with 80%+"}
    --     }
    --   }
    -- }
    template_html       TEXT,                              -- HTML template for PDF certificate rendering
    badge_image_url     TEXT,
    issued_at           TIMESTAMPTZ NOT NULL DEFAULT now(),
    revoked_at          TIMESTAMPTZ,
    revocation_reason   TEXT
);

CREATE INDEX idx_credential_user ON credential (user_id);
CREATE INDEX idx_credential_course ON credential (course_id);

CREATE TABLE xapi_statement (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    statement_id    UUID NOT NULL UNIQUE,
    actor_id        UUID NOT NULL REFERENCES "user" (id),
    verb_iri        VARCHAR(500) NOT NULL,
    object_iri      VARCHAR(500) NOT NULL,
    statement       JSONB NOT NULL,                        -- full xAPI statement JSON
    stored_at       TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_xapi_actor ON xapi_statement (actor_id);
CREATE INDEX idx_xapi_verb ON xapi_statement (verb_iri);
CREATE INDEX idx_xapi_stored ON xapi_statement (stored_at);
CREATE INDEX idx_xapi_statement ON xapi_statement USING GIN (statement jsonb_path_ops);
```

## LTI & Platform Infrastructure

```sql
CREATE TABLE lti_tool (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    config          JSONB NOT NULL,
    -- config example: {
    --   "client_id": "tool-client-123",
    --   "deployment_id": "deploy-456",
    --   "issuer": "https://tool.example.com",
    --   "auth_login_url": "https://tool.example.com/auth/login",
    --   "auth_token_url": "https://tool.example.com/auth/token",
    --   "jwks_url": "https://tool.example.com/.well-known/jwks.json",
    --   "custom_params": {"role": "student"}
    -- }
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE notification (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES "user" (id) ON DELETE CASCADE,
    notification_type VARCHAR(50) NOT NULL,
    title           VARCHAR(500) NOT NULL,
    body            TEXT,
    data            JSONB NOT NULL DEFAULT '{}',           -- action_url, entity references, etc.
    is_read         BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_notification_user ON notification (user_id, is_read);

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID,
    action          VARCHAR(100) NOT NULL,
    entity_type     VARCHAR(100) NOT NULL,
    entity_id       UUID NOT NULL,
    changes         JSONB,                                 -- {field: {old: x, new: y}}
    context         JSONB,                                 -- ip_address, user_agent, request_id
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_entity ON audit_log (entity_type, entity_id);
CREATE INDEX idx_audit_created ON audit_log (created_at);
```

---

## Example JSONB Queries

```sql
-- Find all courses with video content longer than 10 hours
SELECT id, title
FROM course
WHERE (stats->>'total_video_secs')::int > 36000;

-- Find courses with regional pricing for India
SELECT id, title, pricing.region_pricing->'IN' AS india_price
FROM course
JOIN pricing ON pricing.course_id = course.id
WHERE pricing.region_pricing ? 'IN';

-- Get all lessons of type 'quiz' from a course curriculum
SELECT lesson->>'lesson_id' AS lesson_id,
       lesson->>'title' AS title
FROM course,
     jsonb_array_elements(curriculum) AS section,
     jsonb_array_elements(section->'lessons') AS lesson
WHERE course.id = '<course_id>'
  AND lesson->>'type' = 'quiz';

-- Search xAPI statements for completion verbs
SELECT statement_id, statement->'actor'->'mbox' AS actor,
       statement->'object'->'id' AS activity
FROM xapi_statement
WHERE verb_iri = 'http://adlnet.gov/expapi/verbs/completed'
  AND stored_at >= '2026-05-01';
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Users & Auth | 3 | user, organisation, organisation_member |
| Course Catalog | 2 | category, course |
| Content | 1 | content_item (all types via JSONB) |
| Enrollment & Progress | 2 | enrollment, assessment_attempt |
| Payments & Commerce | 4 | pricing, coupon, order, instructor_payout |
| Reviews & Discussion | 2 | review, discussion (self-referencing for replies) |
| Credentials | 1 | credential |
| xAPI | 1 | xapi_statement |
| LTI | 1 | lti_tool |
| Platform | 2 | notification, audit_log |
| **Total** | **~19** | Significantly fewer than normalized (~43 tables) |

---

## Key Design Decisions

1. **Curriculum structure as JSONB on the course table** rather than separate section/lesson tables. The curriculum is always loaded as a unit (never queried for individual lessons across courses), and its structure varies by course type. This eliminates 3-4 tables and simplifies the course authoring API.

2. **Single content_item table** with a `content_type` discriminator and JSONB `content` column. Video metadata, quiz items, SCORM manifest data, and text bodies all live in the same table with type-specific JSON structures. Adding a new content type (e.g., interactive simulations) requires no schema change.

3. **Roles as PostgreSQL array** on the user table rather than a junction table. For a system with 4-5 roles, an array column with a GIN index is simpler and equally performant.

4. **Region-specific pricing as JSONB** on the pricing table. This naturally handles the purchasing-power-parity pricing that platforms like Udemy use (different prices per country) without a separate table per region or a massive junction table.

5. **Order items as JSONB array** within the order. Since orders are immutable after completion and always loaded as a unit, embedding items as JSONB avoids a join table while preserving the data needed for display and reporting.

6. **Self-referencing discussion table** where `parent_id` is NULL for threads and references the parent for replies. This replaces separate thread and reply tables with a single table that supports arbitrary nesting.

7. **SCORM CMI tracking data as JSONB** on the enrollment row. The SCORM CMI data model has ~80 fields that vary by SCORM version, and per-SCO tracking is naturally nested. JSONB is the idiomatic choice.

8. **Full xAPI statement stored as JSONB** alongside extracted relational fields (actor_id, verb_iri) for indexed queries. This preserves the complete xAPI JSON for spec-compliant LRS responses while enabling efficient filtering.

9. **Organisation settings as a single JSONB column** covering branding, tax configuration, payout configuration, and LTI settings. These are always loaded together and vary significantly between organisation types.

10. **Credential data as JSONB** containing the full Open Badges 3.0 / W3C Verifiable Credentials JSON-LD. The JSON-LD structure is needed for external verification and standards compliance, while relational fields (user_id, course_id, issued_at) enable internal queries.
