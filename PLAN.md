# Prompt Dojo — Enterprise Prompt Library
## Complete Implementation Plan

---

## Stack

| Layer | Choice | Reason |
|---|---|---|
| Framework | Next.js 14 (App Router, TypeScript) | SSR + Server Actions, no separate API layer needed |
| ORM | Prisma + PostgreSQL (AWS RDS) | Type-safe queries, migration tooling |
| Auth | NextAuth.js v5 + Okta OIDC | Native Okta issuer support |
| Styling | Tailwind CSS + shadcn/ui | Rapid enterprise UI |
| Search | Postgres `tsvector` + GIN index | No external service needed at this scale |
| Hosting | AWS ECS/Fargate + ALB + RDS Postgres | Containerized, managed infra |
| Secrets | AWS Secrets Manager | Injected as env vars into ECS task |

---

## Data Model (Prisma)

```prisma
enum Role { VIEWER EDITOR ADMIN }
enum PromptStatus { DRAFT PUBLISHED }
enum AuditAction { CREATED UPDATED PUBLISHED RESTORED DELETED }
enum AuditEntity { PROMPT CATEGORY TAG USER COMMENT }

model User {
  id            String   @id @default(cuid())
  email         String   @unique
  name          String?
  image         String?
  role          Role     @default(VIEWER)
  createdAt     DateTime @default(now())

  prompts       Prompt[]       @relation("PromptOwner")
  updatedPrompts Prompt[]      @relation("PromptUpdater")
  versions      PromptVersion[]
  comments      Comment[]
  favorites     Favorite[]
  ratings       PromptRating[]
  auditLogs     AuditLog[]
}

model Category {
  id          String   @id @default(cuid())
  name        String   @unique
  slug        String   @unique
  description String?
  createdAt   DateTime @default(now())
  prompts     Prompt[]
}

model Tag {
  id        String      @id @default(cuid())
  name      String      @unique
  slug      String      @unique
  createdAt DateTime    @default(now())
  prompts   PromptTag[]
}

model Prompt {
  id           String       @id @default(cuid())
  title        String
  description  String?
  content      String       -- plain text, may contain {{variable}} placeholders
  variables    Json         -- [{name: string, description: string}] extracted from content
  status       PromptStatus @default(DRAFT)
  categoryId   String?
  category     Category?    @relation(fields: [categoryId], references: [id])
  createdById  String
  createdBy    User         @relation("PromptOwner", fields: [createdById], references: [id])
  updatedById  String?
  updatedBy    User?        @relation("PromptUpdater", fields: [updatedById], references: [id])
  createdAt    DateTime     @default(now())
  updatedAt    DateTime     @updatedAt
  copyCount    Int          @default(0)

  tags         PromptTag[]
  versions     PromptVersion[]
  comments     Comment[]
  favorites    Favorite[]
  ratings      PromptRating[]
  auditLogs    AuditLog[]
}
-- searchVector tsvector generated always as
--   to_tsvector('english', title || ' ' || coalesce(description,'') || ' ' || content)
--   stored (added via raw SQL migration)
-- GIN index on searchVector

model PromptTag {
  promptId String
  tagId    String
  prompt   Prompt @relation(fields: [promptId], references: [id], onDelete: Cascade)
  tag      Tag    @relation(fields: [tagId], references: [id], onDelete: Cascade)
  @@id([promptId, tagId])
}

model PromptVersion {
  id          String   @id @default(cuid())
  promptId    String
  prompt      Prompt   @relation(fields: [promptId], references: [id], onDelete: Cascade)
  title       String
  description String?
  content     String   -- full snapshot of content at time of save
  variables   Json
  createdById String
  createdBy   User     @relation(fields: [createdById], references: [id])
  createdAt   DateTime @default(now())
  restoredFrom String? -- id of the PromptVersion this was restored from
}

model AuditLog {
  id         String      @id @default(cuid())
  entityType AuditEntity
  entityId   String
  promptId   String?
  prompt     Prompt?     @relation(fields: [promptId], references: [id])
  action     AuditAction
  userId     String
  user       User        @relation(fields: [userId], references: [id])
  metadata   Json        -- {fields: [{field, old, new}], note: string}
  createdAt  DateTime    @default(now())
}

model Comment {
  id        String    @id @default(cuid())
  promptId  String
  prompt    Prompt    @relation(fields: [promptId], references: [id], onDelete: Cascade)
  userId    String
  user      User      @relation(fields: [userId], references: [id])
  content   String
  createdAt DateTime  @default(now())
  updatedAt DateTime  @updatedAt
  deletedAt DateTime? -- soft delete
}

model Favorite {
  userId    String
  promptId  String
  user      User   @relation(fields: [userId], references: [id])
  prompt    Prompt @relation(fields: [promptId], references: [id], onDelete: Cascade)
  createdAt DateTime @default(now())
  @@id([userId, promptId])
}

model PromptRating {
  userId    String
  promptId  String
  user      User   @relation(fields: [userId], references: [id])
  prompt    Prompt @relation(fields: [promptId], references: [id], onDelete: Cascade)
  value     Int    -- 1–5
  updatedAt DateTime @updatedAt
  @@id([userId, promptId])
}
```

---

## Application Routes

```
/                             → redirect to /prompts
/login                        → NextAuth sign-in → Okta redirect → callback → session

/prompts                      → browse PUBLISHED prompts (search, filter by category/tag/author)
/prompts/new                  → create prompt [EDITOR+]
/prompts/[id]                 → view prompt detail, variable fill form, copy, comment, rate
/prompts/[id]/edit            → edit prompt [owner or ADMIN]
/prompts/[id]/history         → full version timeline with content, diff view, restore [owner or ADMIN]

/my-prompts                   → my drafts + published [EDITOR+]
/favorites                    → my bookmarked prompts

/categories/[slug]            → prompts filtered by category

/admin                        → admin dashboard [ADMIN]
/admin/users                  → list users, assign roles
/admin/categories             → CRUD categories (admin-curated)
```

---

## Access Control Matrix

| Action | VIEWER | EDITOR | ADMIN |
|---|:---:|:---:|:---:|
| View published prompts | ✓ | ✓ | ✓ |
| View own drafts | — | ✓ | ✓ |
| Create prompt | — | ✓ | ✓ |
| Edit own prompt | — | ✓ | ✓ |
| Edit any prompt | — | — | ✓ |
| Publish own prompt | — | ✓ | ✓ |
| Restore version | — | owner only | ✓ |
| Delete prompt | — | owner only | ✓ |
| Create tags | — | ✓ | ✓ |
| Manage categories | — | — | ✓ |
| Manage users/roles | — | — | ✓ |
| Comment, rate, favorite | ✓ | ✓ | ✓ |

---

## Key Features — Implementation Notes

### Auth (Okta OIDC)
- NextAuth v5 with `OktaProvider(issuer, clientId, clientSecret)`
- `signIn` callback: upsert User row on first login (name, email, image from ID token claims)
- `session` callback: attach `user.id` and `user.role` to the session object
- Middleware: redirect unauthenticated requests from `/prompts*`, `/admin*`, `/favorites*`, `/my-prompts` to `/login`

### Prompt Templates (variable detection)
- Parse `{{variable_name}}` from content using regex `/\{\{([a-zA-Z_][a-zA-Z0-9_]*)\}\}/g`
- Auto-extract unique variable names on save; merge with existing descriptions
- Variable editor: list of detected names + freetext description field per variable (no types)
- Copy-with-fill: prompt opens a modal with one input per variable → substitutes and copies to clipboard

### Draft / Publish Workflow
- Prompts start as DRAFT (only visible to owner and ADMIN)
- Owner or ADMIN clicks "Publish" → status → PUBLISHED, audit log entry with action PUBLISHED
- Published prompts can be un-published back to DRAFT by owner or ADMIN

### Version History + Restore
- Every `updatePrompt` and `restoreVersion` Server Action writes a `PromptVersion` snapshot in the **same Prisma transaction** as the Prompt update
- `/prompts/[id]/history` page: timeline of versions, newest first
  - Each entry: avatar, timestamp, action label, collapsed diff (old content → new content, character-level diff)
  - "View" button: full-screen side-by-side content viewer
  - "Restore" button (owner / ADMIN): creates a new version from the snapshot, updates Prompt to that content, writes AuditLog entry with action RESTORED and `metadata.restoredFrom = versionId`
- Diffs rendered client-side with `diff` npm package

### Search
- `searchVector` tsvector generated column (raw SQL migration) on Prompt
- GIN index on `searchVector`
- Prisma `$queryRaw` for FTS query: `to_tsquery('english', ...)` with prefix matching for live search
- Filter params encoded in URL query string: `?q=&category=&tag=&author=&status=`

### Social Features
| Feature | Mechanic |
|---|---|
| **Favorites** | Toggle heart → upsert/delete Favorite row; `/favorites` page lists favorited prompts |
| **Comments** | Threaded list on prompt detail; soft-delete (show "deleted" placeholder); EDITOR edits own, ADMIN deletes any |
| **Ratings** | 1–5 star UI; upsert PromptRating; average computed at query time via `_avg` Prisma aggregate |
| **Copy count** | "Copy" button triggers `incrementCopyCount` Server Action → `UPDATE SET copyCount = copyCount + 1` |

### Audit Log
- `auditLog(tx, data)` helper accepts a Prisma transaction client; always runs inside the mutating transaction
- Stored metadata schema: `{ fields: [{field, old, new}], note?: string }`
- For RESTORED action: `{ restoredFrom: versionId, restoredAt: timestamp }`
- Displayed on prompt detail in a collapsible "Activity" section at the bottom

---

## Implementation Phases

### Phase 1 — Scaffold, Auth, DB (Week 1)
1. `npx create-next-app@latest` with TS + Tailwind + App Router + ESLint
2. Install: `prisma @prisma/client next-auth@beta @auth/prisma-adapter`
3. Install shadcn/ui: `npx shadcn@latest init` + core components (Button, Input, Dialog, Dropdown, Badge, Avatar, Textarea, Tabs, Skeleton)
4. Write full Prisma schema; `prisma migrate dev --name init`
5. Raw SQL migration: add `searchVector` generated column + GIN index
6. NextAuth config: Okta provider, Prisma adapter, `signIn` + `session` callbacks
7. Route middleware: session check + role guard
8. Login page: "Sign in with Okta" button, branded layout

### Phase 2 — Prompt CRUD + Templates (Week 2)
9. Layout: sidebar nav (Browse, My Prompts, Favorites, Admin), top bar with user avatar + logout
10. `/prompts` list page: paginated cards (title, description snippet, category, tags, star rating, copy count, author avatar), search bar, filter sidebar
11. `/prompts/new` and `/prompts/[id]/edit`: form with title, description, content textarea, variable detection (live parse on content change), variable description editor, category select, tag multi-input, draft/publish toggle
12. Server Actions: `createPrompt`, `updatePrompt` (with version snapshot + audit log), `publishPrompt`, `deletePrompt`
13. `/prompts/[id]` detail view: content display with variable highlighting, copy-with-fill modal, metadata panel, rating stars, favorite toggle, copy count display

### Phase 3 — Version History + Restore (Week 3)
14. `/prompts/[id]/history`: sorted version list, diff rendering (character-level, side-by-side)
15. `restoreVersion` Server Action: snapshot + audit entry with RESTORED action
16. Inline "Activity" audit log component on prompt detail (collapsed by default)

### Phase 4 — Social Features (Week 3–4)
17. Comments: add/edit/delete on prompt detail, optimistic UI
18. Ratings: star widget with hover state, `upsertRating` Server Action, average on prompt card
19. Favorites: heart toggle on cards and detail, `/favorites` page
20. Copy count: increment on copy, displayed on card and detail

### Phase 5 — Admin + AWS Infra (Week 4)
21. `/admin/users`: paginated user table, role dropdown (ADMIN cannot demote self)
22. `/admin/categories`: create/rename/delete category, prompt count per category
23. `/my-prompts`: tabs for DRAFT and PUBLISHED, sorted by updatedAt
24. Dockerfile: multi-stage (`node:20-alpine` build → `node:20-alpine` runtime)
25. `docker-compose.yml`: app + Postgres for local dev, with seed script
26. `ecs-task-definition.json` template: Fargate, Secrets Manager references for DB URL + Okta secrets
27. GitHub Actions CI: lint → type-check → `prisma migrate deploy` → Docker build + push to ECR
28. `.env.example`: all required variables documented

---

## Environment Variables

```env
# Database
DATABASE_URL=postgresql://user:pass@host:5432/dbname

# NextAuth
NEXTAUTH_SECRET=          # 32+ char random string
NEXTAUTH_URL=             # https://your-app-domain.com

# Okta OIDC
OKTA_ISSUER=              # https://your-org.okta.com
OKTA_CLIENT_ID=
OKTA_CLIENT_SECRET=
```

---

## Scope Boundary (explicit non-goals for v1)

- No LLM execution / prompt testing inside the app
- No multi-tenant / org isolation (single enterprise deployment, one Okta tenant)
- No email notifications
- No GitHub/GitLab SSO (Okta only)
- No import/export (CSV, JSON) — may add in v2
- No team/group scoping of prompts (org-wide visibility model)
