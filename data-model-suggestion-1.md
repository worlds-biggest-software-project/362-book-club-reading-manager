# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Book Club & Reading Manager · Created: 2026-05-25

## Philosophy

Every book club concept — clubs, memberships, books, reading progress, discussions, meetings, challenges, recommendations — gets its own dedicated table with typed columns, foreign keys, and purpose-built indexes. This approach mirrors how social reading platforms (Goodreads, Hardcover) structure their data: books are shared reference entities, users have reading statuses per book, clubs schedule books and host discussions, and spoiler gating depends on per-member progress relative to discussion chapter markers.

The dominant query pattern is "load a club's current book with each member's reading progress, recent discussion threads, and upcoming meeting" — which means a club-centric query joining the current selection, member progress entries, discussion threads, and the next meeting. Normalisation means each entity can be independently indexed and queried: cross-club recommendations, global reading analytics, spoiler-gated content filtering by chapter position.

The trade-off is schema complexity: 16 tables for the full feature set. But for a domain with rich social interactions (discussions, votes, meetings, challenges) where referential integrity between members, books, and progress is critical for spoiler gating, the normalised approach ensures data consistency.

**Best for:** Teams building a production-grade book club platform where spoiler gating accuracy, cross-club analytics, ActivityPub federation, and publisher/API integrations are priorities.

**Trade-offs:**
- Pro: Full referential integrity across clubs, members, books, and progress
- Pro: Spoiler gating is a precise query against member progress vs. thread chapter
- Pro: Cross-club analytics (most-read books, reading speed trends) use indexed queries
- Pro: Maps cleanly to Schema.org Book vocabulary and Hardcover GraphQL model
- Pro: ActivityPub federation operates on well-typed activity records
- Con: 16 tables — more complex schema
- Con: Club dashboard requires JOINs across multiple tables
- Con: Adding new social features requires migration
- Con: Book metadata sync requires careful deduplication

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISBN (ISO 2108) | `books.isbn_13` as canonical identifier; `isbn_10` for legacy |
| ONIX 3.0 | Book metadata import/export format compatibility |
| BISAC Subject Headings | `books.bisac_codes` for standardised genre classification |
| Schema.org Book | Book columns map to Schema.org Book properties |
| ActivityPub (W3C) | Discussion and reading activity federation |
| ActivityStreams 2.0 | Activity types for reviews, reading updates, club announcements |
| OpenAPI 3.1 | REST API documented in OpenAPI |
| GraphQL | Developer-facing API (Hardcover pattern) |
| JSON Schema 2020-12 | Request/response validation |
| OAuth 2.0 / OIDC | User auth; Google Books API authorisation |
| JWT (RFC 7519) | Session tokens |
| WebSocket (RFC 6455) | Real-time discussion updates and progress broadcasts |
| RFC 8030 / VAPID | Web push notifications for reminders |
| FCM / APNs | Mobile push notifications |
| GDPR | Reading history as personal data; right to erasure |
| OWASP Top 10 | Web application security baseline |
| MCP | AI assistant integration for reading queries |

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
    locale              TEXT NOT NULL DEFAULT 'en-US',
    reading_goal_annual INTEGER,
    reading_goal_type   TEXT CHECK (reading_goal_type IN ('books','pages')) DEFAULT 'books',
    taste_profile       TEXT[] NOT NULL DEFAULT '{}',
    preferred_genres    TEXT[] NOT NULL DEFAULT '{}',
    preferred_pace      TEXT CHECK (preferred_pace IN (
                            'slow','moderate','fast'
                        )),
    mcp_enabled         BOOLEAN NOT NULL DEFAULT FALSE,
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_users_username ON users (username);
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
    chapter_count       INTEGER,
    chapters_json       JSONB,
    -- [{"number": 1, "title": "Chapter 1", "start_page": 1, "end_page": 25}, ...]
    source              TEXT NOT NULL CHECK (source IN (
                            'open_library','google_books','isbndb',
                            'hardcover','manual'
                        )),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_books_isbn13 ON books (isbn_13) WHERE isbn_13 IS NOT NULL;
CREATE INDEX idx_books_isbn10 ON books (isbn_10) WHERE isbn_10 IS NOT NULL;
CREATE INDEX idx_books_title ON books USING GIN (to_tsvector('english', title));
CREATE INDEX idx_books_authors ON books USING GIN (authors);
CREATE INDEX idx_books_genres ON books USING GIN (genres);
CREATE INDEX idx_books_olkey ON books (open_library_key) WHERE open_library_key IS NOT NULL;
```

---

## User Books (Reading Status)

```sql
CREATE TABLE user_books (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id             UUID NOT NULL REFERENCES users(id),
    book_id             UUID NOT NULL REFERENCES books(id),
    status              TEXT NOT NULL CHECK (status IN (
                            'want_to_read','currently_reading','read','dnf'
                        )),
    format              TEXT CHECK (format IN (
                            'physical','ebook','audiobook'
                        )),
    current_page        INTEGER,
    current_chapter     INTEGER,
    current_pct         NUMERIC(5,2),
    total_reading_minutes INTEGER NOT NULL DEFAULT 0,
    started_at          DATE,
    finished_at         DATE,
    rating              NUMERIC(2,1) CHECK (rating BETWEEN 0.5 AND 5.0),
    review              TEXT,
    dnf_page            INTEGER,
    dnf_reason          TEXT,
    is_private          BOOLEAN NOT NULL DEFAULT FALSE,
    predicted_finish_date DATE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (user_id, book_id)
);
CREATE INDEX idx_user_books_user ON user_books (user_id);
CREATE INDEX idx_user_books_book ON user_books (book_id);
CREATE INDEX idx_user_books_status ON user_books (user_id, status);
```

---

## Reading Sessions

```sql
CREATE TABLE reading_sessions (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id             UUID NOT NULL REFERENCES users(id),
    book_id             UUID NOT NULL REFERENCES books(id),
    user_book_id        UUID NOT NULL REFERENCES user_books(id),
    started_at          TIMESTAMPTZ NOT NULL,
    ended_at            TIMESTAMPTZ,
    duration_minutes    INTEGER,
    pages_read          INTEGER,
    start_page          INTEGER,
    end_page            INTEGER,
    start_chapter       INTEGER,
    end_chapter         INTEGER,
    format              TEXT CHECK (format IN (
                            'physical','ebook','audiobook'
                        )),
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_sessions_user ON reading_sessions (user_id, started_at DESC);
CREATE INDEX idx_sessions_book ON reading_sessions (user_book_id);
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
    reading_pace        TEXT CHECK (reading_pace IN (
                            'slow','moderate','fast'
                        )),
    max_members         INTEGER,
    member_count        INTEGER NOT NULL DEFAULT 0,
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    created_by          UUID NOT NULL REFERENCES users(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_clubs_visibility ON clubs (visibility) WHERE is_active = TRUE;
CREATE INDEX idx_clubs_genres ON clubs USING GIN (genre_focus);
```

---

## Club Members

```sql
CREATE TABLE club_members (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    club_id             UUID NOT NULL REFERENCES clubs(id) ON DELETE CASCADE,
    user_id             UUID NOT NULL REFERENCES users(id),
    role                TEXT NOT NULL CHECK (role IN (
                            'admin','moderator','member'
                        )) DEFAULT 'member',
    joined_at           TIMESTAMPTZ NOT NULL DEFAULT now(),
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    UNIQUE (club_id, user_id)
);
CREATE INDEX idx_club_members_club ON club_members (club_id);
CREATE INDEX idx_club_members_user ON club_members (user_id);
```

---

## Club Selections (Book Schedule)

```sql
CREATE TABLE club_selections (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    club_id             UUID NOT NULL REFERENCES clubs(id),
    book_id             UUID NOT NULL REFERENCES books(id),
    status              TEXT NOT NULL CHECK (status IN (
                            'nominated','voting','selected','reading',
                            'completed','skipped'
                        )) DEFAULT 'nominated',
    nominated_by        UUID REFERENCES users(id),
    start_date          DATE,
    target_end_date     DATE,
    reading_pace_chapters_per_week INTEGER,
    sort_order          INTEGER NOT NULL DEFAULT 0,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_selections_club ON club_selections (club_id, sort_order);
CREATE INDEX idx_selections_book ON club_selections (book_id);
CREATE INDEX idx_selections_current ON club_selections (club_id)
    WHERE status = 'reading';
```

---

## Book Votes

```sql
CREATE TABLE book_votes (
    club_id             UUID NOT NULL REFERENCES clubs(id),
    selection_id        UUID NOT NULL REFERENCES club_selections(id) ON DELETE CASCADE,
    user_id             UUID NOT NULL REFERENCES users(id),
    vote                INTEGER NOT NULL DEFAULT 1,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (selection_id, user_id)
);
CREATE INDEX idx_votes_club ON book_votes (club_id);
```

---

## Discussion Threads

```sql
CREATE TABLE discussion_threads (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    club_id             UUID NOT NULL REFERENCES clubs(id),
    selection_id        UUID NOT NULL REFERENCES club_selections(id),
    book_id             UUID NOT NULL REFERENCES books(id),
    title               TEXT NOT NULL,
    spoiler_chapter     INTEGER,
    -- threads with a spoiler_chapter are hidden from members who haven't reached it
    is_pinned           BOOLEAN NOT NULL DEFAULT FALSE,
    is_locked           BOOLEAN NOT NULL DEFAULT FALSE,
    is_ai_generated     BOOLEAN NOT NULL DEFAULT FALSE,
    reply_count         INTEGER NOT NULL DEFAULT 0,
    created_by          UUID NOT NULL REFERENCES users(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_threads_club ON discussion_threads (club_id, selection_id);
CREATE INDEX idx_threads_chapter ON discussion_threads (spoiler_chapter)
    WHERE spoiler_chapter IS NOT NULL;
```

---

## Discussion Posts

```sql
CREATE TABLE discussion_posts (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    thread_id           UUID NOT NULL REFERENCES discussion_threads(id) ON DELETE CASCADE,
    parent_post_id      UUID REFERENCES discussion_posts(id),
    author_id           UUID NOT NULL REFERENCES users(id),
    body                TEXT NOT NULL,
    contains_spoiler    BOOLEAN NOT NULL DEFAULT FALSE,
    spoiler_chapter     INTEGER,
    is_flagged          BOOLEAN NOT NULL DEFAULT FALSE,
    is_removed          BOOLEAN NOT NULL DEFAULT FALSE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_posts_thread ON discussion_posts (thread_id, created_at);
CREATE INDEX idx_posts_parent ON discussion_posts (parent_post_id)
    WHERE parent_post_id IS NOT NULL;
```

---

## Meetings

```sql
CREATE TABLE meetings (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    club_id             UUID NOT NULL REFERENCES clubs(id),
    selection_id        UUID REFERENCES club_selections(id),
    title               TEXT NOT NULL,
    description         TEXT,
    meeting_type        TEXT NOT NULL CHECK (meeting_type IN (
                            'in_person','video','hybrid'
                        )) DEFAULT 'video',
    scheduled_at        TIMESTAMPTZ NOT NULL,
    duration_minutes    INTEGER NOT NULL DEFAULT 60,
    location            TEXT,
    video_url           TEXT,
    recording_url       TEXT,
    ai_summary          TEXT,
    status              TEXT NOT NULL CHECK (status IN (
                            'scheduled','in_progress','completed','cancelled'
                        )) DEFAULT 'scheduled',
    created_by          UUID NOT NULL REFERENCES users(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_meetings_club ON meetings (club_id, scheduled_at);
CREATE INDEX idx_meetings_upcoming ON meetings (scheduled_at)
    WHERE status = 'scheduled';

CREATE TABLE meeting_rsvps (
    meeting_id          UUID NOT NULL REFERENCES meetings(id) ON DELETE CASCADE,
    user_id             UUID NOT NULL REFERENCES users(id),
    response            TEXT NOT NULL CHECK (response IN (
                            'attending','maybe','declined'
                        )),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (meeting_id, user_id)
);
```

---

## Reading Challenges

```sql
CREATE TABLE reading_challenges (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    club_id             UUID REFERENCES clubs(id),
    user_id             UUID REFERENCES users(id),
    name                TEXT NOT NULL,
    challenge_type      TEXT NOT NULL CHECK (challenge_type IN (
                            'annual_goal','custom','genre_challenge',
                            'page_count','reading_streak'
                        )),
    target_value        INTEGER NOT NULL,
    current_value       INTEGER NOT NULL DEFAULT 0,
    start_date          DATE NOT NULL,
    end_date            DATE NOT NULL,
    is_group            BOOLEAN NOT NULL DEFAULT FALSE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_challenges_club ON reading_challenges (club_id)
    WHERE club_id IS NOT NULL;
CREATE INDEX idx_challenges_user ON reading_challenges (user_id)
    WHERE user_id IS NOT NULL;
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

### Club dashboard — current book with member progress

```sql
SELECT u.display_name, u.avatar_url,
       ub.current_page, ub.current_chapter, ub.current_pct,
       b.page_count, b.chapter_count
FROM club_members cm
JOIN users u ON u.id = cm.user_id
LEFT JOIN club_selections cs ON cs.club_id = cm.club_id AND cs.status = 'reading'
LEFT JOIN books b ON b.id = cs.book_id
LEFT JOIN user_books ub ON ub.user_id = cm.user_id AND ub.book_id = cs.book_id
WHERE cm.club_id = 'club-uuid' AND cm.is_active = TRUE
ORDER BY ub.current_pct DESC NULLS LAST;
```

### Spoiler-gated threads for a member

```sql
SELECT dt.id, dt.title, dt.spoiler_chapter, dt.reply_count,
       dt.is_ai_generated, dt.created_at
FROM discussion_threads dt
WHERE dt.club_id = 'club-uuid'
  AND dt.selection_id = 'selection-uuid'
  AND (dt.spoiler_chapter IS NULL
       OR dt.spoiler_chapter <= (
           SELECT ub.current_chapter FROM user_books ub
           WHERE ub.user_id = 'user-uuid' AND ub.book_id = dt.book_id
       ))
ORDER BY dt.is_pinned DESC, dt.created_at DESC;
```

### Reading pace and finish-date prediction

```sql
SELECT rs.user_id, u.display_name,
       SUM(rs.pages_read) AS total_pages_7d,
       SUM(rs.duration_minutes) AS total_minutes_7d,
       ROUND(SUM(rs.pages_read)::NUMERIC / NULLIF(SUM(rs.duration_minutes), 0) * 60, 1) AS pages_per_hour,
       b.page_count - ub.current_page AS pages_remaining,
       CEIL((b.page_count - ub.current_page)::NUMERIC /
            NULLIF(SUM(rs.pages_read) / 7.0, 0)) AS days_to_finish
FROM reading_sessions rs
JOIN users u ON u.id = rs.user_id
JOIN user_books ub ON ub.id = rs.user_book_id
JOIN books b ON b.id = rs.book_id
WHERE rs.user_book_id = 'user-book-uuid'
  AND rs.started_at >= now() - INTERVAL '7 days'
GROUP BY rs.user_id, u.display_name, b.page_count, ub.current_page;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Users | 1 | users |
| Books | 1 | books (reference data) |
| Reading | 2 | user_books, reading_sessions |
| Clubs | 3 | clubs, club_members, club_selections |
| Voting | 1 | book_votes |
| Discussions | 2 | discussion_threads, discussion_posts |
| Meetings | 2 | meetings, meeting_rsvps |
| Challenges | 1 | reading_challenges |
| Audit | 1 | audit_log (partitioned) |
| **Total** | **16** | |

---

## Key Design Decisions

1. **`books` as shared reference data** — book metadata is normalised into a shared table sourced from Open Library, Google Books, and ISBNdb. Multiple users and clubs reference the same book record, avoiding duplicate metadata.

2. **`user_books` as the individual reading status** — each user's relationship with a book (status, progress, rating, review) is a separate record. The `current_chapter` field is critical for spoiler gating: it determines which discussion threads a member can see.

3. **`reading_sessions` for pace analytics** — individual reading sessions with duration and pages read enable reading speed calculation, finish-date prediction, and engagement analytics.

4. **`spoiler_chapter` on discussion threads and posts** — the core spoiler-gating mechanism: threads and posts carry a chapter marker, and members only see content up to their current reading position. This is a precise integer comparison against `user_books.current_chapter`.

5. **`club_selections` with status lifecycle** — books move through nominated → voting → selected → reading → completed states. The club's current book is the selection with `status = 'reading'`.

6. **`book_votes` as a junction table** — voting on book nominations is a many-to-many relationship between selections and members, enabling poll tallying and vote history.

7. **`meetings` with `meeting_rsvps`** — meeting scheduling with RSVP tracking, calendar sync, video URL, and AI-generated summaries support the full meeting lifecycle.

8. **`reading_challenges` for both individual and group challenges** — challenges can be user-scoped (personal annual goal) or club-scoped (group reading challenge), with a `target_value` and `current_value` for progress tracking.

9. **`chapters_json` on books** — chapter metadata (title, start/end pages) is stored as JSONB on the book record, enabling chapter-level progress tracking and spoiler gating without a separate chapters table.

10. **16 tables** — the normalised schema separates every social reading concern into its own table, enabling independent queries for club dashboards, reading analytics, spoiler-gated discussions, meeting management, and challenge leaderboards.
