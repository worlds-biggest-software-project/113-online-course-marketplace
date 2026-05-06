# Standards & API Reference

> Project: Online Course Marketplace · Generated: 2026-05-06

## Industry Standards & Specifications

### ISO Standards

**ISO/IEC 40180:2017 — Quality for Learning, Education and Training**
- URL: https://www.iso.org/standard/62825.html
- Provides a Quality Reference Framework for assessing and managing quality in IT-enhanced learning environments at the macro (policy), meso (organisational), and micro (course delivery) levels. Relevant for any course marketplace that makes quality claims to institutional buyers or seeks accreditation alignment.

---

### 1EdTech (IMS Global) Standards

**IMS LTI 1.3 — Learning Tools Interoperability Advantage**
- Specification: https://www.imsglobal.org/spec/lti/v1p3/
- Implementation Guide: https://www.imsglobal.org/spec/lti/v1p3/impl
- 1EdTech Overview: https://www.1edtech.org/standards/lti
- Enables third-party tools (video players, proctoring, interactive content, virtual classrooms) to be securely embedded within an LMS or course marketplace with single sign-on and grade passback. Built on OAuth 2.0, OpenID Connect, and JWT. LTI 1.3 is the current required version for certified interoperability; supersedes LTI 1.1.

**IMS QTI 3.0 — Question and Test Interoperability**
- Overview: https://www.imsglobal.org/spec/qti/v3p0/oview
- Implementation Guide: https://www.imsglobal.org/spec/qti/v3p0/impl
- 1EdTech Standards Page: https://www.1edtech.org/standards/qti
- Portable XML format for representing assessment items, tests, and results. Enables quizzes and graded assessments to be authored once and delivered across any QTI-compliant LMS or marketplace without re-authoring. Version 3.0 adds HTML5 support, computer-adaptive testing, and improved accessibility alignment with WCAG.

**IMS Common Cartridge 1.3/1.4 — Course Content Packaging**
- Specification Index: https://www.imsglobal.org/cc/index.html
- 1EdTech Standards Page: https://www.1edtech.org/standards/cc
- Standard format for packaging and distributing course content (syllabus, readings, quizzes, discussion prompts) across LMS platforms. Common Cartridge 1.4 reached Candidate Final status and is ready to implement. Relevant for content import/export between creator tools and institutional LMS deployments.

**IMS OneRoster 1.2 — Roster and Grade Exchange**
- Specification: https://www.imsglobal.org/spec/oneroster/v1p2
- 1EdTech Standards Page: https://www.1edtech.org/standards/oneroster
- Defines REST API and CSV exchange patterns for securely sharing class rosters, enrollments, and grade data between student information systems (SIS) and learning platforms. Primarily relevant when the marketplace serves accredited institutions or corporate HR systems that require automated roster provisioning.

---

### IEEE Standards

**IEEE 9274.1.1-2023 — xAPI (Experience API)**
- IEEE Standard Page: https://standards.ieee.org/ieee/9274.1.1/7321/
- ADL GitHub Repository: https://github.com/adlnet/xAPI-Spec
- Overview Documentation: https://opensource.ieee.org/xapi/xapi-base-standard-documentation/
- IEEE-ratified standard for capturing learning activity data across any environment (mobile, simulation, game, video, or web). Uses subject-verb-object "statements" (e.g., "learner completed module") sent to a Learning Record Store (LRS). Supersedes SCORM's tracking capability for complex, multi-device, and informal learning; enables richer analytics than SCORM's built-in reporting.

**SCORM 1.2 / SCORM 2004 — Sharable Content Object Reference Model**
- SCORM Explained: https://scorm.com/scorm-explained/
- ADL SCORM Resources: https://adlnet.gov/projects/scorm/
- The de facto legacy standard for packaged eLearning content. Defines how content communicates completion, score, and time with an LMS via JavaScript APIs. SCORM 1.2 remains the most widely deployed version despite being technically superseded; SCORM 2004 added sequencing and navigation rules. Any marketplace importing third-party corporate training content will encounter SCORM packages and must support at minimum SCORM 1.2 launch and tracking.

**cmi5 — xAPI Profile for LMS-Launched Content**
- Specification: https://aicc.github.io/CMI-5_Spec_Current/
- Overview: https://xapi.com/cmi5/
- ADL Overview Paper: https://www.adlnet.gov/assets/uploads/Overview%20and%20Application%20of%20xAPI%20cmi5%20and%20xAPI%20Profiles.pdf
- An xAPI Profile that bridges SCORM and xAPI: defines interoperability rules for content launch, LRS authorisation, and reporting between an LMS and xAPI-enabled learning content. cmi5 is the recommended modern replacement for SCORM for new content authoring. Adoption accelerated post-2020; Moodle 4.x, Docebo, TalentLMS, and Cornerstone have strengthened cmi5 support.

---

### W3C & IETF Standards

**WCAG 2.2 — Web Content Accessibility Guidelines**
- URL: https://www.w3.org/TR/WCAG22/
- Level AA compliance is the de facto legal standard in the US (ADA), EU (EN 301 549), and UK (accessibility regulations). Applies to the course player, checkout flow, and all UI components. QTI 3.0 also incorporates WCAG alignment for accessible assessment rendering.

**RFC 6749 — OAuth 2.0 Authorization Framework**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- Foundation for LTI 1.3, platform SSO, and third-party integrations. Used by Thinkific, Open edX, Moodle, and all reviewed platforms for OAuth-based app authentication.

**RFC 7519 — JSON Web Tokens (JWT)**
- URL: https://datatracker.ietf.org/doc/html/rfc7519
- Used by LTI 1.3 and OpenID Connect for signed authentication and authorisation claims between platform and external tools.

**OpenID Connect 1.0**
- URL: https://openid.net/connect/
- Identity layer on top of OAuth 2.0; used for SSO across enterprise deployments. LTI 1.3 requires OIDC for tool launch authentication.

---

### Data Model & API Specifications

**OpenAPI Specification 3.1 / 3.2**
- URL: https://spec.openapis.org/oas/v3.2.0.html
- OpenAPI Initiative: https://www.openapis.org/
- Industry-standard format for describing RESTful APIs in YAML or JSON. Udemy, Thinkific, Teachable, LearnWorlds, and Kajabi all expose REST APIs; an AI-native course marketplace should publish its API as an OpenAPI 3.1 document to enable automatic client SDK generation, interactive documentation, and gateway integration.

**SCORM Cloud / Rustici Content Engine API**
- API Documentation: https://support.scorm.com/hc/en-us/articles/360029498134-SCORM-Cloud-API-Documentation
- Product Page: https://rusticisoftware.com/products/scorm-cloud/api/
- The de facto reference implementation for SCORM, xAPI, AICC, and cmi5 content hosting and tracking as a managed service. Client libraries available for Java, C#, Python, and PHP. Used by platforms that need SCORM/xAPI support without building their own LRS and content engine.

---

### Security & Compliance Standards

**PCI DSS v4.0.1 — Payment Card Industry Data Security Standard**
- URL: https://www.pcisecuritystandards.org/document_library/
- Required for any platform processing credit/debit card transactions. PCI DSS v4.0.1 was released June 2024 and explicitly addresses API security requirements. Using Stripe or PayPal as the payment processor delegates most PCI scope to those providers (SAQ A or SAQ A-EP compliance for the marketplace operator).

**GDPR (EU) — General Data Protection Regulation**
- URL: https://gdpr.eu/
- Governs collection, processing, and retention of learner personally identifiable information (PII) for EU users. Requires consent management, data subject rights (access, rectification, erasure, portability), data processing agreements with sub-processors, and breach notification within 72 hours.

**CCPA — California Consumer Privacy Act**
- URL: https://oag.ca.gov/privacy/ccpa
- US equivalent of GDPR for California residents. Requires opt-out of data sale, disclosure of data categories collected, and right to deletion on request.

**FERPA — Family Educational Rights and Privacy Act**
- URL: https://studentprivacy.ed.gov/
- Applies when the platform serves accredited US institutions or holds official academic records. March 2025 US Department of Education guidance clarified that gender support plans and similar records constitute education records regardless of storage location. FERPA consent functions differently from GDPR consent.

### MCP Server Specifications

The Model Context Protocol (MCP) is not currently formalised as an industry standard for e-learning platforms but is relevant if the marketplace exposes AI agent integration (e.g., an AI tutor or course creation assistant accessing the catalogue via MCP). The MCP specification is maintained by Anthropic at https://modelcontextprotocol.io/specification. A course marketplace could expose an MCP server to allow AI agents to query course catalogues, enroll learners, or retrieve progress data programmatically.

---

## Similar Products — Developer Documentation & APIs

### Udemy

- **Description:** World's largest open course marketplace (155K+ courses, 480M+ enrollments). Exposes both an Instructor API and enterprise Udemy Business APIs (REST, GraphQL, and xAPI).
- **API Documentation:** https://www.udemy.com/developers/
- **Instructor API Reference:** https://www.udemy.com/developers/instructor/
- **Udemy Business APIs Guide:** https://business-support.udemy.com/hc/en-us/articles/360005792753-Udemy-Business-Web-APIs-Use-Cases-and-Best-Practices
- **Partner Support Docs:** https://partnersupport.udemy.com/hc/en-us/sections/13849005570967--Documentation-Support-Guides
- **Standards:** REST, GraphQL, xAPI; SAML 2.0 for enterprise SSO
- **Authentication:** API client credentials; SAML 2.0 for SSO; Udemy Business API requires an enterprise subscription

---

### Thinkific

- **Description:** Creator SaaS platform for courses, communities, and digital products. Publicly traded (TSX: THNC). Offers both a stable REST API and a newer GraphQL API where new features launch first.
- **Developer Portal:** https://developers.thinkific.com/
- **REST API Reference:** https://developers.thinkific.com/api/api-documentation
- **REST API Introduction:** https://support.thinkific.dev/hc/en-us/articles/4422677315351-REST-API-Introduction
- **Webhooks Reference:** https://developers.thinkific.com/api/webhooks-api/
- **Standards:** REST and GraphQL; OpenAPI-documented endpoints; multi-currency Stripe integration
- **Authentication:** OAuth 2.0 (public and private apps); API key for server-to-server integrations
- **SDKs/Libraries:** Community PHP SDK available (github.com/elliotboney/thinkific-php); official SDKs not published

---

### Teachable

- **Description:** Creator SaaS for courses, coaching, and digital products. 150K+ creators; close to $400M in annual creator payouts. Public API available on Growth plan or higher.
- **Developer Hub:** https://docs.teachable.com/
- **Quickstart Guide:** https://docs.teachable.com/docs/quickstart-guide
- **API Overview:** https://docs.teachable.com/docs/overview
- **Webhooks & API Help:** https://support.teachable.com/hc/en-us/articles/222808927-Webhooks-and-API
- **Standards:** REST/JSON; webhook events for enrollment, completion, and payment triggers
- **Authentication:** API Key in request headers; Growth plan or higher required

---

### LearnWorlds

- **Description:** AI-powered creator LMS with advanced interactive video and white-label mobile app builder. REST API covers courses, users, groups, subscriptions, promotions, payments, and certifications.
- **API Reference:** https://www.learnworlds.dev/docs/api/
- **Developer Portal:** https://www.learnworlds.dev/
- **API Keys Guide:** https://support.learnworlds.com/support/solutions/articles/12000080177-how-to-request-your-api-keys-and-access-tokens
- **Standards:** REST/JSON; SCORM 1.2/2004 and xAPI content import; Zapier and webhook automation
- **Authentication:** API Client ID and Client Secret (OAuth-style); keys generated in Settings → Developers → API
- **SDKs/Libraries:** Community Python wrapper (github.com/whitesmith/learnworlds)

---

### Kajabi

- **Description:** All-in-one creator platform (courses, communities, coaching, email marketing, podcasts). Public API launched Q3 2025 and is available on the Pro Plan or as a $25/month add-on.
- **API Help Documentation:** https://help.kajabi.com/en/collections/16339548-api-integrations
- **Getting Started Guide:** https://help.kajabi.com/en/articles/12696419-getting-started-with-the-kajabi-public-api
- **Standards:** REST/JSON; webhook events for key platform triggers
- **Authentication:** API Keys created in Settings → Public API; Pro Plan required or $25/month add-on
- **SDKs/Libraries:** No official SDKs; third-party integration guides available via Rollout and Make (formerly Integromat)

---

### Coursera

- **Description:** University-partnership marketplace with degrees, professional certificates, and MOOCs. 148M+ registered users. Developer platform exposes REST APIs for third-party integration; enterprise APIs require institutional agreement.
- **Developer Portal:** https://dev.coursera.com/get-started
- **Standards:** REST/JSON; OAuth 2.0 for learner data access with explicit consent
- **Authentication:** OAuth 2.0; enterprise data APIs require Coursera for Business or Campus agreement
- **Notes:** Coursera's developer platform is primarily for institutional partners; a public consumer API is not broadly available. Enterprise integrations connect with Workday and SuccessFactors via SSO and roster APIs.

---

### Open edX

- **Description:** AGPL-3.0 open-source MOOC platform used by edX.org, MIT, Harvard, and 150+ institutions. Full REST API for enrollment, grading, user management, and course operations.
- **API Documentation:** https://docs.openedx.org/projects/edx-platform/en/latest/references/lms_apis.html
- **How to Use the REST API:** https://docs.openedx.org/projects/edx-platform/en/latest/how-tos/use_the_api.html
- **API Conventions:** https://openedx.atlassian.net/wiki/spaces/AC/pages/18350757/Open+edX+REST+API+Conventions
- **REST API Client (Python):** https://github.com/openedx/edx-rest-api-client
- **Standards:** REST/JSON; LTI 1.3; xAPI; OAuth 2.0 and SAML for institutional SSO; OLX format for course export/import
- **Authentication:** JWT Authorization header (preferred); OAuth 2.0 for service-to-service calls
- **SDKs/Libraries:** edx-rest-api-client (Python, official); api-doc-tools for generating OpenAPI documentation

---

### Moodle

- **Description:** GPL-3.0 open-source LMS with 50M+ users and 2,000+ plugins. Exposes a Web Services framework (REST, SOAP, XML-RPC, AMF) with hundreds of built-in functions.
- **Developer Resources:** https://moodledev.io/docs/5.1/apis
- **External Services API:** https://moodledev.io/docs/5.0/apis/subsystems/external
- **Web Service API Functions:** https://docs.moodle.org/dev/Web_service_API_functions
- **Using Web Services:** https://docs.moodle.org/501/en/Using_web_services
- **RESTful Plugin:** https://moodle.org/plugins/webservice_restful
- **Standards:** Custom REST-like protocol (not fully RESTful by default); REST protocol returns XML or JSON; LTI 1.3; IMS OneRoster; xAPI and SCORM
- **Authentication:** Token-based (wstoken query parameter); OAuth 2.0 available for external identity providers
- **Notes:** Moodle's native REST server is not fully RESTful; the community RESTful plugin (Catalyst) provides a more conventional REST interface. All API calls use the endpoint: `https://your.site.com/webservice/rest/server.php?wstoken=...&wsfunction=...&moodlewsrestformat=json`

---

### Stripe

- **Description:** The dominant payment processing API used by Teachable, Thinkific, LearnWorlds, Kajabi, and Podia for course purchases, subscriptions, and payouts. Delegating payment processing to Stripe reduces PCI DSS scope to SAQ A for most marketplace operators.
- **API Reference:** https://docs.stripe.com/api
- **Payments Documentation:** https://docs.stripe.com/payments
- **All Stripe APIs:** https://docs.stripe.com/apis
- **Standards:** REST/JSON; OpenAPI-documented; PCI DSS Level 1 certified; SCA/3DS2 compliant for EU payments
- **Authentication:** Bearer token (secret API key in Authorization header); webhook signatures for event verification
- **SDKs/Libraries:** Official SDKs for Node.js, Python, Ruby, PHP, Java, Go, .NET, and iOS/Android

---

### SCORM Cloud (Rustici Software)

- **Description:** Managed service for hosting, launching, and tracking SCORM, xAPI, AICC, and cmi5 content. Used by platforms that need standards-based content support without building their own LRS and content engine. Operates on a per-registration billing model.
- **API Documentation:** https://support.scorm.com/hc/en-us/articles/360029498134-SCORM-Cloud-API-Documentation
- **API Product Page:** https://rusticisoftware.com/products/scorm-cloud/api/
- **Technical Documentation Hub:** https://rusticisoftware.com/technical-documentation/
- **Standards:** SCORM 1.2, SCORM 2004 (all editions), xAPI (IEEE 9274.1.1), AICC, cmi5; OpenAPI-documented v2 API
- **Authentication:** Application ID + Secret Key pair per realm (billing account)
- **SDKs/Libraries:** Official client libraries for Java, C#, Python, PHP; JavaScript library for browser-side integration; GitHub: https://github.com/RusticiSoftware

---

## Notes

**Emerging: OpenBadges 3.0 and Verifiable Credentials**
1EdTech's Open Badges 3.0 specification (https://www.imsglobal.org/spec/ob/v3p0/) aligns with W3C Verifiable Credentials (https://www.w3.org/TR/vc-data-model/) to produce cryptographically verifiable digital credentials. This is the emerging standard for portable, employer-readable certificates of completion and competency. The OpenID Foundation launched conformance testing for Verifiable Credentials in early 2026. An AI-native marketplace implementing adaptive competency assessment should target Open Badges 3.0 + Verifiable Credentials for certificate issuance rather than simple PDF certificates.

**Learning Record Stores (LRS)**
Any platform consuming xAPI data must either run its own LRS or integrate with a hosted LRS (e.g., SCORM Cloud's LRS, Learning Locker, watershed). The LRS stores, retrieves, and validates xAPI statements. The xAPI specification defines the REST endpoints the LRS must implement.

**SAML 2.0 for Enterprise SSO**
Enterprise buyers (Udemy Business, Coursera for Business) require SAML 2.0 SSO integration with corporate identity providers (Okta, Azure AD, Google Workspace). SAML 2.0 is the current enterprise standard; OIDC is preferred for new integrations but SAML support remains a table-stakes enterprise requirement.
