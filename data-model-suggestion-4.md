# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: Online Course Marketplace · Created: 2026-05-19

## Philosophy

The graph-relational hybrid uses conventional relational tables for operational CRUD (users, courses, enrollments, payments) combined with a property graph layer for relationship-intensive queries: personalized learning path recommendations, prerequisite chains, skill-to-course mapping, learner similarity networks, and instructor collaboration graphs. The graph layer is implemented either as a set of `graph_node` / `graph_edge` tables in PostgreSQL (using recursive CTEs and ltree for traversal) or as a dedicated graph database (Neo4j, Apache AGE for PostgreSQL) for complex traversals.

An online course marketplace is fundamentally a **relationship-rich domain**: learners relate to courses through enrollments, completions, and reviews; courses relate to each other through prerequisites, skill overlap, and topic similarity; instructors relate to learners through teaching and feedback; skills relate to courses and to career paths; and learning journeys are paths through a graph of courses. Traditional relational models flatten these relationships into junction tables and lose the traversal semantics. A graph layer makes questions like "what courses should this learner take next, given their completion history, skill goals, and what similar learners found valuable?" into natural graph traversals rather than complex multi-join SQL.

This pattern is used by LinkedIn Learning (skill graph for career recommendations), Coursera (learning path recommendation engine), and Netflix (content similarity graphs). For an AI-native marketplace, the graph becomes the substrate for recommendation ML models, skill gap analysis, and personalized curriculum sequencing.

**Best for:** AI-powered recommendation engines, personalized learning paths across a multi-instructor catalog, skill-based career pathway mapping, and platforms where discovery and cross-course sequencing are key differentiators.

**Trade-offs:**
- (+) Natural representation of prerequisite chains, skill taxonomies, and learning pathways
- (+) Graph traversal queries (shortest path, neighborhood, similarity) are orders of magnitude faster than recursive SQL joins
- (+) Ideal substrate for AI/ML recommendation models: learner-course interaction graphs, skill embeddings
- (+) Enables "similar courses", "learners also took", and "career pathway" features natively
- (+) Skill gap analysis is a graph operation: compare learner's completed-skill subgraph to target-role skill requirements
- (-) Dual-storage complexity: relational tables + graph layer must be kept in sync
- (-) Graph databases (Neo4j) add infrastructure and licensing cost; PostgreSQL-native approaches (Apache AGE, recursive CTEs) have performance limits on large graphs
- (-) Team must understand both relational and graph query languages (SQL + Cypher or Gremlin)
- (-) Graph schema evolution is less mature than relational migration tooling
- (-) Overkill for simple catalogs with few cross-course relationships

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO 639-1 / BCP 47 | Language nodes in the graph; courses connected to language nodes for multilingual discovery |
| ESCO (European Skills, Competences, Qualifications) | Skill taxonomy modeled as a skill graph; ESCO occupations and skills mapped to course nodes |
| O*NET (US Occupational Information) | Career/occupation nodes linked to skill requirement edges; used for career pathway recommendations |
| IEEE 9274.1.1 (xAPI) | xAPI statements generate edges in the learner-content interaction graph |
| IMS LTI 1.3 | LTI tool nodes connected to course nodes; launch context as edge properties |
| Open Badges 3.0 | Credential nodes linked to learner, course, and skill nodes; credential verification via graph traversal |
| SCORM 1.2 / 2004 | SCORM content nodes with completion edges to learner nodes |
| Schema.org Course | Course node properties align with Schema.org Course type for SEO and linked data |

---

## Relational Core (Operational CRUD)

```sql
-- ============================================================
-- USERS & AUTHENTICATION (standard relational)
-- ============================================================

CREATE TABLE "user" (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(320) NOT NULL UNIQUE,
    email_verified  BOOLEAN NOT NULL DEFAULT FALSE,
    password_hash   VARCHAR(255),
    display_name    VARCHAR(255) NOT NULL,
    avatar_url      TEXT,
    roles           TEXT[] NOT NULL DEFAULT '{learner}',
    profile         JSONB NOT NULL DEFAULT '{}',
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE organisation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(255) NOT NULL UNIQUE,
    org_type        VARCHAR(50) NOT NULL,
    settings        JSONB NOT NULL DEFAULT '{}',
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE organisation_member (
    organisation_id UUID NOT NULL REFERENCES organisation (id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES "user" (id) ON DELETE CASCADE,
    role            VARCHAR(50) NOT NULL DEFAULT 'member',
    joined_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (organisation_id, user_id)
);

-- ============================================================
-- COURSES (operational data)
-- ============================================================

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
    media               JSONB NOT NULL DEFAULT '{}',
    curriculum          JSONB NOT NULL DEFAULT '[]',
    stats               JSONB NOT NULL DEFAULT '{}',
    published_at        TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, slug)
);

CREATE INDEX idx_course_status ON course (status);
CREATE INDEX idx_course_instructor ON course (instructor_id);

CREATE TABLE content_item (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    course_id       UUID NOT NULL REFERENCES course (id) ON DELETE CASCADE,
    content_type    VARCHAR(30) NOT NULL,
    title           VARCHAR(500),
    content         JSONB NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- ENROLLMENT & PROGRESS
-- ============================================================

CREATE TABLE enrollment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES "user" (id) ON DELETE CASCADE,
    course_id       UUID NOT NULL REFERENCES course (id) ON DELETE CASCADE,
    status          VARCHAR(30) NOT NULL DEFAULT 'active',
    enrolled_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ,
    progress_pct    NUMERIC(5,2) NOT NULL DEFAULT 0.00,
    progress_data   JSONB NOT NULL DEFAULT '{}',
    UNIQUE (user_id, course_id)
);

CREATE INDEX idx_enrollment_user ON enrollment (user_id);
CREATE INDEX idx_enrollment_course ON enrollment (course_id);

CREATE TABLE assessment_attempt (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    enrollment_id   UUID NOT NULL REFERENCES enrollment (id) ON DELETE CASCADE,
    content_item_id UUID NOT NULL REFERENCES content_item (id),
    attempt_number  INT NOT NULL DEFAULT 1,
    score           NUMERIC(5,2),
    passed          BOOLEAN,
    responses       JSONB NOT NULL DEFAULT '[]',
    started_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    submitted_at    TIMESTAMPTZ
);

-- ============================================================
-- PAYMENTS (same as hybrid model -- operational data)
-- ============================================================

CREATE TABLE pricing (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    course_id       UUID NOT NULL REFERENCES course (id) ON DELETE CASCADE,
    plan_type       VARCHAR(20) NOT NULL,
    base_price      NUMERIC(10,2) NOT NULL,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    interval_months INT,
    region_pricing  JSONB NOT NULL DEFAULT '{}',
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE "order" (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id             UUID NOT NULL REFERENCES "user" (id),
    status              VARCHAR(20) NOT NULL DEFAULT 'pending',
    currency_code       CHAR(3) NOT NULL,
    total               NUMERIC(10,2) NOT NULL,
    items               JSONB NOT NULL DEFAULT '[]',
    payment             JSONB NOT NULL DEFAULT '{}',
    completed_at        TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE review (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    course_id   UUID NOT NULL REFERENCES course (id) ON DELETE CASCADE,
    user_id     UUID NOT NULL REFERENCES "user" (id),
    rating      SMALLINT NOT NULL CHECK (rating BETWEEN 1 AND 5),
    title       VARCHAR(500),
    body        TEXT,
    is_visible  BOOLEAN NOT NULL DEFAULT TRUE,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (course_id, user_id)
);

CREATE TABLE credential (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    course_id       UUID NOT NULL REFERENCES course (id),
    user_id         UUID NOT NULL REFERENCES "user" (id),
    enrollment_id   UUID NOT NULL REFERENCES enrollment (id),
    credential_type VARCHAR(30) NOT NULL,
    credential_data JSONB NOT NULL,
    verification_url TEXT NOT NULL UNIQUE,
    issued_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    revoked_at      TIMESTAMPTZ
);

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID,
    action          VARCHAR(100) NOT NULL,
    entity_type     VARCHAR(100) NOT NULL,
    entity_id       UUID NOT NULL,
    changes         JSONB,
    context         JSONB,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_entity ON audit_log (entity_type, entity_id);
```

---

## Property Graph Layer

The graph layer represents entities as nodes and relationships as edges. This can be implemented in PostgreSQL using the tables below, or in a dedicated graph database (Neo4j, Apache AGE). The relational core remains the source of truth for operational data; the graph layer is a derived, query-optimized view maintained by event handlers or CDC (Change Data Capture).

```sql
-- ============================================================
-- GRAPH NODES
-- ============================================================

CREATE TABLE graph_node (
    id              UUID PRIMARY KEY,                      -- matches the entity's relational PK
    node_type       VARCHAR(50) NOT NULL,                  -- 'User','Course','Skill','Category','Occupation',
                                                           -- 'Credential','Organisation','LearningPath'
    label           VARCHAR(500) NOT NULL,                 -- display name
    properties      JSONB NOT NULL DEFAULT '{}',           -- type-specific attributes
    -- properties examples by type:
    --
    -- User:       {"roles": ["learner"], "enrolled_count": 5, "completed_count": 3}
    -- Course:     {"level": "intermediate", "language": "en", "avg_rating": 4.5, "enrollment_count": 3200}
    -- Skill:      {"esco_uri": "http://data.europa.eu/esco/skill/abc", "category": "Digital Skills"}
    -- Occupation:  {"onet_code": "15-1252.00", "title": "Software Developers"}
    -- Category:   {"depth": 2, "slug": "machine-learning"}
    -- LearningPath: {"created_by": "system", "estimated_hours": 40, "description": "..."}
    embedding       VECTOR(384),                           -- sentence embedding for similarity search (pgvector)
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_graph_node_type ON graph_node (node_type);
CREATE INDEX idx_graph_node_label ON graph_node USING GIN (to_tsvector('english', label));
-- pgvector index for embedding similarity
-- CREATE INDEX idx_graph_node_embedding ON graph_node USING ivfflat (embedding vector_cosine_ops) WITH (lists = 100);

-- ============================================================
-- GRAPH EDGES (relationships between nodes)
-- ============================================================

CREATE TABLE graph_edge (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_id       UUID NOT NULL REFERENCES graph_node (id) ON DELETE CASCADE,
    target_id       UUID NOT NULL REFERENCES graph_node (id) ON DELETE CASCADE,
    edge_type       VARCHAR(50) NOT NULL,
    -- Edge types:
    -- User -> Course:      ENROLLED_IN, COMPLETED, REVIEWED, TEACHES, WISHLISTED
    -- Course -> Skill:     TEACHES_SKILL (weight = proficiency level)
    -- Course -> Course:    PREREQUISITE_OF, SIMILAR_TO (weight = similarity score)
    -- Course -> Category:  BELONGS_TO
    -- User -> Skill:       HAS_SKILL (weight = proficiency), WANTS_SKILL (target)
    -- Skill -> Skill:      RELATED_TO, PREREQUISITE_OF
    -- Skill -> Occupation:  REQUIRED_FOR (weight = importance)
    -- Occupation -> Occupation: CAREER_PATH (progression)
    -- Course -> LearningPath: STEP_IN (position = sequence order)
    -- User -> User:        FOLLOWS
    -- User -> Organisation: MEMBER_OF
    weight          NUMERIC(5,3) DEFAULT 1.000,            -- relationship strength (0-1 for similarity, rating value, etc.)
    properties      JSONB NOT NULL DEFAULT '{}',
    -- properties examples:
    -- ENROLLED_IN:   {"enrolled_at": "2026-01-15", "progress_pct": 72.5}
    -- TEACHES_SKILL: {"proficiency_level": "intermediate", "lesson_count": 12}
    -- SIMILAR_TO:    {"algorithm": "content_based", "computed_at": "2026-05-01"}
    -- PREREQUISITE_OF: {"is_hard": true, "explanation": "Requires linear algebra knowledge"}
    -- REVIEWED:      {"rating": 5, "reviewed_at": "2026-03-20"}
    -- STEP_IN:       {"position": 3, "is_optional": false}
    -- REQUIRED_FOR:  {"importance": "essential", "source": "onet"}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_graph_edge_source ON graph_edge (source_id, edge_type);
CREATE INDEX idx_graph_edge_target ON graph_edge (target_id, edge_type);
CREATE INDEX idx_graph_edge_type ON graph_edge (edge_type);
CREATE INDEX idx_graph_edge_weight ON graph_edge (edge_type, weight DESC);
CREATE INDEX idx_graph_edge_properties ON graph_edge USING GIN (properties jsonb_path_ops);

-- Prevent duplicate edges of the same type between same nodes
CREATE UNIQUE INDEX idx_graph_edge_unique ON graph_edge (source_id, target_id, edge_type);
```

## Skill & Career Taxonomy (Graph-Native)

```sql
-- ============================================================
-- SKILL TAXONOMY (seeded from ESCO and O*NET, extended by platform)
-- ============================================================

-- Skills, occupations, and their relationships are primarily managed through
-- graph_node and graph_edge, but we maintain a relational reference for
-- taxonomy import and admin management.

CREATE TABLE skill_taxonomy (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    external_uri    VARCHAR(500) UNIQUE,                   -- ESCO URI or O*NET code
    source          VARCHAR(30) NOT NULL,                  -- 'esco', 'onet', 'platform'
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    skill_type      VARCHAR(30),                           -- 'knowledge', 'skill', 'competence'
    category        VARCHAR(100),
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_skill_taxonomy_source ON skill_taxonomy (source);
CREATE INDEX idx_skill_taxonomy_name ON skill_taxonomy USING GIN (to_tsvector('english', name));

CREATE TABLE occupation_taxonomy (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    external_code   VARCHAR(50) UNIQUE,                    -- O*NET SOC code (e.g., '15-1252.00')
    source          VARCHAR(30) NOT NULL,                  -- 'onet', 'esco', 'platform'
    title           VARCHAR(255) NOT NULL,
    description     TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- LEARNING PATHS (curated or AI-generated sequences)
-- ============================================================

CREATE TABLE learning_path (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    title           VARCHAR(500) NOT NULL,
    slug            VARCHAR(500) NOT NULL UNIQUE,
    description     TEXT,
    created_by      UUID REFERENCES "user" (id),           -- NULL for system-generated
    generation_type VARCHAR(20) NOT NULL DEFAULT 'manual', -- 'manual', 'ai_generated', 'ai_personalized'
    target_occupation_id UUID REFERENCES occupation_taxonomy (id),
    target_skills   UUID[],                                -- skill_taxonomy IDs
    estimated_hours INT,
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- metadata example: {
    --   "difficulty_progression": ["beginner", "intermediate", "advanced"],
    --   "ai_model_version": "v2.3",
    --   "confidence_score": 0.87,
    --   "learner_id": "uuid"  (for personalized paths)
    -- }
    is_published    BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE learning_path_step (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    path_id         UUID NOT NULL REFERENCES learning_path (id) ON DELETE CASCADE,
    course_id       UUID NOT NULL REFERENCES course (id),
    position        INT NOT NULL,
    is_optional     BOOLEAN NOT NULL DEFAULT FALSE,
    rationale       TEXT,                                  -- why this course is in this position
    UNIQUE (path_id, position)
);

CREATE INDEX idx_path_step_path ON learning_path_step (path_id, position);
```

## Recommendation Engine Support

```sql
-- ============================================================
-- RECOMMENDATION INFRASTRUCTURE
-- ============================================================

-- Pre-computed similarity scores between courses (updated by ML pipeline)
CREATE TABLE course_similarity (
    course_a_id     UUID NOT NULL REFERENCES course (id) ON DELETE CASCADE,
    course_b_id     UUID NOT NULL REFERENCES course (id) ON DELETE CASCADE,
    algorithm       VARCHAR(30) NOT NULL,                  -- 'content_based', 'collaborative', 'skill_overlap', 'hybrid'
    similarity      NUMERIC(5,4) NOT NULL,                 -- 0.0000 to 1.0000
    computed_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (course_a_id, course_b_id, algorithm),
    CHECK (course_a_id < course_b_id)                      -- canonical ordering
);

CREATE INDEX idx_similarity_a ON course_similarity (course_a_id, algorithm, similarity DESC);
CREATE INDEX idx_similarity_b ON course_similarity (course_b_id, algorithm, similarity DESC);

-- Learner skill profile (derived from completions and assessments)
CREATE TABLE learner_skill_profile (
    user_id         UUID NOT NULL REFERENCES "user" (id) ON DELETE CASCADE,
    skill_id        UUID NOT NULL REFERENCES skill_taxonomy (id),
    proficiency     NUMERIC(3,2) NOT NULL DEFAULT 0.00,    -- 0.00 to 1.00
    evidence_count  INT NOT NULL DEFAULT 0,                -- courses completed that teach this skill
    last_assessed   TIMESTAMPTZ,
    computed_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, skill_id)
);

CREATE INDEX idx_skill_profile_user ON learner_skill_profile (user_id);

-- Career goal tracking
CREATE TABLE learner_career_goal (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES "user" (id) ON DELETE CASCADE,
    occupation_id   UUID NOT NULL REFERENCES occupation_taxonomy (id),
    target_date     DATE,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Example Graph Queries

### PostgreSQL Recursive CTE: Prerequisite Chain

```sql
-- Find all prerequisites (transitive) for a course
WITH RECURSIVE prereqs AS (
    -- Base case: direct prerequisites
    SELECT target_id AS course_id, source_id AS prerequisite_id, 1 AS depth
    FROM graph_edge
    WHERE target_id = '<course_uuid>'
      AND edge_type = 'PREREQUISITE_OF'
    UNION ALL
    -- Recursive: prerequisites of prerequisites
    SELECT p.course_id, e.source_id, p.depth + 1
    FROM prereqs p
    JOIN graph_edge e ON e.target_id = p.prerequisite_id
      AND e.edge_type = 'PREREQUISITE_OF'
    WHERE p.depth < 10  -- prevent infinite loops
)
SELECT DISTINCT c.id, c.title, p.depth
FROM prereqs p
JOIN course c ON c.id = p.prerequisite_id
ORDER BY p.depth;
```

### "Learners Also Took" (Collaborative Filtering via Graph)

```sql
-- Find courses frequently taken by learners who completed course X
SELECT c.id, c.title, COUNT(*) AS co_enrollment_count
FROM graph_edge e1
JOIN graph_edge e2 ON e2.source_id = e1.source_id      -- same learner
  AND e2.edge_type = 'COMPLETED'
  AND e2.target_id <> e1.target_id                      -- different course
JOIN course c ON c.id = e2.target_id AND c.status = 'published'
WHERE e1.target_id = '<course_uuid>'
  AND e1.edge_type = 'COMPLETED'
GROUP BY c.id, c.title
ORDER BY co_enrollment_count DESC
LIMIT 10;
```

### Skill Gap Analysis

```sql
-- What skills does a learner need for a target occupation that they don't yet have?
SELECT st.name AS skill_name,
       re.weight AS importance,
       COALESCE(lsp.proficiency, 0) AS current_proficiency,
       (re.weight - COALESCE(lsp.proficiency, 0)) AS gap
FROM graph_edge re
JOIN graph_node gn ON gn.id = re.source_id AND gn.node_type = 'Skill'
JOIN skill_taxonomy st ON st.id = re.source_id
LEFT JOIN learner_skill_profile lsp ON lsp.skill_id = re.source_id
  AND lsp.user_id = '<learner_uuid>'
WHERE re.target_id = '<occupation_node_uuid>'
  AND re.edge_type = 'REQUIRED_FOR'
  AND (lsp.proficiency IS NULL OR lsp.proficiency < re.weight)
ORDER BY gap DESC;
```

### Course Recommendation via Embedding Similarity (pgvector)

```sql
-- Find courses similar to ones the learner has completed, using vector embeddings
SELECT c.id, c.title, gn.embedding <=> learner_centroid AS distance
FROM (
    -- Compute centroid of learner's completed course embeddings
    SELECT AVG(gn.embedding) AS centroid
    FROM graph_edge e
    JOIN graph_node gn ON gn.id = e.target_id AND gn.node_type = 'Course'
    WHERE e.source_id = '<learner_node_uuid>'
      AND e.edge_type = 'COMPLETED'
) learner_centroid,
graph_node gn
JOIN course c ON c.id = gn.id AND c.status = 'published'
WHERE gn.node_type = 'Course'
  AND gn.id NOT IN (
    SELECT target_id FROM graph_edge
    WHERE source_id = '<learner_node_uuid>' AND edge_type IN ('COMPLETED', 'ENROLLED_IN')
  )
ORDER BY distance
LIMIT 10;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Users & Auth | 3 | user, organisation, organisation_member |
| Courses & Content | 2 | course, content_item |
| Enrollment & Progress | 2 | enrollment, assessment_attempt |
| Payments & Commerce | 2 | pricing, order |
| Reviews & Credentials | 2 | review, credential |
| Graph Core | 2 | graph_node, graph_edge |
| Skill/Career Taxonomy | 2 | skill_taxonomy, occupation_taxonomy |
| Learning Paths | 2 | learning_path, learning_path_step |
| Recommendation Engine | 3 | course_similarity, learner_skill_profile, learner_career_goal |
| Platform | 1 | audit_log |
| **Total** | **~21** | Plus graph nodes/edges scale with data |

---

## Key Design Decisions

1. **Dual-layer architecture**: relational tables for operational CRUD (enrollments, payments, content management) and a graph layer for relationship queries (recommendations, skill gaps, learning paths). The relational core is the source of truth; the graph layer is derived.

2. **Generic graph_node/graph_edge tables** rather than entity-specific relationship tables. This allows new node types (e.g., "LearningPath", "Certificate", "Employer") and edge types to be added without DDL changes, which is critical for a recommendation engine that evolves as the platform grows.

3. **pgvector embeddings on graph nodes**. Course, skill, and user nodes carry vector embeddings (384-dimensional sentence-transformer output) enabling semantic similarity search. This supports "find courses similar to X" without pre-computing all pairwise similarities.

4. **ESCO and O*NET skill taxonomies as graph nodes**. Rather than building a proprietary skill taxonomy, the graph is seeded from established occupational frameworks. Platform-specific skills are added as additional nodes, linked to the standard taxonomy via RELATED_TO edges.

5. **Learning paths as graph traversals**. A learning path is a sequence of STEP_IN edges from a LearningPath node to Course nodes. AI-generated personalized paths create new LearningPath nodes with computed edges, without modifying the underlying course data.

6. **Edge weight for relationship strength**. The `weight` column on graph_edge serves different purposes per edge type: similarity score for SIMILAR_TO, rating value for REVIEWED, proficiency level for TEACHES_SKILL, importance for REQUIRED_FOR. This enables weighted graph algorithms.

7. **Pre-computed similarity table** for recommendation performance. While graph traversals can compute similarity on-the-fly, the `course_similarity` table caches ML-pipeline-computed scores for sub-millisecond recommendation queries. Updated periodically by batch jobs.

8. **Learner skill profile as a derived materialized view**. The `learner_skill_profile` table is computed from enrollment completions and assessment results. It represents the learner's current skill state as a vector that can be compared against occupation skill requirements for gap analysis.

9. **Canonical edge ordering** (`course_a_id < course_b_id` in course_similarity) prevents duplicate symmetric relationships and halves storage for bidirectional similarity.

10. **Relational core mirrors the Hybrid JSONB model** (Suggestion 3) for operational tables. The graph layer is additive -- it does not replace the course, enrollment, or payment tables but augments them with traversal-optimized relationship data. This means the graph layer can be adopted incrementally.
