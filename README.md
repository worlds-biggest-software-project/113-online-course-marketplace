# Online Course Marketplace

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source course marketplace that pairs creator economics with intelligent discovery, pricing, and credentialing.

This project is a candidate platform for course creation, student management, payments, and certificates, designed for independent instructors, training companies, universities, and corporate L&D teams. It addresses the persistent gap between closed marketplaces with weak creator economics (Udemy's 37% revenue share) and creator-SaaS tools with no organic learner discovery (Teachable, Thinkific, Kajabi).

---

## Why Online Course Marketplace?

- Incumbent open marketplaces such as Udemy pay instructors only 37% on organic sales, eroding creator earnings as catalogues grow.
- Creator-SaaS platforms (Teachable, Thinkific, Kajabi) offer better economics but no built-in marketplace discovery — every creator must self-acquire traffic.
- Subscription marketplaces like Skillshare dilute creator royalty pools as the catalogue expands, making earnings increasingly unpredictable.
- Existing AI features (Kajabi AI, LearnWorlds AI course creator) are bolted onto closed SaaS; an open-source AI-native alternative does not yet exist at scale.
- Open-source LMS platforms (Open edX AGPL-3.0, Moodle GPL-3.0) have no native monetisation or marketplace layer, leaving operators to assemble payment, discovery, and credentialing themselves.

---

## Key Features

### Course Authoring & Delivery

- Curriculum builder supporting video, text, PDF, quizzes, and coding exercises
- Adaptive-bitrate video hosting with progress tracking and bookmarks
- SCORM 1.2 / 2004 and xAPI import for packaged enterprise content
- Mobile-responsive learner interface with offline access patterns

### Marketplace, Payments & Creator Economics

- Integrated checkout with one-time purchases, subscriptions, coupons, upsells, and order bumps
- Affiliate and referral tracking with automated commission management
- Instructor dashboard covering revenue analytics, enrollment data, and learner progress
- PCI-compliant payment processing via Stripe and PayPal with multi-currency support

### Discovery, Community & Credentials

- Category browse, search, and personalised recommendations
- Per-course discussion forums and Q&A, plus learner profiles and cohort support
- Completion certificates with verifiable credential links
- LTI 1.3 integration for embedding courses in corporate LMS environments

### Compliance & Standards

- GDPR / CCPA consent management and right-to-erasure workflow
- WCAG 2.2 accessibility compliance for course player and platform UI
- DMCA takedown workflow for user-generated content
- Optional FERPA-aligned configuration for accredited institutional deployments

---

## AI-Native Advantage

AI is positioned as a core layer rather than a feature add-on: automated course quality scoring with actionable instructor improvement suggestions, dynamic pricing optimised against demand signals and promotional history, and AI-generated course scaffolding (outlines, quiz questions, lesson descriptions) from a learning-objective prompt. Personalised cross-catalogue learning pathways sequence courses from multiple instructors based on individual goals and prior knowledge, and adaptive final assessments support competency-mapped credentials that go beyond simple completion badges.

---

## Tech Stack & Deployment

The project targets self-hosted, cloud, and hybrid deployment, with open standards as first-class citizens: SCORM 1.2 / 2004, xAPI, IMS LTI 1.3, IMS Common Cartridge, and IMS QTI 3.0 for content portability and interoperability. Payment scope is delegated to Stripe / PayPal to minimise PCI DSS surface, and SAML 2.0 / OAuth2 SSO supports institutional and enterprise rollouts. SDK and REST API surface follow the patterns established by Open edX and Moodle.

---

## Market Context

Worldwide online education revenue is projected at $203.81B in 2025, growing at 8.2% CAGR to $279.30B by 2029 (Statista); a broader estimate reaches $880.17B by 2033 at 11.6% CAGR. Incumbent pricing spans free open-source (Moodle, Open edX) through creator SaaS at $33–$399/month (Podia, Teachable, Thinkific, Kajabi) up to $15K–$25K Coursera degree programmes. Primary buyers are independent instructors, training companies, universities and bootcamps, corporate L&D teams, and entrepreneurs running cohort-based learning products.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
