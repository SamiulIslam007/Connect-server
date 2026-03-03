## Project Workflow: Structured Mentorship & Reputation Platform

This document explains how the whole system will work and how we will build it.
It is written in simple English so that any developer or stakeholder can read and understand it.

---

## 1. Core Mission

**Goal**: Turn mentorship into measurable outcomes and structured authority.
If we cannot measure outcomes and build mentor reputation in a credible way, the product has failed.

Four main stakeholders:
- **Students** – want clarity, access, and direction
- **Mentors** – want structured authority and leverage for their network
- **Institutions** – want measurable career outcomes
- **Employers** – want filtered, pre-vetted talent signals

The product is **not** a social network, not a content feed, and not a motivational app.
It is a **mentorship and reputation operating system** for the academic-to-career transition.

---

## 2. Tech Stack and High-Level Architecture

**Frontend**:
- Next.js (React) web app
- Uses server components + client components where needed
- Consumes APIs from the backend (and possibly some Next.js API routes for simple cases)

**Backend**:
- Express (Node.js) REST API server
- Prisma as ORM for database access
- Central business logic lives in services (not directly in controllers)

**Database**:
- PostgreSQL hosted on Neon (primary database)
- Prisma migrations for schema changes

**Optional services for later phases** (not mandatory for first MVP):
- Redis for caching
- WebSockets for real-time events (notifications, live sessions)
- Third-party provider for video (WebRTC via managed provider)

**Auth and Security**:
- OAuth + Institutional SSO support (for universities and institutions)
- JWT-based session for web clients
- Encryption for all PII (personally identifiable information) in transit (HTTPS) and sensitive data at rest where needed

**High-level architecture flow**:
- Browser / Mobile client → Next.js frontend
- Next.js frontend → Express API (`/api/...`)
- Express API → Prisma → Neon PostgreSQL

---

## 3. Repository and Folder Structure

Suggested root structure:
- `apps/`
  - `apps/web/` → Next.js frontend
  - `apps/api/` → Express backend
- `packages/` (optional) → shared code (types, utils, design system)
- `prisma/` → Prisma schema and migrations (usually in backend app)
- `docs/` → documentation (including this `workflow.md`)

Inside `apps/web/` (Next.js):
- `app/` or `pages/` → Next.js routes
- `components/` → shared UI components
- `lib/` → frontend utilities (API clients, hooks)

Inside `apps/api/` (Express):
- `src/index.ts` → app entry
- `src/routes/` → route definitions
- `src/controllers/` → request handlers
- `src/services/` → business logic
- `src/repositories/` → Prisma data access
- `src/middleware/` → auth, validation, logging
- `src/config/` → environment configuration

---

## 4. Core Domain Model (Conceptual)

The Prisma schema will implement these concepts.
This section is for understanding the core data model.

**User**
- `id`
- `email`
- `hashedPassword`
- `role` (student, mentor, institution_admin, employer_rep)
- `verifiedStatus` (unverified, pending, verified)
- `domainTags` (e.g. ["AI", "Medicine", "Finance"])
- `availability` (basic availability summary for matching)

**StudentProfile**
- `userId`
+- `targetField`
+- `targetTimeline` (6 months / 1 year / 3 years)
+- `currentLevel`
+- `specificObstacle`
+- `desiredOutcome`
+- `goals`
+- `skills`
+- `projects`
+- `progressTimeline`
+- `mentorHistory` (references to mentorship requests / sessions)

**MentorProfile**
- `userId`
- `verifiedAffiliation` (university / company)
- `domainExpertise` (tags)
- `mentorshipScope`
- `weeklyAvailabilityCap`
- `acceptedMenteeProfile` (rules or tags)
- `officeHourSchedule`
- `impactMetrics` (computed view)
- `responseReliabilityScore`

**Institution**
- `id`
- `name`
- `type` (university, bootcamp, training_institute)
- `domainTags`
- `adminUserId`

**Employer**
- `id`
- `name`
- `domainTags`
- `contactUserId`

**MentorshipRequest**
- `id`
- `mentorId`
- `studentId`
- `topicTags`
- `specificQuestion`
- `contextSummary`
- `targetOutcome`
- `expectedTimeframe`
- `conversationType` (async_thread, one_to_one_session, group_office_hour)
- `status` (pending, accepted, redirected, declined, completed)

**Outcome**
- `id`
- `mentorshipRequestId`
- `studentId`
- `mentorId`
- `type` (interview_secured, internship_obtained, admission_secured, skill_milestone_completed)
- `evidenceUrl` (optional)
- `verificationStatus` (pending, verified, rejected)
- `timestamp`

**ImpactScore** (computed, not directly edited by users)
- `mentorId`
- `domain` (field-specific)
- `scoreValue`
- `lastCalculatedAt`

**Circle (Community Group)**
- `id`
- `type` (field_based, institution_based, transition_group)
- `slug`
- `title`
- `description`
- `domainTags`

**CircleMembership**
- `circleId`
- `userId`
- `role` (member, mentor, moderator)

These entities allow us to support all layers: students, mentors, institutions, employers, and community circles.

---

## 5. API Design Overview

All backend routes are served by the Express API under `/api`.
Authentication is via JWT bearer token (for simplicity in this document).

### 5.1 Auth Routes

- `POST /api/auth/register`
  - Register a new user (student, mentor, institution admin, employer rep).
- `POST /api/auth/login`
  - Login and receive JWT token.
- `POST /api/auth/logout`
  - Invalidate current session (client-side mainly).
- `POST /api/auth/refresh`
  - Get a new token using a refresh token.
- `POST /api/auth/oauth/callback`
  - Handle OAuth or institutional SSO callback.

### 5.2 User and Profile Routes

- `GET /api/users/me`
  - Get current authenticated user profile and role.
- `PATCH /api/users/me`
  - Update basic user info (name, avatar, domain tags, availability).

**Student Profile**
- `POST /api/students/profile`
  - Create or update student profile (onboarding intelligence layer).
- `GET /api/students/profile/:id`
  - Get a student profile by user id (with access control).

**Mentor Profile**
- `POST /api/mentors/profile`
  - Create or update mentor profile (scope, capacity, accepted mentee profile).
- `GET /api/mentors/profile/:id`
  - Get a mentor profile by user id (public mentor asset view, with filtered fields).
- `POST /api/mentors/profile/verify-affiliation`
  - Submit or update verification data for affiliation.

### 5.3 Matching Engine Routes (Phase 1: Rule-Based)

- `POST /api/matching/preview`
  - Input: student id or profile data.
  - Output: list of mentors with match score details.
- `GET /api/matching/recommendations`
  - Get recommended mentors for current student with match percentage.

Matching formula (conceptual):

`matchScore = alumniConnectionWeight + fieldOverlapWeight + goalAlignmentWeight + availabilityMatchWeight`

The exact weighting logic lives in a service layer, not exposed directly to clients.

### 5.4 Mentorship Request Routes

- `POST /api/mentorship-requests`
  - Create a new mentorship request.
  - Requires:
    - specificQuestion
    - contextSummary
    - targetOutcome
    - expectedTimeframe
    - mentorId
    - conversationType
- `GET /api/mentorship-requests`
  - List mentorship requests for the current user (as student or mentor).
- `GET /api/mentorship-requests/:id`
  - Get one request with full details.
- `PATCH /api/mentorship-requests/:id/status`
  - Mentor can accept, redirect, or decline.
- `POST /api/mentorship-requests/:id/redirect`
  - Suggest an alternative mentor.

### 5.5 Mentorship Conversation Routes

For structured conversations (no random chat spam):

- `POST /api/threads`
  - Create a structured thread for a mentorship request (async).
- `GET /api/threads/:id`
  - Get full thread with messages.
- `POST /api/threads/:id/messages`
  - Post a new message in a structured format (no noise-only content).

For sessions:
- `POST /api/sessions`
  - Schedule a 1:1 or group office hour session for a mentorship request.
- `GET /api/sessions/:id`
  - Get session details (time, participants, video link).

Real-time updates and video links can be added in later phases with WebSockets and a video provider.

### 5.6 Outcome Tracking Routes

- `POST /api/outcomes`
  - Mentee reports an outcome for a specific mentorship request.
  - Fields: type, mentorshipRequestId, evidenceUrl (optional).
- `GET /api/outcomes`
  - List outcomes related to the current user.
- `PATCH /api/outcomes/:id/verify`
  - Mentor confirms or rejects an outcome.
- `GET /api/mentors/:id/outcomes`
  - List verified outcomes for a mentor (public-safe view).

All unverified outcomes do not count toward mentor reputation.

### 5.7 Reputation Engine Routes

- `GET /api/mentors/:id/impact-score`
  - Get the current impact score and breakdown for a mentor.
- `POST /api/reputation/recalculate/:mentorId` (admin or background job)
  - Recalculate impact score for a mentor based on new outcomes and metrics.

Impact score formula (conceptual only):

`impactScore = (verifiedOutcomes × weightOutcome) + (responseReliability × weightResponse) + (peerEndorsements × weightEndorsements) + (longTermMenteeRetention × weightRetention)`

The exact weights depend on difficulty level of outcomes and are defined in configuration.

### 5.8 Authority Surface (Mentor Public Page)

- `GET /api/mentors/:id/public`
  - Public mentor asset page data:
    - verifiedBadge
    - domainFocus
    - impactMetrics
    - featuredInsights
    - successStories
    - availabilitySummary

This is consumed by the frontend route like `/mentors/[id]`.

### 5.9 Community and Circles Routes

- `GET /api/circles`
  - List circles (field-based, institution-based, transition groups).
- `POST /api/circles` (admin)
  - Create a new circle.
- `GET /api/circles/:id`
  - Get circle details, guides, and mentor list.
- `GET /api/circles/:id/members`
  - List members and mentors in a circle.
- `POST /api/circles/:id/join`
  - Join a circle.
- `POST /api/circles/:id/leave`
  - Leave a circle.

Structured threads inside circles can reuse the `/api/threads` model with a `circleId` field.

### 5.10 Institution Layer Routes

- `POST /api/institutions`
  - Create a new institution (admin or system only).
- `GET /api/institutions/:id/dashboard`
  - Outcome dashboards and metrics for an institution:
    - matchAcceptanceRate
    - timeToFirstResponse
    - outcomeConversionRate
    - mentorRetentionRate
    - studentEngagement
- `GET /api/institutions/:id/alumni`
  - Alumni verification tools and mentor/mentee mapping.

These routes support the main revenue stream: institutional subscriptions.

### 5.11 Employer Layer Routes

- `POST /api/employers`
  - Create an employer account (admin or controlled signup).
- `GET /api/employers/:id/dashboard`
  - View access to mentor networks and student pipelines.
- `POST /api/employers/:id/office-hours`
  - Host office hours via sessions.
- `POST /api/employers/:id/opportunities`
  - Post internship or job opportunities.
- `GET /api/employer-opportunities`
  - List opportunities for students (filtered by verified progress and mentor endorsements).

### 5.12 Admin and Metrics Routes

- `GET /api/admin/metrics`
  - Global KPIs: match acceptance rate, response time, outcome conversion, revenue per institution.
- `POST /api/admin/moderation/actions`
  - Apply moderation decisions based on reports.

---

## 6. Frontend Flows and Pages (Next.js)

Main web routes (example Next.js `app` routes):

- `/` – Landing page, explains product value and stakeholders.
- `/auth/login` – Login page.
- `/auth/register` – Role-based registration (student, mentor, institution, employer).
- `/onboarding/student` – Onboarding form for students (target field, timeline, obstacle, desired outcome).
- `/onboarding/mentor` – Onboarding for mentors (scope, availability, mentee profile).
- `/dashboard/student` – Student dashboard:
  - suggested mentors
  - active mentorship requests
  - outcomes timeline
- `/dashboard/mentor` – Mentor dashboard:
  - incoming requests
  - active threads
  - impact metrics
- `/dashboard/institution` – Institution dashboard:
  - outcome dashboards
  - program performance
- `/dashboard/employer` – Employer dashboard:
  - office hours
  - candidate pools

Other routes:
- `/mentors/[id]` – Public mentor authority page.
- `/circles` and `/circles/[slug]` – Community layer.
- `/opportunities` – Employer opportunities surface.

Each page uses a clear API client layer that calls the Express backend.

---

## 7. Development Workflow (Step by Step)

1. **Pick a feature**  
   Example: "Student onboarding" or "Outcome reporting".

2. **Define data changes**  
   - Update Prisma schema if new fields or models are needed.
   - Run `prisma migrate dev` to create and apply migrations.

3. **Add backend logic**  
   - Create or update service functions in `apps/api/src/services`.
   - Add routes and controllers in `apps/api/src/routes` and `apps/api/src/controllers`.
   - Add validation (request body, params) and auth middleware.

4. **Add frontend integration**  
   - Add or update API client functions in `apps/web/lib/api`.
   - Build or update React components and pages in Next.js.

5. **Test the feature**  
   - Write unit tests for services.
   - Manual test the full flow in a local environment.

6. **Review and refactor**  
   - Clean up any duplication and improve naming.
   - Make sure error messages are clear and helpful.

7. **Commit and document**  
   - Write a clear commit message.
   - Update relevant docs (e.g. this `workflow.md` or `README.md`) if something important changed.

---

## 8. Phased Launch Strategy (Mapped to System)

**Phase 1: Controlled Pilot**
- Target: 1 university, around 30 mentors, 100 students.
- Focus: core onboarding, matching, mentorship requests, and basic outcome tracking.

**Phase 2: Measure Track**
- Implement dashboards for:
  - match acceptance rate
  - time to first response
  - outcome reporting rate
- Use `/api/institutions/:id/dashboard` and `/api/admin/metrics` to show this.

**Phase 3: Institutional Expansion**
- Support multi-institution setup.
- Add more robust admin tools and reporting for institutions.

**Phase 4: Employer Entry**
- Add employer dashboards.
- Enable office hours and internship pipelines.

**Phase 5: Scale and AI Matching**
- Add AI-powered matching and intelligent brief summaries for mentors.
- Integrate advanced search (e.g. Postgres full-text or ElasticSearch later).

---

## 9. Monetization and Business Logic (System View)

Monetization tiers:
- **Tier 1**: Institutional subscriptions (main revenue stream).
- **Tier 2**: Paid mentor office hours (revenue split between platform and mentor).
- **Tier 3**: Premium analytics for employers.
- **Tier 4**: Sponsored skill programs.

We avoid:
- Banner ads
- Selling user data
- Dark-pattern gamified monetization

From a technical point of view, monetization is implemented by:
- Institution subscription records (linked to Institution).
- Payment records for office hours and sponsored programs.
- Role-based access checks in middleware for premium analytics and reports.

---

## 10. KPIs and Observability

Key metrics to track (via logs, analytics, and dashboards):
- Match acceptance rate (should be > 40%).
- Average response time (should be < 48 hours).
- Outcome conversion rate (mentorship requests → verified outcomes).
- Mentor retention rate.
- Institutional contract renewal rate.
- Revenue per institution.

From the system perspective, these are computed by scheduled jobs or analytics queries.
Results are exposed through `/api/institutions/:id/dashboard` and `/api/admin/metrics`.

---

## 11. Compliance, Safety, and Moderation

The platform must:
- Protect user data according to data protection laws.
- Use clear consent flows for data usage.
- Enforce strict terms against harassment and exploitation.
- Require identity verification for mentors and institutional actors.
- Provide a clear outcome verification disclaimer (what is and is not guaranteed).

Moderation tools include:
- Reporting endpoints.
- Admin review queue.
- Audit logs for reputation-related changes.

---

## 12. Summary

This workflow file defines:
- The mission and stakeholders.
- The tech stack (Next.js, Express, Prisma, Neon PostgreSQL).
- The main data models and all key API routes.
- The frontend flows and development workflow.
- The launch phases, monetization, KPIs, and safety layer.

Any time the system design changes in a meaningful way, this document should be updated so that a new developer can read it and understand how the platform is supposed to work end to end.
