# Book Club & Reading Manager — Feature & Functionality Survey

> Candidate #362 · Researched: 2026-05-04

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| Bookclubs.com | Web + Mobile (iOS/Android) | Freemium; $65/year per club for premium | https://bookclubs.com |
| Fable | Mobile (iOS/Android) | Freemium; $69.99/year or $5.99–$9.99/month per club | https://fable.co |
| Goodreads | Web + Mobile | Free (ad-supported, Amazon-owned) | https://goodreads.com |
| StoryGraph | Web + Mobile | Freemium; Plus tier for advanced analytics | https://thestorygraph.com |
| Novellic | Mobile (iOS/Android) | Free | https://novellic.com |
| Hardcover | Web + Mobile | Free (open-source, community-driven) | https://hardcover.app |
| Literal Club | Web + Mobile | Free | https://literal.club |
| Bookly | Mobile (iOS/Android) | Freemium; premium for advanced stats | https://getbookly.com |
| BookWyrm | Web (self-hostable) | Free and open-source (AGPL-3.0) | https://joinbookwyrm.com |

---

## Feature Analysis by Solution

### Bookclubs.com

**Core features**
- Club creation and management with organiser/member roles
- Book scheduling and voting via in-app polls
- Meeting RSVP and attendance tracking
- In-app video meetings with recording capability
- Automated personal calendar sync
- Group discussion boards and direct messaging
- Club discovery directory
- Barcode scanning to add books to shelves
- Discussion guides and reading questions per book
- Reading goal tracking for members

**Differentiating features**
- Strongest dedicated organisational tooling of any platform (admin delegation, custom event titles, meeting reminders)
- In-app hosted video meetings with recording — no third-party integration required
- Multiple club admin roles, reducing single-organiser bottleneck

**UX patterns**
- Simple card-based book and event layout accessible to non-technical users
- Colour-coded club shelves for at-a-glance progress
- Progressive disclosure: basic features free; premium features revealed on upgrade prompts

**Integration points**
- Calendar sync (Google Calendar, Apple Calendar)
- Native video meeting hosting
- Email reminders and notifications

**Known gaps**
- No per-member reading progress tracking (only book-level scheduling)
- No AI-driven recommendations
- No spoiler-gated discussion threads tied to reading chapter position
- Audiobook integration absent
- Notification granularity requested by users (e.g., poll submissions, new posts)
- Cannot schedule meetings for books not yet in the database

**Licence / IP notes**
- Proprietary SaaS; no public API or SDK

---

### Fable

**Core features**
- In-app interactive ebook reader with highlights, notes, and reactions
- Club creation with author-led and influencer-led clubs
- Chapter-level discussion threads
- Personal reading tracker with annual goals and streaks
- Social feed of friends' activity
- Curated and community reading lists
- Half-star ratings and emoji reactions on reviews
- Content warnings on books

**Differentiating features**
- Built-in ebook reader — the only major platform combining reading and discussion in one interface
- Author and BookTok influencer-led premium clubs as a content model
- Annotation and highlight sharing within club threads

**UX patterns**
- Social-media-style scrollable feed (Gen-Z oriented)
- Rich reading progress visualisation (charts, streaks, annual wrap-up)
- Premium clubs revealed via discovery tab to drive upsells

**Integration points**
- White-label B2B SDK available for publishers and media partners
- No public developer API

**Known gaps**
- Limited custom club tooling for independent organisers
- Duplicate book listings in catalogue reported by users
- No author pages (unlike Goodreads)
- No meeting scheduling or RSVP tooling
- Audiobook tracking not supported

**Licence / IP notes**
- Proprietary commercial; white-label SDK available under negotiated terms

---

### Goodreads

**Core features**
- Massive book catalogue (150 million+ registered users)
- Reading shelves (Read, Currently Reading, Want to Read)
- Annual reading challenge with progress bar
- Group-based discussion forums
- User-written reviews and star ratings
- Author pages with Q&A
- Book lists curated by users and editorial staff
- Friend activity feed and book comparisons

**Differentiating features**
- Largest social reading community globally
- Most comprehensive author page ecosystem
- Long-established review corpus used for discovery

**UX patterns**
- Dense, legacy web interface — desktop-first design
- Mobile app is a pared-down version of the web experience
- Minimal onboarding guidance

**Integration points**
- Goodreads public API deprecated (no new keys issued since December 2020)
- Amazon ecosystem (Kindle reading sync available)
- Goodreads CSV export for data portability

**Known gaps**
- Group/club UX is outdated and rarely updated
- No reading pace or analytics beyond annual goal count
- No video meetings, RSVP, or scheduling tooling
- No spoiler-gated discussions
- API fully deprecated; developers cannot build on it

**Licence / IP notes**
- Proprietary; owned by Amazon. API is deprecated. Data export available via CSV.

---

### StoryGraph

**Core features**
- Detailed reading analytics (mood charts, pace graphs, genre breakdowns, format splits, ratings histograms, monthly trends)
- Annual reading goals with page and hour targets
- DNF (Did Not Finish) tracking with page number
- Content warnings on books
- AI-powered mood-based book recommendations
- Half- and quarter-star ratings
- Progress updates with built-in reading journal
- Buddy reads and read-along features
- Book club functionality (in active development)

**Differentiating features**
- Best-in-class reading statistics and data visualisation
- Mood-based recommendation engine (not purely genre/author-based)
- Strong privacy-conscious positioning (no Amazon affiliation, no ads)

**UX patterns**
- Data-driven dashboard as landing screen
- Filters and custom charts available on Plus tier
- Clean, accessible design with dark mode

**Integration points**
- Goodreads import (CSV)
- No public API (feature requested on community roadmap)
- Community-built Python scraper available as unofficial workaround

**Known gaps**
- Book club v2 still requested on public roadmap — current club tools are minimal
- No in-app meeting scheduling or RSVP
- No video integration
- No official API for third-party developers

**Licence / IP notes**
- Proprietary SaaS; independent, Black-woman-owned business

---

### Novellic

**Core features**
- Personalised book recommendations based on reading taste profile
- Club creation (public and private)
- Shared club reading history
- Member management (public/private membership settings)
- Reading goal setting and tracking
- Social progress sharing
- Book discovery via community ratings

**Differentiating features**
- Recommendation engine built on real reader data rather than algorithms alone
- Small, engaged community with taste-driven discovery

**UX patterns**
- Mobile-first; lightweight UI
- Onboarding centres on taste profiling to seed recommendations

**Integration points**
- Bookshop.org affiliate integration for book purchasing
- No public API

**Known gaps**
- Very small user base limiting social discovery
- No meeting scheduler or RSVP
- No discussion guides or prompts
- No analytics beyond goal tracking

**Licence / IP notes**
- Proprietary mobile app

---

### Hardcover

**Core features**
- Reading status tracking (Want to Read, Currently Reading, Read, Did Not Finish)
- Half-star ratings and reviews
- Social reading feed (friends' activity, trending picks)
- Custom reading lists
- Series and edition tracking
- Audiobook tracking (format tagging)
- Annual reading goals
- Book discovery via curated lists and community trends
- Open GraphQL API

**Differentiating features**
- Only major platform with a free, open GraphQL API for developers
- Amazon-free, ad-free, open-source philosophy
- Engaged tech-oriented community

**UX patterns**
- Clean modern design; dark mode
- Progressive feature reveal; power features available via GraphQL console

**Integration points**
- GraphQL API (beta): full read/write access to user data and book catalogue
- Goodreads import
- JavaScript/Python SDK available via community

**Known gaps**
- No club organisational tools (scheduling, RSVP, group discussions)
- No discussion guides or generated prompts
- Smaller catalogue than Goodreads or ISBNdb

**Licence / IP notes**
- Open-source; GraphQL API is free to use (beta)

---

### Literal Club

**Core features**
- Reading tracker (shelves, reading log)
- Social feed from trusted friends
- Club creation and book-specific discussion forums
- Highlights and notes via camera scan
- Social sharing of highlights to Twitter, Instagram, WhatsApp
- Book discovery through trusted social graph

**Differentiating features**
- Camera-based highlight capture — scan a book page to extract a quote
- Trust-graph social model (recommendations from people you know, not strangers)
- Minimalist, aesthetic UI targeted at engaged literary readers

**UX patterns**
- Invite-based onboarding (historically invite-only)
- Minimalist card layout; emphasis on quotes and highlights over statistics
- Social feed prioritises personal connections over algorithmic content

**Integration points**
- No public API
- Social sharing to mainstream platforms

**Known gaps**
- No meeting scheduler or RSVP
- No reading pace or analytics
- No AI recommendations
- Small user base; discovery limited to personal social graph

**Licence / IP notes**
- Proprietary SaaS

---

### Bookly

**Core features**
- Real-time reading session timer
- Reading speed calculation and finish-date prediction
- Reading statistics (total read time, pages read, streaks, daily reading time)
- Infographic-style session and book reports
- Annual, monthly, and weekly reading goals
- Quote and note capture during sessions
- Bookshelf organisation
- Reading challenges ("Bookly Readathons")
- Achievement system for motivation

**Differentiating features**
- Timer-first UX — tracks the act of reading rather than just cataloguing books
- Predicted finish date based on personal reading speed
- Infographic export for social sharing

**UX patterns**
- Mobile-first; session timer is the primary action on the home screen
- Achievement badges for habit reinforcement
- Weekly/monthly infographic reports as shareable social content

**Integration points**
- No public API
- Social sharing of reports

**Known gaps**
- No social or community features
- No book club or group functionality
- No recommendations engine
- No discussion threads or prompts

**Licence / IP notes**
- Proprietary; freemium mobile app

---

### BookWyrm

**Core features**
- Federated social reading (ActivityPub compatible — interoperates with Mastodon, Pleroma)
- Book reviews, comments, and quotes
- Reading lists and shelves
- Instance-level federation and moderation controls
- Decentralised book metadata database (collaboratively built across instances)
- Community-moderated data

**Differentiating features**
- Only reading platform built on ActivityPub federation
- Self-hostable; each instance is autonomous with its own community rules
- Fully open-source (AGPL-3.0); no corporate ownership

**UX patterns**
- Mastodon-like interface; familiar to fediverse users
- Open to non-technical users on public instances; self-hosting for technical users
- Community-driven moderation visible in UI

**Integration points**
- ActivityPub (interoperates with full fediverse ecosystem)
- Open-source codebase (contributors can extend via pull requests)
- No formal SDK or REST API beyond ActivityPub endpoints

**Known gaps**
- No club organisational tools (scheduling, RSVP, meeting management)
- UI less polished than commercial competitors
- Book catalogue depends on community contributions; gaps in metadata
- No AI features

**Licence / IP notes**
- AGPL-3.0 — open-source and freely forkable, but derivative works must also be AGPL

---

## Cross-Cutting Feature Themes

### Table-Stakes Features
- Book search and cataloguing by title, author, ISBN
- Reading shelves (currently reading, read, want to read)
- Personal reading goal setting (annual book or page count)
- Basic progress tracking per book
- Star ratings and written reviews
- Friend/follower social graph

### Differentiating Features
- In-app ebook reader integrated with discussion (Fable)
- Real-time reading session timer with finish-date prediction (Bookly)
- Deep reading analytics and data visualisation (StoryGraph)
- Full club organisational suite: scheduling, RSVP, video meetings (Bookclubs.com)
- ActivityPub federation and self-hosting (BookWyrm)
- Open GraphQL API for developers (Hardcover)
- Camera-based highlight capture (Literal)
- Author- and influencer-led premium clubs (Fable)

### Underserved Areas / Opportunities
- **Unified platform**: No single tool combines strong club organisation (scheduling, RSVP, meetings) with deep individual reading analytics
- **Spoiler-gated discussions**: Chapter-level discussion gating tied to individual reading position is absent or rudimentary across all platforms
- **Audiobook parity**: None of the platforms offer seamless progress sync with Audible or Libby for members who listen rather than read
- **AI discussion facilitation**: Auto-generated, context-aware discussion questions per book and chapter are missing; existing "discussion guides" are generic or manual
- **Cross-club reading insights**: Aggregated anonymised analytics showing how clubs read (pace, completion rates, most-discussed chapters) are not available
- **Member accountability**: Reading streak and progress accountability mechanisms that work at group level (not just individual) are weak across all platforms
- **Moderation tooling**: Tools for managing inappropriate content in group discussions are primitive; no AI-assisted moderation

### AI-Augmentation Candidates
- **Recommendation engine**: Mood, pace, and thematic similarity matching — superior to genre-only filtering (StoryGraph's current approach is closest but limited)
- **Discussion prompt generation**: Per-book, per-chapter AI-generated discussion questions contextualised to the club's reading history
- **Spoiler detection**: AI classification of spoiler content in discussion threads without requiring manual tagging
- **Reading pace prediction**: AI-personalised finish-date estimates accounting for historical reading speed variations (weekends, busy periods)
- **Meeting summary generation**: Auto-transcription and structured summary of club meeting recordings
- **Sentiment analysis on reviews**: Surface what readers loved or struggled with across a book's community reviews before a club votes to read it

---

## Legal & IP Summary

No patent or copyright concerns were identified with building an independent book club and reading manager platform. The Goodreads API has been deprecated since December 2020; any data obtained from Goodreads must use their authorised CSV export. The Hardcover GraphQL API is available under a free-use beta licence. Open Library, Google Books, and ISBNdb all offer documented public APIs with permissive terms for application developers. BookWyrm is AGPL-3.0 licensed, meaning any code derived directly from it must also be released under AGPL; building a compatible ActivityPub implementation from scratch carries no such obligation. No patented features were identified in the platforms surveyed.

---

## Recommended Feature Scope

**Must-have (MVP)**
- Club creation, membership management, and book scheduling with polls
- Per-member reading progress tracker (page/chapter milestones)
- Meeting scheduler with RSVP, reminders, and calendar sync
- Threaded discussion boards with chapter-level spoiler gating
- Book search via Open Library and Google Books APIs
- Basic personal reading goal and annual challenge tracking

**Should-have (v1.1)**
- AI-generated discussion questions per book and chapter
- In-app video meeting hosting with recording
- AI-powered book recommendations based on group reading history and taste profiles
- Audiobook progress sync (Audible/Libby integration)
- Reading pace analytics and personalised finish-date prediction
- Club discovery directory with genre and pace filters

**Nice-to-have (backlog)**
- ActivityPub federation for cross-platform interoperability
- Camera-based highlight and quote capture
- AI-assisted moderation for discussion content
- Voice discussion rooms / recorded audio clips for meetings
- Cross-club aggregated reading insights dashboard
- Group leaderboards for reading challenges
