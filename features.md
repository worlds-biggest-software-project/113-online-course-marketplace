# Online Course Marketplace — Feature & Functionality Survey

> Candidate #113 · Researched: 2026-05-01

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| Udemy | Open marketplace (B2C + B2B) | Commercial (proprietary) | udemy.com |
| Coursera | University-partnership marketplace + SaaS | Commercial (proprietary) | coursera.org |
| Teachable | Creator SaaS + marketplace | Commercial (proprietary) | teachable.com |
| Thinkific | Creator SaaS + marketplace | Commercial (proprietary) | thinkific.com |
| Kajabi | All-in-one creator platform | Commercial (proprietary) | kajabi.com |
| LearnWorlds | AI-powered creator LMS | Commercial (proprietary) | learnworlds.com |
| Podia | All-in-one digital product platform | Commercial (proprietary) | podia.com |
| Open edX | Open-source LMS / MOOC platform | AGPL-3.0 | openedx.org |
| Moodle | Open-source LMS | GPL-3.0 | moodle.org |
| Skillshare | Creative-skills subscription marketplace | Commercial (proprietary) | skillshare.com |

## Feature Analysis by Solution

### Udemy

**Core features**
- Open instructor marketplace: any subject-matter expert can publish a course after quality review
- Course creation tools: video uploads, quizzes, assignments, and coding exercises (in browser)
- Learner discovery: search, category browse, curated collections, and personalised recommendations
- Udemy Business: curated B2B catalogue with SSO, team analytics, and custom learning paths
- Certificate of completion for every course (non-accredited)
- Mobile app with offline content download

**Differentiating features**
- Largest catalogue in any reviewed marketplace (155K+ courses, 70K+ instructors) — network effect drives discovery
- Udemy Business enterprise analytics: skill gap analysis, team progress, and compliance tracking
- Dynamic pricing engine: automated promotional pricing aligned with local purchasing power parity
- Instructor revenue share on organic marketplace sales (37%) supplemented by promotional revenue

**UX patterns**
- Learner home: personalised recommendations based on purchase history and browsing behaviour
- Course player: video with bookmarks, notes, Q&A, and closed captions
- Instructor dashboard: revenue analytics, student reviews, and course performance metrics

**Integration points**
- LTI integration for Udemy Business embedding in corporate LMS
- SSO (SAML 2.0) for enterprise seat management
- Udemy Business API for custom reporting and content cataloguing

**Known gaps**
- Instructor revenue share (37% on organic sales) is among the lowest in the creator economy
- No native community or cohort course features — learner experience is largely asynchronous and isolated
- Certificate carries no academic credit or employer-recognised credential framework
- Quality is highly variable — low publishing bar creates content quality discovery problem

**Licence / IP notes**
- Proprietary platform; instructors retain content IP but grant Udemy a broad licence to sell and distribute

---

### Coursera

**Core features**
- University-partnered courses, specialisations, professional certificates, and full online degrees
- Structured learning paths: multi-course specialisations with capstone projects and peer review
- Coursera for Campus and Coursera for Business: institutional and enterprise access
- Graded assignments, peer-graded projects, and proctored exams for verified certificates
- Coursera Plus subscription: unlimited access to most content for a flat monthly/annual fee

**Differentiating features**
- Only marketplace with employer-recognised professional certificates (Google, IBM, Meta, Amazon) mapped to specific job roles
- Full online degree programmes ($15K–$25K) from accredited universities — unique in the reviewed set
- Coursera for Business provides skill benchmarking, custom learning paths, and team analytics
- AI-powered content recommendations based on career goal selection

**UX patterns**
- Structured weekly module pacing with deadlines for guided learners; self-paced option available
- Discussion forums moderated by course staff and teaching assistants
- Progress tracker showing completion percentage and assignment deadlines

**Integration points**
- SSO/SAML and LTI for enterprise campus deployment
- API for enterprise HR system integration (Workday, SuccessFactors)
- Skills taxonomy mapped to job market data for career pathway recommendations

**Known gaps**
- Heavy dependence on university partner content limits responsiveness to emerging skill areas
- High cost of degree programmes vs. alternatives (bootcamps, self-paced platforms)
- Instructor creator tools are not available for independent content creators outside of university partnerships
- Profitability pressure has led to course catalogue pruning and reduced investment in some subject areas

**Licence / IP notes**
- Proprietary platform; content owned by university/institutional partners; learner data governed by Coursera privacy policy

---

### Teachable

**Core features**
- Course creation: drag-and-drop curriculum builder supporting video, audio, PDF, quizzes, and coding challenges
- Sales and payments: integrated checkout with coupons, upsells, order bumps, and affiliate tracking
- Student management: enrollment, progress tracking, and certificate of completion issuance
- Community: integrated discussion forums per course (Teachable Community add-on)
- Email marketing: automated drip sequences triggered by enrollment, completion, or inactivity

**Differentiating features**
- Close to $400M in annual creator payouts — validates platform's creator revenue ecosystem
- Coaching product: one-on-one session booking and payment alongside course offerings
- Strong affiliate programme management natively in the platform
- No transaction fees on paid plans — creator keeps full revenue minus platform subscription

**UX patterns**
- Creator experience: guided course builder with content library and curriculum organiser
- Student experience: clean course player with progress bar and community tab
- Admin dashboard: revenue, enrollment, and conversion funnel analytics

**Integration points**
- Zapier and API webhooks for automation workflows
- ConvertKit, Mailchimp, and ActiveCampaign for email marketing
- Stripe and PayPal for payment processing

**Known gaps**
- No built-in marketplace discovery — creators must drive their own traffic; no organic audience
- Live session tools (webinar/video calls) require third-party integrations (Zoom, StreamYard)
- Analytics depth is basic; no AI-powered learner performance insights
- Free plan charges 5% transaction fee, which erodes margins for new creators

**Licence / IP notes**
- Proprietary SaaS; creator owns all content; Teachable holds a licence to host and deliver it

---

### Thinkific

**Core features**
- Course and community creation with drag-and-drop builder
- Multi-product support: courses, memberships, cohort courses, and digital downloads
- Thinkific Apps marketplace for third-party extensions
- Community features: discussion boards, live events, and member directories
- White-label option for branded course academy

**Differentiating features**
- Thinkific Plus: enterprise white-label multi-tenancy for training companies and brands
- Community + course bundling as a core product differentiator (not an add-on)
- App marketplace strategy — more open integration ecosystem than competitors
- Publicly traded on TSX (THNC) — financial transparency unusual in the creator SaaS segment

**UX patterns**
- Unified creator dashboard managing courses, community, and analytics in one view
- Student experience: community-integrated course player with social learning elements
- Event scheduling for live cohort sessions with automated reminder workflows

**Integration points**
- Thinkific Apps marketplace (Zapier, ConvertKit, Typeform, and 50+ direct integrations)
- REST API for custom integrations
- Stripe for payment processing; multi-currency support

**Known gaps**
- No native marketplace discovery — platform is a course creation and delivery tool, not a learner acquisition channel
- Analytics and reporting less sophisticated than enterprise LMS platforms
- Video hosting through third-party CDN; no native interactive video authoring

**Licence / IP notes**
- Proprietary SaaS; creator retains content ownership; platform licence for hosting and delivery

---

### Kajabi

**Core features**
- All-in-one platform: courses, communities, coaching, memberships, podcasts, and email marketing
- Website and landing page builder with integrated checkout and sales funnels
- Email marketing automation: sequences, broadcasts, and behavioural triggers
- Affiliate management: tracking, commission configuration, and payout management
- Analytics: revenue, churn, page conversion, and learner engagement in one dashboard

**Differentiating features**
- Only reviewed platform that natively replaces website builder, email marketing, and course delivery in one subscription — highest per-creator monthly revenue of reviewed platforms
- No transaction fees at any plan level
- Kajabi AI: AI-generated course outlines, landing page copy, and email sequences (launched 2023)
- Creator live-stream and podcast hosting as native features

**UX patterns**
- Business-oriented creator dashboard with pipeline-style sales funnel visibility
- Learner experience: branded member portal with course + community in a single destination
- Mobile app (Kajabi app) for learner access; creator management via mobile admin

**Integration points**
- Zapier for automation workflows
- Stripe and PayPal for payment processing
- Drip, ConvertKit integration (though native email marketing reduces need)

**Known gaps**
- No marketplace discovery — requires creator-driven traffic acquisition
- High subscription cost ($69–$399/month) is a barrier for early-stage creators
- Community features less mature than Circle or dedicated community platforms
- No LTI or enterprise SSO integration for institutional deployment

**Licence / IP notes**
- Proprietary SaaS; creator owns all content; no open-source components

---

### LearnWorlds

**Core features**
- Interactive video player with overlaid questions, clickable areas, and chapter markers
- SCORM and xAPI support for importing and tracking packaged content
- AI course creator: AI-generated course outlines and learning objectives from a topic prompt
- White-label mobile app builder (no-code) for branded iOS and Android learner apps
- Community features: social profiles, discussion feeds, and live events

**Differentiating features**
- Most advanced interactive video authoring of any reviewed creator platform — questions embedded in video timeline change based on learner response
- White-label mobile app without requiring native app development skills — unique in the reviewed set
- SCORM/xAPI import capability bridges the gap between enterprise LMS and creator platform markets
- Built-in AI course creator as part of the standard subscription

**UX patterns**
- Creator experience: multi-step course builder with interactive video editor and assessment authoring
- Student experience: immersive course player with interactive video, social learning feed, and progress indicators
- White-label deployment: fully branded learner portal with custom domain and no LearnWorlds branding

**Integration points**
- SCORM 1.2 / 2004 and xAPI content import
- Zapier and webhooks for automation
- Stripe and PayPal; EU VAT compliance for European sales

**Known gaps**
- No organic marketplace discovery; creator drives own traffic
- Community features not as robust as dedicated community platforms (Circle, Mighty Networks)
- Pricing ($24–$299/month) is competitive but the highest-value features require the highest tier

**Licence / IP notes**
- Proprietary SaaS; creator owns content; platform holds hosting and delivery licence

---

### Open edX

**Core features**
- Full-featured MOOC and course delivery platform used by edX.org, MIT, Harvard, and 150+ institutions
- Course authoring (Studio): video, text, assessments, discussions, and problem types including code execution
- Learning Management: cohorts, instructor-paced and self-paced tracks, grade book, and certificates
- Comprehensive discussion forums with moderation tools
- XBlock plugin architecture for extending course content types
- Bulk enrollment, analytics, and course export APIs

**Differentiating features**
- Only reviewed platform with AGPL-3.0 open-source licence — fully auditable and self-hostable
- Course content portability via course export/import (OLX format)
- Proven at MOOC scale: millions of concurrent learners across edX.org deployments
- Active contributor community (2U, Axim Collaborative) with enterprise support available from third parties

**UX patterns**
- Learner experience: structured weekly course with discussion forums and progress dashboard
- Author experience: Studio drag-and-drop course builder with assessment preview
- Administrator experience: site administration for user management, enrollment, and analytics

**Integration points**
- LTI 1.3 for external tool embedding
- xAPI for learner analytics export
- REST API for programmatic enrollment, grading, and user management
- OAuth2/SAML for institutional SSO

**Known gaps**
- Complex self-hosted infrastructure (Docker-based Tutor deployment) requires significant DevOps expertise
- Default UI is functional but dated compared to commercial alternatives
- No built-in marketplace or monetisation layer — operators must build payment and discovery features
- Analytics beyond basic completion data require additional tooling (Aspects data pipeline)

**Licence / IP notes**
- AGPL-3.0: any modifications to Open edX deployed as a network service must be made publicly available under AGPL-3.0; XBlocks can be licensed separately under any licence

---

### Moodle

**Core features**
- Course management: content pages, activities, and resources with configurable completion criteria
- Assessment: quizzes (QTI-compatible), assignments, workshops, and H5P interactive content
- Gradebook: flexible grade calculation with categories, scales, and outcome mapping
- Forums, wikis, databases, and glossaries as collaborative learning activities
- MoodleCloud hosted option; self-hosted via standard LAMP/LEMP stack

**Differentiating features**
- Largest open-source LMS install base globally (50M+ users); widest plugin ecosystem (2,000+ plugins)
- H5P integration for interactive content (drag-and-drop, video annotation, branching scenarios)
- Moodle Workplace: commercial extension with multi-tenancy, competencies, and advanced reporting for corporate use
- MoodleNet: federated social network for educator resource sharing

**UX patterns**
- Traditional LMS course structure with section-based navigation
- Classic and Boost/Moove themes for modern responsive UI
- Mobile app (Moodle Mobile) for offline content access and push notifications

**Integration points**
- LTI 1.3 for external tool embedding (including adaptive tools, proctoring, turnitin)
- IMS OneRoster for roster management
- xAPI and SCORM for content import/tracking
- REST API and web services framework for custom integrations

**Known gaps**
- Default UI is functionally complex; requires significant configuration to achieve modern UX
- No native marketplace or monetisation layer — requires plugins (WooCommerce, Moodle eCommerce)
- Self-hosted infrastructure management is substantial overhead for small organisations
- AI-native features are nascent; AI subsystem plugin is in early adoption

**Licence / IP notes**
- GPL-3.0: all modifications and plugins distributed externally must be licensed under GPL-3.0; MoodleCloud hosted service is commercial

---

### Skillshare

**Core features**
- Creative-skills subscription marketplace: design, photography, illustration, music, writing, business
- Project-based learning: each class has an accompanying hands-on project with community gallery
- Learner subscription model: flat monthly fee gives unlimited access to all classes
- Creator royalty pool: instructors earn monthly payments based on minutes watched relative to total platform watch time
- Community features: class discussion threads and project feedback

**Differentiating features**
- Project-based learning is core to every class — not an optional add-on; strongest creative skills community of reviewed platforms
- Learner subscription model removes purchase friction and enables serendipitous discovery
- Creator royalty pool incentivises engagement-quality content rather than just enrollment numbers

**UX patterns**
- Class discovery: curated categories, trending classes, and personalised recommendations
- Class player: video-centric with project brief tab and community discussion
- Creator studio: simplified upload and chapter-marking interface

**Integration points**
- Skillshare for Teams: SSO and team analytics for corporate creative skills training
- No LTI or deep enterprise LMS integration

**Known gaps**
- Creator earnings are unpredictable and declining as catalogue grows (fixed royalty pool diluted by more instructors)
- Focused exclusively on creative and soft skills — no STEM, compliance, or professional certification content
- No certificates or credentials beyond platform completion badges
- No instructor-controlled pricing; subscription model means creators cannot sell individual courses at premium prices

**Licence / IP notes**
- Proprietary platform; instructor owns content IP; broad hosting and distribution licence granted to Skillshare

---

## Cross-Cutting Feature Themes

### Table-Stakes Features
- Video content hosting and adaptive-bitrate streaming with progress tracking
- Course curriculum builder (sections, lessons, quizzes) with drag-and-drop organisation
- Learner progress tracking and completion certificate generation
- Integrated payment processing with multiple currencies and coupon/discount management
- Student discussion forums or Q&A per course
- Mobile-responsive learner interface (or native mobile app)
- GDPR / CCPA consent management and right-to-erasure workflow
- WCAG 2.2 accessibility compliance for course player and platform UI

### Differentiating Features
- Organic marketplace discovery with audience network (Udemy, Coursera, Skillshare lead; creator-SaaS platforms have no organic audience)
- University-accredited or employer-recognised credentials (Coursera is unique in reviewed set)
- AI-assisted course creation (outline generation, quiz drafting, landing page copy) — Kajabi AI and LearnWorlds lead
- Interactive video with embedded assessments that adapt to response (LearnWorlds leads)
- All-in-one business platform (website + email marketing + course + community) in one subscription — Kajabi leads
- White-label mobile app builder without native development (LearnWorlds is unique in reviewed set)
- Open-source self-hostable option (Open edX AGPL-3.0, Moodle GPL-3.0)

### Underserved Areas / Opportunities
- AI-powered dynamic pricing optimised per-course based on demand signals, learner segment, and promotional history
- Personalised cross-catalogue learning journeys that sequence courses from multiple instructors based on individual goals
- Competency-mapped credentials verified by AI adaptive assessment rather than simple completion tracking
- Instructor coaching tools: AI-generated actionable improvement suggestions from learner completion and review signals
- Community-driven discovery with social proof signals beyond star ratings (peer project portfolios, cohort outcome data)
- SCORM/xAPI import + delivery natively in a creator marketplace — bridges enterprise content and creator economy

### AI-Augmentation Candidates
- AI-generated course scaffolding (outlines, quiz questions, slide structures) from a learning outcome prompt
- Dynamic pricing engine optimising course prices based on demand, competitor signals, and promotional performance
- Personalised cross-catalogue learning path builder based on learner goals and prior knowledge
- Automated course quality scoring from completion rates, quiz performance, and review sentiment with instructor improvement suggestions
- Adaptive final assessment for competency-mapped certificate issuance (replacing simple completion badges)

---

## Legal & IP Summary

- **AGPL-3.0 (Open edX):** Network deployment of modified Open edX source code requires publishing modifications under AGPL-3.0; XBlock plugins can carry separate licences; strong copyleft with network use clause
- **GPL-3.0 (Moodle):** Modifications to Moodle core distributed externally must be GPL-3.0; plugins distributed to third parties must also be GPL-3.0; self-hosted internal use may keep modifications private
- **Content IP:** All reviewed commercial platforms grant the platform a hosting and distribution licence while instructors retain content ownership; new platforms must define this licence clearly to attract high-quality instructors
- **PCI DSS:** Any platform processing payment card transactions directly must achieve and maintain PCI DSS compliance; using Stripe or PayPal as the payment processor delegates most PCI scope to those providers
- **GDPR / CCPA:** Global course marketplaces must implement consent management, data subject rights (access, erasure, portability), and documented data processing agreements with sub-processors
- **FERPA:** Applies if the platform hosts accredited institutional courses or holds official academic records; most creator platforms are not FERPA-covered entities unless they serve accredited institutions
- **Copyright:** User-generated course content carries inherent IP risk; platforms must implement DMCA takedown workflows (US) and equivalent processes for other jurisdictions

---

## Recommended Feature Scope

**Must-have (MVP)**:
- Course curriculum builder with video, text, PDF, and quiz lesson types
- Adaptive-bitrate video hosting with progress tracking and bookmarks
- Integrated checkout: one-time purchase, subscription, and coupon/discount management
- Instructor dashboard: revenue analytics, enrollment data, and learner progress summaries
- Student discussion forum and Q&A per course
- Completion certificate generation with verifiable credential link
- GDPR / CCPA consent management and PCI-compliant payment processing via Stripe

**Should-have (v1.1)**:
- Marketplace discovery layer with category browse, search, and personalised recommendations
- AI-assisted course creation: outline generation, quiz drafting, and lesson descriptions from a learning objective prompt
- Community features: learner profiles, discussion feeds, and cohort course support
- LTI 1.3 integration for embedding courses in corporate LMS environments
- Affiliate and referral tracking with automated commission management

**Nice-to-have (backlog)**:
- Interactive video player with embedded questions and adaptive branching
- AI-powered dynamic pricing recommendations per course based on demand and promotional signals
- Personalised cross-catalogue learning path builder based on declared learner goals
- White-label branded academy option for enterprise and training company customers
- Adaptive final assessment for competency-mapped certificate issuance beyond simple completion tracking
