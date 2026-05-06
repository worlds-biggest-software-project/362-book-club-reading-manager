# Book Club & Reading Manager

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source platform that unifies reading tracking, group discussion, and club organisation in a single tool for engaged book club organisers and members.

Today, running a book club means stitching together a reading tracker, a group chat, a scheduling tool, and a shared document for notes. Book Club & Reading Manager combines per-member progress tracking, spoiler-aware discussions, meeting management, and AI-driven recommendations into one purpose-built platform for organised reading communities.

---

## Why Book Club & Reading Manager?

- **No unified incumbent.** Goodreads has the catalogue but a stagnant, outdated club UX; Bookclubs.com has organisational tooling but no per-member reading progress; StoryGraph has the analytics but its book club v2 remains on the public roadmap.
- **Subscription fatigue at the high end.** Bookclubs.com charges $65/year per club for premium and Fable charges up to $69.99/year, yet neither offers spoiler-gated chapter discussions or audiobook progress sync.
- **No spoiler-gated discussions anywhere.** Chapter-level discussion gating tied to individual reading position is absent or rudimentary across every platform surveyed.
- **Audiobook readers are second-class citizens.** None of the surveyed platforms offer seamless progress sync with Audible or Libby for members who listen rather than read.
- **Closed ecosystems.** The Goodreads API has been deprecated since December 2020 and most competitors expose no public API at all, leaving developers and clubs with no way to extend or self-host.

---

## Key Features

### Club Organisation

- Club creation with public/private membership and admin role delegation
- Book scheduling with in-app polls and voting
- Meeting scheduler with RSVP, reminders, and calendar sync
- In-app video meeting hosting with recording
- Club discovery directory filterable by genre and reading pace

### Reading Progress & Analytics

- Per-member reading progress bars with page and chapter milestones
- Reading streaks, annual goals, and group-level accountability
- Personalised finish-date prediction from historical reading speed
- Audiobook progress sync via Audible and Libby

### Discussion & Engagement

- Threaded chapter-by-chapter discussion boards
- Chapter-level spoiler gating tied to each member's reading position
- AI-generated discussion questions per book and per chapter
- Moderator controls including flagging, muting, and removal
- Optional voice rooms and recorded audio clips for podcast-style engagement

### Discovery & Recommendations

- AI-driven recommendations based on group reading history and individual taste profiles
- Mood, pace, and thematic similarity matching beyond genre-only filtering
- Book search backed by Open Library, Google Books, and ISBN databases
- Sentiment analysis on community reviews to surface book strengths and pain points

### Challenges & Community

- Annual and custom reading challenges with group leaderboards
- Reading streaks and progress accountability at the group level
- Curated and auto-generated discussion guides per book

---

## AI-Native Advantage

Existing platforms treat AI as a bolt-on; Book Club & Reading Manager builds it into the core loop. AI generates context-aware discussion questions for each book and chapter, classifies spoiler content automatically without manual tagging, predicts personal finish dates accounting for real reading patterns, and powers mood- and pace-based recommendations that go beyond genre matching. AI-assisted moderation and meeting summary generation handle the operational overhead that drains volunteer organisers.

---

## Tech Stack & Deployment

The platform is designed for a freemium SaaS deployment with self-hosting on the roadmap. Book metadata is sourced from Open Library, Google Books, and ISBNdb public APIs. ActivityPub federation is a backlog item to enable interoperability with the BookWyrm fediverse and Mastodon. Audiobook integrations target Audible and Libby. The pricing model is free for clubs up to 10 members with paid tiers for larger clubs and premium analytics.

---

## Market Context

Subscription-based apps account for over 40% of book-app revenue in 2026, yet no dominant all-in-one platform for organised clubs has emerged. Incumbent pricing ranges from free (Goodreads, Novellic, Hardcover, BookWyrm) to $65–$70 per year per club (Bookclubs.com, Fable). Primary buyers are book club organisers running 5–30 member clubs who currently juggle multiple tools and would pay for an integrated solution.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
