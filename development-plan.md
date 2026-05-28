# Online Course Marketplace -- Phased Development Plan

> Project: 113-online-course-marketplace
> Created: 2026-05-25
> Status: Planning

---

## Technology Decisions

### Core Stack

| Layer | Choice | Rationale |
|-------|--------|-----------|
| **Language** | TypeScript (full-stack) | Type safety across API and UI; largest hiring pool for web; strong ecosystem for the integrations we need (Stripe SDK, xAPI libraries, LTI libraries) |
| **Backend framework** | Node.js + Fastify | High-throughput JSON API server; schema-based request validation (Ajv) pairs well with OpenAPI generation; native async/await for video pipeline orchestration |
| **Frontend framework** | Next.js 15 (App Router) | SSR for marketplace SEO (course catalog pages must be crawlable); RSC for course player performance; built-in image/video optimization; middleware for auth gating |
| **Database** | PostgreSQL 16 + pgvector | Hybrid Relational+JSONB model (Data Model Suggestion 3) as the primary schema, with graph tables from Suggestion 4 added in Phase 8 for recommendations. PostgreSQL handles relational integrity, JSONB flexibility, full-text search, and vector similarity in a single engine |
| **Cache** | Redis (Valkey) | Session store, rate limiting, job queues (BullMQ), real-time presence for discussions, and leaderboard/ranking caches |
| **Object storage** | S3-compatible (AWS S3 / MinIO for self-hosted) | Video source files, SCORM packages, PDF attachments, certificate images. CDN-fronted for delivery |
| **Video pipeline** | MediaConvert (AWS) or Mux | Adaptive-bitrate HLS encoding, thumbnail extraction, caption generation. Mux simplifies but increases cost; MediaConvert offers self-managed economics |
| **Search** | Meilisearch | Typo-tolerant faceted search for the course catalog. Lower operational burden than Elasticsearch; sufficient for marketplace discovery. PostgreSQL full-text search as fallback |
| **Payments** | Stripe Connect (Express accounts) | Multi-party payments (learner pays, platform takes fee, instructor receives payout). Express accounts handle KYC/AML for instructors. Stripe Checkout for PCI SAQ-A compliance |
| **Auth** | NextAuth.js (Auth.js) v5 + custom SAML/OIDC | Credential + social login for B2C; SAML 2.0 / OIDC for enterprise SSO. JWT sessions with refresh tokens |
| **AI/ML** | Python microservices (FastAPI) | Course quality scoring, recommendation engine, content generation. Deployed as separate services behind internal API; uses sentence-transformers for embeddings, scikit-learn/PyTorch for models |
| **Job queue** | BullMQ (Redis-backed) | Video encoding callbacks, payout calculations, email digests, SCORM import processing, AI pipeline triggers |
| **Email** | Resend or AWS SES | Transactional emails (enrollment confirmation, payout notifications, password reset); marketing via integration with ConvertKit/Mailchimp |
| **Monitoring** | OpenTelemetry + Grafana stack | Distributed tracing, metrics, and logging. OTel collector to Prometheus/Loki/Tempo |
| **Infrastructure** | Docker Compose (dev) / Kubernetes (prod) | Self-hosted deployment target as stated in README; Helm charts for k8s; Terraform for cloud provisioning |

### Data Model Decision

We adopt **Data Model Suggestion 3 (Hybrid Relational + JSONB)** as the primary schema because:

1. **Fewer tables (~19)** vs fully normalized (~43) reduces migration overhead during rapid MVP iteration
2. **JSONB columns** naturally handle variable content types (video, quiz, SCORM, coding exercises) without schema changes for each new type
3. **Relational integrity** is preserved on business-critical fields (users, enrollments, orders, payments) via foreign keys and constraints
4. **Curriculum as JSONB** on the course table matches the access pattern: curriculum is always loaded as a unit during course playback
5. **Regional pricing as JSONB** supports purchasing-power-parity pricing without a region junction table
6. **Standards-aligned**: SCORM CMI data, xAPI statements, and Open Badges 3.0 JSON-LD are naturally stored as JSONB

In Phase 8, we add the **graph tables from Suggestion 4** (graph_node, graph_edge, skill_taxonomy, learning_path) to power the AI recommendation engine, skill gap analysis, and personalized learning paths. These are additive -- they augment the Suggestion 3 core without replacing it.

### API Design

- **REST API** documented via OpenAPI 3.1, following conventions from Open edX and Thinkific APIs
- **Versioned endpoints**: `/api/v1/courses`, `/api/v1/enrollments`, etc.
- **Webhook events** for external integrations (enrollment.created, payment.completed, course.published)
- **LTI 1.3 endpoints** conforming to the 1EdTech specification for tool launch and grade passback
- **xAPI endpoints** implementing the LRS statement API (IEEE 9274.1.1) for learning record ingestion

### Project Structure

```
online-course-marketplace/
  apps/
    web/                     # Next.js 15 frontend (marketplace, course player, dashboards)
    api/                     # Fastify REST API server
    ai-services/             # Python FastAPI microservices (recommendations, quality scoring, content generation)
  packages/
    database/                # Prisma schema, migrations, seed data
    shared/                  # Shared TypeScript types, validation schemas (Zod), constants
    ui/                      # Shared UI component library (Radix + Tailwind)
    email-templates/         # React Email templates
    scorm-runtime/           # SCORM 1.2/2004 JavaScript runtime adapter
    lti-provider/            # LTI 1.3 provider library
    xapi-client/             # xAPI statement builder and LRS client
  infrastructure/
    docker/                  # Docker Compose for local development
    k8s/                     # Kubernetes Helm charts
    terraform/               # Cloud infrastructure provisioning
  docs/
    api/                     # Generated OpenAPI documentation
    architecture/            # Architecture Decision Records (ADRs)
```

Monorepo managed by **Turborepo** with pnpm workspaces.

---

## Phase Dependency Graph

```
Phase 1: Foundation & Auth
    |
    v
Phase 2: Course Authoring
    |
    v
Phase 3: Enrollment & Course Player -----+
    |                                     |
    v                                     |
Phase 4: Payments & Creator Economics     |
    |                                     |
    v                                     v
Phase 5: Reviews, Discussion & Community  Phase 6: Assessments & Credentials
    |                                     |
    +-------------+-----------------------+
                  |
                  v
Phase 7: Marketplace Discovery & Search
                  |
                  v
Phase 8: AI Recommendation Engine & Learning Paths
                  |
                  v
Phase 9: SCORM, xAPI & LTI Integration
                  |
                  v
Phase 10: Enterprise Features (SSO, White-Label, Analytics)
                  |
                  v
Phase 11: AI Content Generation & Dynamic Pricing
                  |
                  v
Phase 12: Production Hardening & Compliance
```

**Key dependencies:**
- Phase 3 requires Phase 2 (courses must exist before enrollment)
- Phase 4 requires Phase 3 (enrollment triggers payment)
- Phase 5 requires Phase 3 (discussions are per-enrollment/course)
- Phase 6 requires Phase 3 (assessments are part of course progress)
- Phase 7 requires Phase 2 (catalog data) and Phase 5 (reviews for ranking)
- Phase 8 requires Phase 7 (search) and Phase 6 (skill/completion data for recommendations)
- Phase 9 can begin after Phase 3 but benefits from Phase 6 (assessment tracking)
- Phase 10 requires Phase 4 (payments) and Phase 1 (auth) to be mature
- Phase 11 requires Phase 8 (AI infrastructure) and Phase 4 (pricing data)
- Phase 12 requires all prior phases to be functionally complete

---

## Phase 1: Foundation & Auth

**Goal:** Establish project scaffolding, database schema, authentication, and user management. At the end of this phase, users can register, log in, manage profiles, and be assigned roles.

**Duration estimate:** 3 weeks

### Task 1.1: Project Scaffolding & Monorepo Setup

**What:** Initialize Turborepo monorepo with pnpm workspaces. Create the `apps/web`, `apps/api`, and `packages/*` workspace structure. Configure TypeScript, ESLint, Prettier, and Husky pre-commit hooks.

**Design:**

```typescript
// turbo.json
{
  "pipeline": {
    "build": { "dependsOn": ["^build"], "outputs": [".next/**", "dist/**"] },
    "dev": { "cache": false, "persistent": true },
    "lint": {},
    "test": { "dependsOn": ["build"] },
    "db:migrate": { "cache": false }
  }
}

// apps/api/src/server.ts
import Fastify from 'fastify';
import cors from '@fastify/cors';
import helmet from '@fastify/helmet';
import rateLimit from '@fastify/rate-limit';

const app = Fastify({ logger: true });

await app.register(cors, { origin: process.env.WEB_URL });
await app.register(helmet);
await app.register(rateLimit, { max: 100, timeWindow: '1 minute' });

app.get('/health', async () => ({ status: 'ok', timestamp: new Date().toISOString() }));

await app.listen({ port: parseInt(process.env.PORT || '3001'), host: '0.0.0.0' });
```

```yaml
# infrastructure/docker/docker-compose.yml
services:
  postgres:
    image: pgvector/pgvector:pg16
    environment:
      POSTGRES_DB: coursemarket
      POSTGRES_USER: coursemarket
      POSTGRES_PASSWORD: dev_password
    ports: ["5432:5432"]
    volumes: ["pgdata:/var/lib/postgresql/data"]
  redis:
    image: valkey/valkey:8
    ports: ["6379:6379"]
  meilisearch:
    image: getmeili/meilisearch:v1.11
    ports: ["7700:7700"]
    environment:
      MEILI_MASTER_KEY: dev_master_key
volumes:
  pgdata:
```

**Testing:**

| # | Test case | Type | Expected result |
|---|-----------|------|-----------------|
| 1.1.1 | `pnpm install` completes without errors | Build | All workspaces resolved, no peer dependency conflicts |
| 1.1.2 | `pnpm turbo build` builds all packages and apps | Build | Exit code 0; `apps/api/dist` and `apps/web/.next` exist |
| 1.1.3 | `docker compose up` starts PostgreSQL, Redis, Meilisearch | Integration | Health checks pass for all services; `pg_isready` returns 0 |
| 1.1.4 | API health endpoint returns 200 | Integration | `GET /health` returns `{ "status": "ok" }` |
| 1.1.5 | ESLint and Prettier pass on all workspaces | Lint | `pnpm turbo lint` exits 0 |

---

### Task 1.2: Database Schema & Migrations

**What:** Set up Prisma ORM with the Hybrid JSONB schema (Data Model Suggestion 3). Create initial migration for user, organisation, organisation_member tables. Seed reference data.

**Design:**

```prisma
// packages/database/prisma/schema.prisma
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

generator client {
  provider        = "prisma-client-js"
  previewFeatures = ["postgresqlExtensions"]
}

model User {
  id            String   @id @default(uuid()) @db.Uuid
  email         String   @unique @db.VarChar(320)
  emailVerified Boolean  @default(false) @map("email_verified")
  passwordHash  String?  @map("password_hash") @db.VarChar(255)
  displayName   String   @map("display_name") @db.VarChar(255)
  avatarUrl     String?  @map("avatar_url")
  roles         String[] @default(["learner"])
  profile       Json     @default("{}")
  ssoProviders  Json     @default("[]") @map("sso_providers")
  isActive      Boolean  @default(true) @map("is_active")
  lastLoginAt   DateTime? @map("last_login_at")
  createdAt     DateTime @default(now()) @map("created_at")
  updatedAt     DateTime @updatedAt @map("updated_at")

  organisationMemberships OrganisationMember[]
  enrollments             Enrollment[]
  orders                  Order[]
  reviews                 Review[]
  credentials             Credential[]

  @@map("user")
}

model Organisation {
  id          String   @id @default(uuid()) @db.Uuid
  name        String   @db.VarChar(255)
  slug        String   @unique @db.VarChar(255)
  orgType     String   @map("org_type") @db.VarChar(50)
  countryCode String?  @map("country_code") @db.Char(2)
  settings    Json     @default("{}")
  isActive    Boolean  @default(true) @map("is_active")
  createdAt   DateTime @default(now()) @map("created_at")
  updatedAt   DateTime @updatedAt @map("updated_at")

  members OrganisationMember[]
  courses Course[]

  @@map("organisation")
}

model OrganisationMember {
  organisationId String   @map("organisation_id") @db.Uuid
  userId         String   @map("user_id") @db.Uuid
  role           String   @default("member") @db.VarChar(50)
  permissions    Json     @default("[]")
  joinedAt       DateTime @default(now()) @map("joined_at")

  organisation Organisation @relation(fields: [organisationId], references: [id], onDelete: Cascade)
  user         User         @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@id([organisationId, userId])
  @@map("organisation_member")
}
```

**Testing:**

| # | Test case | Type | Expected result |
|---|-----------|------|-----------------|
| 1.2.1 | `prisma migrate dev` applies initial migration | Migration | Tables `user`, `organisation`, `organisation_member` created; `\dt` lists all three |
| 1.2.2 | `prisma db seed` inserts test users and org | Seed | SELECT count returns expected rows for each table |
| 1.2.3 | Create user with duplicate email fails | Unit | Prisma throws `P2002` unique constraint violation |
| 1.2.4 | Organisation member cascade delete | Unit | Deleting org removes all organisation_member rows |
| 1.2.5 | JSONB profile field stores and retrieves nested data | Unit | User.profile round-trips with nested objects intact |

---

### Task 1.3: Authentication & Authorization

**What:** Implement Auth.js v5 with credential (email/password) and social (Google, GitHub) providers. Add role-based middleware for API routes. Password hashing with bcrypt. Email verification flow.

**Design:**

```typescript
// apps/web/src/lib/auth.ts
import NextAuth from 'next-auth';
import Credentials from 'next-auth/providers/credentials';
import Google from 'next-auth/providers/google';
import { PrismaAdapter } from '@auth/prisma-adapter';
import { prisma } from '@coursemarket/database';
import bcrypt from 'bcryptjs';

export const { handlers, signIn, signOut, auth } = NextAuth({
  adapter: PrismaAdapter(prisma),
  providers: [
    Google({ clientId: process.env.GOOGLE_CLIENT_ID!, clientSecret: process.env.GOOGLE_CLIENT_SECRET! }),
    Credentials({
      credentials: { email: { type: 'email' }, password: { type: 'password' } },
      async authorize(credentials) {
        const user = await prisma.user.findUnique({ where: { email: credentials.email as string } });
        if (!user?.passwordHash) return null;
        const valid = await bcrypt.compare(credentials.password as string, user.passwordHash);
        return valid ? { id: user.id, email: user.email, name: user.displayName, roles: user.roles } : null;
      },
    }),
  ],
  callbacks: {
    async jwt({ token, user }) {
      if (user) { token.roles = (user as any).roles; token.userId = user.id; }
      return token;
    },
    async session({ session, token }) {
      session.user.id = token.userId as string;
      session.user.roles = token.roles as string[];
      return session;
    },
  },
});

// apps/api/src/middleware/auth.ts
import { FastifyRequest, FastifyReply } from 'fastify';

export function requireRole(...roles: string[]) {
  return async (request: FastifyRequest, reply: FastifyReply) => {
    const userRoles = request.user?.roles ?? [];
    if (!roles.some(r => userRoles.includes(r))) {
      return reply.code(403).send({ error: 'Insufficient permissions' });
    }
  };
}
```

**Testing:**

| # | Test case | Type | Expected result |
|---|-----------|------|-----------------|
| 1.3.1 | Register with valid email/password creates user | Integration | POST /api/auth/register returns 201; user row created with hashed password |
| 1.3.2 | Register with existing email returns 409 | Integration | POST /api/auth/register returns 409 Conflict |
| 1.3.3 | Login with correct credentials returns JWT session | Integration | POST /api/auth/callback/credentials returns session cookie with roles |
| 1.3.4 | Login with wrong password returns 401 | Integration | POST /api/auth/callback/credentials returns 401 |
| 1.3.5 | Protected route rejects unauthenticated request | Integration | GET /api/v1/me without session returns 401 |
| 1.3.6 | Role-gated route rejects learner accessing instructor-only endpoint | Integration | GET /api/v1/instructor/dashboard with learner role returns 403 |
| 1.3.7 | Google OAuth flow creates user on first login | Integration | User row created with email from Google profile; role defaults to `['learner']` |
| 1.3.8 | Password hash is bcrypt with cost factor >= 12 | Unit | Hash starts with `$2a$12$` or `$2b$12$` |

---

### Task 1.4: User Profile & Organisation Management

**What:** CRUD API endpoints for user profiles and organisations. Organisation creation, member invite, and role assignment. Profile avatar upload to S3.

**Design:**

```typescript
// apps/api/src/routes/users.ts
import { z } from 'zod';

const updateProfileSchema = z.object({
  displayName: z.string().min(1).max(255).optional(),
  profile: z.object({
    bio: z.string().max(2000).optional(),
    headline: z.string().max(200).optional(),
    website: z.string().url().optional(),
    social: z.record(z.string()).optional(),
    timezone: z.string().optional(),
  }).optional(),
});

app.patch('/api/v1/users/me', { preHandler: [requireAuth] }, async (request, reply) => {
  const data = updateProfileSchema.parse(request.body);
  const user = await prisma.user.update({
    where: { id: request.user.id },
    data: {
      displayName: data.displayName,
      profile: data.profile ? { ...existingProfile, ...data.profile } : undefined,
    },
  });
  return reply.send(user);
});

// apps/api/src/routes/organisations.ts
app.post('/api/v1/organisations', { preHandler: [requireAuth] }, async (request, reply) => {
  const data = createOrgSchema.parse(request.body);
  const org = await prisma.$transaction(async (tx) => {
    const org = await tx.organisation.create({ data });
    await tx.organisationMember.create({
      data: { organisationId: org.id, userId: request.user.id, role: 'owner' },
    });
    return org;
  });
  return reply.code(201).send(org);
});
```

**Testing:**

| # | Test case | Type | Expected result |
|---|-----------|------|-----------------|
| 1.4.1 | GET /api/v1/users/me returns authenticated user | Integration | Returns user object with profile, roles, and organisation memberships |
| 1.4.2 | PATCH /api/v1/users/me updates display name | Integration | Returns updated user; database reflects change |
| 1.4.3 | POST /api/v1/organisations creates org and sets creator as owner | Integration | Org created; organisation_member row with role='owner' |
| 1.4.4 | POST /api/v1/organisations/:id/members invites a user | Integration | Member row created with role='member'; invitation email queued |
| 1.4.5 | Non-owner cannot change organisation settings | Integration | PATCH /api/v1/organisations/:id by member returns 403 |
| 1.4.6 | Avatar upload to S3 returns CDN URL | Integration | PUT /api/v1/users/me/avatar with image file returns `{ avatarUrl: "https://cdn..." }` |

### Phase 1 Definition of Done

- [ ] Turborepo monorepo builds and lints cleanly
- [ ] Docker Compose starts PostgreSQL 16 + pgvector, Redis, and Meilisearch
- [ ] Database migration creates user, organisation, and organisation_member tables
- [ ] Users can register with email/password and login with Google OAuth
- [ ] JWT session includes user ID and roles
- [ ] Role-based middleware blocks unauthorized access
- [ ] User profile CRUD with avatar upload works end-to-end
- [ ] Organisation CRUD with member management works end-to-end
- [ ] API health check, structured logging, and error handling are in place
- [ ] All test cases pass (unit and integration)
- [ ] CI pipeline runs lint, type-check, and test on every push

---

## Phase 2: Course Authoring

**Goal:** Instructors can create, edit, and publish courses with sections, lessons (video, text, PDF), and a curriculum structure. Video uploads are processed into adaptive-bitrate HLS streams.

**Duration estimate:** 4 weeks

### Task 2.1: Course CRUD & Curriculum Builder API

**What:** API endpoints for creating courses, managing sections and lessons within the curriculum JSONB structure, and transitioning course status (draft -> in_review -> published -> archived).

**Design:**

```typescript
// packages/shared/src/schemas/course.ts
import { z } from 'zod';

export const lessonSchema = z.object({
  lesson_id: z.string().uuid(),
  title: z.string().min(1).max(500),
  type: z.enum(['video', 'text', 'pdf', 'quiz', 'coding_exercise', 'assignment']),
  duration_secs: z.number().int().nonnegative().optional(),
  is_preview: z.boolean().default(false),
  is_mandatory: z.boolean().default(true),
  content_ref: z.string().uuid(), // references content_item.id
});

export const sectionSchema = z.object({
  section_id: z.string().uuid(),
  title: z.string().min(1).max(500),
  lessons: z.array(lessonSchema),
});

export const curriculumSchema = z.array(sectionSchema);

export const createCourseSchema = z.object({
  title: z.string().min(1).max(500),
  subtitle: z.string().max(500).optional(),
  description: z.string().optional(),
  languageCode: z.string().default('en'),
  level: z.enum(['beginner', 'intermediate', 'advanced', 'all']).default('beginner'),
  learningOutcomes: z.array(z.string()).optional(),
  requirements: z.array(z.string()).optional(),
  targetAudience: z.array(z.string()).optional(),
  categoryIds: z.array(z.string().uuid()).optional(),
  skillTags: z.array(z.string()).optional(),
});

// apps/api/src/routes/courses.ts
app.post('/api/v1/courses', {
  preHandler: [requireAuth, requireRole('instructor', 'admin')]
}, async (request, reply) => {
  const data = createCourseSchema.parse(request.body);
  const slug = generateSlug(data.title);
  const course = await prisma.course.create({
    data: {
      organisationId: request.user.organisationId,
      instructorId: request.user.id,
      title: data.title,
      slug,
      subtitle: data.subtitle,
      description: data.description,
      languageCode: data.languageCode,
      level: data.level,
      learningOutcomes: data.learningOutcomes ?? [],
      requirements: data.requirements ?? [],
      targetAudience: data.targetAudience ?? [],
      categoryIds: data.categoryIds ?? [],
      skillTags: data.skillTags ?? [],
      curriculum: [],
      stats: {},
      media: {},
      seo: {},
    },
  });
  return reply.code(201).send(course);
});

// Curriculum reorder endpoint (full curriculum replacement with validation)
app.put('/api/v1/courses/:courseId/curriculum', {
  preHandler: [requireAuth, requireCourseOwner]
}, async (request, reply) => {
  const curriculum = curriculumSchema.parse(request.body);
  // Validate all content_refs exist in content_item table
  const contentRefs = curriculum.flatMap(s => s.lessons.map(l => l.content_ref));
  const existingItems = await prisma.contentItem.findMany({
    where: { id: { in: contentRefs }, courseId: request.params.courseId },
    select: { id: true },
  });
  const existingIds = new Set(existingItems.map(i => i.id));
  const missing = contentRefs.filter(ref => !existingIds.has(ref));
  if (missing.length > 0) {
    return reply.code(400).send({ error: 'Invalid content references', missing });
  }
  const course = await prisma.course.update({
    where: { id: request.params.courseId },
    data: { curriculum },
  });
  return reply.send(course);
});
```

**Testing:**

| # | Test case | Type | Expected result |
|---|-----------|------|-----------------|
| 2.1.1 | POST /api/v1/courses creates a draft course | Integration | Returns course with status='draft', empty curriculum, generated slug |
| 2.1.2 | Duplicate slug within same org appends suffix | Unit | Second course with same title gets slug `intro-to-ml-2` |
| 2.1.3 | PUT /api/v1/courses/:id/curriculum saves valid curriculum | Integration | Course.curriculum JSONB updated; response includes sections and lessons |
| 2.1.4 | PUT curriculum with invalid content_ref returns 400 | Integration | Returns `{ error: 'Invalid content references', missing: [...] }` |
| 2.1.5 | Learner cannot create a course | Integration | POST /api/v1/courses with learner role returns 403 |
| 2.1.6 | Instructor A cannot edit Instructor B's course | Integration | PATCH /api/v1/courses/:id by non-owner returns 403 |
| 2.1.7 | Course status transitions enforce valid flow | Unit | draft->published fails (must go through in_review); in_review->published succeeds |

---

### Task 2.2: Content Item Management

**What:** CRUD for content items (video, text, PDF, quiz, coding exercise). Content items are referenced by the curriculum JSONB via content_ref UUID. Each content type has a type-specific JSONB structure validated by Zod schemas.

**Design:**

```typescript
// apps/api/src/routes/content-items.ts
const videoContentSchema = z.object({
  video_url: z.string().url(),
  hls_url: z.string().url().optional(),
  duration_secs: z.number().int().nonnegative(),
  transcript_url: z.string().url().optional(),
  captions: z.array(z.object({
    language: z.string(),
    url: z.string().url(),
  })).optional(),
  encoding_status: z.enum(['pending', 'processing', 'ready', 'failed']).default('pending'),
});

const textContentSchema = z.object({
  body_html: z.string(),
  attachments: z.array(z.object({
    name: z.string(),
    url: z.string().url(),
    size_bytes: z.number().int(),
    mime_type: z.string(),
  })).optional(),
});

const contentSchemaMap = {
  video: videoContentSchema,
  text: textContentSchema,
  pdf: pdfContentSchema,
  quiz: quizContentSchema,
  coding_exercise: codingContentSchema,
};

app.post('/api/v1/courses/:courseId/content', {
  preHandler: [requireAuth, requireCourseOwner]
}, async (request, reply) => {
  const { contentType, title, content } = request.body;
  const schema = contentSchemaMap[contentType];
  if (!schema) return reply.code(400).send({ error: `Unknown content type: ${contentType}` });
  schema.parse(content); // validate type-specific structure
  const item = await prisma.contentItem.create({
    data: {
      courseId: request.params.courseId,
      contentType,
      title,
      content,
    },
  });
  return reply.code(201).send(item);
});
```

**Testing:**

| # | Test case | Type | Expected result |
|---|-----------|------|-----------------|
| 2.2.1 | Create video content item with valid payload | Integration | Content item created with contentType='video' and JSONB content stored |
| 2.2.2 | Create text content item with HTML body | Integration | Content item created; body_html retrievable |
| 2.2.3 | Invalid quiz content (missing items array) returns 400 | Integration | Zod validation error returned |
| 2.2.4 | Content item deletion removes it from course | Integration | DELETE returns 204; item no longer in database |
| 2.2.5 | Content items are scoped to course | Integration | GET /api/v1/courses/:id/content returns only items for that course |

---

### Task 2.3: Video Upload & Processing Pipeline

**What:** Presigned URL generation for direct S3 upload from the browser. On upload completion, a BullMQ job triggers video encoding (HLS adaptive bitrate via MediaConvert or Mux). Webhook callback updates the content item with HLS URL and encoding status.

**Design:**

```typescript
// apps/api/src/routes/uploads.ts
import { S3Client, PutObjectCommand } from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';

app.post('/api/v1/uploads/presigned-url', {
  preHandler: [requireAuth, requireRole('instructor')]
}, async (request, reply) => {
  const { fileName, contentType, courseId } = request.body;
  const key = `courses/${courseId}/raw/${crypto.randomUUID()}-${fileName}`;
  const command = new PutObjectCommand({
    Bucket: process.env.S3_BUCKET,
    Key: key,
    ContentType: contentType,
  });
  const url = await getSignedUrl(s3Client, command, { expiresIn: 3600 });
  return reply.send({ uploadUrl: url, key });
});

// apps/api/src/jobs/video-encoding.ts
import { Queue, Worker } from 'bullmq';

export const videoEncodingQueue = new Queue('video-encoding', { connection: redis });

const worker = new Worker('video-encoding', async (job) => {
  const { contentItemId, s3Key } = job.data;
  // Update status to processing
  await prisma.contentItem.update({
    where: { id: contentItemId },
    data: { content: { ...existing, encoding_status: 'processing' } },
  });
  // Trigger MediaConvert job (or Mux asset creation)
  const encodingJobId = await mediaConvert.createJob({
    input: `s3://${process.env.S3_BUCKET}/${s3Key}`,
    outputGroup: {
      type: 'HLS_GROUP_SETTINGS',
      destination: `s3://${process.env.S3_BUCKET}/courses/${job.data.courseId}/hls/`,
    },
    // Renditions: 360p, 720p, 1080p
  });
  await job.updateData({ ...job.data, encodingJobId });
}, { connection: redis });

// Webhook handler for encoding completion
app.post('/api/v1/webhooks/encoding-complete', async (request, reply) => {
  const { contentItemId, hlsUrl, duration, thumbnailUrl, status } = request.body;
  await prisma.contentItem.update({
    where: { id: contentItemId },
    data: {
      content: {
        ...existing,
        hls_url: hlsUrl,
        duration_secs: duration,
        encoding_status: status, // 'ready' or 'failed'
      },
    },
  });
  return reply.code(200).send({ ok: true });
});
```

**Testing:**

| # | Test case | Type | Expected result |
|---|-----------|------|-----------------|
| 2.3.1 | Presigned URL generation returns valid S3 upload URL | Integration | URL is valid; uploading a test file to it succeeds |
| 2.3.2 | Upload completion triggers video-encoding job | Integration | BullMQ job appears in video-encoding queue with correct payload |
| 2.3.3 | Encoding webhook updates content item status to 'ready' | Integration | Content item content JSONB has encoding_status='ready' and hls_url set |
| 2.3.4 | Failed encoding sets status to 'failed' | Integration | Content item content JSONB has encoding_status='failed' |
| 2.3.5 | Non-instructor cannot generate presigned URLs | Integration | POST /api/v1/uploads/presigned-url by learner returns 403 |

---

### Task 2.4: Category & Skill Tag Management

**What:** Admin CRUD for hierarchical categories (adjacency list with parent_id). Instructor-assignable skill tags. Seed initial categories from a curated taxonomy.

**Design:**

```typescript
// apps/api/src/routes/categories.ts
app.get('/api/v1/categories', async (request, reply) => {
  const categories = await prisma.category.findMany({
    where: { isActive: true },
    orderBy: [{ sortOrder: 'asc' }, { name: 'asc' }],
  });
  // Build tree structure from flat list
  const tree = buildCategoryTree(categories);
  return reply.send(tree);
});

function buildCategoryTree(categories: Category[]): CategoryNode[] {
  const map = new Map<string | null, CategoryNode[]>();
  for (const cat of categories) {
    const parentKey = cat.parentId ?? null;
    if (!map.has(parentKey)) map.set(parentKey, []);
    map.get(parentKey)!.push({ ...cat, children: [] });
  }
  function attach(nodes: CategoryNode[]): CategoryNode[] {
    for (const node of nodes) {
      node.children = map.get(node.id) ?? [];
      attach(node.children);
    }
    return nodes;
  }
  return attach(map.get(null) ?? []);
}
```

**Testing:**

| # | Test case | Type | Expected result |
|---|-----------|------|-----------------|
| 2.4.1 | GET /api/v1/categories returns tree structure | Integration | Top-level categories have `children` arrays; depth <= 3 |
| 2.4.2 | POST /api/v1/categories creates subcategory | Integration | Category created with parentId pointing to parent |
| 2.4.3 | Deleting parent category cascades to children | Integration | Parent and all child categories deactivated |
| 2.4.4 | Course can be assigned to multiple categories | Integration | Course.categoryIds updated; GET returns correct categories |
| 2.4.5 | Skill tags are unique, case-insensitive | Unit | Creating "Machine Learning" and "machine learning" produces one tag |

### Phase 2 Definition of Done

- [ ] Instructors can create, edit, and manage courses via API
- [ ] Curriculum JSONB stores sections and lessons with sort ordering
- [ ] Content items (video, text, PDF) are created and linked to curriculum
- [ ] Video uploads use presigned S3 URLs with background encoding to HLS
- [ ] Encoding status tracked and updated via webhook callbacks
- [ ] Categories support hierarchical tree structure (2-3 levels)
- [ ] Skill tags assignable to courses
- [ ] Course status transitions enforced (draft -> in_review -> published -> archived)
- [ ] All content creation validates type-specific JSONB schemas via Zod
- [ ] All test cases pass

---

## Phase 3: Enrollment & Course Player

**Goal:** Learners can browse the course catalog, enroll in free courses, and progress through lessons with tracked completion. The course player renders video, text, and PDF content with progress bookmarking.

**Duration estimate:** 3 weeks

### Task 3.1: Enrollment API

**What:** Endpoints for enrolling in a course (free courses only at this stage; paid enrollment added in Phase 4), listing enrolled courses, and managing enrollment status.

**Design:**

```typescript
// apps/api/src/routes/enrollments.ts
app.post('/api/v1/courses/:courseId/enroll', {
  preHandler: [requireAuth]
}, async (request, reply) => {
  const courseId = request.params.courseId;
  const course = await prisma.course.findUnique({ where: { id: courseId } });
  if (!course || course.status !== 'published') {
    return reply.code(404).send({ error: 'Course not found or not published' });
  }
  // Check if free (paid enrollment handled by checkout in Phase 4)
  const pricing = await prisma.pricing.findFirst({
    where: { courseId, isActive: true },
  });
  if (pricing && pricing.planType !== 'free') {
    return reply.code(402).send({ error: 'Course requires payment', checkoutUrl: `/checkout/${courseId}` });
  }
  const enrollment = await prisma.enrollment.upsert({
    where: { userId_courseId: { userId: request.user.id, courseId } },
    create: {
      userId: request.user.id,
      courseId,
      status: 'active',
      progressData: { lessons_completed: [], lessons_total: countLessons(course.curriculum) },
    },
    update: {}, // no-op if already enrolled
  });
  return reply.code(201).send(enrollment);
});
```

**Testing:**

| # | Test case | Type | Expected result |
|---|-----------|------|-----------------|
| 3.1.1 | Enroll in free published course succeeds | Integration | Enrollment created with status='active', progress_pct=0 |
| 3.1.2 | Enrolling in same course twice returns existing enrollment | Integration | Returns 201 with same enrollment ID; no duplicate row |
| 3.1.3 | Enroll in unpublished course returns 404 | Integration | 404 with "Course not found or not published" |
| 3.1.4 | Enroll in paid course returns 402 | Integration | Returns 402 with checkout URL |
| 3.1.5 | GET /api/v1/enrollments lists user's enrolled courses | Integration | Returns array with course title, progress, and last accessed |
| 3.1.6 | Unauthenticated enrollment attempt returns 401 | Integration | 401 Unauthorized |

---

### Task 3.2: Course Player UI

**What:** Next.js course player page with video playback (HLS via hls.js), text/PDF rendering, section navigation sidebar, progress tracking, and lesson bookmarking.

**Design:**

```tsx
// apps/web/src/app/courses/[slug]/learn/page.tsx
import { VideoPlayer } from '@/components/course-player/video-player';
import { CurriculumSidebar } from '@/components/course-player/curriculum-sidebar';
import { TextLesson } from '@/components/course-player/text-lesson';
import { ProgressBar } from '@/components/course-player/progress-bar';

export default async function CoursePlayerPage({ params, searchParams }) {
  const enrollment = await getEnrollment(params.slug);
  const lesson = getCurrentLesson(enrollment, searchParams.lesson);

  return (
    <div className="flex h-screen">
      <CurriculumSidebar
        curriculum={enrollment.course.curriculum}
        completedLessons={enrollment.progressData.lessons_completed}
        currentLessonId={lesson.lesson_id}
      />
      <main className="flex-1 overflow-y-auto">
        <ProgressBar percentage={enrollment.progressPct} />
        {lesson.type === 'video' && (
          <VideoPlayer
            hlsUrl={lesson.content.hls_url}
            captions={lesson.content.captions}
            startPosition={enrollment.progressData.current_video_position}
            onProgress={handleVideoProgress}
          />
        )}
        {lesson.type === 'text' && (
          <TextLesson html={lesson.content.body_html} attachments={lesson.content.attachments} />
        )}
      </main>
    </div>
  );
}

// apps/web/src/components/course-player/video-player.tsx
'use client';
import Hls from 'hls.js';
import { useEffect, useRef } from 'react';

export function VideoPlayer({ hlsUrl, captions, startPosition, onProgress }) {
  const videoRef = useRef<HTMLVideoElement>(null);

  useEffect(() => {
    if (!videoRef.current || !hlsUrl) return;
    const hls = new Hls();
    hls.loadSource(hlsUrl);
    hls.attachMedia(videoRef.current);
    hls.on(Hls.Events.MANIFEST_PARSED, () => {
      videoRef.current!.currentTime = startPosition || 0;
    });
    return () => hls.destroy();
  }, [hlsUrl, startPosition]);

  return (
    <video
      ref={videoRef}
      controls
      onTimeUpdate={(e) => onProgress(e.currentTarget.currentTime)}
      className="w-full aspect-video bg-black"
    />
  );
}
```

**Testing:**

| # | Test case | Type | Expected result |
|---|-----------|------|-----------------|
| 3.2.1 | Course player page loads with curriculum sidebar | E2E | Sidebar shows all sections and lessons; current lesson is highlighted |
| 3.2.2 | Video lesson plays HLS stream | E2E | hls.js initializes; video plays adaptive bitrate |
| 3.2.3 | Text lesson renders HTML content safely | E2E | HTML rendered; XSS payloads in body_html are sanitized |
| 3.2.4 | Clicking a lesson in sidebar navigates to that lesson | E2E | URL updates; main content area shows correct lesson |
| 3.2.5 | Progress bar reflects enrollment progress percentage | E2E | Bar width matches progressPct value |
| 3.2.6 | Non-enrolled user redirected to course landing page | E2E | Accessing /learn without enrollment redirects to /courses/[slug] |

---

### Task 3.3: Progress Tracking API

**What:** Endpoint to mark lessons complete, update video position, and recalculate enrollment progress percentage. Debounced video position updates.

**Design:**

```typescript
// apps/api/src/routes/progress.ts
app.post('/api/v1/enrollments/:enrollmentId/lessons/:lessonId/complete', {
  preHandler: [requireAuth, requireEnrollmentOwner]
}, async (request, reply) => {
  const { enrollmentId, lessonId } = request.params;
  const enrollment = await prisma.enrollment.findUnique({ where: { id: enrollmentId } });
  const progressData = enrollment.progressData as ProgressData;

  if (!progressData.lessons_completed.includes(lessonId)) {
    progressData.lessons_completed.push(lessonId);
  }
  progressData.last_accessed = new Date().toISOString();

  const progressPct = (progressData.lessons_completed.length / progressData.lessons_total) * 100;
  const isComplete = progressPct >= 100;

  await prisma.enrollment.update({
    where: { id: enrollmentId },
    data: {
      progressData,
      progressPct: Math.min(progressPct, 100),
      completedAt: isComplete ? new Date() : null,
      status: isComplete ? 'completed' : 'active',
    },
  });

  return reply.send({ progressPct, isComplete });
});

// Debounced video position update (called every 10 seconds from player)
app.put('/api/v1/enrollments/:enrollmentId/video-position', {
  preHandler: [requireAuth, requireEnrollmentOwner]
}, async (request, reply) => {
  const { lessonId, position } = request.body;
  await prisma.enrollment.update({
    where: { id: request.params.enrollmentId },
    data: {
      progressData: {
        ...existingProgressData,
        current_lesson_id: lessonId,
        current_video_position: position,
        last_accessed: new Date().toISOString(),
      },
    },
  });
  return reply.code(204).send();
});
```

**Testing:**

| # | Test case | Type | Expected result |
|---|-----------|------|-----------------|
| 3.3.1 | Mark lesson complete updates progress percentage | Integration | progressPct recalculated; lesson_id added to lessons_completed array |
| 3.3.2 | Marking same lesson complete twice is idempotent | Integration | lessons_completed array unchanged; progressPct same |
| 3.3.3 | Completing all lessons sets enrollment status to 'completed' | Integration | status='completed', completedAt set, progressPct=100 |
| 3.3.4 | Video position update saves current position | Integration | progressData.current_video_position updated |
| 3.3.5 | User cannot update progress on another user's enrollment | Integration | Returns 403 |

### Phase 3 Definition of Done

- [ ] Learners can enroll in free published courses
- [ ] Course player renders video (HLS), text, and PDF lessons
- [ ] Curriculum sidebar shows sections, lessons, and completion status
- [ ] Lesson completion tracked with automatic progress percentage calculation
- [ ] Video position bookmarked and restored on return
- [ ] Enrollment list API returns user's courses with progress
- [ ] WCAG 2.2 AA compliance verified for course player (keyboard navigation, focus management, captions)
- [ ] All test cases pass

---

## Phase 4: Payments & Creator Economics

**Goal:** Learners can purchase courses via Stripe Checkout. Instructors receive payouts via Stripe Connect. Coupon/discount management. Revenue analytics dashboard.

**Duration estimate:** 4 weeks

### Task 4.1: Stripe Integration & Checkout Flow

**What:** Stripe Connect onboarding for instructors (Express accounts). Stripe Checkout for course purchases with automatic platform fee deduction. Webhook handling for payment completion, which triggers enrollment creation.

**Design:**

```typescript
// apps/api/src/routes/checkout.ts
import Stripe from 'stripe';
const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!);

app.post('/api/v1/checkout', { preHandler: [requireAuth] }, async (request, reply) => {
  const { courseId, pricingId, couponCode } = request.body;
  const pricing = await prisma.pricing.findUnique({ where: { id: pricingId } });
  const course = await prisma.course.findUnique({ where: { id: courseId } });
  const org = await prisma.organisation.findUnique({ where: { id: course.organisationId } });

  let discount = 0;
  if (couponCode) {
    const coupon = await validateCoupon(couponCode, courseId);
    discount = calculateDiscount(coupon, pricing.basePrice);
  }

  const platformFeePct = 0.15; // 15% platform fee
  const amount = Math.round((pricing.basePrice - discount) * 100); // cents
  const platformFee = Math.round(amount * platformFeePct);

  const session = await stripe.checkout.sessions.create({
    mode: pricing.planType === 'subscription' ? 'subscription' : 'payment',
    customer_email: request.user.email,
    line_items: [{
      price_data: {
        currency: pricing.currencyCode.toLowerCase(),
        product_data: { name: course.title },
        unit_amount: amount,
      },
      quantity: 1,
    }],
    payment_intent_data: {
      application_fee_amount: platformFee,
      transfer_data: { destination: org.settings.payout_config.stripe_account_id },
    },
    metadata: { courseId, userId: request.user.id, pricingId, couponCode },
    success_url: `${process.env.WEB_URL}/courses/${course.slug}/learn?enrolled=true`,
    cancel_url: `${process.env.WEB_URL}/courses/${course.slug}`,
  });

  return reply.send({ checkoutUrl: session.url });
});

// Stripe webhook handler
app.post('/api/v1/webhooks/stripe', async (request, reply) => {
  const event = stripe.webhooks.constructEvent(
    request.rawBody, request.headers['stripe-signature'], process.env.STRIPE_WEBHOOK_SECRET!
  );
  switch (event.type) {
    case 'checkout.session.completed': {
      const session = event.data.object;
      await prisma.$transaction(async (tx) => {
        const order = await tx.order.create({
          data: {
            userId: session.metadata.userId,
            status: 'completed',
            currencyCode: session.currency.toUpperCase(),
            total: session.amount_total / 100,
            items: [{ course_id: session.metadata.courseId, amount: session.amount_total / 100 }],
            payment: { stripe_checkout_session_id: session.id, stripe_payment_intent_id: session.payment_intent },
            completedAt: new Date(),
          },
        });
        await tx.enrollment.create({
          data: {
            userId: session.metadata.userId,
            courseId: session.metadata.courseId,
            status: 'active',
            progressData: { lessons_completed: [], lessons_total: 0 },
          },
        });
      });
      break;
    }
  }
  return reply.code(200).send({ received: true });
});
```

**Testing:**

| # | Test case | Type | Expected result |
|---|-----------|------|-----------------|
| 4.1.1 | POST /api/v1/checkout returns Stripe Checkout URL | Integration | Valid Stripe session URL returned; metadata includes courseId and userId |
| 4.1.2 | Checkout with valid coupon applies discount | Integration | Stripe session amount reflects discounted price |
| 4.1.3 | Checkout with invalid coupon returns 400 | Integration | Error message identifies invalid coupon |
| 4.1.4 | Stripe webhook creates order and enrollment on success | Integration | Order with status='completed' and enrollment with status='active' created |
| 4.1.5 | Duplicate webhook delivery is idempotent | Integration | Second delivery does not create duplicate order or enrollment |
| 4.1.6 | Platform fee calculated correctly at 15% | Unit | For $100 course, platform fee is $15, instructor receives $85 |

---

### Task 4.2: Coupon & Discount Management

**What:** Instructor-created coupons with percentage or fixed discounts, usage limits, expiry dates, and course-specific or category-wide applicability.

**Design:**

```typescript
// apps/api/src/routes/coupons.ts
app.post('/api/v1/coupons', {
  preHandler: [requireAuth, requireRole('instructor')]
}, async (request, reply) => {
  const data = createCouponSchema.parse(request.body);
  const coupon = await prisma.coupon.create({
    data: {
      code: data.code.toUpperCase(),
      discountType: data.discountType,
      discountValue: data.discountValue,
      config: {
        currency_code: data.currencyCode,
        max_uses: data.maxUses,
        max_uses_per_user: data.maxUsesPerUser ?? 1,
        applicable_course_ids: data.courseIds,
        valid_from: data.validFrom,
        valid_until: data.validUntil,
      },
      createdBy: request.user.id,
    },
  });
  return reply.code(201).send(coupon);
});

async function validateCoupon(code: string, courseId: string): Promise<Coupon> {
  const coupon = await prisma.coupon.findUnique({ where: { code } });
  if (!coupon) throw new CouponError('Coupon not found');
  const config = coupon.config as CouponConfig;
  if (config.valid_until && new Date(config.valid_until) < new Date()) throw new CouponError('Coupon expired');
  if (config.max_uses && coupon.usesCount >= config.max_uses) throw new CouponError('Coupon usage limit reached');
  if (config.applicable_course_ids?.length && !config.applicable_course_ids.includes(courseId)) {
    throw new CouponError('Coupon not applicable to this course');
  }
  return coupon;
}
```

**Testing:**

| # | Test case | Type | Expected result |
|---|-----------|------|-----------------|
| 4.2.1 | Create coupon with percentage discount | Integration | Coupon created; code stored uppercase |
| 4.2.2 | Validate expired coupon throws error | Unit | CouponError with 'Coupon expired' |
| 4.2.3 | Validate coupon at max uses throws error | Unit | CouponError with 'Coupon usage limit reached' |
| 4.2.4 | Validate coupon for wrong course throws error | Unit | CouponError with 'Coupon not applicable' |
| 4.2.5 | Using coupon increments usesCount | Integration | After checkout, coupon.usesCount incremented by 1 |

---

### Task 4.3: Instructor Payout System

**What:** Monthly payout calculation job that aggregates instructor earnings, deducts platform fees and affiliate commissions, and initiates Stripe Connect transfers. Payout dashboard API.

**Design:**

```typescript
// apps/api/src/jobs/payout-calculation.ts
// Runs on 1st of each month via cron
export async function calculatePayouts(periodStart: Date, periodEnd: Date) {
  const instructors = await prisma.user.findMany({
    where: { roles: { has: 'instructor' } },
    include: { organisationMemberships: { include: { organisation: true } } },
  });

  for (const instructor of instructors) {
    const orders = await prisma.order.findMany({
      where: {
        status: 'completed',
        completedAt: { gte: periodStart, lt: periodEnd },
        // Filter by courses owned by this instructor
      },
    });

    const grossRevenue = orders.reduce((sum, o) => sum + o.total, 0);
    const platformFee = grossRevenue * 0.15;
    const affiliateFees = calculateAffiliateFees(orders);
    const refunds = await calculateRefunds(instructor.id, periodStart, periodEnd);
    const netPayout = grossRevenue - platformFee - affiliateFees - refunds;

    if (netPayout > 0) {
      await prisma.instructorPayout.create({
        data: {
          instructorId: instructor.id,
          organisationId: instructor.organisationMemberships[0].organisationId,
          periodStart,
          periodEnd,
          summary: { gross_revenue: grossRevenue, platform_fee: platformFee,
                     affiliate_fees: affiliateFees, refunds, net_payout: netPayout,
                     currency: 'USD', order_count: orders.length },
          status: 'pending',
        },
      });
    }
  }
}
```

**Testing:**

| # | Test case | Type | Expected result |
|---|-----------|------|-----------------|
| 4.3.1 | Payout calculation aggregates correct revenue | Unit | Gross = sum of orders; net = gross - 15% - affiliates - refunds |
| 4.3.2 | Zero-revenue month creates no payout record | Unit | No instructor_payout row created |
| 4.3.3 | GET /api/v1/instructor/payouts returns payout history | Integration | Array of payouts with period, amounts, and status |
| 4.3.4 | Stripe transfer initiated for approved payouts | Integration | stripe_transfer_id populated; status changes to 'processing' |

### Phase 4 Definition of Done

- [ ] Stripe Connect onboarding flow for instructors
- [ ] Stripe Checkout creates sessions with platform fee and instructor destination
- [ ] Webhook handler creates orders and enrollments on payment success
- [ ] Coupon CRUD with validation (expiry, usage limits, course scope)
- [ ] Monthly payout calculation job aggregates and creates payout records
- [ ] Instructor payout dashboard shows earnings history
- [ ] Refund flow reverts enrollment and updates order status
- [ ] PCI DSS compliance achieved via Stripe (SAQ-A level)
- [ ] All test cases pass

---

## Phase 5: Reviews, Discussion & Community

**Goal:** Learners can review courses, participate in per-course and per-lesson discussion forums, and build learner profiles.

**Duration estimate:** 2 weeks

### Task 5.1: Course Reviews

**What:** Submit, edit, and display course reviews with star ratings. One review per user per course. Verified-purchase badge. Review aggregation updates course stats. Instructor can respond to reviews.

**Design:**

```typescript
// apps/api/src/routes/reviews.ts
app.post('/api/v1/courses/:courseId/reviews', {
  preHandler: [requireAuth]
}, async (request, reply) => {
  const enrollment = await prisma.enrollment.findUnique({
    where: { userId_courseId: { userId: request.user.id, courseId: request.params.courseId } },
  });
  if (!enrollment) return reply.code(403).send({ error: 'Must be enrolled to review' });

  const data = createReviewSchema.parse(request.body);
  const review = await prisma.review.create({
    data: {
      courseId: request.params.courseId,
      userId: request.user.id,
      rating: data.rating,
      title: data.title,
      body: data.body,
      metadata: { is_verified_purchase: true, completion_pct_at_review: enrollment.progressPct },
    },
  });

  // Update course aggregate stats
  await updateCourseRatingStats(request.params.courseId);
  return reply.code(201).send(review);
});
```

**Testing:**

| # | Test case | Type | Expected result |
|---|-----------|------|-----------------|
| 5.1.1 | Enrolled user creates review | Integration | Review created with verified_purchase=true |
| 5.1.2 | Non-enrolled user cannot review | Integration | Returns 403 |
| 5.1.3 | Second review by same user returns 409 | Integration | Unique constraint violation |
| 5.1.4 | Course avg_rating recalculated after review | Integration | Course stats JSONB updated with correct avg and count |
| 5.1.5 | Rating must be 1-5 | Unit | Rating of 0 or 6 rejected by validation |

---

### Task 5.2: Discussion Forums

**What:** Per-course discussion threads with replies (self-referencing discussion table). Optional lesson scope. Thread pinning, resolution marking, and instructor answers.

**Design:**

```typescript
// apps/api/src/routes/discussions.ts
app.post('/api/v1/courses/:courseId/discussions', {
  preHandler: [requireAuth, requireEnrolledOrInstructor]
}, async (request, reply) => {
  const data = createDiscussionSchema.parse(request.body);
  const thread = await prisma.discussion.create({
    data: {
      courseId: request.params.courseId,
      userId: request.user.id,
      lessonId: data.lessonId, // optional
      title: data.title,
      body: data.body,
      metadata: {},
    },
  });
  return reply.code(201).send(thread);
});

// Reply to a thread
app.post('/api/v1/discussions/:threadId/replies', {
  preHandler: [requireAuth]
}, async (request, reply) => {
  const data = createReplySchema.parse(request.body);
  const replyPost = await prisma.discussion.create({
    data: {
      courseId: request.body.courseId,
      parentId: request.params.threadId,
      userId: request.user.id,
      body: data.body,
      metadata: {},
    },
  });
  return reply.code(201).send(replyPost);
});
```

**Testing:**

| # | Test case | Type | Expected result |
|---|-----------|------|-----------------|
| 5.2.1 | Create discussion thread in a course | Integration | Thread created with parentId=null |
| 5.2.2 | Reply to thread creates child post | Integration | Discussion created with parentId pointing to thread |
| 5.2.3 | Thread scoped to lesson returns only for that lesson | Integration | GET with lessonId filter returns matching threads only |
| 5.2.4 | Instructor can pin a thread | Integration | metadata.is_pinned set to true |
| 5.2.5 | Instructor can mark reply as answer | Integration | metadata.is_answer set to true; thread metadata.is_resolved=true |
| 5.2.6 | Non-enrolled user cannot create thread | Integration | Returns 403 |

### Phase 5 Definition of Done

- [ ] Course reviews with 1-5 star rating and text body
- [ ] One review per user per course enforced
- [ ] Review aggregation updates course stats
- [ ] Discussion threads per course with optional lesson scope
- [ ] Threaded replies with nesting support
- [ ] Instructor can pin threads and mark answers
- [ ] All test cases pass

---

## Phase 6: Assessments & Credentials

**Goal:** Quizzes with auto-grading (multiple choice, true/false, short answer). Assessment attempts tracked. Completion certificates generated as PDFs with verifiable URLs.

**Duration estimate:** 3 weeks

### Task 6.1: Quiz Engine

**What:** Quiz rendering and auto-grading for multiple-choice, multiple-select, and true/false items. Items stored in content_item JSONB following QTI patterns. Attempt tracking with score calculation.

**Design:**

```typescript
// apps/api/src/routes/assessments.ts
app.post('/api/v1/enrollments/:enrollmentId/assessments/:contentItemId/submit', {
  preHandler: [requireAuth, requireEnrollmentOwner]
}, async (request, reply) => {
  const contentItem = await prisma.contentItem.findUnique({ where: { id: request.params.contentItemId } });
  const quizConfig = contentItem.content as QuizContent;

  // Check attempt limits
  const attemptCount = await prisma.assessmentAttempt.count({
    where: { enrollmentId: request.params.enrollmentId, contentItemId: request.params.contentItemId },
  });
  if (quizConfig.max_attempts > 0 && attemptCount >= quizConfig.max_attempts) {
    return reply.code(403).send({ error: 'Maximum attempts reached' });
  }

  const responses = request.body.responses as SubmittedResponse[];
  const gradedResponses = gradeResponses(quizConfig.items, responses);
  const totalPoints = gradedResponses.reduce((sum, r) => sum + r.points, 0);
  const maxPoints = quizConfig.items.reduce((sum, i) => sum + i.points, 0);
  const score = (totalPoints / maxPoints) * 100;
  const passed = score >= quizConfig.passing_score;

  const attempt = await prisma.assessmentAttempt.create({
    data: {
      enrollmentId: request.params.enrollmentId,
      contentItemId: request.params.contentItemId,
      attemptNumber: attemptCount + 1,
      score,
      passed,
      responses: gradedResponses,
      submittedAt: new Date(),
    },
  });

  return reply.send({ attempt, score, passed, gradedResponses });
});

function gradeResponses(items: QuizItem[], responses: SubmittedResponse[]): GradedResponse[] {
  return responses.map(response => {
    const item = items.find(i => i.item_id === response.item_id);
    if (!item) return { ...response, is_correct: false, points: 0 };
    switch (item.type) {
      case 'multiple_choice':
      case 'true_false': {
        const correct = item.options.filter(o => o.is_correct).map(o => o.id);
        const isCorrect = JSON.stringify(response.selected.sort()) === JSON.stringify(correct.sort());
        return { ...response, is_correct: isCorrect, points: isCorrect ? item.points : 0 };
      }
      // ... other types
    }
  });
}
```

**Testing:**

| # | Test case | Type | Expected result |
|---|-----------|------|-----------------|
| 6.1.1 | Submit quiz with all correct answers scores 100% | Integration | score=100, passed=true |
| 6.1.2 | Submit quiz with mixed answers calculates partial score | Integration | Score reflects correct/incorrect ratio |
| 6.1.3 | Exceeding max_attempts returns 403 | Integration | Error "Maximum attempts reached" |
| 6.1.4 | Quiz with passing_score=70 fails at 65% | Integration | passed=false, score=65 |
| 6.1.5 | Graded responses include per-item is_correct and explanation | Integration | Each response has is_correct boolean and associated points |

---

### Task 6.2: Certificate Generation

**What:** PDF certificate generation on course completion using HTML templates rendered to PDF. Verifiable credential URL with Open Badges 3.0 JSON-LD metadata.

**Design:**

```typescript
// apps/api/src/services/certificate-generator.ts
import puppeteer from 'puppeteer';

export async function generateCertificate(
  enrollment: Enrollment, course: Course, user: User
): Promise<Credential> {
  const verificationUrl = `${process.env.WEB_URL}/verify/${crypto.randomUUID()}`;

  const credentialData = {
    '@context': [
      'https://www.w3.org/ns/credentials/v2',
      'https://purl.imsglobal.org/spec/ob/v3p0/context-3.0.3.json',
    ],
    type: ['VerifiableCredential', 'OpenBadgeCredential'],
    issuer: { id: process.env.WEB_URL, name: process.env.PLATFORM_NAME },
    validFrom: new Date().toISOString(),
    credentialSubject: {
      type: 'AchievementSubject',
      identifier: { type: 'email', value: user.email },
      achievement: {
        name: course.title,
        description: `Completed ${course.title}`,
        criteria: { narrative: 'Completed all required lessons and assessments' },
      },
    },
  };

  // Generate PDF from HTML template
  const html = renderCertificateTemplate({ user, course, date: new Date(), verificationUrl });
  const browser = await puppeteer.launch({ headless: true });
  const page = await browser.newPage();
  await page.setContent(html);
  const pdfBuffer = await page.pdf({ format: 'A4', landscape: true });
  await browser.close();

  // Upload PDF to S3
  const pdfUrl = await uploadToS3(pdfBuffer, `certificates/${enrollment.id}.pdf`);

  return prisma.credential.create({
    data: {
      courseId: course.id,
      userId: user.id,
      enrollmentId: enrollment.id,
      credentialType: 'completion',
      verificationUrl,
      credentialData,
      templateHtml: html,
      badgeImageUrl: course.media?.thumbnail_url,
    },
  });
}

// Verification endpoint
app.get('/verify/:id', async (request, reply) => {
  const credential = await prisma.credential.findUnique({
    where: { verificationUrl: `${process.env.WEB_URL}/verify/${request.params.id}` },
    include: { user: { select: { displayName: true } }, course: { select: { title: true } } },
  });
  if (!credential) return reply.code(404).send({ error: 'Credential not found' });
  if (credential.revokedAt) return reply.send({ valid: false, revoked: true, reason: credential.revocationReason });
  return reply.send({ valid: true, credential: credential.credentialData });
});
```

**Testing:**

| # | Test case | Type | Expected result |
|---|-----------|------|-----------------|
| 6.2.1 | Course completion triggers certificate generation | Integration | Credential row created; PDF uploaded to S3; verification URL set |
| 6.2.2 | Certificate PDF renders with correct name and course title | Unit | PDF contains user display name and course title text |
| 6.2.3 | Verification URL returns valid credential JSON-LD | Integration | GET /verify/:id returns Open Badges 3.0 JSON-LD with valid=true |
| 6.2.4 | Revoked credential returns valid=false | Integration | After revocation, GET returns valid=false with reason |
| 6.2.5 | Invalid verification URL returns 404 | Integration | GET /verify/nonexistent returns 404 |

### Phase 6 Definition of Done

- [ ] Quiz engine grades multiple-choice, multiple-select, and true/false items
- [ ] Attempt tracking with configurable max attempts and passing score
- [ ] PDF certificates generated on course completion
- [ ] Open Badges 3.0 JSON-LD credential metadata stored
- [ ] Public verification endpoint validates credentials
- [ ] Credential revocation supported
- [ ] All test cases pass

---

## Phase 7: Marketplace Discovery & Search

**Goal:** Browsable course catalog with faceted search, category filtering, sorting, and personalized recommendations (basic, before AI engine).

**Duration estimate:** 3 weeks

### Task 7.1: Meilisearch Integration

**What:** Index published courses into Meilisearch. Typo-tolerant search with facets (category, level, language, price range, rating). Real-time index updates on course publish/unpublish.

**Design:**

```typescript
// apps/api/src/services/search-indexer.ts
import { MeiliSearch } from 'meilisearch';
const meili = new MeiliSearch({ host: process.env.MEILI_URL, apiKey: process.env.MEILI_API_KEY });

const coursesIndex = meili.index('courses');

export async function indexCourse(course: Course) {
  await coursesIndex.addDocuments([{
    id: course.id,
    title: course.title,
    subtitle: course.subtitle,
    description: course.description,
    instructorName: course.instructor?.displayName,
    level: course.level,
    languageCode: course.languageCode,
    categoryNames: course.categories?.map(c => c.name),
    skillTags: course.skillTags,
    avgRating: course.stats?.avg_rating ?? 0,
    enrollmentCount: course.stats?.enrollment_count ?? 0,
    basePrice: course.pricing?.basePrice ?? 0,
    isFree: course.pricing?.planType === 'free',
    thumbnailUrl: course.media?.thumbnail_url,
    publishedAt: course.publishedAt?.toISOString(),
  }]);
}

// Configure index settings
await coursesIndex.updateSettings({
  searchableAttributes: ['title', 'subtitle', 'description', 'instructorName', 'skillTags'],
  filterableAttributes: ['level', 'languageCode', 'categoryNames', 'isFree', 'avgRating', 'basePrice'],
  sortableAttributes: ['avgRating', 'enrollmentCount', 'publishedAt', 'basePrice'],
  rankingRules: ['words', 'typo', 'proximity', 'attribute', 'sort', 'exactness'],
});
```

**Testing:**

| # | Test case | Type | Expected result |
|---|-----------|------|-----------------|
| 7.1.1 | Published course indexed in Meilisearch | Integration | Document retrievable by ID from Meilisearch |
| 7.1.2 | Search by title returns matching courses | Integration | Query "machine learning" returns courses with ML in title |
| 7.1.3 | Typo-tolerant search works | Integration | Query "mahcine lerning" returns ML courses |
| 7.1.4 | Filter by level=intermediate returns correct subset | Integration | Only intermediate courses in results |
| 7.1.5 | Sort by avgRating descending works | Integration | Results ordered by rating high-to-low |
| 7.1.6 | Unpublished course removed from index | Integration | After archiving, course no longer in search results |

---

### Task 7.2: Catalog Browse UI

**What:** Next.js catalog pages with server-side rendering for SEO. Category browse, search bar with autocomplete, faceted filters, and course cards with ratings, price, and enrollment count.

**Design:**

```tsx
// apps/web/src/app/courses/page.tsx
export default async function CourseCatalogPage({ searchParams }) {
  const { q, category, level, sort, page } = searchParams;
  const results = await searchCourses({ q, category, level, sort, page: Number(page) || 1 });

  return (
    <div className="container mx-auto px-4 py-8">
      <SearchBar defaultValue={q} />
      <div className="flex gap-8 mt-6">
        <FilterSidebar
          categories={await getCategories()}
          levels={['beginner', 'intermediate', 'advanced']}
          selectedCategory={category}
          selectedLevel={level}
        />
        <div className="flex-1">
          <SortControls value={sort} />
          <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6 mt-4">
            {results.hits.map(course => (
              <CourseCard key={course.id} course={course} />
            ))}
          </div>
          <Pagination total={results.totalHits} page={Number(page) || 1} perPage={20} />
        </div>
      </div>
    </div>
  );
}

// apps/web/src/components/course-card.tsx
export function CourseCard({ course }) {
  return (
    <Link href={`/courses/${course.slug}`} className="block border rounded-lg overflow-hidden hover:shadow-lg">
      <img src={course.thumbnailUrl} alt={course.title} className="w-full aspect-video object-cover" />
      <div className="p-4">
        <h3 className="font-semibold text-lg line-clamp-2">{course.title}</h3>
        <p className="text-sm text-gray-600 mt-1">{course.instructorName}</p>
        <div className="flex items-center gap-2 mt-2">
          <StarRating value={course.avgRating} />
          <span className="text-sm text-gray-500">({course.enrollmentCount})</span>
        </div>
        <p className="font-bold mt-2">{course.isFree ? 'Free' : `$${course.basePrice}`}</p>
      </div>
    </Link>
  );
}
```

**Testing:**

| # | Test case | Type | Expected result |
|---|-----------|------|-----------------|
| 7.2.1 | Catalog page server-renders with course cards | E2E | Page HTML contains course titles (SEO-visible) |
| 7.2.2 | Search filters update URL params and results | E2E | Selecting "intermediate" adds ?level=intermediate; results update |
| 7.2.3 | Category browse shows subcategories | E2E | Clicking "Programming" shows subcategories and filtered courses |
| 7.2.4 | Course card displays rating, price, instructor | E2E | Card shows all metadata; clicking navigates to course detail |
| 7.2.5 | Pagination loads next page of results | E2E | Page 2 shows different courses; URL updates to ?page=2 |
| 7.2.6 | Empty search shows "No courses found" message | E2E | Query "xyznonexistent" shows empty state |

---

### Task 7.3: Course Landing Page & SEO

**What:** Public course detail page with structured data (Schema.org Course), social meta tags, instructor bio, curriculum preview, reviews, and enrollment CTA.

**Design:**

```tsx
// apps/web/src/app/courses/[slug]/page.tsx
export async function generateMetadata({ params }): Promise<Metadata> {
  const course = await getCourseBySlug(params.slug);
  return {
    title: `${course.title} | CourseMarket`,
    description: course.subtitle || course.description?.slice(0, 160),
    openGraph: { images: [course.media?.og_image_url || course.media?.thumbnail_url] },
    other: {
      'application/ld+json': JSON.stringify({
        '@context': 'https://schema.org',
        '@type': 'Course',
        name: course.title,
        description: course.description,
        provider: { '@type': 'Organization', name: course.organisation?.name },
        hasCourseInstance: {
          '@type': 'CourseInstance',
          courseMode: 'online',
          instructor: { '@type': 'Person', name: course.instructor?.displayName },
        },
        aggregateRating: {
          '@type': 'AggregateRating',
          ratingValue: course.stats?.avg_rating,
          reviewCount: course.stats?.rating_count,
        },
      }),
    },
  };
}
```

**Testing:**

| # | Test case | Type | Expected result |
|---|-----------|------|-----------------|
| 7.3.1 | Course landing page contains Schema.org JSON-LD | E2E | `<script type="application/ld+json">` present with Course type |
| 7.3.2 | OG meta tags render for social sharing | E2E | og:title, og:description, og:image present in head |
| 7.3.3 | Curriculum preview shows sections and lesson count | E2E | Sections listed with lesson titles; preview lessons marked as "Preview" |
| 7.3.4 | Reviews section shows ratings and recent reviews | E2E | Star ratings, review text, and verified badges displayed |

### Phase 7 Definition of Done

- [ ] Meilisearch indexes all published courses with faceted search
- [ ] Catalog page with search, filters, sorting, and pagination
- [ ] Course cards display rating, price, instructor, and enrollment count
- [ ] Course landing page with Schema.org structured data for SEO
- [ ] Category browse with hierarchical navigation
- [ ] Server-side rendering for all catalog pages (SEO requirement)
- [ ] All test cases pass

---

## Phase 8: AI Recommendation Engine & Learning Paths

**Goal:** Graph-based recommendation engine providing "similar courses", "learners also took", skill gap analysis, and personalized learning path generation. This phase adds the graph layer from Data Model Suggestion 4.

**Duration estimate:** 4 weeks

### Task 8.1: Graph Schema & Sync Pipeline

**What:** Create graph_node and graph_edge tables. Build CDC pipeline that syncs course, user, enrollment, review, and skill data from relational tables into the graph layer.

**Design:**

```sql
-- Migration: add graph tables (from data-model-suggestion-4)
CREATE TABLE graph_node (
    id          UUID PRIMARY KEY,
    node_type   VARCHAR(50) NOT NULL,
    label       VARCHAR(500) NOT NULL,
    properties  JSONB NOT NULL DEFAULT '{}',
    embedding   VECTOR(384),
    is_active   BOOLEAN NOT NULL DEFAULT TRUE,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_graph_node_type ON graph_node (node_type);

CREATE TABLE graph_edge (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_id   UUID NOT NULL REFERENCES graph_node (id) ON DELETE CASCADE,
    target_id   UUID NOT NULL REFERENCES graph_node (id) ON DELETE CASCADE,
    edge_type   VARCHAR(50) NOT NULL,
    weight      NUMERIC(5,3) DEFAULT 1.000,
    properties  JSONB NOT NULL DEFAULT '{}',
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX idx_graph_edge_unique ON graph_edge (source_id, target_id, edge_type);
CREATE INDEX idx_graph_edge_source ON graph_edge (source_id, edge_type);
CREATE INDEX idx_graph_edge_target ON graph_edge (target_id, edge_type);
```

```typescript
// apps/api/src/jobs/graph-sync.ts
// Triggered by enrollment, review, and course publish events
export async function syncEnrollmentToGraph(enrollment: Enrollment) {
  // Ensure user node exists
  await upsertGraphNode(enrollment.userId, 'User', enrollment.user.displayName, {
    roles: enrollment.user.roles,
    enrolled_count: await countEnrollments(enrollment.userId),
  });
  // Ensure course node exists
  await upsertGraphNode(enrollment.courseId, 'Course', enrollment.course.title, {
    level: enrollment.course.level,
    avg_rating: enrollment.course.stats?.avg_rating,
  });
  // Create ENROLLED_IN edge
  await upsertGraphEdge(enrollment.userId, enrollment.courseId, 'ENROLLED_IN', 1.0, {
    enrolled_at: enrollment.enrolledAt,
  });
}
```

**Testing:**

| # | Test case | Type | Expected result |
|---|-----------|------|-----------------|
| 8.1.1 | Graph tables created successfully | Migration | graph_node and graph_edge exist with correct columns and indexes |
| 8.1.2 | New enrollment creates User and Course nodes plus ENROLLED_IN edge | Integration | Three rows created across graph_node and graph_edge |
| 8.1.3 | Course completion updates edge type to COMPLETED | Integration | ENROLLED_IN edge updated or COMPLETED edge created |
| 8.1.4 | Review creates REVIEWED edge with rating weight | Integration | Edge weight matches review rating / 5 |
| 8.1.5 | Full sync job populates graph from existing relational data | Integration | All published courses, active users, and enrollments represented in graph |

---

### Task 8.2: Course Similarity & Embedding Pipeline

**What:** Python microservice that generates sentence embeddings for course titles and descriptions using sentence-transformers. Computes content-based and collaborative filtering similarity scores. Stores embeddings in pgvector and similarity scores in course_similarity table.

**Design:**

```python
# apps/ai-services/src/embeddings.py
from sentence_transformers import SentenceTransformer
from fastapi import FastAPI
import asyncpg

app = FastAPI()
model = SentenceTransformer('all-MiniLM-L6-v2')  # 384-dim

@app.post("/embeddings/generate")
async def generate_embeddings(course_ids: list[str] = None):
    pool = await asyncpg.create_pool(dsn=settings.DATABASE_URL)
    async with pool.acquire() as conn:
        query = "SELECT id, title, subtitle, description FROM course WHERE status = 'published'"
        if course_ids:
            query += f" AND id = ANY($1::uuid[])"
            courses = await conn.fetch(query, course_ids)
        else:
            courses = await conn.fetch(query)

        for course in courses:
            text = f"{course['title']}. {course['subtitle'] or ''}. {course['description'] or ''}"
            embedding = model.encode(text).tolist()
            await conn.execute(
                "UPDATE graph_node SET embedding = $1 WHERE id = $2",
                str(embedding), course['id']
            )
    return {"embedded": len(courses)}

@app.post("/similarity/compute")
async def compute_similarities():
    pool = await asyncpg.create_pool(dsn=settings.DATABASE_URL)
    async with pool.acquire() as conn:
        courses = await conn.fetch(
            "SELECT id, embedding FROM graph_node WHERE node_type = 'Course' AND embedding IS NOT NULL"
        )
        # Compute pairwise cosine similarity and store top-K
        for i, a in enumerate(courses):
            similarities = []
            for j, b in enumerate(courses):
                if i >= j: continue
                sim = cosine_similarity(a['embedding'], b['embedding'])
                if sim > 0.3:  # threshold
                    similarities.append((b['id'], sim))
            similarities.sort(key=lambda x: x[1], reverse=True)
            for target_id, score in similarities[:20]:  # top 20
                await conn.execute("""
                    INSERT INTO course_similarity (course_a_id, course_b_id, algorithm, similarity, computed_at)
                    VALUES (LEAST($1, $2), GREATEST($1, $2), 'content_based', $3, NOW())
                    ON CONFLICT (course_a_id, course_b_id, algorithm) DO UPDATE SET similarity = $3, computed_at = NOW()
                """, a['id'], target_id, score)
    return {"status": "computed"}
```

**Testing:**

| # | Test case | Type | Expected result |
|---|-----------|------|-----------------|
| 8.2.1 | Embedding generation produces 384-dim vectors | Unit | Vector length is 384; values are floats |
| 8.2.2 | Similar courses have high cosine similarity | Unit | Two ML courses score > 0.7; ML vs. cooking scores < 0.3 |
| 8.2.3 | course_similarity table populated with top-K pairs | Integration | Rows exist with similarity > threshold |
| 8.2.4 | GET /api/v1/courses/:id/similar returns similar courses | Integration | Returns ordered list of similar courses with similarity scores |

---

### Task 8.3: Personalized Learning Paths

**What:** API for generating personalized learning paths based on learner's completed courses, declared goals, and skill gaps. Uses graph traversal to sequence courses.

**Design:**

```typescript
// apps/api/src/routes/learning-paths.ts
app.post('/api/v1/learning-paths/generate', {
  preHandler: [requireAuth]
}, async (request, reply) => {
  const { targetSkills, targetOccupationId } = request.body;

  // Call AI service to generate path
  const response = await fetch(`${process.env.AI_SERVICE_URL}/learning-paths/generate`, {
    method: 'POST',
    body: JSON.stringify({
      userId: request.user.id,
      targetSkills,
      targetOccupationId,
    }),
  });
  const pathData = await response.json();

  const learningPath = await prisma.learningPath.create({
    data: {
      title: pathData.title,
      slug: generateSlug(pathData.title),
      description: pathData.description,
      createdBy: request.user.id,
      generationType: 'ai_personalized',
      targetOccupationId,
      targetSkills,
      estimatedHours: pathData.estimatedHours,
      metadata: { ai_model_version: pathData.modelVersion, confidence_score: pathData.confidence },
    },
  });

  for (const step of pathData.steps) {
    await prisma.learningPathStep.create({
      data: {
        pathId: learningPath.id,
        courseId: step.courseId,
        position: step.position,
        isOptional: step.isOptional,
        rationale: step.rationale,
      },
    });
  }

  return reply.code(201).send(learningPath);
});
```

**Testing:**

| # | Test case | Type | Expected result |
|---|-----------|------|-----------------|
| 8.3.1 | Generate learning path for target occupation | Integration | Path created with ordered course steps |
| 8.3.2 | Path excludes already-completed courses | Integration | Completed courses not in generated path |
| 8.3.3 | Path respects prerequisite ordering | Integration | Prerequisites appear before dependent courses |
| 8.3.4 | Skill gap analysis returns missing skills | Integration | Returns skills required for occupation minus learner's current skills |

### Phase 8 Definition of Done

- [ ] Graph tables (graph_node, graph_edge) created and populated via sync pipeline
- [ ] Course embeddings generated and stored in pgvector
- [ ] Content-based similarity scores computed and cached
- [ ] "Similar courses" API endpoint returns relevant results
- [ ] "Learners also took" collaborative filtering works
- [ ] Learning path generation from skill gaps and target occupation
- [ ] Skill gap analysis API functional
- [ ] All test cases pass

---

## Phase 9: SCORM, xAPI & LTI Integration

**Goal:** Import and deliver SCORM 1.2/2004 packages. Capture xAPI statements as learning records. LTI 1.3 provider for embedding courses in external LMS platforms.

**Duration estimate:** 4 weeks

### Task 9.1: SCORM Package Import & Runtime

**What:** Upload SCORM ZIP packages, parse imsmanifest.xml, store as content items, and serve the SCORM runtime adapter (JavaScript bridge between SCORM content and our tracking API).

**Design:**

```typescript
// packages/scorm-runtime/src/scorm-api-adapter.ts
// SCORM 1.2 API adapter injected into the iframe hosting the SCORM content
export class ScormApiAdapter {
  private cmiData: Record<string, string> = {};
  private enrollmentId: string;
  private contentItemId: string;

  constructor(enrollmentId: string, contentItemId: string) {
    this.enrollmentId = enrollmentId;
    this.contentItemId = contentItemId;
  }

  // SCORM 1.2 API
  LMSInitialize(): string { return 'true'; }
  LMSGetValue(key: string): string { return this.cmiData[key] || ''; }
  LMSSetValue(key: string, value: string): string {
    this.cmiData[key] = value;
    return 'true';
  }
  LMSCommit(): string {
    // Persist CMI data to server
    fetch(`/api/v1/scorm/tracking`, {
      method: 'POST',
      body: JSON.stringify({
        enrollmentId: this.enrollmentId,
        contentItemId: this.contentItemId,
        cmiData: this.cmiData,
      }),
    });
    return 'true';
  }
  LMSFinish(): string { this.LMSCommit(); return 'true'; }
  LMSGetLastError(): string { return '0'; }
  LMSGetErrorString(): string { return 'No error'; }
  LMSGetDiagnostic(): string { return ''; }
}
```

**Testing:**

| # | Test case | Type | Expected result |
|---|-----------|------|-----------------|
| 9.1.1 | Upload valid SCORM 1.2 ZIP extracts and parses manifest | Integration | Content item created with SCORM metadata in JSONB |
| 9.1.2 | SCORM content launches in iframe with API adapter injected | E2E | Content loads; LMSInitialize called; content renders |
| 9.1.3 | LMSSetValue persists CMI data to enrollment | Integration | enrollment.scorm_tracking JSONB updated with CMI values |
| 9.1.4 | SCORM completion status tracked | Integration | After LMSFinish, enrollment progress updated based on completion_status |
| 9.1.5 | Invalid SCORM package (missing manifest) returns error | Integration | Upload returns 400 with descriptive error |

---

### Task 9.2: xAPI Statement API (LRS)

**What:** Implement the xAPI statement endpoint (IEEE 9274.1.1) for receiving, storing, and querying learning activity statements.

**Design:**

```typescript
// apps/api/src/routes/xapi.ts
// xAPI Statement API (subset of LRS spec for internal use)
app.post('/api/v1/xapi/statements', {
  preHandler: [requireAuth]
}, async (request, reply) => {
  const statement = request.body;
  // Validate xAPI statement structure
  xapiStatementSchema.parse(statement);

  const stored = await prisma.xapiStatement.create({
    data: {
      statementId: statement.id || crypto.randomUUID(),
      actorId: request.user.id,
      verbIri: statement.verb.id,
      objectIri: statement.object.id,
      statement: statement, // full JSON
    },
  });

  return reply.code(200).send([stored.statementId]);
});

app.get('/api/v1/xapi/statements', {
  preHandler: [requireAuth]
}, async (request, reply) => {
  const { agent, verb, activity, since, until, limit } = request.query;
  const statements = await prisma.xapiStatement.findMany({
    where: {
      actorId: agent ? await resolveAgentId(agent) : undefined,
      verbIri: verb,
      objectIri: activity,
      storedAt: { gte: since ? new Date(since) : undefined, lte: until ? new Date(until) : undefined },
    },
    take: Math.min(Number(limit) || 100, 500),
    orderBy: { storedAt: 'desc' },
  });
  return reply.send({ statements: statements.map(s => s.statement) });
});
```

**Testing:**

| # | Test case | Type | Expected result |
|---|-----------|------|-----------------|
| 9.2.1 | POST xAPI statement stores and returns statement ID | Integration | Statement stored; response is array with UUID |
| 9.2.2 | GET statements filtered by verb returns matching records | Integration | Only statements with matching verb_iri returned |
| 9.2.3 | GET statements filtered by date range works | Integration | Only statements within since/until range returned |
| 9.2.4 | Invalid xAPI statement structure returns 400 | Integration | Missing required verb field returns validation error |

---

### Task 9.3: LTI 1.3 Provider

**What:** Implement LTI 1.3 tool provider endpoints (OIDC login initiation, launch, and grade passback) so courses can be embedded in external LMS platforms.

**Design:**

```typescript
// packages/lti-provider/src/lti-launch.ts
import { createRemoteJWKSet, jwtVerify } from 'jose';

export async function handleLtiLaunch(idToken: string, toolConfig: LtiToolConfig) {
  const JWKS = createRemoteJWKSet(new URL(toolConfig.config.jwks_url));
  const { payload } = await jwtVerify(idToken, JWKS, {
    issuer: toolConfig.config.issuer,
    audience: toolConfig.config.client_id,
  });

  // Extract LTI claims
  const ltiClaims = {
    deploymentId: payload['https://purl.imsglobal.org/spec/lti/claim/deployment_id'],
    targetLinkUri: payload['https://purl.imsglobal.org/spec/lti/claim/target_link_uri'],
    roles: payload['https://purl.imsglobal.org/spec/lti/claim/roles'],
    context: payload['https://purl.imsglobal.org/spec/lti/claim/context'],
    resourceLink: payload['https://purl.imsglobal.org/spec/lti/claim/resource_link'],
  };

  return { userId: payload.sub, email: payload.email, claims: ltiClaims };
}
```

**Testing:**

| # | Test case | Type | Expected result |
|---|-----------|------|-----------------|
| 9.3.1 | OIDC login initiation redirects to tool auth URL | Integration | 302 redirect with correct OIDC parameters |
| 9.3.2 | Valid LTI launch JWT creates session and redirects to course | Integration | User authenticated; course player loads |
| 9.3.3 | Invalid JWT signature returns 401 | Integration | JWT verification fails; 401 returned |
| 9.3.4 | Grade passback sends score to external LMS | Integration | AGS lineitem endpoint receives correct score |

### Phase 9 Definition of Done

- [ ] SCORM 1.2/2004 packages upload, parse, and launch in iframe
- [ ] SCORM runtime adapter persists CMI tracking data
- [ ] xAPI statement API receives, stores, and queries statements
- [ ] LTI 1.3 OIDC launch flow works with external LMS
- [ ] LTI grade passback sends scores to external LMS
- [ ] All test cases pass

---

## Phase 10: Enterprise Features

**Goal:** SAML 2.0 / OIDC SSO for enterprise customers. White-label branded academies. Advanced analytics dashboards for L&D teams.

**Duration estimate:** 3 weeks

### Task 10.1: Enterprise SSO (SAML 2.0 & OIDC)

**What:** SAML 2.0 and OIDC SSO integration for enterprise customers connecting to Okta, Azure AD, or Google Workspace. Organisation-level SSO configuration stored in organisation.settings JSONB.

**Design:**

```typescript
// apps/api/src/routes/sso.ts
import { SAML } from '@node-saml/node-saml';

app.post('/api/v1/sso/saml/:orgSlug/callback', async (request, reply) => {
  const org = await prisma.organisation.findUnique({ where: { slug: request.params.orgSlug } });
  const samlConfig = org.settings.sso?.saml;
  const saml = new SAML({
    callbackUrl: `${process.env.API_URL}/api/v1/sso/saml/${org.slug}/callback`,
    entryPoint: samlConfig.entryPoint,
    issuer: samlConfig.issuer,
    cert: samlConfig.cert,
  });

  const profile = await saml.validatePostResponseAsync(request.body);
  const user = await findOrCreateSSOUser(profile, org.id);
  const session = await createSession(user);
  return reply.redirect(`${process.env.WEB_URL}/dashboard?token=${session.token}`);
});
```

**Testing:**

| # | Test case | Type | Expected result |
|---|-----------|------|-----------------|
| 10.1.1 | SAML login redirect to IdP with correct parameters | Integration | Redirect URL includes SAML request and org-specific callback |
| 10.1.2 | SAML callback creates user on first login | Integration | User created with SSO provider data; linked to org |
| 10.1.3 | SAML callback logs in existing user | Integration | Session created; no duplicate user |
| 10.1.4 | Invalid SAML assertion returns 401 | Integration | Signature verification fails; 401 returned |

---

### Task 10.2: White-Label & Custom Domains

**What:** Organisation-level branding (logo, colors, custom domain) applied via Next.js middleware. Multi-tenant subdomain routing.

**Testing:**

| # | Test case | Type | Expected result |
|---|-----------|------|-----------------|
| 10.2.1 | Custom domain resolves to org-branded UI | E2E | Logo, colors, and domain match org settings |
| 10.2.2 | Default domain shows platform branding | E2E | Platform logo and default theme applied |
| 10.2.3 | Courses scoped to org on white-label domain | E2E | Only org's courses visible on custom domain |

---

### Task 10.3: Enterprise Analytics Dashboard

**What:** L&D team analytics: team enrollment rates, completion rates, skill gap reports, and compliance tracking. Powered by aggregation queries on enrollment and assessment data.

**Testing:**

| # | Test case | Type | Expected result |
|---|-----------|------|-----------------|
| 10.3.1 | Team completion report shows per-member progress | Integration | API returns array of users with course progress and completion dates |
| 10.3.2 | Skill gap report shows team deficiencies | Integration | API returns skills below threshold with member counts |
| 10.3.3 | CSV export of analytics data | Integration | Download returns valid CSV with correct headers and data |

### Phase 10 Definition of Done

- [ ] SAML 2.0 SSO integration with Okta and Azure AD
- [ ] OIDC SSO for Google Workspace
- [ ] White-label branding per organisation (logo, colors, custom domain)
- [ ] Multi-tenant subdomain routing in Next.js middleware
- [ ] Enterprise analytics dashboard with team progress and skill gap reports
- [ ] CSV export of analytics data
- [ ] All test cases pass

---

## Phase 11: AI Content Generation & Dynamic Pricing

**Goal:** AI-generated course scaffolding (outlines, quiz questions, lesson descriptions). Dynamic pricing recommendations based on demand signals and promotional history.

**Duration estimate:** 3 weeks

### Task 11.1: AI Course Scaffolding

**What:** Instructor provides a learning outcome prompt; AI generates module outlines, quiz questions, and lesson descriptions. Uses Claude API for generation with structured output.

**Design:**

```python
# apps/ai-services/src/course_generator.py
from anthropic import Anthropic

client = Anthropic()

@app.post("/generate/course-outline")
async def generate_course_outline(request: CourseOutlineRequest):
    message = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=4096,
        messages=[{
            "role": "user",
            "content": f"""Generate a detailed course outline for: {request.learning_outcome}
            Target level: {request.level}
            Estimated duration: {request.duration_hours} hours

            Return a JSON structure with sections, each containing lessons with:
            - title, type (video/text/quiz), estimated_duration_minutes, description, learning_objectives
            For quiz lessons, also include 3-5 sample questions with options and correct answers."""
        }],
    )
    outline = parse_outline(message.content[0].text)
    return outline

@app.post("/generate/quiz-questions")
async def generate_quiz_questions(request: QuizGenerationRequest):
    message = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=2048,
        messages=[{
            "role": "user",
            "content": f"""Generate {request.count} quiz questions for the topic: {request.topic}
            Difficulty: {request.difficulty}
            Types: multiple_choice, true_false, short_answer

            Return as JSON array following QTI item patterns with question_html, options (with is_correct),
            explanation_html, and points."""
        }],
    )
    questions = parse_questions(message.content[0].text)
    return questions
```

**Testing:**

| # | Test case | Type | Expected result |
|---|-----------|------|-----------------|
| 11.1.1 | Course outline generation returns valid structure | Integration | JSON contains sections with lessons; each lesson has required fields |
| 11.1.2 | Quiz generation produces correct answer format | Integration | Questions have options with exactly one is_correct=true for single-choice |
| 11.1.3 | Generated outline matches specified level and duration | Unit | Beginner outline has simpler language; duration roughly matches estimate |
| 11.1.4 | Instructor can edit and adopt generated outline | E2E | Generated outline populates curriculum builder; instructor modifies and saves |

---

### Task 11.2: AI-Powered Dynamic Pricing

**What:** Pricing recommendations based on demand signals (enrollment velocity, competitor prices, time-of-year), promotional history, and learner segment analysis.

**Design:**

```python
# apps/ai-services/src/pricing_engine.py
@app.post("/pricing/recommend")
async def recommend_price(request: PricingRequest):
    # Gather signals
    enrollment_velocity = await get_enrollment_velocity(request.course_id, days=30)
    similar_courses = await get_similar_course_prices(request.course_id)
    promo_history = await get_promotional_history(request.course_id)
    seasonal_factor = get_seasonal_factor(datetime.now())

    # Compute recommendation
    avg_competitor_price = np.mean([c['price'] for c in similar_courses]) if similar_courses else request.current_price
    demand_factor = min(enrollment_velocity / 100, 2.0)  # cap at 2x

    recommended_price = avg_competitor_price * demand_factor * seasonal_factor
    recommended_price = round(max(recommended_price, request.min_price), 2)

    return {
        "recommended_price": recommended_price,
        "signals": {
            "enrollment_velocity": enrollment_velocity,
            "avg_competitor_price": avg_competitor_price,
            "seasonal_factor": seasonal_factor,
        },
        "confidence": calculate_confidence(len(similar_courses), enrollment_velocity),
    }
```

**Testing:**

| # | Test case | Type | Expected result |
|---|-----------|------|-----------------|
| 11.2.1 | Pricing recommendation returns within expected range | Unit | Price between min and 3x competitor average |
| 11.2.2 | High demand increases recommended price | Unit | enrollment_velocity=200 yields higher price than velocity=10 |
| 11.2.3 | No competitor data defaults to current price | Unit | recommended_price equals current_price when no similar courses |
| 11.2.4 | Confidence score reflects data quality | Unit | More similar courses = higher confidence |

### Phase 11 Definition of Done

- [ ] AI course outline generation from learning outcome prompt
- [ ] AI quiz question generation following QTI patterns
- [ ] Generated content importable into curriculum builder
- [ ] Dynamic pricing recommendations with demand signal analysis
- [ ] Instructor dashboard shows pricing recommendations with explanations
- [ ] All test cases pass

---

## Phase 12: Production Hardening & Compliance

**Goal:** GDPR/CCPA compliance, WCAG 2.2 AA accessibility audit, performance optimization, security hardening, monitoring, and deployment automation.

**Duration estimate:** 3 weeks

### Task 12.1: GDPR & CCPA Compliance

**What:** Cookie consent banner, data subject rights (access, export, deletion), privacy policy acceptance tracking, and data processing agreements.

**Design:**

```typescript
// apps/api/src/routes/privacy.ts
// Data subject access request
app.get('/api/v1/users/me/data-export', {
  preHandler: [requireAuth]
}, async (request, reply) => {
  const userId = request.user.id;
  const data = {
    profile: await prisma.user.findUnique({ where: { id: userId } }),
    enrollments: await prisma.enrollment.findMany({ where: { userId } }),
    orders: await prisma.order.findMany({ where: { userId } }),
    reviews: await prisma.review.findMany({ where: { userId } }),
    credentials: await prisma.credential.findMany({ where: { userId } }),
    discussions: await prisma.discussion.findMany({ where: { userId } }),
  };
  return reply.send(data);
});

// Right to erasure
app.delete('/api/v1/users/me', {
  preHandler: [requireAuth]
}, async (request, reply) => {
  const userId = request.user.id;
  await prisma.$transaction(async (tx) => {
    // Anonymize rather than hard-delete to preserve data integrity
    await tx.user.update({
      where: { id: userId },
      data: {
        email: `deleted-${userId}@anonymized.local`,
        displayName: 'Deleted User',
        passwordHash: null,
        avatarUrl: null,
        profile: {},
        ssoProviders: [],
        isActive: false,
      },
    });
    // Record erasure for compliance
    await tx.auditLog.create({
      data: {
        userId,
        action: 'user.gdpr_erasure',
        entityType: 'User',
        entityId: userId,
        changes: { anonymized: true },
        context: { requested_at: new Date().toISOString() },
      },
    });
  });
  return reply.code(204).send();
});
```

**Testing:**

| # | Test case | Type | Expected result |
|---|-----------|------|-----------------|
| 12.1.1 | Data export contains all user PII and activity | Integration | Export includes profile, enrollments, orders, reviews, credentials |
| 12.1.2 | Data erasure anonymizes user profile | Integration | Email, name, avatar replaced with anonymous values; isActive=false |
| 12.1.3 | Erasure preserves enrollment/order records with anonymized user | Integration | Enrollment and order rows still exist but user display name shows "Deleted User" |
| 12.1.4 | Cookie consent banner appears for EU visitors | E2E | Banner shown; preferences saved; analytics scripts blocked until consent |
| 12.1.5 | Audit log records erasure event | Integration | audit_log row with action='user.gdpr_erasure' created |

---

### Task 12.2: WCAG 2.2 AA Accessibility Audit

**What:** Comprehensive accessibility audit and remediation of all UI components. Automated testing with axe-core, manual testing with screen readers.

**Testing:**

| # | Test case | Type | Expected result |
|---|-----------|------|-----------------|
| 12.2.1 | axe-core reports zero critical violations on course player | Automated | No critical or serious accessibility violations |
| 12.2.2 | All interactive elements keyboard-navigable | Manual | Tab order follows visual order; focus visible on all elements |
| 12.2.3 | Video player captions toggle works | Manual | Captions can be enabled/disabled; text is readable |
| 12.2.4 | Color contrast meets AA ratio (4.5:1 for text) | Automated | All text passes contrast ratio check |
| 12.2.5 | Screen reader announces course progress | Manual | NVDA/VoiceOver reads "Progress: 65% complete" on player page |

---

### Task 12.3: Performance Optimization

**What:** Database query optimization (N+1 prevention, index analysis), CDN configuration for static assets and video, API response caching, and frontend bundle optimization.

**Testing:**

| # | Test case | Type | Expected result |
|---|-----------|------|-----------------|
| 12.3.1 | Course catalog page loads under 1.5s (LCP) | Performance | Lighthouse LCP < 1500ms on 4G simulation |
| 12.3.2 | API responses use appropriate cache headers | Integration | GET /api/v1/courses returns Cache-Control with max-age |
| 12.3.3 | No N+1 queries in enrollment list endpoint | Performance | Query count stays constant regardless of enrollment count |
| 12.3.4 | Video starts playing within 2 seconds | Performance | HLS first segment loads under 2s on broadband |

---

### Task 12.4: Security Hardening & Monitoring

**What:** Rate limiting, CSRF protection, input sanitization, dependency vulnerability scanning, structured logging, distributed tracing, alerting, and health monitoring.

**Testing:**

| # | Test case | Type | Expected result |
|---|-----------|------|-----------------|
| 12.4.1 | Rate limiter blocks excessive requests | Integration | 101st request within 1 minute returns 429 |
| 12.4.2 | XSS payload in course description sanitized | Security | `<script>alert(1)</script>` rendered as text, not executed |
| 12.4.3 | SQL injection in search query prevented | Security | Search query `'; DROP TABLE course;--` returns normal results |
| 12.4.4 | Dependency scan reports no critical vulnerabilities | CI | `npm audit` and `snyk test` report zero critical findings |
| 12.4.5 | OpenTelemetry traces visible in Grafana | Integration | API request traces appear with span details |

---

### Task 12.5: CI/CD & Deployment

**What:** GitHub Actions CI pipeline (lint, type-check, test, build). Docker multi-stage builds. Kubernetes Helm chart for production deployment. Database migration automation.

**Design:**

```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]
jobs:
  lint-and-test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: pgvector/pgvector:pg16
        env:
          POSTGRES_DB: test
          POSTGRES_PASSWORD: test
        ports: [5432:5432]
      redis:
        image: valkey/valkey:8
        ports: [6379:6379]
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: pnpm }
      - run: pnpm install --frozen-lockfile
      - run: pnpm turbo lint
      - run: pnpm turbo type-check
      - run: pnpm turbo test
      - run: pnpm turbo build
```

**Testing:**

| # | Test case | Type | Expected result |
|---|-----------|------|-----------------|
| 12.5.1 | CI pipeline passes on clean main branch | CI | All jobs green; build artifacts produced |
| 12.5.2 | Docker images build successfully | CI | API and web images built and pass health checks |
| 12.5.3 | Helm chart deploys to staging cluster | Deployment | All pods healthy; services reachable; migrations applied |
| 12.5.4 | Database migration runs automatically on deploy | Deployment | New migrations applied without manual intervention |
| 12.5.5 | Rollback to previous version succeeds | Deployment | Helm rollback restores previous version; data intact |

### Phase 12 Definition of Done

- [ ] GDPR data export and erasure endpoints functional
- [ ] CCPA opt-out and disclosure endpoints functional
- [ ] Cookie consent banner with preference management
- [ ] WCAG 2.2 AA audit passed with zero critical violations
- [ ] Lighthouse performance scores: LCP < 1.5s, CLS < 0.1, FID < 100ms
- [ ] Rate limiting, CSRF, and XSS protections in place
- [ ] Dependency vulnerability scanning in CI
- [ ] OpenTelemetry tracing and Grafana dashboards operational
- [ ] CI/CD pipeline runs lint, test, build, and deploy
- [ ] Kubernetes Helm chart deployable to staging and production
- [ ] DMCA takedown workflow documented and implemented
- [ ] All test cases pass across all phases
- [ ] Load testing validates 1000 concurrent learners on course player

---

## Summary

| Phase | Duration | Key Deliverable |
|-------|----------|-----------------|
| 1. Foundation & Auth | 3 weeks | Monorepo, database, auth, user/org management |
| 2. Course Authoring | 4 weeks | Course CRUD, content items, video pipeline, categories |
| 3. Enrollment & Player | 3 weeks | Enrollment, course player (video/text/PDF), progress tracking |
| 4. Payments & Economics | 4 weeks | Stripe Connect, checkout, coupons, payouts |
| 5. Reviews & Discussion | 2 weeks | Reviews, discussion forums, community |
| 6. Assessments & Credentials | 3 weeks | Quiz engine, certificates, Open Badges 3.0 |
| 7. Discovery & Search | 3 weeks | Meilisearch catalog, faceted browse, SEO |
| 8. AI Recommendations | 4 weeks | Graph layer, embeddings, learning paths |
| 9. SCORM/xAPI/LTI | 4 weeks | SCORM runtime, xAPI LRS, LTI 1.3 provider |
| 10. Enterprise | 3 weeks | SAML SSO, white-label, analytics |
| 11. AI Content & Pricing | 3 weeks | Course scaffolding, quiz generation, dynamic pricing |
| 12. Production Hardening | 3 weeks | GDPR, WCAG 2.2, performance, security, CI/CD |
| **Total** | **~39 weeks** | Full-featured AI-native course marketplace |

**Data model:** Hybrid Relational + JSONB (Suggestion 3) as primary schema, augmented with graph tables (Suggestion 4) in Phase 8.

**Standards compliance:** SCORM 1.2/2004, xAPI (IEEE 9274.1.1), LTI 1.3, Open Badges 3.0, QTI 3.0 patterns, PCI DSS (via Stripe), GDPR/CCPA, WCAG 2.2 AA.
