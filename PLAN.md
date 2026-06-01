# Prompt Dojo — Enterprise Prompt Library

## Goal
A web application where enterprise users can store, share, search, tag, and track changes to prompts. No LLM execution. Auth via SSO (OIDC/OAuth). Audit log tracks who changed what and when.

---

## Tech Stack
| Layer | Choice |
|---|---|
| Framework | Next.js 14 (App Router, TypeScript) |
| Database | PostgreSQL via Prisma ORM |
| Auth | NextAuth.js v5 with a generic OIDC provider |
| Styling | Tailwind CSS + shadcn/ui |
| Search | Postgres full-text search (no external service needed) |

---

## Data Model (Prisma schema)

```
User
  id, email, name, image, role (VIEWER | EDITOR | ADMIN)
  createdAt

Category
  id, name, slug, description, createdAt

Tag
  id, name, slug

Prompt
  id, title, content (text), description
  categoryId -> Category
  tags -> Tag[] (many-to-many via PromptTag)
  createdById -> User
  updatedById -> User
  createdAt, updatedAt
  isPublic (boolean — visible to all enterprise users)
  searchVector (tsvector, generated column for FTS)

AuditLog
  id, entityType (PROMPT | CATEGORY | TAG | USER)
  entityId, action (CREATED | UPDATED | DELETED)
  userId -> User
  metadata (JSON — what fields changed, old/new values)
  createdAt
```

---

## Application Routes

```
/                         → redirect to /prompts
/login                    → NextAuth sign-in (SSO redirect)

/prompts                  → browse + search all prompts (paginated, filterable by category/tag)
/prompts/new              → create a new prompt
/prompts/[id]             → view prompt detail + audit log timeline
/prompts/[id]/edit        → edit prompt

/categories               → manage categories (ADMIN / EDITOR)
/admin                    → user management, role assignment (ADMIN only)
```

---

## Key Features

### 1. Prompt CRUD
- Rich text area for prompt content (no markdown renderer needed — prompts are plain text)
- Title, description, category, tags
- Public/private toggle (private = only creator can see)
- Server Actions for create/update/delete

### 2. Search & Filter
- Postgres `tsvector` full-text search across title + content + description
- Filter sidebar: category, tags, author, date range
- URL-driven filters (shareable search URLs)

### 3. SSO Auth (NextAuth OIDC)
- Single OIDC provider configured via env vars (`OIDC_ISSUER`, `OIDC_CLIENT_ID`, `OIDC_CLIENT_SECRET`)
- On first login, user row is auto-created with VIEWER role
- ADMIN promotes users via `/admin`

### 4. Audit Log
- Middleware-layer hook: every Server Action that mutates a Prompt writes an AuditLog row
- Stores: userId, timestamp, action, JSON diff (field → {old, new})
- Displayed as a timeline on `/prompts/[id]` (who, when, what changed)

### 5. Role-Based Access
| Role | Can do |
|---|---|
| VIEWER | Read any public prompt |
| EDITOR | Create, edit own prompts; create categories/tags |
| ADMIN | Edit/delete any prompt; manage users and roles |

---

## Implementation Phases

### Phase 1 — Scaffold & Auth
1. `npx create-next-app@latest` with TypeScript + Tailwind + App Router
2. Install: `prisma`, `@prisma/client`, `next-auth@beta`, `shadcn/ui`
3. Write Prisma schema (all models above)
4. Configure NextAuth with OIDC provider + Prisma adapter
5. Protect all `/prompts` and `/admin` routes via middleware

### Phase 2 — Prompt Library Core
6. `/prompts` page: paginated list with search bar + filter sidebar
7. `/prompts/new` and `/prompts/[id]/edit`: form with category + tag pickers
8. `/prompts/[id]`: detail view (read-only display of content)
9. Server Actions: `createPrompt`, `updatePrompt`, `deletePrompt`
10. FTS: add `searchVector` generated column + GIN index to migration

### Phase 3 — Audit Log
11. `auditLog()` helper that writes to AuditLog inside the same Prisma transaction as the mutation
12. Diff utility: compare old/new prompt fields, serialize changed fields to JSON
13. Audit timeline component on prompt detail page

### Phase 4 — Admin & Polish
14. `/admin` user table: list users, change role
15. `/categories` management page
16. Empty states, loading skeletons, error boundaries
17. Docker Compose file for local dev (Next.js + Postgres)
18. `.env.example` documenting all required variables

---

## Environment Variables
```
DATABASE_URL=
NEXTAUTH_SECRET=
NEXTAUTH_URL=
OIDC_ISSUER=
OIDC_CLIENT_ID=
OIDC_CLIENT_SECRET=
```

---

## What's NOT included (scope boundary)
- LLM prompt execution / testing
- Version history / rollback (audit log only — no restore)
- Multi-tenant / org isolation (single enterprise deployment)
- Email notifications
