# Standards & API Reference

> Project: Book Club & Reading Manager · Generated: 2026-05-04

---

## Industry Standards & Specifications

### Book Metadata & Identification Standards

**ISBN (International Standard Book Number)**
- Standard: ISO 2108
- URL: https://www.isbn-international.org/
- The universal identifier for book editions and formats. Every book lookup API (Google Books, Open Library, ISBNdb) resolves books by ISBN. The app must accept ISBN-10 and ISBN-13 inputs and normalise them to ISBN-13 for internal storage and interoperability.

**ONIX for Books**
- Standard: EDItEUR ONIX 3.0
- URL: https://www.editeur.org/83/overview/
- The international XML-based standard for communicating book product information (title, contributor, description, price, availability) through the publishing supply chain. Relevant for any publisher partnership, catalogue import/export, or integration with distributors and retailers. Widely adopted in North America, Europe, and Australasia.

**BISAC Subject Headings**
- Standard: Book Industry Study Group (BISG)
- URL: https://www.bisg.org/bisac-subject-codes
- The industry-standard genre and subject taxonomy for English-language books, used by publishers, booksellers, and catalogues. Using BISAC codes for internal genre classification ensures compatibility with book data from major APIs and supports consistent recommendation filtering.

---

### Web & Communication Standards

**HTTP/1.1 and HTTP/2**
- Standards: RFC 7230–7235 (HTTP/1.1), RFC 7540 (HTTP/2)
- URLs: https://www.rfc-editor.org/rfc/rfc7230, https://www.rfc-editor.org/rfc/rfc7540
- Foundation for all REST API communication. HTTP/2 multiplexing is particularly relevant for mobile clients fetching multiple resources (book covers, user progress, club discussions) in parallel.

**WebSocket Protocol**
- Standard: RFC 6455
- URL: https://www.rfc-editor.org/rfc/rfc6455
- Defines the full-duplex persistent connection standard used for real-time features: live discussion updates, reading progress broadcasts to club members, and meeting room presence signals. Supported by 99%+ of browsers and all major mobile platforms. Relevant for the real-time discussion threading feature.

**Web Push Notifications**
- Standards: RFC 8030 (transport), RFC 8292 (VAPID authorisation), RFC 8291 (encryption)
- URLs: https://www.rfc-editor.org/rfc/rfc8030, https://www.rfc-editor.org/rfc/rfc8292, https://www.rfc-editor.org/rfc/rfc8291
- Defines the open standard for delivering push notifications to web browsers without a proprietary push gateway. Works alongside platform-specific services:
  - Apple Push Notification Service (APNs) — iOS/macOS
  - Firebase Cloud Messaging (FCM) — Android and cross-platform web
- Critical for meeting reminders, reading milestone nudges, and discussion reply alerts.

**ActivityPub**
- Standard: W3C Recommendation (January 2018)
- URL: https://www.w3.org/TR/activitypub/
- The decentralised social networking protocol powering the fediverse (Mastodon, BookWyrm, Pleroma). Provides both a client-to-server (C2S) API and a federated server-to-server (S2S) protocol using ActivityStreams 2.0 (JSON-LD). Relevant if implementing cross-instance federation or interoperability with BookWyrm communities.

**ActivityStreams 2.0**
- Standard: W3C Recommendation
- URL: https://www.w3.org/TR/activitystreams-core/
- The JSON-LD vocabulary used by ActivityPub to represent social objects (posts, reviews, comments) and activities (Create, Like, Announce). If building a federated reading social graph, ActivityStreams 2.0 defines the data model for reviews, reading status updates, and club announcements.

---

### API & Data Specifications

**OpenAPI 3.1**
- Standard: OpenAPI Specification (Linux Foundation)
- URL: https://spec.openapis.org/oas/v3.1.0
- The industry-standard machine-readable format for documenting REST APIs. Any public or partner-facing API the app exposes should be documented with an OpenAPI 3.1 spec to enable auto-generated client SDKs and developer portal tooling.

**GraphQL**
- Standard: GraphQL Specification (GraphQL Foundation)
- URL: https://spec.graphql.org/
- A query language and runtime for APIs enabling clients to request exactly the data they need. Relevant as the query paradigm used by the Hardcover API. Consider GraphQL for the app's own developer-facing API to allow flexible queries over books, clubs, members, and reading progress.

**JSON Schema**
- Standard: IETF Draft (draft-bhutton-json-schema-01)
- URL: https://json-schema.org/specification
- Defines the structure, validation, and documentation of JSON documents. Use for validating book metadata payloads, reading progress updates, and club event objects exchanged between client and server.

**JSON-LD**
- Standard: W3C Recommendation
- URL: https://www.w3.org/TR/json-ld11/
- Linked data format using JSON, used by ActivityStreams 2.0 and Schema.org. Relevant for representing book and reading activity data in a semantically interoperable way.

**Schema.org Book Vocabulary**
- Standard: Schema.org (community/W3C)
- URL: https://schema.org/Book
- Structured data vocabulary for books used in SEO and web interoperability. Implementing Schema.org Book markup on public-facing club and book pages improves search engine discoverability and rich-result display.

---

### Authentication & Security Standards

**OAuth 2.0**
- Standard: RFC 6749
- URL: https://www.rfc-editor.org/rfc/rfc6749
- The industry-standard authorisation framework for delegated access. Required for Google Books API integration (user bookshelves) and any third-party OAuth provider (Google, Apple Sign-In). The app's own public API should use OAuth 2.0 for developer access tokens.

**OpenID Connect (OIDC)**
- Standard: OpenID Foundation
- URL: https://openid.net/specs/openid-connect-core-1_0.html
- An identity layer on top of OAuth 2.0 that provides verified user authentication and standardised identity tokens (JWT). Use for social login (Sign in with Google, Sign in with Apple) and for any SSO integration with library systems or institutional accounts.

**JSON Web Token (JWT)**
- Standard: RFC 7519
- URL: https://www.rfc-editor.org/rfc/rfc7519
- The token format used by OIDC and widely used in REST API authentication. Reading progress, club membership, and user preferences should be scoped to validated JWT claims.

**OWASP Top 10**
- Standard: OWASP (community, non-profit)
- URL: https://owasp.org/www-project-top-ten/
- The authoritative list of the most critical web application security risks. Particular relevance for a social reading platform: Broken Access Control (private club content must not leak to non-members), Injection (SQL/NoSQL injection via book search inputs), and Cryptographic Failures (protecting reading history as sensitive personal data).

**GDPR (General Data Protection Regulation)**
- Standard: EU Regulation 2016/679
- URL: https://gdpr-info.eu/
- Applies to any app with EU users. Reading history, club membership, and personal taste profiles constitute personal data under GDPR. Key requirements: explicit consent for data processing, right to be forgotten (user data deletion), data minimisation, and data portability (export of reading data). Non-compliance fines reach €20 million or 4% of global annual revenue.

---

### MCP Server Specifications

Model Context Protocol (MCP) is relevant for exposing book club data to AI assistants and LLM-powered features. Key use cases for an MCP server implementation:
- Allow AI assistants to query a user's reading history to generate personalised recommendations
- Expose club discussion threads and reading progress to an LLM for generating meeting summaries or discussion prompts
- MCP specification: https://modelcontextprotocol.io/specification

---

## Similar Products — Developer Documentation & APIs

### Google Books API
- **Description:** REST API providing access to Google's book catalogue, including full-text search, volume metadata, book cover images, and user My Library management.
- **API Documentation:** https://developers.google.com/books/docs/v1/using
- **Getting Started:** https://developers.google.com/books/docs/v1/getting_started
- **API Reference:** https://developers.google.com/books/docs/v1/reference
- **SDKs/Libraries:** Google API client libraries for JavaScript, Python, Java, Go (https://developers.google.com/api-client-library)
- **Standards:** REST/JSON; OAuth 2.0 for user data; API key for public data
- **Authentication:** API key (public data) or OAuth 2.0 (user My Library)
- **Base URI:** https://www.googleapis.com/books/v1
- **Notes:** Most widely used book API; covers metadata, search, and user library management. Does not provide access to full book text unless the book is in the public domain.

### Open Library API
- **Description:** Free, open REST API from the Internet Archive providing access to over 20 million book records, including author data, editions, covers, and reading lists.
- **API Documentation:** https://openlibrary.org/developers/api
- **Search API:** https://openlibrary.org/dev/docs/api/search
- **RESTful API Guide:** https://openlibrary.org/dev/docs/restful_api
- **SDKs/Libraries:** Official Python client; community Ruby and Elixir clients (https://openlibrary.org/developers)
- **Standards:** REST; JSON, YAML, and RDF/XML formats
- **Authentication:** No API key required for public GET requests; session cookie for write operations
- **Notes:** Best source for open, freely reusable book metadata. Covers API fetches book cover images by ISBN. No rate-limit documentation for low-volume use; bulk data dumps available for high-volume use cases.

### ISBNdb API
- **Description:** Commercial REST API providing access to over 108 million book titles, including metadata, pricing, and availability. Established 2001.
- **API Documentation:** https://isbndb.com/isbndb-api-documentation-v2
- **API Reference (Postman):** https://www.postman.com/api-evangelist/isbndb/collection/86oo60r/isbndb-api
- **SDKs/Libraries:** Community Dart/Flutter package; Python library (https://github.com/scholnicks/isbndb)
- **Standards:** REST/JSON
- **Authentication:** API key (Authorization header)
- **Rate Limits:** 1 req/s (Basic), 3 req/s (Premium), 5 req/s (Pro)
- **Notes:** Best catalogue coverage for obscure or international titles. Commercial pricing applies. Useful as a fallback when Google Books or Open Library metadata is incomplete.

### Hardcover GraphQL API
- **Description:** Free GraphQL API (beta) providing full read/write access to Hardcover's book catalogue, user reading statuses, ratings, reviews, and lists. Used by the Hardcover web and mobile apps themselves.
- **API Documentation:** https://docs.hardcover.app/api/getting-started/
- **GitHub (schema and docs):** https://github.com/hardcoverapp/hardcover-docs
- **GraphQL Schema:** https://github.com/hardcoverapp/hardcover-docs/blob/main/schema.graphql
- **Standards:** GraphQL
- **Authentication:** Bearer token (obtained from account settings page)
- **Notes:** Strongest open developer API in the book app ecosystem. Supports querying and mutating reading status, books, lists, and reviews. Still in beta; subject to change. Free to use. A strong candidate for data enrichment and community integration.

### Bookclubs.com
- **Description:** Web and mobile platform for book club organisation with scheduling, RSVP, video meetings, and discussion tools.
- **API Documentation:** No public API documented
- **SDKs/Libraries:** None publicly available
- **Standards:** Proprietary
- **Authentication:** N/A (no public API)
- **Notes:** Closest functional competitor for the club-management feature set. No developer integration path available.

### Fable (fable.co)
- **Description:** Social reading app with built-in ebook reader, club features, and author-led premium clubs.
- **API Documentation:** No public REST/GraphQL API documented for fable.co (social reading); white-label B2B SDK available under negotiated terms
- **SDKs/Libraries:** White-label SDK (B2B only)
- **Standards:** Proprietary
- **Authentication:** N/A (no public API)
- **Notes:** B2B white-label SDK may be accessible for publishers or media partners; not available for independent developers.

### BookWyrm
- **Description:** Federated, self-hostable, open-source social reading platform built on ActivityPub. Interoperates with the fediverse.
- **API Documentation:** https://joinbookwyrm.com/api/ (ActivityPub endpoints)
- **GitHub:** https://github.com/bookwyrm-social/bookwyrm
- **Standards:** ActivityPub (W3C), ActivityStreams 2.0
- **Authentication:** OAuth 2.0 for API access
- **Licence:** AGPL-3.0
- **Notes:** Source code is fully open for study and contribution. ActivityPub endpoints can be used for federation. Code should not be incorporated directly without complying with AGPL-3.0 copyleft terms.

### StoryGraph
- **Description:** Independent reading tracker and analytics platform with mood-based recommendations and detailed reading statistics.
- **API Documentation:** No official public API (community-requested on roadmap; no ETA given)
- **SDKs/Libraries:** Community Python scraper (unofficial): https://pypi.org/project/storygraph-api/
- **Standards:** N/A (no public API)
- **Authentication:** N/A
- **Notes:** Official API is planned but not prioritised. Only Goodreads CSV import and export are available for data portability.

---

## Notes

**Goodreads API deprecation**: The Goodreads public API has been closed to new registrations since December 2020. Developers should not rely on Goodreads data. User data portability from Goodreads is limited to CSV export. The Hardcover GraphQL API is the most viable open alternative for community book data.

**Audiobook integration gap**: Neither Audible nor Libby exposes a public API for reading/listening progress. Progress sync with these platforms would require either a manual progress entry from the user or a browser extension/companion app approach. This remains an open technical challenge across all competitors.

**Emerging standard — MCP for reading tools**: The Model Context Protocol is gaining traction as a way to expose reading and library data to LLM-powered assistants. Building an MCP server exposing club reading data, progress, and discussion history would differentiate the platform as AI-native infrastructure rather than a standalone app.

**Push notification infrastructure**: For cross-platform push notifications, FCM (Google Firebase Cloud Messaging) is the most common choice, as it handles both Android natively and iOS via APNs relay, reducing infrastructure complexity to a single integration point.
