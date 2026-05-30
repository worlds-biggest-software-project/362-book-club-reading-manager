# Data Model Suggestion 2: Hybrid Relational + JSONB

> Project: Book Club & Reading Manager · Created: 2026-05-25

## Philosophy

Core operational entities — users, books, and clubs — are relational tables with indexed columns for cross-entity queries (spoiler gating, reading analytics, club discovery). Variable-structure data — reading sessions, club memberships, selections with votes, discussions, meetings with RSVPs, challenges, AI recommendations — lives in JSONB columns with GIN indexes.

Book clubs have two dominant access patterns: (1) "load my club's dashboard showing the current book, member progress, upcoming meeting, and recent discussions" and (2) "show my personal reading dashboard with progress, streaks, and goals." Embedding member progress and discussion data on the club row, and embedding reading sessions and goals on the user row, means both dashboards are single-row reads.

The trade-off is that spoiler gating — the platform's key differentiator — requires comparing a member's chapter position (from the user's JSONB) against a thread's spoiler chapter marker (from the club's JSONB). This is a JSONB extraction operation rather than an indexed column comparison. But for a social reading app where data is always accessed within a single club or user context, the schema simplicity pays for itself.

**Best for:** Teams building an MVP where rapid iteration on club features, minimal schema migrations, and fast dashboard loading are priorities.

**Trade-offs:**
- Pro: 5 tables — simple schema, fast to deploy
- Pro: Club dashboard is a single-row read with full member progress
- Pro: New discussion types, challenge formats, and meeting features require no migration
- Pro: User reading data (sessions, goals, streaks) embedded for personal dashboard
- Con: Spoiler gating requires JSONB extraction for chapter comparison
- Con: Active clubs with many members and discussions can produce oversized JSONB
- Con: No FK enforcement on member references within JSONB
- Con: Cross-club analytics require JSONB extraction

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISBN (ISO 2108) | `books.isbn_13` as canonical identifier |
| BISAC Subject Headings | `books.bisac_codes` for genre classification |
| Schema.org Book | Book columns map to Schema.org properties |
| OpenAPI 3.1 | REST API documented in OpenAPI |
| JSON Schema 2020-12 | Club and reading data validation |
| OAuth 2.0 / OIDC | User auth; Google Books API authorisation |
| JWT (RFC 7519) | Session tokens |
| WebSocket (RFC 6455) | Real-time discussion and progress updates |
| FCM / APNs | Push notifications |
| GDPR | Reading history as personal data |
| OWASP Top 10 | Web application security baseline |
| MCP | AI assistant integration |

---

## Users

```sql
CREATE TABLE users (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email               TEXT UNIQUE NOT NULL,
    display_name        TEXT NOT NULL,
    username            TEXT UNIQUE NOT NULL,
    avatar_url          TEXT,
    auth_provider       TEXT NOT NULL CHECK (auth_provider IN (
                            'email_password','google','apple'
                        )),
    timezone            TEXT NOT NULL DEFAULT 'America/New_York',
    reading_json        JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "currently_reading": [{
    --     "book_id": "uuid", "title": "Project Hail Mary",
    --     "format": "audiobook", "current_page": 180,
    --     "current_chapter": 12, "current_pct": 58.4,
    --     "started_at": "2026-05-10", "predicted_finish": "2026-06-02"
    --   }],
    --   "want_to_read": ["uuid", "uuid"],
    --   "read_count_2026": 14,
    --   "annual_goal": 24, "goal_type": "books",
    --   "reading_streak_days": 12, "longest_streak": 28,
    --   "total_pages_2026": 4250, "total_minutes_2026": 8400,
    --   "avg_pages_per_hour": 42,
    --   "recent_sessions": [{
    --     "book_id": "uuid", "date": "2026-05-25",
    --     "duration_minutes": 35, "pages_read": 28,
    --     "start_page": 152, "end_page": 180
    --   }]
    -- }
    taste_json          JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "preferred_genres": ["science_fiction","literary_fiction"],
    --   "preferred_pace": "moderate",
    --   "mood_preferences": ["thought_provoking","hopeful"],
    --   "favorite_authors": ["Andy Weir","Ursula K. Le Guin"],
    --   "disliked_genres": ["romance"],
    --   "ratings_distribution": {"5": 4, "4": 8, "3": 5, "2": 1, "1": 0}
    -- }
    clubs_json          JSONB NOT NULL DEFAULT '[]',
    -- [{"club_id": "uuid", "role": "admin", "joined_at": "2026-01-15"}]
    settings_json       JSONB NOT NULL DEFAULT '{}',
    mcp_json            JSONB,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_users_username ON users (username);
CREATE INDEX idx_users_reading ON users USING GIN (reading_json);
```

---

## Books (Reference Data)

```sql
CREATE TABLE books (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    title               TEXT NOT NULL,
    subtitle            TEXT,
    authors             TEXT[] NOT NULL,
    isbn_13             TEXT,
    isbn_10             TEXT,
    open_library_key    TEXT,
    google_books_id     TEXT,
    cover_url           TEXT,
    description         TEXT,
    publisher           TEXT,
    published_date      DATE,
    page_count          INTEGER,
    language            TEXT NOT NULL DEFAULT 'en',
    bisac_codes         TEXT[] NOT NULL DEFAULT '{}',
    genres              TEXT[] NOT NULL DEFAULT '{}',
    avg_rating          NUMERIC(3,2),
    ratings_count       INTEGER NOT NULL DEFAULT 0,
    chapters_json       JSONB,
    -- [{"number": 1, "title": "Chapter 1", "start_page": 1, "end_page": 25}]
    source              TEXT NOT NULL CHECK (source IN (
                            'open_library','google_books','isbndb',
                            'hardcover','manual'
                        )),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_books_isbn13 ON books (isbn_13) WHERE isbn_13 IS NOT NULL;
CREATE INDEX idx_books_title ON books USING GIN (to_tsvector('english', title));
CREATE INDEX idx_books_authors ON books USING GIN (authors);
CREATE INDEX idx_books_genres ON books USING GIN (genres);
```

---

## User Books (Reviews & Ratings)

```sql
CREATE TABLE user_books (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id             UUID NOT NULL REFERENCES users(id),
    book_id             UUID NOT NULL REFERENCES books(id),
    status              TEXT NOT NULL CHECK (status IN (
                            'want_to_read','currently_reading','read','dnf'
                        )),
    format              TEXT CHECK (format IN ('physical','ebook','audiobook')),
    rating              NUMERIC(2,1) CHECK (rating BETWEEN 0.5 AND 5.0),
    review              TEXT,
    started_at          DATE,
    finished_at         DATE,
    dnf_page            INTEGER,
    is_private          BOOLEAN NOT NULL DEFAULT FALSE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (user_id, book_id)
);
CREATE INDEX idx_user_books_user ON user_books (user_id, status);
CREATE INDEX idx_user_books_book ON user_books (book_id);
```

---

## Clubs

```sql
CREATE TABLE clubs (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name                TEXT NOT NULL,
    description         TEXT,
    cover_url           TEXT,
    visibility          TEXT NOT NULL CHECK (visibility IN (
                            'public','private','invite_only'
                        )) DEFAULT 'private',
    genre_focus         TEXT[] NOT NULL DEFAULT '{}',
    reading_pace        TEXT CHECK (reading_pace IN ('slow','moderate','fast')),
    created_by          UUID NOT NULL REFERENCES users(id),
    members_json        JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "user_id": "uuid", "display_name": "Alice",
    --   "avatar_url": "...", "role": "admin",
    --   "joined_at": "2026-01-15",
    --   "current_book_progress": {
    --     "book_id": "uuid", "current_page": 180,
    --     "current_chapter": 12, "current_pct": 58.4
    --   }
    -- }]
    selections_json     JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "id": "uuid", "book_id": "uuid", "book_title": "Project Hail Mary",
    --   "status": "reading", "nominated_by": "uuid",
    --   "start_date": "2026-05-01", "target_end_date": "2026-05-31",
    --   "pace_chapters_per_week": 5,
    --   "votes": [{"user_id": "uuid", "vote": 1}],
    --   "sort_order": 1
    -- }]
    discussions_json    JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "id": "uuid", "selection_id": "uuid",
    --   "title": "What did you think of the Hail Mary's AI?",
    --   "spoiler_chapter": 15, "is_pinned": false,
    --   "is_ai_generated": true, "reply_count": 8,
    --   "created_by": "uuid", "created_at": "2026-05-20T10:00:00Z",
    --   "posts": [{
    --     "id": "uuid", "author_id": "uuid", "author_name": "Alice",
    --     "body": "I loved the way Rocky...",
    --     "contains_spoiler": true, "spoiler_chapter": 15,
    --     "created_at": "2026-05-20T10:15:00Z"
    --   }]
    -- }]
    meetings_json       JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "id": "uuid", "title": "May Book Discussion",
    --   "selection_id": "uuid", "type": "video",
    --   "scheduled_at": "2026-05-28T19:00:00Z",
    --   "duration_minutes": 60, "video_url": "...",
    --   "status": "scheduled",
    --   "rsvps": [
    --     {"user_id": "uuid", "response": "attending"},
    --     {"user_id": "uuid", "response": "maybe"}
    --   ],
    --   "recording_url": null, "ai_summary": null
    -- }]
    challenges_json     JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "id": "uuid", "name": "Summer Reading Sprint",
    --   "type": "custom", "target_value": 5,
    --   "start_date": "2026-06-01", "end_date": "2026-08-31",
    --   "leaderboard": [
    --     {"user_id": "uuid", "display_name": "Alice", "current": 3},
    --     {"user_id": "uuid", "display_name": "Bob", "current": 2}
    --   ]
    -- }]
    member_count        INTEGER NOT NULL DEFAULT 0,
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_clubs_visibility ON clubs (visibility) WHERE is_active = TRUE;
CREATE INDEX idx_clubs_genres ON clubs USING GIN (genre_focus);
CREATE INDEX idx_clubs_members ON clubs USING GIN (members_json);
CREATE INDEX idx_clubs_discussions ON clubs USING GIN (discussions_json);
```

---

## Audit Log

```sql
CREATE TABLE audit_log (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id             UUID REFERENCES users(id),
    actor_type          TEXT NOT NULL CHECK (actor_type IN (
                            'user','system','ai','moderator'
                        )),
    action              TEXT NOT NULL,
    entity_type         TEXT NOT NULL,
    entity_id           UUID NOT NULL,
    changes_json        JSONB,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_audit_user ON audit_log (user_id, created_at);
CREATE INDEX idx_audit_entity ON audit_log (entity_type, entity_id);
```

---

## Example Queries

### Club dashboard — single row read

```sql
SELECT name, description, member_count,
       members_json, selections_json, discussions_json,
       meetings_json, challenges_json
FROM clubs
WHERE id = 'club-uuid';
```

### Spoiler-gated discussions from JSONB

```sql
SELECT d->>'id' AS thread_id,
       d->>'title' AS title,
       (d->>'spoiler_chapter')::INTEGER AS spoiler_chapter,
       (d->>'reply_count')::INTEGER AS replies,
       (d->>'is_ai_generated')::BOOLEAN AS ai_generated
FROM clubs c,
     jsonb_array_elements(c.discussions_json) AS d
WHERE c.id = 'club-uuid'
  AND (d->>'spoiler_chapter' IS NULL
       OR (d->>'spoiler_chapter')::INTEGER <= 12)
ORDER BY (d->>'created_at')::TIMESTAMPTZ DESC;
```

### Personal reading dashboard

```sql
SELECT display_name,
       reading_json->'currently_reading' AS currently_reading,
       (reading_json->>'read_count_2026')::INTEGER AS books_read,
       (reading_json->>'annual_goal')::INTEGER AS goal,
       (reading_json->>'reading_streak_days')::INTEGER AS streak,
       reading_json->'recent_sessions' AS recent_sessions
FROM users
WHERE id = 'user-uuid';
```

### Most-read books across all clubs

```sql
SELECT b.title, b.authors, b.avg_rating,
       COUNT(*) AS club_selections
FROM clubs c,
     jsonb_array_elements(c.selections_json) AS s
JOIN books b ON b.id = (s->>'book_id')::UUID
WHERE s->>'status' = 'completed'
GROUP BY b.id
ORDER BY club_selections DESC
LIMIT 20;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Users | 1 | users (embeds reading progress, sessions, goals, taste, club memberships) |
| Books | 1 | books (reference data with chapters) |
| User Books | 1 | user_books (ratings, reviews, status per book) |
| Clubs | 1 | clubs (embeds members, selections, discussions, meetings, challenges) |
| Audit | 1 | audit_log (partitioned) |
| **Total** | **5** | |

---

## Key Design Decisions

1. **`members_json` on clubs** — a club typically has 5-30 members; embedding all members with their current book progress means the club dashboard showing member progress bars is a single-row read.

2. **`discussions_json` with embedded posts on clubs** — discussion threads with their posts are embedded on the club row. The `spoiler_chapter` on each thread enables JSONB-level spoiler filtering. For clubs with very active discussions, posts beyond the most recent 50 per thread could be paginated via a separate query.

3. **`reading_json` on users** — currently-reading books, reading sessions, streaks, and annual progress are embedded on the user row for instant personal dashboard loading. The `current_chapter` field on each currently-reading entry is the spoiler-gating reference.

4. **`user_books` as a separate table** — ratings and reviews are public social data that needs cross-user aggregation (average rating, review feed). A separate relational table enables efficient aggregation with indexed queries.

5. **`selections_json` with embedded votes** — book nominations, voting, and scheduling are club-scoped and typically number 5-20 per year. Embedding keeps the book selection lifecycle within the club row.

6. **`meetings_json` with embedded RSVPs** — meetings are club-scoped events with RSVPs, recording URLs, and AI summaries. A club typically has 1-2 meetings per month; embedding keeps the meeting lifecycle self-contained.

7. **`challenges_json` with leaderboard** — reading challenges are club-scoped with member progress tracked on a leaderboard. Embedding keeps the challenge view within the club dashboard.

8. **`taste_json` on users** — taste profile data (preferred genres, mood preferences, favorite authors) is embedded for AI recommendation queries without a separate table.

9. **`chapters_json` on books** — chapter metadata enables chapter-level progress tracking and spoiler gating without a separate chapters table.

10. **5 tables** — book clubs have a strongly club-centric and user-centric data model; embedding club activity (discussions, meetings, challenges) and user activity (reading progress, sessions) into their parent rows minimises joins for the dominant dashboard views.
