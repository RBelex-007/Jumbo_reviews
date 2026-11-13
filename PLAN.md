# Tufts Jumbo Reviews — HASS Course Discovery & Evaluation Platform
## Master Plan & Technical Blueprint

> **Mission**: Build a student-led, verified review platform for Tufts University—specifically engineered to help undergraduates (especially engineering students) discover, evaluate, and choose HASS (Humanities, Arts, and Social Sciences) courses based on real workload, grading styles, and assessment structures (e.g., exams vs. papers).

---

## Table of Contents
1. [Core Problems & Solutions](#1-core-problems--solutions)
2. [Tech Stack Breakdown](#2-tech-stack-breakdown)
3. [Tufts Authentication Strategy](#3-tufts-authentication-strategy)
4. [Solving the SIS "Hidden Professors" Problem](#4-solving-the-sis-hidden-professors-problem)
5. [Database Schema (PostgreSQL)](#5-database-schema-postgresql)
6. [Dual-Axis Rating System (Courses × Professors)](#6-dual-axis-rating-system-courses--professors)
7. [Faceted Filter & Search Taxonomy](#7-faceted-filter--search-taxonomy)
8. [Step-by-Step Implementation Roadmap](#8-step-by-step-implementation-roadmap)
9. [Campus Pitch & University Adoption Strategy](#9-campus-pitch--university-adoption-strategy)

---

## 1. Core Problems & Solutions

### The Problem at Tufts
* Tufts School of Engineering requires **24 HASS credits** across Humanities, Arts, and Social Sciences.
* Engineering students are often overloaded with heavy STEM courses and need to avoid classes with surprise midterms, weekly 10-page papers, or punitive attendance policies.
* **RateMyProfessors (RMP)** only rates the professor, not the specific course, and misses visiting faculty.
* **Coursicle** is focused on schedule conflicts and seat tracking, not course structure or workload.
* **Tufts SIS** gives course descriptions, but zero qualitative data or honest workload breakdowns.

### The Solution: Jumbo Reviews
* **Verified Tufts-Only**: Log in using Tufts credentials to prevent spam, while keeping student names completely anonymous on reviews.
* **Granular Preference Filters**:
  - `[ ] No Exams (No Midterms / No Finals)`
  - `[ ] No Essays (Project/quiz-based only)`
  - `[ ] Max Hours per Week Slider`
  - `[ ] Attendance Optional`
* **Course × Professor Historical Tracking**: Compare how `PHIL-0001` differs when taught by Professor A vs. Professor B.

---

## 2. Tech Stack Breakdown

### Selected Stack: Full-Stack Next.js (TypeScript) + PostgreSQL
* **Frontend**: Next.js 14+ (App Router), React, Tailwind CSS, shadcn/ui, Lucide Icons.
* **Backend**: Next.js Server Actions & Route Handlers (unified in the same codebase).
* **Database**: PostgreSQL hosted on **Supabase** or **Neon** (generous free tier, serverless).
* **ORM**: **Prisma** or **Drizzle ORM** (type-safe database queries and automated schema migrations).
* **Authentication**: **Auth.js (NextAuth v5)** with Google OAuth restricted to `@tufts.edu`.
* **Deployment**: **Vercel** (one-click deployments, fast edge network).

### Why Next.js over Traditional Node (Express) + React?
* **Node.js** is just the runtime engine that lets JavaScript run on a server. With traditional Node + Express + React, you must build and maintain two separate projects, manage CORS headers, and build custom API endpoints for everything.
* **Next.js** is a full-stack framework built *on top of* Node and React. It houses your UI components and server-side database actions inside one single repository. You run one command (`npm run dev`) and deploy everything in one click.

### Why PostgreSQL over MongoDB?
* Course catalogs are fundamentally **relational**: `Departments -> Courses -> Offerings (Semesters) -> Professors -> Reviews -> TagVotes`.
* Calculating real-time averages (e.g., average hours per week, percentage reporting "No Exams", and instructor rating across multiple classes) is substantially faster, safer, and cleaner with SQL joins and foreign keys than with MongoDB document collections.

---

## 3. Tufts Authentication Strategy

### Official SIS/Canvas SSO vs. Google `@tufts.edu` OAuth
* **Official Tufts SSO (Shibboleth / Canvas Login)**: Managed strictly by Tufts Technology Services (TTS). Getting SAML credentials requires bureaucratic approvals, university sponsorships, and cybersecurity reviews.
* **Tufts Google Workspace OAuth**: All Tufts student email accounts (`@tufts.edu`) are Google accounts. By implementing standard Google OAuth and restricting login to the `tufts.edu` domain, you get **100% verified Tufts-only authentication on Day 1 without asking TTS for permission**.

```
[Student clicks 'Log in with Tufts']
                │
                ▼
[Google OAuth Prompt: user selects @tufts.edu]
                │
                ▼
[Server verifies: email.endsWith('@tufts.edu')]
       ├── Valid: Issue session cookie (User can review & vote)
       └── Invalid: Reject with error ("Must use a valid Tufts email")
```

> **Privacy Wall**: The database stores the student's email solely to prevent duplicate reviews for the same course offering. Publicly, reviews are displayed as `Anonymous Jumbo (Class of '26)`.

---

## 4. Solving the SIS "Hidden Professors" Problem

When you browse Tufts SIS as a guest without logging in, the catalog lists **Course Codes, Titles, and HASS Designations**, but conceals the specific **Professors / Instructors** for class sections.

To solve this sustainably:

### 1. Crowdsourced Professor Attachment (Core Longevity Architecture)
* Seed your database with the **courses** and **HASS designations** from the public catalog.
* Pre-seed known faculty names from the public [Tufts Faculty Directory](https://facultyprofiles.tufts.edu).
* When a student submits a review:
  - They select the course (e.g., `PHIL-0001`).
  - An autocomplete field asks: *"Who was your professor?"*
  - If the professor isn't in the list, the student clicks `+ Add Professor` and types their name.
  - The app automatically links that Professor to the Course for that term.
* **Advantage**: Requires zero brittle scrapers and continues working smoothly years after you graduate.

### 2. One-Time Authenticated Export (Dev Kickoff)
* Log into SIS as a student in your browser.
* Open DevTools (`F12`) -> **Network** tab -> Search for classes in Arts & Sciences.
* Right-click the internal JSON response containing the table data -> **Copy Response**.
* Save as `seed.json` to immediately populate your development database with hundreds of active course-professor pairings.

---

## 5. Database Schema (PostgreSQL)

```sql
-- Core Courses
CREATE TABLE courses (
    id VARCHAR(20) PRIMARY KEY, -- e.g., 'PHIL-0001', 'PS-0021'
    title VARCHAR(255) NOT NULL,
    department VARCHAR(100) NOT NULL,
    description TEXT,
    is_humanities BOOLEAN DEFAULT FALSE,
    is_arts BOOLEAN DEFAULT FALSE,
    is_social_science BOOLEAN DEFAULT FALSE
);

-- Professors
CREATE TABLE professors (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    full_name VARCHAR(255) NOT NULL,
    department VARCHAR(100) NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Course Offerings (Bridges Course + Professor + Semester)
CREATE TABLE course_offerings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    course_id VARCHAR(20) REFERENCES courses(id) ON DELETE CASCADE,
    professor_id UUID REFERENCES professors(id) ON DELETE CASCADE,
    term VARCHAR(50) NOT NULL, -- e.g., 'Fall 2025'
    UNIQUE(course_id, professor_id, term)
);

-- Verified Tufts Users
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tufts_email VARCHAR(255) UNIQUE NOT NULL,
    hashed_email VARCHAR(255) NOT NULL,
    graduation_year INT,
    is_engineering BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Detailed Course Reviews
CREATE TABLE reviews (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    offering_id UUID REFERENCES course_offerings(id) ON DELETE CASCADE,
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    overall_rating INT CHECK (overall_rating BETWEEN 1 AND 5),
    difficulty_rating INT CHECK (difficulty_rating BETWEEN 1 AND 5),
    weekly_hours NUMERIC(4, 1) NOT NULL, -- e.g., 4.5
    has_exams BOOLEAN NOT NULL DEFAULT FALSE,
    exam_type VARCHAR(50), -- 'NONE', 'MIDTERM_ONLY', 'FINAL_ONLY', 'BOTH', 'QUIZZES'
    has_essays BOOLEAN NOT NULL DEFAULT FALSE,
    essay_frequency VARCHAR(50), -- 'NONE', '1_OR_2_TOTAL', 'WEEKLY', 'TERM_PAPER'
    attendance_mandatory BOOLEAN NOT NULL DEFAULT FALSE,
    group_projects BOOLEAN NOT NULL DEFAULT FALSE,
    qualitative_review TEXT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    UNIQUE(offering_id, user_id) -- Prevents review spamming
);

-- Professor-Specific Feedback
CREATE TABLE professor_ratings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    professor_id UUID REFERENCES professors(id) ON DELETE CASCADE,
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    teaching_quality INT CHECK (teaching_quality BETWEEN 1 AND 5),
    accessibility INT CHECK (accessibility BETWEEN 1 AND 5), -- Office hours availability
    tags TEXT[], -- e.g. ['Passionate', 'Lenient Grader', 'Heavy Reading', 'Clear Slides']
    notes TEXT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

---

## 6. Dual-Axis Rating System (Courses × Professors)

The platform evaluates education along two complementary dimensions:

### Dimension 1: The Course Detail View (`/courses/PHIL-0001`)
* Shows the course description and HASS category tags (`HUM`, `ART`, or `SOC`).
* Visual breakdown of average weekly workload outside class.
* **Professor Comparison Table**:
  | Professor | Semester | % Reporting Exams | % Reporting Essays | Attendance | Rating |
  | :--- | :--- | :--- | :--- | :--- | :--- |
  | **Prof. Alice Smith** | Fall 2025 | ❌ 0% (No Exams) | 📝 100% (3 Papers) | Optional | ⭐ 4.6 |
  | **Prof. Bob Jones** | Spring 2025 | 📝 85% (Midterm+Final)| ❌ 10% (Short quizzes)| Mandatory | ⭐ 3.7 |

### Dimension 2: The Professor Detail View (`/professors/alice-smith`)
* Instructor teaching quality and office-hours accessibility scores.
* Most common student tags (`Passionate`, `Clear Feedback`, `Cold Caller`).
* **All Courses Taught by this Professor**:
  - `PHIL-0001`: Intro to Philosophy (Rating: 4.6, Workload: 4.8 hrs)
  - `PHIL-0038`: Philosophy of Law (Rating: 4.8, Workload: 6.5 hrs)

---

## 7. Faceted Filter & Search Taxonomy

To deliver what Tufts students actually care about, the search page provides:

### 1. HASS Requirements (Filter by Tufts Degree Audit Requirements)
* `[ ] Humanities (HUM)`
* `[ ] Arts (ART)`
* `[ ] Social Sciences (SOC)`

### 2. Assessment Types (The Dealbreakers)
* `[ ] No Exams` (No midterm, no final exam)
* `[ ] No Essays` (No long papers / thesis requirements)
* `[ ] Exam-focused Only` (For students who prefer multiple-choice/midterms over writing)
* `[ ] Project-based / Portfolio`

### 3. Workload & Logistics
* **Max Hours / Week Slider**: `[ 0 hrs ] ────────●───── [ 15+ hrs ]` (Set ceiling, e.g. 5 hrs/wk)
* `[ ] Attendance Not Mandatory`
* `[ ] Recorded Lectures Available`
* `[ ] Generous Grader` (High proportion of A / A- grades reported)

---

## 8. Step-by-Step Implementation Roadmap

### Phase 1: Project Setup & Database Scaffolding
- [ ] Initialize repository: `npx create-next-app@latest jumbo-reviews --typescript --tailwind --app`
- [ ] Install UI library: `npx shadcn@latest init`
- [ ] Install component primitives: `button`, `card`, `input`, `badge`, `slider`, `checkbox`, `dialog`, `select`, `table`
- [ ] Set up free PostgreSQL database on **Supabase** or **Neon**.
- [ ] Install Prisma ORM: `npm install prisma @prisma/client`
- [ ] Run initial migrations to build the schema.

### Phase 2: Verified Tufts Authentication
- [ ] Create an OAuth 2.0 Client ID in **Google Cloud Console**.
- [ ] Configure **Auth.js / NextAuth v5** in Next.js.
- [ ] Add domain constraint callback in `auth.ts`:
  ```typescript
  callbacks: {
    async signIn({ account, profile }) {
      if (account?.provider === "google") {
        return !!profile?.email?.endsWith("@tufts.edu");
      }
      return false;
    },
  }
  ```
- [ ] Add user session indicator and login/logout button to the navigation bar.

### Phase 3: Review Submission & Dynamic Course Linking
- [ ] Build the interactive review modal with structured checkboxes and sliders.
- [ ] Create a searchable course selector with an inline `+ Add Missing Course` modal.
- [ ] Create a professor selector with an inline `+ Add Missing Professor` modal.
- [ ] Write the Next.js Server Action to validate inputs and insert rows into `reviews` and `course_offerings`.

### Phase 4: Search Engine & Aggregated Dashboards
- [ ] Build the search page with URL query parameters (e.g. `?hass=humanities&no_exams=true&max_hours=6`).
- [ ] Write SQL queries to aggregate review metrics:
  - Average weekly hours: `AVG(weekly_hours)`
  - Exam probability: `COUNT(CASE WHEN has_exams THEN 1 END) / COUNT(*)`
  - Overall score: `AVG(overall_rating)`
- [ ] Build Course Detail (`/courses/[id]`) and Professor Detail (`/professors/[id]`) pages.

### Phase 5: Testing, Content Moderation & Launch
- [ ] Add basic client/server profanity and personal attack filtering on qualitative text.
- [ ] Add a "Flag Review" button for community reporting.
- [ ] Seed initial reviews with 20–30 fellow engineering students.
- [ ] Deploy live to **Vercel** with a clean domain.

---

## 9. Campus Pitch & University Adoption Strategy

When your platform has working traction (100–300 honest reviews), pitch it to university leadership:

### Target Audiences
1. **Tufts Community Union (TCU) Senate**: The student government funds student software projects and can grant official recognition.
2. **Associate Dean of Undergraduate Education (School of Engineering)**: Frame this as a **student well-being and retention tool**—helping engineers fulfill their 8 HASS credits without academic burnout.
3. **Tufts Technology Services (TTS)**: Transition the login from Google OAuth to official Tufts Shibboleth SSO once you have official sponsorship.

### Key Points for the Pitch Deck
* **Constructive Culture**: Focuses on **course mechanics** (workload hours, exam formats) rather than personal attacks on faculty.
* **Privacy & FERPA Compliant**: Completely independent, anonymous peer feedback with no private student record access.
* **Community Benefit**: Solves a real dilemma that every single Tufts engineer encounters during course registration.
