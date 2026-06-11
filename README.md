# High Level Design (HLD)
# School Marks & Attendance Management System
Version 1.0 | June 2026

---

## 1. System Overview

A single-school, single-class web application serving three roles: super admin, teachers, and parents. Built as a monolithic Next.js application deployed on Vercel, with Aiven as the database and storage backend. No separate backend server.

---

## 2. System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        CLIENT BROWSER                        │
│                                                             │
│   Next.js App Router (React)                                │
│   Tailwind CSS + shadcn/ui + i18next                        │
└────────────────────────┬────────────────────────────────────┘
                         │ HTTPS
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                     VERCEL EDGE NETWORK                      │
│                                                             │
│   ┌─────────────────────────────────────────────────────┐   │
│   │           Next.js Serverless Functions               │   │
│   │                                                     │   │
│   │   /app/api/*  (API Routes)                          │   │
│   │   NextAuth.js (Session Management)                  │   │
│   │   Prisma Client (ORM)                               │   │
│   │   Zod (Input Validation)                            │   │
│   │   SheetJS (Excel Generation)                        │   │
│   └──────────────────┬──────────────────────────────────┘   │
└─────────────────────-│──────────────────────────────────────┘
                       │
          ┌────────────┴────────────┐
          │                         │
          ▼                         ▼
┌─────────────────┐       ┌─────────────────────┐
│    Aiven     │       │   Aiven Storage  │
│   PostgreSQL    │       │                     │
│                 │       │   /student-photos/  │
│   - users       │       │   (max 1GB free)    │
│   - students    │       └─────────────────────┘
│   - teachers    │
│   - parents     │
│   - subjects    │
│   - exam_types  │
│   - marks       │
│   - attendance  │
│   - grading_    │
│     scale       │
│   - audit_logs  │
└─────────────────┘
```

---

## 3. User Roles & Access Boundaries

```
┌──────────────────────────────────────────────────────────────┐
│                        SUPER ADMIN                            │
│  Full access to all data, configuration, and reports          │
│  Single account, seeded manually                              │
└──────────────────────────────────────────────────────────────┘
              │
              ├──────────────────────────────────┐
              ▼                                  ▼
┌─────────────────────────┐          ┌───────────────────────┐
│        TEACHERS (6)      │          │      PARENTS (80)      │
│                          │          │                        │
│  Regular Teacher:        │          │  Login: roll number    │
│  - Own subject marks     │          │  Read-only access:     │
│  - Own subject analysis  │          │  - Child's marks       │
│  - Own subject report    │          │  - Child's attendance  │
│                          │          │  - Child's analysis    │
│  Class Teacher (+):      │          │  - Change password     │
│  - Daily attendance      │          └───────────────────────┘
│  - Full class analysis   │
└─────────────────────────┘
```

---

## 4. Data Flow

### 4.1 Mark Upload Flow (Excel)
```
Teacher selects exam → Downloads blank template
→ Fills marks offline
→ Uploads filled Excel
→ API validates (reject all on any error)
→ Show errors OR upsert all marks to DB
→ Confirmation shown to teacher
```

### 4.2 Mark Calculation Flow
```
Raw marks stored in DB
→ Query fetches raw marks
→ Application layer computes:
   - Kannada FA conversion (×25/20)
   - SA1 total (written + FA contribution)
   - Final total (written + FA contribution)
   - Grade (from grading scale table)
   - Rank (percentage-based, per exam)
→ Computed values returned to client
→ Never stored in DB
```

### 4.3 Parent View Flow
```
Parent logs in with roll number + password
→ Session created with role: PARENT
→ All API calls scoped to parent's studentId
→ Parent sees only their child's data
```

### 4.4 Academic Year Reset Flow
```
Admin navigates to reset page
→ System shows consequences (what gets deleted)
→ Admin types "RESET" to confirm
→ API hard deletes: marks, attendance
→ Audit log entry created (never deleted)
→ System ready for new academic year
```

---

## 5. Authentication Flow

```
User visits /login
→ Enters credentials (loginId + password)
→ NextAuth CredentialsProvider validates against DB
→ bcrypt.compare(password, hash)
→ Session created with: { id, name, role, subjectId?, isClassTeacher? }
→ Middleware redirects to role-specific dashboard:
   - ADMIN  → /admin/dashboard
   - TEACHER → /teacher/dashboard
   - PARENT → /parent/dashboard
```

---

## 6. Language Switching

```
User toggles language (EN/KN)
→ i18next switches active namespace
→ UI labels re-render in selected language
→ Subject names fetched from DB in selected language
→ Preference saved to localStorage
→ Excel generated in language selected at download time
```

---

## 7. Deployment Architecture

```
GitHub Repository
       │
       │ push to main
       ▼
   Vercel CI/CD
       │
       ├── Build: next build
       ├── Lint + Type check
       └── Deploy to Vercel Edge Network
                │
                ├── Static assets (CDN cached)
                ├── Server components (SSR)
                └── API routes (serverless functions)
                              │
                              └── Connects to Aiven
                                  (DATABASE_URL via env vars)
```

---

## 8. Security Model

| Concern | Solution |
|---------|----------|
| Auth | NextAuth.js session cookies (httpOnly, secure) |
| Password storage | bcrypt, min 10 rounds |
| Role enforcement | Server-side middleware on every route |
| Subject ownership | API verifies teacher's subjectId on every marks call |
| SQL injection | Prisma parameterized queries |
| File upload | Size limit (500KB), type validation, Aiven Storage |
| Excel upload | Full validation before any DB write |
| Session expiry | NextAuth default (30 days), configurable |

---

## 9. Free Tier Constraints & Mitigations

| Service | Free Tier Limit | Expected Usage | Risk |
|---------|----------------|----------------|------|
| Vercel | 100GB bandwidth/month | ~1GB/month | None |
| Vercel | 100k serverless invocations/month | ~5k/month | None |
| Aiven DB | 1024MB storage | ~50MB | None |
| Aiven Storage | 1GB | ~40MB (photos) | None |
| Aiven | 50k monthly active users | 90 users | None |

---

## 10. Key Design Decisions

| Decision | Choice | Reason |
|----------|--------|--------|
| Backend | Next.js API routes | No separate server, free tier, single deployment |
| Database | Aiven PostgreSQL | Free, managed, 500MB sufficient |
| ORM | Prisma | Type safety, easy migrations |
| Auth | NextAuth.js | Built for Next.js, session management included |
| Computed values | At query time | Avoids data staleness, simpler writes |
| Excel | Client-side SheetJS | No server memory needed for generation |
| Reset | Hard delete | School requirement, simpler than archiving |
| Multi-school | Not supported | Single school scope |