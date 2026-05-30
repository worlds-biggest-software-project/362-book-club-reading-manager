# Data Model Suggestion 3: Event-Sourced / Audit-First

> Project: Book Club & Reading Manager · Created: 2026-05-25

## Philosophy

Every reading and club action — updating reading progress, posting a discussion reply, voting on a book nomination, RSVPing to a meeting, completing a challenge milestone — is captured as an immutable event in a single append-only event store. The current state of a club's dashboard, a member's reading progress, and the spoiler-gating configuration is derived by replaying or projecting events into purpose-built read models (CQRS pattern). The event store is the source of truth; read models are disposable and rebuildable.

Social reading platforms benefit from event sourcing because the temporal dimension is integral to the user experience: "when did each member finish each chapter?" drives spoiler gating, "how has my reading pace changed over time?" drives analytics, and "what happened in the club this week?" drives engagement summaries. An event-sourced architecture makes these temporal queries natural — every state transition is preserved with its timestamp, actor, and context.

The trade-off is query complexity: the club dashboard can't be answered by a direct SELECT — it requires a materialised read model. But for a book club platform where reading pace analytics, spoiler-gating accuracy, AI-generated meeting summaries from discussion history, and ActivityPub federation (which is event-based) are core requirements, event sourcing provides capabilities that relational snapshots cannot replicate.

**Best for:** Teams building a book club platform where reading analytics, temporal spoiler gating, AI-powered discussion summaries, ActivityPub federation, and the ability to retroactively recompute recommendations from historical reading data are priorities.

**Trade-offs:**
- Pro: Complete reading history — every progress update and session preserved
- Pro: Temporal spoiler gating: "what had each member read at the time this thread was posted?"
- Pro: AI meeting summaries can replay the full discussion event stream
- Pro: ActivityPub federation maps naturally to event streams
- Pro: Reading analytics can be retroactively recomputed with new algorithms
- Con: Club dashboard requires materialised read models
- Con: Event replay for active clubs can be slow without snapshots
- Con: Higher storage costs — events are never deleted
- Con: More complex application code

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| CloudEvents 1.0 | Event envelope format (ce_source, ce_type, ce_specversion, ce_time) |
| ISBN (ISO 2108) | Book events carry ISBN for identification |
| BISAC Subject Headings | Genre classification in book events |
| Schema.org Book | Book event data follows Schema.org properties |
| ActivityPub / ActivityStreams 2.0 | Events map to ActivityStreams activities for federation |
| OpenAPI 3.1 | REST API for commands and read model queries |
| JSON Schema 2020-12 | Event data validation |
| OAuth 2.0 / OIDC | User auth |
| JWT (RFC 7519) | Session tokens |
| WebSocket (RFC 6455) | Real-time event broadcasting |
| GDPR | Crypto-shredding for user data erasure |
| OWASP Top 10 | Web application security baseline |
| MCP | AI assistant integration via event-derived read models |

---

## Event Store (Infrastructure)

```sql
CREATE TABLE event_store (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_type         TEXT NOT NULL CHECK (stream_type IN (
                            'user','book','reading','club',
                            'discussion','meeting','challenge',
                            'recommendation','config'
                        )),
    stream_id           UUID NOT NULL,
    sequence_num        BIGINT NOT NULL,
    event_type          TEXT NOT NULL,
    event_data          JSONB NOT NULL,
    metadata            JSONB NOT NULL DEFAULT '{}',
    ce_source           TEXT NOT NULL DEFAULT '/book-club-reading-manager',
    ce_specversion      TEXT NOT NULL DEFAULT '1.0',
    ce_type             TEXT NOT NULL,
    ce_time             TIMESTAMPTZ NOT NULL DEFAULT now(),
    actor_id            UUID,
    actor_type          TEXT NOT NULL CHECK (actor_type IN (
                            'user','system','ai','moderator'
                        )),
    encryption_key_ref  TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_id, sequence_num)
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_events_stream ON event_store (stream_id, sequence_num);
CREATE INDEX idx_events_type ON event_store (event_type, created_at);
CREATE INDEX idx_events_actor ON event_store (actor_id, created_at);
CREATE INDEX idx_events_ce_type ON event_store (ce_type, ce_time);
```

---

## Stream Snapshots (Infrastructure)

```sql
CREATE TABLE stream_snapshots (
    stream_id           UUID NOT NULL,
    sequence_num        BIGINT NOT NULL,
    snapshot_data       JSONB NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_id, sequence_num)
);
```

---

## Projection Checkpoints (Infrastructure)

```sql
CREATE TABLE projection_checkpoints (
    projection_name     TEXT PRIMARY KEY,
    last_event_id       UUID NOT NULL,
    last_sequence_num   BIGINT NOT NULL,
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Event Types by Stream

### User Stream
- `user_registered` — profile, auth method, timezone
- `user_profile_updated` — display name, avatar, preferences
- `taste_profile_updated` — genres, moods, pace preferences
- `annual_goal_set` — target books/pages for the year
- `user_gdpr_erasure_requested`
- `user_deactivated`

### Reading Stream
- `book_added_to_shelf` — book_id, status (want_to_read/currently_reading)
- `reading_started` — book_id, format, started_at
- `progress_updated` — current_page, current_chapter, current_pct
- `reading_session_logged` — duration_minutes, pages_read, start/end page
- `book_finished` — finished_at, total_reading_minutes
- `book_rated` — rating (0.5-5.0)
- `book_reviewed` — review text
- `book_dnf` — dnf_page, reason
- `reading_streak_continued` — streak day count
- `reading_streak_broken`
- `finish_date_predicted` — AI-computed prediction

### Club Stream
- `club_created` — name, description, visibility, genre_focus
- `club_updated` — changed fields
- `member_joined` — user_id, role
- `member_role_changed` — new role
- `member_left` — reason
- `member_removed` — by admin, reason
- `book_nominated` — book_id, nominated_by
- `vote_cast` — selection_id, vote
- `book_selected` — selection_id, start_date, target_end_date, pace
- `book_completed_by_club` — selection_id, completion stats

### Discussion Stream
- `thread_created` — title, spoiler_chapter, is_ai_generated
- `post_created` — thread_id, body, contains_spoiler, spoiler_chapter
- `post_edited` — changed body
- `post_flagged` — flagged_by, reason
- `post_removed` — by moderator, reason
- `thread_pinned` / `thread_unpinned`
- `thread_locked` / `thread_unlocked`
- `ai_discussion_questions_generated` — book_id, chapter, questions

### Meeting Stream
- `meeting_scheduled` — title, type, scheduled_at, duration, location/video_url
- `meeting_rsvp_submitted` — user_id, response
- `meeting_started`
- `meeting_ended`
- `meeting_recording_uploaded` — recording_url
- `meeting_summary_generated` — AI summary from recording/discussion
- `meeting_cancelled` — reason

### Challenge Stream
- `challenge_created` — name, type, target_value, dates
- `challenge_progress_updated` — user_id, current_value
- `challenge_milestone_reached` — user_id, milestone
- `challenge_completed` — user_id or club_id
- `leaderboard_updated`

### Recommendation Stream
- `recommendation_generated` — user_id or club_id, books, reasoning
- `recommendation_accepted` — book added to shelf or nominated
- `recommendation_dismissed`
- `sentiment_analysis_completed` — book_id, sentiment scores

---

## Read Model: Club Dashboard

```sql
CREATE TABLE rm_club_dashboard (
    club_id             UUID NOT NULL PRIMARY KEY,
    name                TEXT NOT NULL,
    description         TEXT,
    visibility          TEXT NOT NULL,
    member_count        INTEGER NOT NULL DEFAULT 0,
    members_json        JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "user_id": "uuid", "display_name": "Alice", "role": "admin",
    --   "current_progress": {"page": 180, "chapter": 12, "pct": 58.4}
    -- }]
    current_selection_json JSONB,
    -- {
    --   "book_id": "uuid", "title": "Project Hail Mary",
    --   "cover_url": "...", "status": "reading",
    --   "start_date": "2026-05-01", "target_end_date": "2026-05-31",
    --   "avg_progress_pct": 62.5
    -- }
    upcoming_meeting_json JSONB,
    recent_discussions_json JSONB NOT NULL DEFAULT '[]',
    active_challenges_json JSONB NOT NULL DEFAULT '[]',
    selection_history_json JSONB NOT NULL DEFAULT '[]',
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Read Model: Reading Analytics

```sql
CREATE TABLE rm_reading_analytics (
    user_id             UUID NOT NULL,
    period_type         TEXT NOT NULL CHECK (period_type IN ('weekly','monthly','yearly')),
    period_start        DATE NOT NULL,
    books_json          JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "started": 3, "finished": 2, "dnf": 0,
    --   "total_pages": 650, "total_minutes": 1200,
    --   "avg_pages_per_hour": 42,
    --   "genres": {"science_fiction": 2, "literary_fiction": 1},
    --   "formats": {"physical": 1, "audiobook": 1, "ebook": 1},
    --   "ratings": [4.5, 3.0]
    -- }
    pace_json           JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "avg_session_minutes": 32,
    --   "sessions_count": 18,
    --   "most_active_day": "sunday",
    --   "streak_current": 12, "streak_longest": 28
    -- }
    goal_json           JSONB NOT NULL DEFAULT '{}',
    -- {"target": 24, "current": 14, "pct": 58.3, "on_track": true}
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, period_type, period_start)
);
```

---

## Read Model: Discussion Feed

```sql
CREATE TABLE rm_discussion_feed (
    club_id             UUID NOT NULL,
    thread_id           UUID NOT NULL,
    selection_id        UUID NOT NULL,
    book_id             UUID NOT NULL,
    book_title          TEXT NOT NULL,
    title               TEXT NOT NULL,
    spoiler_chapter     INTEGER,
    is_pinned           BOOLEAN NOT NULL DEFAULT FALSE,
    is_locked           BOOLEAN NOT NULL DEFAULT FALSE,
    is_ai_generated     BOOLEAN NOT NULL DEFAULT FALSE,
    reply_count         INTEGER NOT NULL DEFAULT 0,
    last_post_at        TIMESTAMPTZ,
    created_by          UUID NOT NULL,
    created_by_name     TEXT NOT NULL,
    posts_json          JSONB NOT NULL DEFAULT '[]',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (club_id, thread_id)
);
CREATE INDEX idx_rm_discussions_chapter ON rm_discussion_feed (spoiler_chapter)
    WHERE spoiler_chapter IS NOT NULL;
CREATE INDEX idx_rm_discussions_selection ON rm_discussion_feed (selection_id);
```

---

## Read Model: Recommendation Feed

```sql
CREATE TABLE rm_recommendation_feed (
    user_id             UUID NOT NULL,
    recommendation_id   UUID NOT NULL,
    source              TEXT NOT NULL CHECK (source IN (
                            'personal_taste','club_history','trending',
                            'similar_readers','ai_curated'
                        )),
    books_json          JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "book_id": "uuid", "title": "...", "authors": [...],
    --   "cover_url": "...", "match_score": 0.87,
    --   "reasoning": "Similar themes to books you rated 4+ stars"
    -- }]
    is_dismissed        BOOLEAN NOT NULL DEFAULT FALSE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, recommendation_id)
);
CREATE INDEX idx_rm_recs_active ON rm_recommendation_feed (user_id)
    WHERE is_dismissed = FALSE;
```

---

## Read Model: Book Stats

```sql
CREATE TABLE rm_book_stats (
    book_id             UUID NOT NULL PRIMARY KEY,
    title               TEXT NOT NULL,
    authors             TEXT[] NOT NULL,
    avg_rating          NUMERIC(3,2),
    ratings_count       INTEGER NOT NULL DEFAULT 0,
    reviews_count       INTEGER NOT NULL DEFAULT 0,
    currently_reading   INTEGER NOT NULL DEFAULT 0,
    total_readers       INTEGER NOT NULL DEFAULT 0,
    club_selections     INTEGER NOT NULL DEFAULT 0,
    avg_reading_days    NUMERIC(5,1),
    sentiment_json      JSONB,
    -- {
    --   "positive_themes": ["engaging characters","fast pacing"],
    --   "negative_themes": ["slow start","abrupt ending"],
    --   "overall_sentiment": 0.78
    -- }
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Example Event Sequences

### Member reads and unlocks spoiler-gated discussions

```
1. reading_session_logged {stream: reading, user: U1}
   → pages_read: 30, end_page: 180, end_chapter: 12
   → rm_reading_analytics updated

2. progress_updated      {stream: reading, user: U1}
   → current_chapter: 12
   → rm_club_dashboard.members_json updated for U1
   → discussions up to chapter 12 now visible to U1

3. reading_streak_continued {stream: reading, user: U1, actor: system}
   → streak: 12 days
```

### AI generates discussion questions after chapter milestone

```
1. progress_updated (multiple members reach chapter 10)

2. ai_discussion_questions_generated {stream: discussion, actor: ai}
   → book: "Project Hail Mary", chapter: 10
   → questions: ["What do you think of Rocky's communication approach?", ...]
   → thread_created event emitted
   → rm_discussion_feed row created with spoiler_chapter: 10
   → rm_club_dashboard.recent_discussions_json updated
```

### Meeting lifecycle with AI summary

```
1. meeting_scheduled     {stream: meeting, actor: user}
   → "May Book Discussion", video, 2026-05-28 7pm
   → rm_club_dashboard.upcoming_meeting_json set

2. meeting_rsvp_submitted {stream: meeting}
   → 3x attending, 1x maybe, 1x declined

3. meeting_ended         {stream: meeting, actor: system}

4. meeting_summary_generated {stream: meeting, actor: ai}
   → AI summary from recording + discussion events
   → rm_club_dashboard.upcoming_meeting_json updated with summary
```

### GDPR erasure via crypto-shredding

```
1. user_gdpr_erasure_requested {stream: user, user: U1}
   → encryption_key_ref destroyed
   → all U1's events undecryptable
   → read models for U1 deleted
   → club membership events preserved anonymously
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Infrastructure | 3 | event_store (partitioned), stream_snapshots, projection_checkpoints |
| Read Models | 5 | rm_club_dashboard, rm_reading_analytics, rm_discussion_feed, rm_recommendation_feed, rm_book_stats |
| **Total** | **8** | |

---

## Key Design Decisions

1. **Reading stream as the core user aggregate** — every reading action (shelf add, progress update, session log, rating, review) is an event on the reading stream. This enables fine-grained reading analytics and reading speed computation from historical event data.

2. **`spoiler_chapter` as a temporal gate** — discussion threads carry a `spoiler_chapter` marker. The `progress_updated` event updates each member's current chapter in the club dashboard read model. Spoiler gating is a comparison: thread.spoiler_chapter <= member.current_chapter.

3. **`rm_discussion_feed` as a separate read model** — discussions are the most frequently accessed club data (real-time updates, spoiler filtering). A dedicated read model per thread enables efficient chapter-filtered queries without processing the full club event stream.

4. **AI discussion generation as events** — when AI generates discussion questions, the generated content is an event. The thread and posts created from AI content are separate events with `actor_type: ai`, distinguishing AI-generated from user-generated discussions.

5. **`rm_reading_analytics` for weekly/monthly/yearly periods** — reading statistics (books read, pages, sessions, pace, streaks, genre distribution) are pre-computed per period so the analytics dashboard doesn't require on-the-fly event replay.

6. **`rm_book_stats` with sentiment analysis** — book-level statistics (ratings, reader count, average reading time, sentiment analysis) are aggregated from events across all users, enabling informed book selection for clubs.

7. **`rm_recommendation_feed` as a persistent read model** — AI-generated recommendations are persisted as read model rows with match scores and reasoning, enabling the recommendation feed without recomputing on each load.

8. **CloudEvents envelope with ActivityPub mapping** — every event carries standard CloudEvents fields. Reading activities and club events map to ActivityStreams 2.0 types (Create, Like, Announce), enabling ActivityPub federation with BookWyrm and the broader fediverse.

9. **`encryption_key_ref` for GDPR** — per-user encryption keys enable right-to-erasure via crypto-shredding, destroying the key renders reading history undecryptable while preserving aggregate book statistics.

10. **8 tables (3 infrastructure + 5 read models)** — the event-sourced architecture separates the write path (event store) from the read path (materialised views), with read models tailored to each view: club dashboard, reading analytics, discussion feed, recommendations, and book stats.
