# Book Club & Reading Manager — Phased Development Plan

> Project: 362-book-club-reading-manager · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and `data-model-suggestion-1.md` (the Entity-Centric Normalized Relational model, which is adopted wholesale as the persistence layer). It targets the **MVP scope** from `features.md` first, then layers the AI-native differentiators (spoiler detection, discussion-question generation, recommendations, finish-date prediction) and integration features (video, audiobook, federation) in later phases.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language | TypeScript (Node.js 22 LTS) | Project is API + real-time + web-frontend heavy, not ML-heavy. Shared types between server, WebSocket layer, and a future React Native client reduce drift. LLM calls go through provider SDKs, so Python's ML edge is irrelevant. |
| API framework | NestJS 11 | Module/provider DI maps cleanly onto the 16-table domain (clubs, books, discussions, meetings). First-class OpenAPI 3.1 generation (`@nestjs/swagger`), WebSocket gateways, guards for the access-control rules OWASP A01 demands. |
| Database | PostgreSQL 16 | `data-model-suggestion-1.md` is Postgres-specific: `UUID`, `TEXT[]`, `JSONB`, GIN full-text indexes, range-partitioned `audit_log`. Spoiler gating is a precise integer subquery (see model §Spoiler-gated threads) that needs relational integrity. |
| ORM / migrations | Drizzle ORM + drizzle-kit | Thin, SQL-first; preserves the hand-tuned DDL and partial/GIN indexes from the data model rather than hiding them. Type-safe queries match the TS-everywhere choice. Generates SQL migrations checked into the repo. |
| Cache / queue | Redis 7 + BullMQ | Async workloads: LLM discussion-question generation, finish-date prediction batch jobs, push-notification fan-out, meeting reminders, book-metadata enrichment. Redis also backs the WebSocket pub/sub adapter for horizontal scaling. |
| Real-time | Socket.IO (NestJS WebSocket gateway) + Redis adapter | RFC 6455 WebSockets for live discussion posts and reading-progress broadcasts. Socket.IO gives reconnection + room semantics (one room per club selection) out of the box. |
| LLM provider | Anthropic Claude via `@anthropic-ai/sdk`, behind a provider interface | AI is the core differentiator (discussion questions, spoiler classification, recommendations, meeting summaries). Provider interface keeps it swappable. Prompt caching reduces cost on repeated book-context prompts. |
| Auth | OIDC (Sign in with Google + Apple) + email/password, JWT sessions | `standards.md`: OAuth 2.0 (RFC 6749), OIDC, JWT (RFC 7519). Google OAuth doubles as the Google Books "My Library" authorisation grant. |
| Frontend | Next.js 15 (App Router, React 19) + Tailwind + shadcn/ui | Server-rendered club/book pages get Schema.org Book JSON-LD for SEO (standards.md). shadcn/ui gives accessible primitives. SPA-like interactivity for discussion boards and progress dashboards. |
| Book metadata | Open Library (primary, no key) → Google Books (OAuth/key) → ISBNdb (paid fallback) | Cascade matches `standards.md` guidance: Open Library for free reusable metadata, Google Books for breadth, ISBNdb for obscure titles. |
| Push notifications | FCM (web + Android via VAPID, iOS via APNs relay) | `standards.md` recommends FCM as the single integration point covering RFC 8030/8291/8292 web push and mobile. |
| Validation | Zod (shared schemas) + `nestjs-zod` | JSON Schema 2020-12 alignment; one Zod schema validates the HTTP boundary and produces the OpenAPI component schema. |
| Testing | Vitest (unit), Supertest (HTTP integration), Playwright (E2E), Testcontainers (real Postgres/Redis) | Fast unit runner; Testcontainers gives real Postgres for spoiler-gating queries that cannot be meaningfully mocked. |
| Code quality | ESLint (typescript-eslint) + Prettier + `tsc --noEmit` | Standard TS toolchain; type checking is a Definition-of-Done gate. |
| Package manager / monorepo | pnpm workspaces + Turborepo | Monorepo holds `api`, `web`, `worker`, and shared `packages/*` (types, db, llm). pnpm for disk-efficient installs. |
| Containerisation | Docker + docker-compose (Postgres, Redis, api, worker, web) | Self-hosting is on the README roadmap; compose gives a one-command dev/self-host stack. |

### Project Structure

```
book-club-reading-manager/
├── package.json                  # pnpm workspace root
├── pnpm-workspace.yaml
├── turbo.json
├── docker-compose.yml            # postgres, redis, api, worker, web
├── .env.example
├── packages/
│   ├── db/                       # Drizzle schema + migrations + client
│   │   ├── src/schema/           # one file per domain group (users, books, clubs…)
│   │   ├── src/client.ts
│   │   ├── migrations/           # generated SQL migrations
│   │   └── drizzle.config.ts
│   ├── types/                    # shared Zod schemas + inferred TS types
│   │   └── src/                  # book.ts, club.ts, discussion.ts, progress.ts…
│   ├── llm/                      # provider interface + Claude impl + prompt templates
│   │   ├── src/provider.ts
│   │   ├── src/anthropic.ts
│   │   └── src/prompts/          # discussion-questions.ts, spoiler-classify.ts…
│   └── book-metadata/            # Open Library / Google Books / ISBNdb cascade client
│       └── src/
├── apps/
│   ├── api/                      # NestJS HTTP + WebSocket
│   │   ├── src/
│   │   │   ├── main.ts
│   │   │   ├── app.module.ts
│   │   │   ├── auth/             # guards, JWT, OIDC strategies
│   │   │   ├── users/
│   │   │   ├── books/
│   │   │   ├── reading/          # user_books, reading_sessions, progress
│   │   │   ├── clubs/            # clubs, members, selections, votes
│   │   │   ├── discussions/      # threads, posts, spoiler gating, ws gateway
│   │   │   ├── meetings/         # meetings, rsvps, video, summaries
│   │   │   ├── challenges/
│   │   │   ├── recommendations/
│   │   │   ├── notifications/
│   │   │   ├── common/           # guards, interceptors, pagination, audit
│   │   │   └── mcp/              # MCP server (Phase 11)
│   │   └── test/
│   ├── worker/                   # BullMQ processors (LLM jobs, reminders, enrichment)
│   │   └── src/
│   └── web/                      # Next.js 15 App Router
│       ├── app/
│       └── components/
└── tests/
    └── e2e/                      # Playwright specs
```

The structure is grouped by domain concern, not by phase; every phase adds modules/files without restructuring.

---

## Phase 1: Foundation — Monorepo, Database, Auth

### Purpose
Establish the monorepo, the full database schema, and authenticated identity. Nothing in the product works without users, a migrated schema, and JWT-scoped requests. After this phase a developer can sign up, sign in, receive a JWT, and hit a protected health endpoint backed by a real migrated Postgres database.

### Tasks

#### 1.1 — Monorepo, tooling, and Docker baseline

**What**: Scaffold the pnpm/Turborepo workspace with `api`, `worker`, `web`, and shared packages, plus a docker-compose stack for Postgres and Redis.

**Design**:
- `pnpm-workspace.yaml` includes `apps/*` and `packages/*`.
- `turbo.json` pipelines: `build`, `lint`, `test`, `typecheck` (with `dependsOn: ["^build"]`).
- `docker-compose.yml` services: `postgres` (16, volume `pgdata`, healthcheck `pg_isready`), `redis` (7, healthcheck `redis-cli ping`), `api`, `worker`, `web`.
- `.env.example` keys: `DATABASE_URL`, `REDIS_URL`, `JWT_SECRET`, `JWT_EXPIRES_IN=15m`, `REFRESH_EXPIRES_IN=30d`, `GOOGLE_OAUTH_CLIENT_ID`, `GOOGLE_OAUTH_CLIENT_SECRET`, `APPLE_*`, `ANTHROPIC_API_KEY`, `OPEN_LIBRARY_BASE`, `GOOGLE_BOOKS_API_KEY`, `ISBNDB_API_KEY`, `FCM_*`, `VAPID_PUBLIC_KEY`, `VAPID_PRIVATE_KEY`, `WEB_ORIGIN`.
- Root scripts wire ESLint, Prettier, and `tsc --noEmit` across all packages.

**Testing**:
- `Unit: turbo lint` passes on the empty scaffold.
- `Integration (real): docker-compose up postgres redis` → both containers report healthy within 30s.
- `Smoke: pnpm -r typecheck` → exit 0.

#### 1.2 — Drizzle schema for all 16 tables

**What**: Implement the complete schema from `data-model-suggestion-1.md` in `packages/db`, including enums (as CHECK constraints), array/JSONB columns, and every index.

**Design**:
- One schema file per group: `users.ts`, `books.ts`, `reading.ts` (user_books, reading_sessions), `clubs.ts` (clubs, club_members, club_selections), `votes.ts`, `discussions.ts` (threads, posts), `meetings.ts` (meetings, rsvps), `challenges.ts`, `audit.ts`.
- Reproduce DDL faithfully: `gen_random_uuid()` defaults, `TEXT[]` via `text().array()`, `chapters_json` as `jsonb` typed to `Chapter[]` where `Chapter = { number: int; title: string; start_page: int; end_page: int }`.
- Preserve partial/GIN indexes (e.g. `idx_books_title GIN to_tsvector`, `idx_selections_current WHERE status='reading'`) using raw SQL in migration files where Drizzle's index builder is insufficient.
- `audit_log` is `PARTITION BY RANGE (created_at)`; create the parent plus a current-month partition in the migration, and a documented helper to roll partitions.
- `packages/db/src/client.ts` exports a configured `drizzle(pool)` instance and the typed schema object.

**Testing**:
- `Integration (real Postgres via Testcontainers): run all migrations → \d shows 16 tables + audit partition`.
- `Integration: insert user, book, user_book → UNIQUE(user_id, book_id) enforced on duplicate (expect error)`.
- `Integration: insert discussion_thread with spoiler_chapter=3 → row persists; GIN title search on books returns inserted book`.
- `Unit: Chapter JSONB round-trips through the typed column`.

#### 1.3 — Auth module (email/password, Google + Apple OIDC, JWT)

**What**: Registration, login, OIDC social login, JWT access + refresh tokens, and an `AuthGuard` that populates `req.user` from validated claims.

**Design**:
- Endpoints:
  - `POST /auth/register` body `{ email, password, displayName, username }` → `201 { user, accessToken, refreshToken }`.
  - `POST /auth/login` body `{ email, password }` → `200 { accessToken, refreshToken }`.
  - `POST /auth/refresh` body `{ refreshToken }` → `200 { accessToken }`.
  - `GET /auth/oidc/:provider` → redirect; `GET /auth/oidc/:provider/callback` → issues tokens, upserts `users.auth_provider`.
- Passwords hashed with argon2id. JWT payload `{ sub: userId, username }`, signed HS256, 15-minute access, 30-day refresh (refresh tokens stored hashed, rotated on use).
- `AuthGuard` validates Bearer JWT; `@CurrentUser()` param decorator returns the typed user. `@Public()` decorator opts endpoints out.
- Errors: invalid creds → 401 generic message (no user-enumeration); duplicate email/username → 409.

**Testing**:
- `Integration (mocked OIDC): register → login → access protected GET /users/me returns the user`.
- `Integration: login with wrong password → 401, body has no field indicating which was wrong`.
- `Integration: expired access token → 401; valid refresh → new access token; reused (rotated) refresh → 401`.
- `Unit: argon2 verify true for correct password, false otherwise`.
- `Integration (mocked Google OIDC): callback with valid code → user upserted with auth_provider='google', tokens issued`.

#### 1.4 — Common infrastructure: error handling, pagination, audit interceptor

**What**: Global exception filter, cursor pagination helper, request validation pipe (Zod), and an audit interceptor writing to `audit_log`.

**Design**:
- Global `HttpExceptionFilter` → RFC 7807-style `{ type, title, status, detail, instance }` JSON.
- `ZodValidationPipe` rejects bad bodies with 422 listing failing paths.
- Cursor pagination: `?limit=&cursor=` where cursor is a base64 `(created_at,id)` tuple; responses `{ data, nextCursor }`.
- `AuditInterceptor` writes `{ user_id, actor_type, action, entity_type, entity_id, changes_json }` for mutating routes (decorated with `@Audited('club','create')`).

**Testing**:
- `Unit: ZodValidationPipe with missing required field → 422, error.paths includes the field`.
- `Integration: a decorated mutating request → one audit_log row with correct entity_type/action`.
- `Unit: cursor encode/decode round-trips; tampered cursor → 400`.

---

## Phase 2: Books & Metadata Cascade

### Purpose
Books are shared reference data that every other feature depends on (clubs schedule them, members track progress against them, spoiler gating reads their chapter map). This phase builds search, the three-source metadata cascade, ISBN normalisation, and deduplication so any later phase can resolve "the book this club is reading."

### Tasks

#### 2.1 — ISBN normalisation and the book-metadata cascade client

**What**: `packages/book-metadata` client that searches/resolves books via Open Library → Google Books → ISBNdb and returns a normalised `BookMetadata`.

**Design**:
```ts
interface BookMetadata {
  title: string; subtitle?: string; authors: string[];
  isbn13?: string; isbn10?: string;
  openLibraryKey?: string; googleBooksId?: string;
  coverUrl?: string; description?: string; publisher?: string;
  publishedDate?: string; pageCount?: number; language: string;
  bisacCodes: string[]; genres: string[];
  source: 'open_library' | 'google_books' | 'isbndb' | 'manual';
}
interface BookMetadataProvider {
  searchByQuery(q: string, limit: number): Promise<BookMetadata[]>;
  resolveByIsbn(isbn13: string): Promise<BookMetadata | null>;
}
```
- ISBN util: validate + convert ISBN-10 ↔ ISBN-13 (ISO 2108), store canonical ISBN-13 internally.
- Cascade: try Open Library first; if a field set is incomplete (no pageCount/cover), enrich from Google Books; ISBNdb only on full miss (respect its 1 req/s basic rate limit via a Redis token bucket).
- All outbound calls wrapped with timeout (5s), retry (2x exp backoff), and circuit breaker per provider.

**Testing**:
- `Unit: ISBN-10 "0306406152" → ISBN-13 "9780306406157"; invalid checksum → throws`.
- `Integration (mocked HTTP): Open Library hit with full metadata → no Google Books call made`.
- `Integration (mocked HTTP): Open Library miss → falls through to Google Books → ISBNdb; ISBNdb rate limit enforced`.
- `Unit: provider timeout → circuit opens after threshold, next call short-circuits`.

#### 2.2 — Book ingestion, deduplication, and chapter map

**What**: Persist resolved metadata into `books`, deduplicating by ISBN-13 then by normalised title+author, and capture the `chapters_json` map.

**Design**:
- `BooksService.findOrCreate(meta)`: lookup by `isbn_13`; else by `lower(title)+authors[0]` similarity; else insert.
- `chapters_json` populated from source where available; otherwise null (chapter-level spoiler gating degrades gracefully to page-based — see 4.3).
- Background enrichment job (BullMQ) backfills `avg_rating`, `cover_url`, missing pageCount post-insert.

**Testing**:
- `Integration: ingest same ISBN twice → one books row`.
- `Integration: ingest two editions with same ISBN-13 → deduped; different ISBN-13 same title → two rows`.
- `Unit: chapters_json parses to Chapter[] and is queryable`.

#### 2.3 — Books API

**What**: REST endpoints for search and detail.

**Design**:
- `GET /books/search?q=&isbn=&limit=` → cascade search, persists hits, returns `BookMetadata[]` with internal `id`.
- `GET /books/:id` → full record including `chapters_json`.
- `POST /books/manual` (auth) → create a `source='manual'` book for titles absent from all APIs (closes the Bookclubs.com gap "cannot schedule meetings for books not yet in the database").
- Public book pages emit Schema.org `Book` JSON-LD (rendered in web Phase 6).

**Testing**:
- `Integration: GET /books/search?q=dune → ≥1 result, response validates against Zod BookMetadata schema`.
- `Integration: GET /books/:id unknown → 404`.
- `Integration: POST /books/manual minimal body → 201, source='manual'`.

---

## Phase 3: Clubs, Membership, Selections & Voting

### Purpose
The organisational core that differentiates this product from pure trackers. After this phase users can create clubs, manage members and roles, nominate books, run voting polls, and advance a book through its selection lifecycle to `reading` — the state that everything in Phases 4–5 keys off.

### Tasks

#### 3.1 — Clubs and membership with role-based access control

**What**: CRUD for clubs, join/leave, role management (admin/moderator/member), visibility rules.

**Design**:
- Endpoints: `POST /clubs`, `GET /clubs/:id`, `PATCH /clubs/:id` (admin), `GET /clubs` (discovery filters in Phase 8), `POST /clubs/:id/join`, `DELETE /clubs/:id/members/:userId`, `PATCH /clubs/:id/members/:userId` (role change, admin only).
- `ClubRoleGuard` reads `club_members.role`; decorators `@RequireRole('admin'|'moderator')`. Enforces OWASP A01: private/invite_only club content is rejected (404, not 403, to avoid leaking existence) for non-members.
- `member_count` maintained transactionally on join/leave; `max_members` enforced (freemium: free tier capped at 10 members per README).
- Creator auto-inserted as `admin`.

**Testing**:
- `Integration: non-member GET /clubs/:id for a private club → 404`.
- `Integration: member (not admin) PATCH /clubs/:id → 403`.
- `Integration: join private club without invite → 403; join public club → member_count increments by 1`.
- `Integration: join when member_count == max_members (10, free tier) → 409`.
- `Unit: last admin cannot demote self → 409`.

#### 3.2 — Book selections lifecycle

**What**: Nominate books, schedule reading, and manage the `nominated → voting → selected → reading → completed → skipped` state machine.

**Design**:
- State transitions enforced server-side; only one selection per club may be `reading` (matches `idx_selections_current` partial index).
- `POST /clubs/:id/selections` (nominate, any member), `PATCH /clubs/:id/selections/:sid` (admin/mod: change status, set `start_date`, `target_end_date`, `reading_pace_chapters_per_week`).
- Advancing to `reading` computes a default `target_end_date` from `page_count` / `reading_pace_chapters_per_week`.

**Testing**:
- `Integration: nominate book → selection status='nominated'`.
- `Integration: set second selection to 'reading' while one already reading → 409`.
- `Unit: invalid transition (completed → voting) → 422`.

#### 3.3 — Voting / polls

**What**: In-app polls for selecting the next book.

**Design**:
- Admin opens voting (`status='voting'` on candidate selections). `POST /clubs/:id/selections/:sid/vote` upserts into `book_votes` (PK `(selection_id,user_id)`). `GET /clubs/:id/poll` tallies votes per candidate.
- Closing the poll (admin) sets the winner to `selected`, others to `skipped`.

**Testing**:
- `Integration: two votes by same user on same selection → second overwrites first (one row)`.
- `Integration: tally returns counts ordered desc; close poll → top selection 'selected', rest 'skipped'`.
- `Integration: vote on a non-voting selection → 409`.

---

## Phase 4: Reading Progress & Spoiler-Gating Foundation

### Purpose
This phase delivers the per-member reading progress tracking that incumbents like Bookclubs.com lack, and lays the `current_chapter` foundation that the spoiler-gating engine (Phase 5) depends on. After this phase each member has a live progress position per book and the club dashboard shows everyone's progress.

### Tasks

#### 4.1 — Reading status (`user_books`) and progress updates

**What**: Per-user reading status, format, and live progress (page/chapter/percent).

**Design**:
- `PUT /me/books/:bookId/status` body `{ status, format }` → upserts `user_books`.
- `PATCH /me/books/:bookId/progress` body `{ currentPage?, currentChapter?, currentPct? }` → updates position; derives the unspecified fields from `chapters_json`/`page_count` where possible (page → chapter via chapter page ranges).
- Status transition to `read` sets `finished_at`; `dnf` requires `dnf_page`/`dnf_reason`.
- Progress change emits a domain event `progress.updated {userId, bookId, currentChapter}` (consumed by WebSocket broadcast 5.4 and challenge progress 7.1).

**Testing**:
- `Integration: set currentPage within chapter 3's range → currentChapter derived as 3`.
- `Integration: mark status='read' → finished_at set to today`.
- `Unit: dnf without dnf_page → 422`.
- `Integration: progress update emits progress.updated event once`.

#### 4.2 — Reading sessions and pace data capture

**What**: Record reading sessions (start/end, pages, minutes) feeding pace analytics in Phase 9.

**Design**:
- `POST /me/books/:bookId/sessions` body `{ startedAt, endedAt, startPage, endPage, format, notes? }` → computes `duration_minutes`, `pages_read`, links `user_book_id`, and advances `user_books` progress to `endPage`.
- Indexed for the 7-day pace window query from the data model (`idx_sessions_user`).

**Testing**:
- `Integration: log session 30 pages / 60 min → pages_read=30, duration_minutes=60, user_books.current_page advanced`.
- `Unit: endPage < startPage → 422`.

#### 4.3 — Club progress dashboard query

**What**: The headline "current book with each member's reading progress" query from the data model.

**Design**:
- `GET /clubs/:id/progress` runs the club-dashboard JOIN (members × current `reading` selection × `user_books`), returning per-member `{ displayName, avatarUrl, currentPage, currentChapter, currentPct }` ordered by `current_pct desc nulls last`.
- Result cached in Redis (30s TTL) keyed by club+selection, invalidated on `progress.updated`.

**Testing**:
- `Integration (real Postgres): club with 3 members at chapters 1/5/10 → ordered by pct desc; member with no user_books row → nulls last`.
- `Integration: progress.updated for a member → cache invalidated, next read reflects new position`.

---

## Phase 5: Discussions & Spoiler Gating (Core Differentiator)

### Purpose
Chapter-level spoiler-gated discussions are absent across every surveyed competitor — this is the product's signature feature. This phase ships threaded discussion boards where thread/post visibility is gated by each member's `current_chapter`, plus real-time updates over WebSocket.

### Tasks

#### 5.1 — Discussion threads with spoiler-chapter gating

**What**: Create/list threads per club selection, hiding threads whose `spoiler_chapter` exceeds the requester's progress.

**Design**:
- `POST /clubs/:id/selections/:sid/threads` body `{ title, spoilerChapter?, isPinned? }`.
- `GET /clubs/:id/selections/:sid/threads` runs the spoiler-gated query from the data model: a thread is visible if `spoiler_chapter IS NULL OR spoiler_chapter <= requester.current_chapter`. Members with no progress row see only ungated threads.
- Moderators/admins may pass `?includeSpoilers=true` to see all (for moderation).

**Testing**:
- `Integration (real Postgres): member at chapter 4, threads gated at NULL/3/7 → sees NULL and 3, not 7`.
- `Integration: member with no user_books row → sees only NULL-gated threads`.
- `Integration: moderator with includeSpoilers → sees all threads`.

#### 5.2 — Posts, threading, and reply counts

**What**: Threaded replies (`parent_post_id`), with spoiler flags on individual posts.

**Design**:
- `POST /threads/:tid/posts` body `{ body, parentPostId?, containsSpoiler?, spoilerChapter? }`. `reply_count` on the thread incremented transactionally.
- `GET /threads/:tid/posts` paginated; posts with `spoiler_chapter > requester.current_chapter` are returned with `body` redacted to `null` and `spoilerHidden:true` (so the thread structure shows but content is gated, matching spoiler-aware UX).
- `is_removed` posts return a tombstone.

**Testing**:
- `Integration: reply increments thread.reply_count`.
- `Integration: post flagged spoilerChapter=8, requester at chapter 5 → body null, spoilerHidden true`.
- `Integration: removed post → tombstone, body null`.

#### 5.3 — Moderation controls

**What**: Flagging, muting, post removal, thread lock (community-health tooling, weak across competitors).

**Design**:
- `POST /posts/:id/flag` (any member) → `is_flagged=true`, audit entry.
- `DELETE /posts/:id` (moderator/admin) → `is_removed=true`.
- `PATCH /threads/:id` lock/pin (moderator/admin). `POST /clubs/:id/members/:userId/mute` prevents posting for a duration.

**Testing**:
- `Integration: member flags post → is_flagged true, audit row`.
- `Integration: member tries DELETE post → 403; moderator → 200, is_removed true`.
- `Integration: muted member POST → 403 until mute expires`.

#### 5.4 — Real-time discussion + progress over WebSocket

**What**: Live thread updates and progress broadcasts via a Socket.IO gateway, scaled with the Redis adapter.

**Design**:
- Gateway namespace `/clubs`; clients join room `selection:{sid}` after a membership + JWT handshake check.
- Server emits `post.created`, `post.removed`, `progress.updated` to the room. Spoiler gating re-applied per recipient on emit (a post gated above a recipient's chapter is sent redacted).
- Redis pub/sub adapter so multiple `api` instances broadcast consistently.

**Testing**:
- `Integration (real Redis): two clients in a room; one posts → other receives post.created`.
- `Integration: client at chapter 2 receives a chapter-9 post redacted`.
- `Integration: non-member handshake → connection rejected`.

---

## Phase 6: Web Frontend (MVP UI)

### Purpose
Surface the MVP backend through a usable, accessible web app: auth, club dashboard with progress bars, spoiler-aware discussion boards, book search, and reading-goal tracking. After this phase the MVP feature set from `features.md` is fully usable end-to-end.

### Tasks

#### 6.1 — App shell, auth flows, and API client

**What**: Next.js App Router shell, login/register/OIDC pages, session handling, typed API client from the shared Zod types.

**Design**:
- Server components for public/club pages; client components for interactive boards. Access token in memory + httpOnly refresh cookie. Middleware redirects unauthenticated users from protected routes.
- API client generated/derived from `packages/types` Zod schemas for end-to-end type safety.

**Testing**:
- `E2E (Playwright): register → land on dashboard; logout → redirected to login`.
- `E2E: protected route while logged out → redirect to /login`.

#### 6.2 — Club dashboard, book search, progress UI

**What**: Club page with current book, per-member progress bars, book search/add, and a member's progress-update control.

**Design**:
- Dashboard consumes `GET /clubs/:id/progress`; colour-coded progress bars (Bookclubs.com UX pattern). Subscribes to `progress.updated` over WebSocket for live bars.
- Book search modal hits `GET /books/search`; selecting a book nominates it.
- Public book pages render Schema.org `Book` JSON-LD.

**Testing**:
- `E2E: open club → see current book + member progress bars`.
- `E2E: update own progress → bar moves without reload (WS)`.
- `E2E: search + nominate a book → appears in selections`.

#### 6.3 — Discussion board UI with spoiler gating

**What**: Threaded board where gated threads/posts show spoiler-blur with a "reveal anyway" affordance respecting server gating.

**Design**:
- Threads list from the gated endpoint; spoiler-hidden posts render a blurred placeholder ("Spoilers past chapter N"). New posts arrive live via WebSocket.
- Moderation actions (flag, remove, lock) shown contextually by role.

**Testing**:
- `E2E: member at chapter 3 sees a chapter-8 thread hidden; advancing progress reveals it`.
- `E2E: post a reply → appears live for a second browser context in the same room`.

---

## Phase 7: Reading Challenges & Goals

### Purpose
Add the gamification/accountability layer (annual goals, custom challenges, group leaderboards) that competitors implement only individually. Reuses the `progress.updated` event stream from Phase 4.

### Tasks

#### 7.1 — Challenges (individual + group) with progress tracking

**What**: Create challenges scoped to a user or a club; auto-increment `current_value` from reading events.

**Design**:
- `POST /challenges` body `{ scope:'user'|'club', clubId?, name, challengeType, targetValue, startDate, endDate, isGroup }`.
- Listener on `progress.updated` / book `read` events increments `current_value` per challenge type (`annual_goal`/`page_count`/`genre_challenge`/`reading_streak`).
- Streak logic: consecutive days with ≥1 reading session.

**Testing**:
- `Integration: annual_goal target 12; finishing a book increments current_value`.
- `Unit: streak resets after a day with no session`.
- `Integration: group challenge sums all members' contributions`.

#### 7.2 — Leaderboards

**What**: Group leaderboard ranking members by contribution to a club challenge.

**Design**:
- `GET /challenges/:id/leaderboard` → members ranked by their contribution to the group challenge window.

**Testing**:
- `Integration: 3 members with different page counts → ranked desc`.

---

## Phase 8: Meetings, RSVP, Reminders & Club Discovery

### Purpose
Complete the organisational suite (meeting scheduling, RSVP, calendar sync, reminders) and the discovery directory, matching and exceeding Bookclubs.com's strongest area. Notification infrastructure here is reused by Phase 10.

### Tasks

#### 8.1 — Meetings and RSVP

**What**: Schedule meetings tied to a selection, collect RSVPs.

**Design**:
- `POST /clubs/:id/meetings` body `{ title, meetingType, scheduledAt, durationMinutes, location?, selectionId? }` (admin/mod).
- `POST /meetings/:id/rsvp` body `{ response: 'attending'|'maybe'|'declined' }` upserts `meeting_rsvps`.
- `GET /clubs/:id/meetings?upcoming=true` uses the `idx_meetings_upcoming` partial index.

**Testing**:
- `Integration: create meeting → status 'scheduled'; member RSVP attending → one rsvp row, upsert on change`.
- `Integration: member tries to create meeting → 403`.

#### 8.2 — Calendar sync (iCal) and reminders

**What**: iCalendar export/feed and scheduled reminder notifications.

**Design**:
- `GET /clubs/:id/meetings.ics` → RFC 5545 VEVENT feed (subscribable in Google/Apple Calendar).
- On meeting create, enqueue BullMQ delayed jobs at T-24h and T-1h that fan out notifications (Phase 10 transport).

**Testing**:
- `Integration: .ics feed validates as RFC 5545, one VEVENT per upcoming meeting`.
- `Integration: scheduling a meeting enqueues two delayed reminder jobs`.

#### 8.3 — Club discovery directory

**What**: Browse/filter public clubs by genre and reading pace.

**Design**:
- `GET /clubs?visibility=public&genre=&pace=&cursor=` uses `idx_clubs_genres` (GIN) and `idx_clubs_visibility`. Only `public` clubs listed; `invite_only`/`private` excluded.

**Testing**:
- `Integration: filter genre='sci-fi' → only matching public clubs; private clubs never appear`.

---

## Phase 9: AI Features I — Discussion Questions, Spoiler Classification, Finish-Date Prediction

### Purpose
Deliver the first wave of AI-native differentiators. AI moves from bolt-on to core loop: auto-generated discussion questions, automatic spoiler classification (no manual tagging), and personalised finish-date prediction. Requires the LLM provider abstraction and the reading-session data from Phase 4.

### Tasks

#### 9.1 — LLM provider abstraction with prompt caching

**What**: `packages/llm` provider interface, Claude implementation, structured-output helpers, and prompt caching of book context.

**Design**:
```ts
interface LlmProvider {
  generate<T>(opts: { system: string; messages: Msg[]; schema: ZodSchema<T>;
                      cacheKey?: string }): Promise<T>;
}
```
- Anthropic impl uses prompt caching on the (large, stable) book-context block keyed by book id. Structured output validated against the Zod schema; one retry on validation failure.

**Testing**:
- `Unit (mocked SDK): generate returns parsed object matching schema; malformed JSON → one retry then throws`.
- `Unit: identical book-context block reused across calls sets cache control`.

#### 9.2 — AI discussion-question generation

**What**: Generate per-book and per-chapter discussion questions on demand, persisted as `is_ai_generated` threads.

**Design**:
- Prompt template (in `packages/llm/src/prompts/discussion-questions.ts`): system establishes a thoughtful book-club facilitator; user provides title/author/description and, for chapter scope, the chapter summary and `spoiler_chapter` boundary. Output schema `{ questions: { text: string; chapter?: number }[] }`.
- `POST /clubs/:id/selections/:sid/ai/questions` body `{ scope:'book'|'chapter', chapter? }` → BullMQ job → creates `is_ai_generated=true` thread(s) with `spoiler_chapter` set to the requested chapter so generated questions respect gating.

**Testing**:
- `Integration (mocked LLM): generate chapter-3 questions → thread created with spoiler_chapter=3, is_ai_generated true`.
- `Unit: generated questions for chapter 3 never reference plot points tagged beyond chapter 3 (prompt asserts boundary; verify spoiler_chapter set)`.

#### 9.3 — Automatic spoiler classification

**What**: Classify whether a post contains spoilers and infer its `spoiler_chapter`, without manual tagging.

**Design**:
- On `POST /threads/:tid/posts`, if the author did not set `containsSpoiler`, enqueue a classification job: prompt returns `{ containsSpoiler: boolean; inferredChapter?: number; confidence: number }`.
- High-confidence spoilers update the post's `contains_spoiler`/`spoiler_chapter`, retroactively gating it (broadcast a `post.updated` redaction over WebSocket).

**Testing**:
- `Integration (mocked LLM): post "the villain dies in the finale" → classified spoiler, spoiler_chapter set to final chapter, gated`.
- `Integration: benign post → not gated`.
- `Unit: low-confidence result → left ungated, flagged for moderator review`.

#### 9.4 — Reading-pace analytics and finish-date prediction

**What**: Compute personal reading pace from `reading_sessions` and predict per-member finish dates.

**Design**:
- `GET /me/books/:bookId/prediction` runs the data model's 7-day pace query (pages/hour, pages remaining, days-to-finish), accounting for weekday/weekend variance, and writes `user_books.predicted_finish_date`.
- A nightly BullMQ job recomputes predictions for all `currently_reading` books.

**Testing**:
- `Integration (real Postgres, seeded sessions): member reading 20 pp/day with 100 pages left → ~5-day prediction`.
- `Unit: zero recent sessions → prediction null, not divide-by-zero`.

---

## Phase 10: Notifications & Push

### Purpose
Centralise the notification transport used by reminders (Phase 8), discussion replies, and milestones, balancing engagement without overwhelming users (a documented competitor pain point). Implements Web Push (VAPID) and FCM.

### Tasks

#### 10.1 — Notification service, preferences, and push transport

**What**: Unified notification dispatch with per-user preferences over Web Push (RFC 8030/8291/8292) and FCM.

**Design**:
- `notification_preferences` (new small table or JSONB on users): toggles per category (`meeting_reminder`, `discussion_reply`, `poll_open`, `milestone`).
- `POST /me/push-subscriptions` stores Web Push subscription / FCM token. Dispatcher consumes a `notifications` BullMQ queue, respects preferences and quiet hours, dedupes, and batches digestible nudges.

**Testing**:
- `Integration (mocked push): reply to a thread → author receives discussion_reply push iff preference enabled`.
- `Integration: user with meeting_reminder disabled → no push on reminder job`.
- `Unit: two milestone events within the dedupe window → one push`.

---

## Phase 11: AI Features II — Recommendations, Sentiment, Meeting Summaries & MCP

### Purpose
Ship the remaining AI differentiators and expose reading data to AI assistants via MCP, positioning the platform as AI-native infrastructure (per standards.md). Depends on accumulated reading history (Phase 4) and discussion/meeting data (Phases 5, 8).

### Tasks

#### 11.1 — Mood/pace/theme recommendations

**What**: Recommend books from group reading history and individual taste profiles, beyond genre matching.

**Design**:
- `GET /clubs/:id/recommendations` and `GET /me/recommendations`: build a taste vector from `user_books` ratings, genres, `preferred_pace`, and club history; LLM ranks candidate books (fetched via metadata cascade) by mood/pace/theme fit with cited rationale. Results cached per user/club (daily).

**Testing**:
- `Integration (mocked LLM): user who rated 3 fast-paced sci-fi highly → recommendations skew fast/sci-fi with rationale strings`.
- `Unit: empty history → falls back to popular-in-genre, no LLM error`.

#### 11.2 — Review sentiment analysis

**What**: Summarise what readers loved/struggled with across a book's reviews before a club votes.

**Design**:
- `GET /books/:id/sentiment` aggregates `user_books.review` text → LLM returns `{ strengths: string[]; painPoints: string[]; overallSentiment: number }`, cached.

**Testing**:
- `Integration (mocked LLM): mixed reviews → both strengths and painPoints populated`.

#### 11.3 — Meeting summary generation

**What**: Generate structured summaries from meeting recordings/transcripts.

**Design**:
- On `recording_url` set, enqueue transcription (pluggable STT) → LLM summary → store `meetings.ai_summary`. Surfaced on the meeting page.

**Testing**:
- `Integration (mocked STT + LLM): recording → ai_summary populated with agenda/decisions/next-book sections`.

#### 11.4 — MCP server

**What**: An MCP server exposing reading history, club discussions, and progress to LLM assistants (gated by `users.mcp_enabled` and OAuth scopes).

**Design**:
- Tools: `get_reading_history(userId)`, `get_club_discussions(clubId, sinceChapter)`, `get_reading_progress(userId)`. All enforce the same access-control + spoiler gating as the REST API.
- Disabled unless `users.mcp_enabled=true`; scoped tokens (OWASP A01).

**Testing**:
- `Integration: get_club_discussions respects spoiler gating for the calling identity`.
- `Integration: mcp_enabled=false → tools return authorization error`.

---

## Phase 12: Integrations & Backlog — Video, Audiobook, Federation

### Purpose
Layer the remaining "should/nice-to-have" integrations. Each is independent and optional; none block earlier phases.

### Tasks

#### 12.1 — In-app video meetings with recording

**What**: Embed a video room per meeting with recording, feeding 11.3 summaries.

**Design**:
- Integrate a WebRTC SFU provider (e.g. LiveKit) behind a `VideoProvider` interface; `meetings.video_url`/`recording_url` populated. RSVP attendees admitted via short-lived room tokens.

**Testing**:
- `Integration (mocked provider): create video meeting → video_url issued; recording webhook → recording_url stored, summary job enqueued`.

#### 12.2 — Audiobook progress sync (Audible/Libby)

**What**: Best-effort audiobook progress entry, acknowledging no public API exists (standards.md).

**Design**:
- Manual time-based progress entry (`format='audiobook'`, minutes → percent → derived page/chapter via `chapters_json`) plus a documented companion-extension ingestion endpoint `POST /me/books/:bookId/audio-progress { positionSeconds, totalSeconds }`.

**Testing**:
- `Integration: audio-progress 50% of total → current_pct≈50, derived chapter set`.

#### 12.3 — ActivityPub federation (backlog)

**What**: Federate reviews/reading-status/club announcements as ActivityStreams 2.0 objects over ActivityPub for BookWyrm interoperability.

**Design**:
- Implement S2S inbox/outbox and actor documents (JSON-LD); map `discussion_posts`/reviews to `Create`/`Note`. Clean-room implementation (no AGPL BookWyrm code copied — see features.md Legal summary).

**Testing**:
- `Integration: outbox serves valid ActivityStreams 2.0 JSON-LD; signed inbox delivery accepted, unsigned rejected`.

---

## Phase Summary & Dependencies

```
Phase 1: Foundation (monorepo, DB, auth)        ─── required by everything
    │
Phase 2: Books & Metadata Cascade               ─── requires P1
    │
Phase 3: Clubs, Membership, Selections, Voting  ─── requires P1, P2
    │
Phase 4: Reading Progress & Spoiler Foundation  ─── requires P2, P3
    │
Phase 5: Discussions & Spoiler Gating           ─── requires P4 (current_chapter)
    │
Phase 6: Web Frontend (MVP UI)                  ─── requires P3, P4, P5  ◀ MVP complete here
    │
    ├── Phase 7: Challenges & Goals              ─── requires P4 (event stream)   ┐
    ├── Phase 8: Meetings, RSVP, Discovery       ─── requires P3                   ├ parallelisable
    └── Phase 10: Notifications & Push           ─── requires P8 (reminder hooks) ┘
         │
Phase 9: AI Features I (questions, spoiler ML, prediction) ─── requires P4, P5
    │
Phase 11: AI Features II (recs, sentiment, summaries, MCP) ─── requires P4, P5, P8, P9
    │
Phase 12: Integrations (video, audiobook, federation)      ─── each independent; requires P8 (video↔meetings)
```

Parallelism: after Phase 6 (MVP), Phases 7, 8, and 9 can proceed concurrently by separate developers; Phase 10 follows Phase 8's reminder hooks; Phase 12's three tasks are mutually independent.

---

## Definition of Done (per phase)

A phase is complete only when all of the following hold:

1. All tasks in the phase are implemented.
2. All unit and integration tests for the phase pass (`pnpm -r test`), including the real-Postgres/Redis Testcontainers suites where specified.
3. `pnpm -r lint` and `pnpm -r typecheck` (`tsc --noEmit`) pass with zero errors.
4. `docker-compose build` succeeds for any new/changed service.
5. New/changed REST endpoints appear in the auto-generated OpenAPI 3.1 spec and validate against their Zod component schemas.
6. New database tables/columns/indexes have a checked-in drizzle-kit migration that applies cleanly from empty and is reversible or documented.
7. The phase's headline capability works end-to-end (E2E Playwright spec green for UI-facing phases; Supertest flow for API-only phases).
8. New config/env keys are added to `.env.example` and documented.
9. Access-control and spoiler-gating invariants are covered by an explicit negative test (e.g. non-member 404, gated-content redaction) — OWASP A01.
10. Any new personal-data fields are covered by the export/erasure paths (GDPR) before the phase merges.
