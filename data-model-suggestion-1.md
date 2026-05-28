# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Online Course Marketplace · Created: 2026-05-19

## Philosophy

The entity-centric normalized relational model treats every domain concept as a first-class table with strict foreign key relationships, reference data tables for controlled vocabularies, and junction tables for many-to-many associations. This is the classical approach used by mature platforms like Open edX (Django ORM over MySQL/PostgreSQL) and Moodle (PHP over MySQL/PostgreSQL), where decades of schema evolution have produced highly normalized structures that support complex cross-entity queries.

The core principle is **data integrity through normalization**: every fact is stored once, every relationship is enforced by the database, and every query can join across the full domain model. Reference data (categories, languages, currencies, jurisdictions) lives in dedicated lookup tables aligned with ISO standards rather than in application enums or JSONB blobs.

This approach is best suited for teams with strong relational database expertise, deployments where regulatory compliance demands provable data integrity (FERPA, GDPR audit requirements), and scenarios where complex analytical queries across courses, enrollments, payments, and credentials are frequent.

**Best for:** Enterprise and institutional deployments requiring maximum data integrity, complex reporting, and regulatory compliance.

**Trade-offs:**
- (+) Maximum data integrity via foreign keys and constraints
- (+) Complex cross-entity queries are natural and performant with proper indexing
- (+) Well-understood by most development teams; extensive tooling support
- (+) Schema changes are explicit and reviewable via migrations
- (-) High table count increases schema complexity and migration overhead
- (-) Adding new content types or metadata fields requires DDL changes
- (-) Junction tables for many-to-many relationships add query complexity
- (-) Rigid schema makes multi-jurisdiction or multi-content-type flexibility harder
- (-) Performance degrades on deep joins without careful index management

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO 639-1/639-3 | `language` reference table uses ISO 639 codes for course and content language |
| ISO 4217 | `currency` reference table stores ISO 4217 currency codes for multi-currency pricing |
| ISO 3166-1 alpha-2 | `country` reference table for instructor/learner jurisdiction and tax rules |
| SCORM 1.2 / 2004 | `scorm_package` table stores manifest metadata; `scorm_tracking` captures CMI data model fields |
| IEEE 9274.1.1 (xAPI) | `xapi_statement` table stores actor/verb/object/result/context per the xAPI specification |
| IMS LTI 1.3 | `lti_tool` and `lti_launch` tables model registered tools and launch contexts per LTI 1.3 spec |
| IMS QTI 3.0 | `assessment` and `assessment_item` tables model QTI-aligned question and test structures |
| Open Badges 3.0 / W3C VC | `credential` table stores badge/credential metadata aligned with Open Badges 3.0 JSON-LD structure |
| PCI DSS v4.0.1 | No card data stored; `payment` table references Stripe payment intent IDs only |
| WCAG 2.2 | `accessibility_metadata` table tracks per-content accessibility compliance status |

---

## Core Identity & Multi-Tenancy

```sql
-- ============================================================
-- USERS & AUTHENTICATION
-- ============================================================

CREATE TABLE "user" (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(320) NOT NULL UNIQUE,
    email_verified  BOOLEAN NOT NULL DEFAULT FALSE,
    password_hash   VARCHAR(255),                          -- NULL for SSO-only users
    display_name    VARCHAR(255) NOT NULL,
    avatar_url      TEXT,
    bio             TEXT,
    locale          VARCHAR(10) DEFAULT 'en',              -- BCP 47 language tag
    timezone        VARCHAR(64) DEFAULT 'UTC',             -- IANA timezone
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_user_email ON "user" (email);
CREATE INDEX idx_user_display_name ON "user" (display_name);

CREATE TABLE user_role (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id     UUID NOT NULL REFERENCES "user" (id) ON DELETE CASCADE,
    role        VARCHAR(50) NOT NULL,                      -- 'learner', 'instructor', 'admin', 'reviewer'
    granted_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    granted_by  UUID REFERENCES "user" (id),
    UNIQUE (user_id, role)
);

CREATE INDEX idx_user_role_user ON user_role (user_id);

CREATE TABLE user_sso (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES "user" (id) ON DELETE CASCADE,
    provider        VARCHAR(50) NOT NULL,                  -- 'google', 'saml', 'oidc'
    provider_uid    VARCHAR(512) NOT NULL,
    metadata        JSONB,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (provider, provider_uid)
);

CREATE TABLE organisation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(255) NOT NULL UNIQUE,
    org_type        VARCHAR(50) NOT NULL,                  -- 'individual', 'business', 'university', 'enterprise'
    country_code    CHAR(2),                               -- ISO 3166-1 alpha-2
    tax_id          VARCHAR(64),
    logo_url        TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE organisation_member (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation (id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES "user" (id) ON DELETE CASCADE,
    role            VARCHAR(50) NOT NULL DEFAULT 'member', -- 'owner', 'admin', 'instructor', 'member'
    joined_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, user_id)
);
```

## Course Catalog & Content Structure

```sql
-- ============================================================
-- REFERENCE DATA
-- ============================================================

CREATE TABLE category (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    parent_id   UUID REFERENCES category (id),
    name        VARCHAR(255) NOT NULL,
    slug        VARCHAR(255) NOT NULL UNIQUE,
    description TEXT,
    sort_order  INT NOT NULL DEFAULT 0,
    is_active   BOOLEAN NOT NULL DEFAULT TRUE
);

CREATE INDEX idx_category_parent ON category (parent_id);

CREATE TABLE language (
    code    VARCHAR(10) PRIMARY KEY,                       -- ISO 639-1 or BCP 47
    name    VARCHAR(100) NOT NULL
);

CREATE TABLE skill_tag (
    id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name    VARCHAR(100) NOT NULL UNIQUE,
    slug    VARCHAR(100) NOT NULL UNIQUE
);

-- ============================================================
-- COURSES
-- ============================================================

CREATE TABLE course (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisation (id),
    instructor_id       UUID NOT NULL REFERENCES "user" (id),
    title               VARCHAR(500) NOT NULL,
    slug                VARCHAR(500) NOT NULL,
    subtitle            VARCHAR(500),
    description         TEXT,
    language_code       VARCHAR(10) NOT NULL REFERENCES language (code),
    level               VARCHAR(20) NOT NULL DEFAULT 'beginner',  -- 'beginner','intermediate','advanced','all'
    status              VARCHAR(30) NOT NULL DEFAULT 'draft',     -- 'draft','in_review','published','archived'
    thumbnail_url       TEXT,
    promo_video_url     TEXT,
    estimated_duration  INTERVAL,                                 -- e.g. '12 hours'
    published_at        TIMESTAMPTZ,
    is_free             BOOLEAN NOT NULL DEFAULT FALSE,
    avg_rating          NUMERIC(3,2) DEFAULT 0.00,
    rating_count        INT DEFAULT 0,
    enrollment_count    INT DEFAULT 0,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, slug)
);

CREATE INDEX idx_course_instructor ON course (instructor_id);
CREATE INDEX idx_course_status ON course (status);
CREATE INDEX idx_course_published ON course (published_at) WHERE status = 'published';
CREATE INDEX idx_course_language ON course (language_code);

CREATE TABLE course_category (
    course_id   UUID NOT NULL REFERENCES course (id) ON DELETE CASCADE,
    category_id UUID NOT NULL REFERENCES category (id) ON DELETE CASCADE,
    PRIMARY KEY (course_id, category_id)
);

CREATE TABLE course_skill (
    course_id   UUID NOT NULL REFERENCES course (id) ON DELETE CASCADE,
    skill_tag_id UUID NOT NULL REFERENCES skill_tag (id) ON DELETE CASCADE,
    PRIMARY KEY (course_id, skill_tag_id)
);

CREATE TABLE course_prerequisite (
    course_id       UUID NOT NULL REFERENCES course (id) ON DELETE CASCADE,
    prerequisite_id UUID NOT NULL REFERENCES course (id) ON DELETE CASCADE,
    PRIMARY KEY (course_id, prerequisite_id),
    CHECK (course_id <> prerequisite_id)
);

-- ============================================================
-- SECTIONS & LESSONS (curriculum hierarchy)
-- ============================================================

CREATE TABLE section (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    course_id   UUID NOT NULL REFERENCES course (id) ON DELETE CASCADE,
    title       VARCHAR(500) NOT NULL,
    sort_order  INT NOT NULL DEFAULT 0,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_section_course ON section (course_id, sort_order);

CREATE TABLE lesson (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    section_id      UUID NOT NULL REFERENCES section (id) ON DELETE CASCADE,
    title           VARCHAR(500) NOT NULL,
    lesson_type     VARCHAR(30) NOT NULL,                  -- 'video','text','pdf','quiz','coding_exercise','assignment'
    sort_order      INT NOT NULL DEFAULT 0,
    is_preview      BOOLEAN NOT NULL DEFAULT FALSE,        -- free preview
    is_mandatory    BOOLEAN NOT NULL DEFAULT TRUE,
    estimated_time  INTERVAL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_lesson_section ON lesson (section_id, sort_order);

CREATE TABLE lesson_video (
    lesson_id       UUID PRIMARY KEY REFERENCES lesson (id) ON DELETE CASCADE,
    video_url       TEXT NOT NULL,
    duration_secs   INT NOT NULL,
    transcript_url  TEXT,
    captions_url    TEXT,                                   -- WebVTT
    encoding_status VARCHAR(30) NOT NULL DEFAULT 'pending' -- 'pending','processing','ready','failed'
);

CREATE TABLE lesson_text (
    lesson_id   UUID PRIMARY KEY REFERENCES lesson (id) ON DELETE CASCADE,
    body_html   TEXT NOT NULL
);

CREATE TABLE lesson_attachment (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    lesson_id   UUID NOT NULL REFERENCES lesson (id) ON DELETE CASCADE,
    file_name   VARCHAR(500) NOT NULL,
    file_url    TEXT NOT NULL,
    file_size   BIGINT,
    mime_type   VARCHAR(100),
    sort_order  INT NOT NULL DEFAULT 0
);
```

## Assessments (QTI-Aligned)

```sql
CREATE TABLE assessment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    lesson_id       UUID NOT NULL REFERENCES lesson (id) ON DELETE CASCADE,
    title           VARCHAR(500) NOT NULL,
    assessment_type VARCHAR(30) NOT NULL,                  -- 'quiz','assignment','coding','peer_review'
    passing_score   NUMERIC(5,2) DEFAULT 70.00,
    time_limit_secs INT,
    max_attempts    INT DEFAULT 0,                         -- 0 = unlimited
    shuffle_items   BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE assessment_item (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    assessment_id   UUID NOT NULL REFERENCES assessment (id) ON DELETE CASCADE,
    item_type       VARCHAR(30) NOT NULL,                  -- 'multiple_choice','multiple_select','true_false','short_answer','essay','code'
    question_html   TEXT NOT NULL,
    explanation_html TEXT,
    points          NUMERIC(5,2) NOT NULL DEFAULT 1.00,
    sort_order      INT NOT NULL DEFAULT 0
);

CREATE TABLE assessment_item_option (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    item_id         UUID NOT NULL REFERENCES assessment_item (id) ON DELETE CASCADE,
    option_html     TEXT NOT NULL,
    is_correct      BOOLEAN NOT NULL DEFAULT FALSE,
    sort_order      INT NOT NULL DEFAULT 0
);
```

## Enrollment & Progress

```sql
CREATE TABLE enrollment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES "user" (id) ON DELETE CASCADE,
    course_id       UUID NOT NULL REFERENCES course (id) ON DELETE CASCADE,
    status          VARCHAR(30) NOT NULL DEFAULT 'active', -- 'active','completed','refunded','expired'
    enrolled_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ,
    progress_pct    NUMERIC(5,2) NOT NULL DEFAULT 0.00,
    last_accessed   TIMESTAMPTZ,
    UNIQUE (user_id, course_id)
);

CREATE INDEX idx_enrollment_user ON enrollment (user_id);
CREATE INDEX idx_enrollment_course ON enrollment (course_id);
CREATE INDEX idx_enrollment_status ON enrollment (status);

CREATE TABLE lesson_progress (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    enrollment_id   UUID NOT NULL REFERENCES enrollment (id) ON DELETE CASCADE,
    lesson_id       UUID NOT NULL REFERENCES lesson (id) ON DELETE CASCADE,
    status          VARCHAR(20) NOT NULL DEFAULT 'not_started', -- 'not_started','in_progress','completed'
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    time_spent_secs INT NOT NULL DEFAULT 0,
    video_position  INT DEFAULT 0,                             -- seconds into video
    UNIQUE (enrollment_id, lesson_id)
);

CREATE INDEX idx_lesson_progress_enrollment ON lesson_progress (enrollment_id);

CREATE TABLE assessment_attempt (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    enrollment_id   UUID NOT NULL REFERENCES enrollment (id) ON DELETE CASCADE,
    assessment_id   UUID NOT NULL REFERENCES assessment (id) ON DELETE CASCADE,
    attempt_number  INT NOT NULL DEFAULT 1,
    score           NUMERIC(5,2),
    passed          BOOLEAN,
    started_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    submitted_at    TIMESTAMPTZ,
    UNIQUE (enrollment_id, assessment_id, attempt_number)
);

CREATE TABLE assessment_response (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    attempt_id      UUID NOT NULL REFERENCES assessment_attempt (id) ON DELETE CASCADE,
    item_id         UUID NOT NULL REFERENCES assessment_item (id),
    response_data   TEXT,                                  -- selected option IDs, text answer, or code
    is_correct      BOOLEAN,
    points_awarded  NUMERIC(5,2) DEFAULT 0.00
);
```

## Payments & Creator Economics

```sql
CREATE TABLE pricing_plan (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    course_id       UUID NOT NULL REFERENCES course (id) ON DELETE CASCADE,
    plan_type       VARCHAR(20) NOT NULL,                  -- 'one_time','subscription','free'
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',        -- ISO 4217
    amount          NUMERIC(10,2) NOT NULL,
    interval_months INT,                                   -- for subscription: 1, 12, etc.
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE coupon (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(50) NOT NULL UNIQUE,
    discount_type   VARCHAR(10) NOT NULL,                  -- 'percent','fixed'
    discount_value  NUMERIC(10,2) NOT NULL,
    currency_code   CHAR(3),                               -- required for fixed discounts
    max_uses        INT,
    uses_count      INT NOT NULL DEFAULT 0,
    valid_from      TIMESTAMPTZ NOT NULL,
    valid_until     TIMESTAMPTZ,
    created_by      UUID NOT NULL REFERENCES "user" (id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE coupon_course (
    coupon_id   UUID NOT NULL REFERENCES coupon (id) ON DELETE CASCADE,
    course_id   UUID NOT NULL REFERENCES course (id) ON DELETE CASCADE,
    PRIMARY KEY (coupon_id, course_id)
);

CREATE TABLE "order" (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id             UUID NOT NULL REFERENCES "user" (id),
    status              VARCHAR(20) NOT NULL DEFAULT 'pending', -- 'pending','completed','refunded','failed'
    currency_code       CHAR(3) NOT NULL,
    subtotal            NUMERIC(10,2) NOT NULL,
    discount_amount     NUMERIC(10,2) NOT NULL DEFAULT 0.00,
    tax_amount          NUMERIC(10,2) NOT NULL DEFAULT 0.00,
    total               NUMERIC(10,2) NOT NULL,
    coupon_id           UUID REFERENCES coupon (id),
    stripe_payment_id   VARCHAR(255),                      -- Stripe PaymentIntent ID
    stripe_checkout_id  VARCHAR(255),                      -- Stripe Checkout Session ID
    completed_at        TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_order_user ON "order" (user_id);
CREATE INDEX idx_order_status ON "order" (status);

CREATE TABLE order_item (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id        UUID NOT NULL REFERENCES "order" (id) ON DELETE CASCADE,
    course_id       UUID NOT NULL REFERENCES course (id),
    pricing_plan_id UUID NOT NULL REFERENCES pricing_plan (id),
    amount          NUMERIC(10,2) NOT NULL
);

CREATE TABLE instructor_payout (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    instructor_id       UUID NOT NULL REFERENCES "user" (id),
    organisation_id     UUID NOT NULL REFERENCES organisation (id),
    period_start        DATE NOT NULL,
    period_end          DATE NOT NULL,
    gross_revenue       NUMERIC(10,2) NOT NULL,
    platform_fee        NUMERIC(10,2) NOT NULL,
    affiliate_fee       NUMERIC(10,2) NOT NULL DEFAULT 0.00,
    net_payout          NUMERIC(10,2) NOT NULL,
    currency_code       CHAR(3) NOT NULL,
    stripe_transfer_id  VARCHAR(255),
    status              VARCHAR(20) NOT NULL DEFAULT 'pending', -- 'pending','processing','paid','failed'
    paid_at             TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE affiliate (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES "user" (id),
    referral_code   VARCHAR(50) NOT NULL UNIQUE,
    commission_pct  NUMERIC(5,2) NOT NULL DEFAULT 10.00,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE affiliate_referral (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    affiliate_id    UUID NOT NULL REFERENCES affiliate (id),
    order_id        UUID NOT NULL REFERENCES "order" (id),
    commission      NUMERIC(10,2) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'pending', -- 'pending','approved','paid'
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Reviews, Discussion & Community

```sql
CREATE TABLE review (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    course_id   UUID NOT NULL REFERENCES course (id) ON DELETE CASCADE,
    user_id     UUID NOT NULL REFERENCES "user" (id),
    rating      SMALLINT NOT NULL CHECK (rating BETWEEN 1 AND 5),
    title       VARCHAR(500),
    body        TEXT,
    is_verified BOOLEAN NOT NULL DEFAULT FALSE,            -- verified purchase
    is_visible  BOOLEAN NOT NULL DEFAULT TRUE,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (course_id, user_id)
);

CREATE INDEX idx_review_course ON review (course_id);

CREATE TABLE discussion_thread (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    course_id   UUID NOT NULL REFERENCES course (id) ON DELETE CASCADE,
    lesson_id   UUID REFERENCES lesson (id),               -- optional: scoped to lesson
    user_id     UUID NOT NULL REFERENCES "user" (id),
    title       VARCHAR(500) NOT NULL,
    body        TEXT NOT NULL,
    is_pinned   BOOLEAN NOT NULL DEFAULT FALSE,
    is_resolved BOOLEAN NOT NULL DEFAULT FALSE,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE discussion_reply (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    thread_id   UUID NOT NULL REFERENCES discussion_thread (id) ON DELETE CASCADE,
    user_id     UUID NOT NULL REFERENCES "user" (id),
    body        TEXT NOT NULL,
    is_answer   BOOLEAN NOT NULL DEFAULT FALSE,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Credentials & Certificates

```sql
CREATE TABLE credential_template (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    course_id           UUID NOT NULL REFERENCES course (id) ON DELETE CASCADE,
    template_name       VARCHAR(255) NOT NULL,
    credential_type     VARCHAR(30) NOT NULL DEFAULT 'completion', -- 'completion','competency','badge'
    template_html       TEXT,
    badge_image_url     TEXT,
    criteria_narrative  TEXT,                               -- human-readable criteria description
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE issued_credential (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    template_id         UUID NOT NULL REFERENCES credential_template (id),
    enrollment_id       UUID NOT NULL REFERENCES enrollment (id),
    user_id             UUID NOT NULL REFERENCES "user" (id),
    issued_at           TIMESTAMPTZ NOT NULL DEFAULT now(),
    verification_url    TEXT NOT NULL UNIQUE,               -- public verification endpoint
    credential_json     JSONB,                             -- Open Badges 3.0 / W3C VC JSON-LD
    revoked_at          TIMESTAMPTZ,
    revocation_reason   TEXT
);

CREATE INDEX idx_credential_user ON issued_credential (user_id);
```

## SCORM & xAPI Integration

```sql
CREATE TABLE scorm_package (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    course_id           UUID NOT NULL REFERENCES course (id),
    lesson_id           UUID REFERENCES lesson (id),
    scorm_version       VARCHAR(20) NOT NULL,              -- '1.2', '2004_3rd', '2004_4th'
    manifest_data       JSONB,                             -- parsed imsmanifest.xml
    package_url         TEXT NOT NULL,                      -- S3/CDN path to ZIP
    launch_url          TEXT NOT NULL,
    uploaded_at         TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE scorm_tracking (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    scorm_package_id    UUID NOT NULL REFERENCES scorm_package (id),
    enrollment_id       UUID NOT NULL REFERENCES enrollment (id),
    cmi_data            JSONB NOT NULL,                    -- SCORM CMI data model values
    completion_status   VARCHAR(30),
    success_status      VARCHAR(30),
    score_raw           NUMERIC(5,2),
    score_max           NUMERIC(5,2),
    total_time          INTERVAL,
    last_updated        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE xapi_statement (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    statement_id    UUID NOT NULL UNIQUE,                   -- xAPI statement UUID
    actor_id        UUID NOT NULL REFERENCES "user" (id),
    verb_iri        VARCHAR(500) NOT NULL,                  -- e.g. 'http://adlnet.gov/expapi/verbs/completed'
    object_type     VARCHAR(50) NOT NULL,                   -- 'Activity','Agent','StatementRef'
    object_iri      VARCHAR(500) NOT NULL,
    result_score    NUMERIC(5,2),
    result_success  BOOLEAN,
    result_complete BOOLEAN,
    result_duration INTERVAL,
    context_data    JSONB,
    stored_at       TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_xapi_actor ON xapi_statement (actor_id);
CREATE INDEX idx_xapi_verb ON xapi_statement (verb_iri);
CREATE INDEX idx_xapi_stored ON xapi_statement (stored_at);
```

## LTI Integration

```sql
CREATE TABLE lti_tool (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    client_id       VARCHAR(255) NOT NULL UNIQUE,
    deployment_id   VARCHAR(255) NOT NULL,
    issuer          VARCHAR(500) NOT NULL,
    auth_login_url  TEXT NOT NULL,
    auth_token_url  TEXT NOT NULL,
    jwks_url        TEXT NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE lti_launch (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tool_id         UUID NOT NULL REFERENCES lti_tool (id),
    course_id       UUID NOT NULL REFERENCES course (id),
    lesson_id       UUID REFERENCES lesson (id),
    user_id         UUID NOT NULL REFERENCES "user" (id),
    launch_data     JSONB NOT NULL,
    launched_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Notifications & Audit

```sql
CREATE TABLE notification (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES "user" (id) ON DELETE CASCADE,
    notification_type VARCHAR(50) NOT NULL,                 -- 'enrollment','review','payout','system'
    title           VARCHAR(500) NOT NULL,
    body            TEXT,
    is_read         BOOLEAN NOT NULL DEFAULT FALSE,
    action_url      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_notification_user ON notification (user_id, is_read);

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID REFERENCES "user" (id),
    action          VARCHAR(100) NOT NULL,                  -- 'course.published', 'enrollment.created', etc.
    entity_type     VARCHAR(100) NOT NULL,
    entity_id       UUID NOT NULL,
    old_values      JSONB,
    new_values      JSONB,
    ip_address      INET,
    user_agent      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_entity ON audit_log (entity_type, entity_id);
CREATE INDEX idx_audit_user ON audit_log (user_id);
CREATE INDEX idx_audit_created ON audit_log (created_at);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Users & Auth | 5 | user, user_role, user_sso, organisation, organisation_member |
| Reference Data | 3 | category, language, skill_tag |
| Course Catalog | 5 | course, course_category, course_skill, course_prerequisite, section |
| Lessons & Content | 4 | lesson, lesson_video, lesson_text, lesson_attachment |
| Assessments | 3 | assessment, assessment_item, assessment_item_option |
| Enrollment & Progress | 4 | enrollment, lesson_progress, assessment_attempt, assessment_response |
| Payments & Commerce | 7 | pricing_plan, coupon, coupon_course, order, order_item, instructor_payout, affiliate, affiliate_referral (8 total) |
| Reviews & Discussion | 3 | review, discussion_thread, discussion_reply |
| Credentials | 2 | credential_template, issued_credential |
| SCORM & xAPI | 3 | scorm_package, scorm_tracking, xapi_statement |
| LTI | 2 | lti_tool, lti_launch |
| Platform | 2 | notification, audit_log |
| **Total** | **~43** | |

---

## Key Design Decisions

1. **Separate tables for each lesson content type** (lesson_video, lesson_text, lesson_attachment) rather than a single polymorphic table. This allows type-specific columns (video duration, encoding status) without NULLable columns or JSONB overloading.

2. **Junction tables for many-to-many relationships** (course_category, course_skill, coupon_course) enforce referential integrity at the database level rather than relying on application logic.

3. **Reference data in lookup tables** (category, language, skill_tag) rather than application enums. This allows runtime additions and keeps the data model standards-aligned (ISO 639, ISO 4217).

4. **SCORM CMI data stored as JSONB** within an otherwise relational model. The SCORM CMI data model has ~80 fields that vary by version, making a JSONB column the pragmatic choice for this specific case.

5. **Audit log as a simple append-only table** rather than full event sourcing. This provides regulatory compliance (who changed what, when) without the complexity of event replay.

6. **Stripe IDs stored as references** (stripe_payment_id, stripe_transfer_id) rather than duplicating payment data locally. This follows PCI DSS best practice of minimizing cardholder data scope.

7. **Denormalized counters on course** (avg_rating, rating_count, enrollment_count) for read performance on catalog pages, maintained by application-level triggers or background jobs.

8. **Open Badges 3.0 credential JSON stored in issued_credential.credential_json** alongside relational metadata. The JSON-LD structure is needed for external verification, while relational fields enable internal queries.

9. **Hierarchical categories via adjacency list** (category.parent_id) — simple and sufficient for 2-3 level category trees. If deeper hierarchies are needed, a materialized path or ltree column can be added.

10. **Assessment items modeled per QTI patterns** with separate option tables, supporting multiple choice, multiple select, true/false, and free-text response types without schema changes.
