# Architecture principles

This document is the project's fixed reference. Every technical decision must
be consistent with these principles. A principle changes only by agreement and
by updating this file.

## 1. Database: PostgreSQL only

- **PostgreSQL** is the only database (MySQL is not supported).
- Supabase is used only as a plain Postgres host. We do **not** use Supabase
  Auth, Storage, Realtime or RLS; all of that lives in our own server.
- The connection is configured by a single `DATABASE_URL`. Moving between
  Supabase, a self-hosted server or any other Postgres provider is done with
  `pg_dump` / `pg_restore` and changing that URL, with no code changes.
- Only features of standard Postgres (and common extensions such as PostGIS and
  pgvector) are used, never provider-specific features.

## 2. Lightweight: minimal load on database, RAM and CPU

- Every frequent query has a suitable index.
- Every list is paginated; no endpoint returns an unbounded list.
- Files (photos) are stored outside the database; the database keeps only the
  file reference.
- Realtime events (chat, matches) are pushed over WebSocket, never polled.
- Small, configurable connection pool.
- A cache layer (e.g. Redis) is added only when measurements show it is needed.

## 3. Database load rules (mandatory)

Keeping the database light is a top priority. Every piece of code is checked
against these rules before it is merged.

### 3.1 Avoid unnecessary queries
- **Settings, translations, plans and brand config are cached in server
  memory** and reloaded only when an admin changes them. They never cost a
  query per request.
- **Authentication without the database:** the user's identity comes from the
  JWT; no "who is this user" query per request.
- **No polling:** messages and matches are pushed over WebSocket.
- **Photos are served directly from disk / MinIO** with long-lived browser
  caching; the database is not involved.
- **Admin analytics** read from hourly/daily summary tables, never by counting
  whole tables each time the panel opens.

### 3.2 Every query does the minimum work
- **No N+1 queries:** related data for a list is fetched in one query (e.g. all
  photos for 20 profiles at once).
- **Only needed columns are selected;** `SELECT *` is not allowed.
- **Every query is backed by an index** and checked with `EXPLAIN` before
  merging.
- **Cursor (keyset) pagination** instead of `OFFSET`.
- **Counters are stored, not recomputed:** e.g. today's like count is kept in a
  column or in memory, not computed with `COUNT` on each request.

### 3.3 Prevent load spikes
- **Per-user rate limits,** so no single user or bot can overload the database.
- **Discover cards are fetched in batches** (e.g. 20 at a time) and kept by the
  app; no discovery query per swipe.
- **Heavy work runs in the background:** notifications, image processing and
  cleanup run outside the request path, cleanup during low-traffic hours.
- **Small connection pool** (e.g. 5–10 connections).
- **Query time cap** (`statement_timeout`, e.g. 2 seconds).

### 3.4 Data does not grow unchecked
- Old "pass" swipes are deleted after a set period.
- Large tables (swipes, messages) are partitioned by time.
- Data of deleted accounts is actually removed.
- Logs and raw analytics are not kept in the main database.

### 3.5 Continuous measurement
- `pg_stat_statements` is enabled so the slowest and most frequent queries are
  always visible.
- Each endpoint has a maximum query count (e.g. 3), enforced in tests.
- Queries slower than a set threshold are logged automatically.

## 4. Everything self-hosted

- The API server, file storage and database all run on our own infrastructure.
- The whole system starts with `docker compose`.

## 5. Professional super admin

The admin panel is a web app, separate from the user apps. Every configurable
detail of the system must be controllable from this panel, without code
changes or a new release.

- **Roles and permissions:** super admin plus narrower roles (e.g. content
  moderator, support, finance) with definable permissions.
- **Users:** search, full profile view, edit, suspend, ban, delete.
- **Content moderation:** review reports, approve or reject photos and
  profiles, handle violations.
- **Finance:** manage plans and prices, view transactions, refunds, discount
  codes.
- **Remote app configuration:** feature flags, limits (e.g. daily likes),
  forced updates, global messages and announcements.
- **Texts and translations:** edit the app's Persian and English strings from
  the panel.
- **Analytics and reports:** active users, sign-ups, matches, revenue.
- **Audit log:** every admin action is recorded and traceable.
- **Panel security:** two-factor authentication (2FA) for all admins.

## 6. Platforms and release order

1. **Phase one: Android app** (Flutter), published on **Cafe Bazaar**.
   - In-app purchases through Cafe Bazaar's billing (official Poolakey SDK).
2. **Phase two: iOS.**
   - Payments for iOS users are made on the **website**, not inside the app.
   - The website is a **professional PWA** that looks and feels like an iOS
     app (installable to the home screen, full screen, iOS-like animations and
     navigation).
   - The website must be **lightweight**: fast loading, small bundle, smooth on
     low-end phones.

### iOS distribution and payments

| Channel | Role | Payments |
|---|---|---|
| **PWA** (website) | Main iOS channel; works in any browser and needs no store | Website payment gateway |
| **Iranian iOS stores** | Complementary; certificates can be revoked by Apple | Website payment gateway |
| **App Store** | Optional, for users outside Iran; requires a legal developer account outside Iran (with legal advice) | **Apple in-app purchase** (required by Apple) |

- An App Store version must offer premium items through Apple in-app purchase
  (Guideline 3.1.3(b)). Purchases made on Android or the website are also
  honored there, but the app never mentions or links to external purchase
  (except where Apple allows it, e.g. US/EU).
- An App Store version must also meet Apple's rules for dating apps: report and
  block, content filtering, in-app account deletion, 18+ age rating, Sign in
  with Apple if any third-party login exists, privacy labels, a demo account
  for review, and a clear unique feature (Guideline 4.3).
- **No feature is ever hidden from Apple's review** with feature flags
  (Guideline 2.3.1).

## 7. Languages

- The apps, website and admin panel are in **Persian (RTL) and English (LTR)**.
- Users can switch language at any moment; the change applies instantly,
  without restarting.
- No user-facing text is hardcoded; all strings come from translation files or
  the translation panel.

## 8. UI design

- The UI of every app is built on **the best and most beautiful open-source UIs
  available on GitHub** (well-known, highly starred kits, components and
  examples).
- **The license of every source is checked** before use, to make sure
  commercial use is allowed.
- A single design system (colors, typography, spacing, components) keeps the
  Android app, the PWA and the admin panel consistent.
- Full support for light and dark mode, and for RTL.

## 9. App name and domains are never hardcoded

The app's name (brand) and domains are **never written directly in code,
designs or texts**, and can be changed at any time without rewriting code.

- **Single brand source:** Persian and English names, logo, icon, primary
  colors, support email and links, and all domains are defined in one central
  configuration, editable from the super admin panel.
- **Texts:** the app name appears in translations as a variable (e.g.
  `{appName}`), never as fixed text.
- **Domains:** the API, website/PWA, admin panel and file URLs are read from
  configuration and environment variables. Installed apps carry a list of
  fallback domains and fetch the current address from the server, so they keep
  working when a domain changes.
- **Share links, emails and notifications** are always built with the current
  name and domain.
- **Database:** no name or domain is stored in data; e.g. for photos only the
  file name is stored and the full URL is built at response time.
- **Neutral technical identifiers:** folder, service and table names are not
  tied to the brand.

> **Store exception:** the Android package name on Cafe Bazaar and the Bundle
> ID on the App Store **cannot be changed** after the first release. A neutral
> identifier unrelated to the brand is chosen. The display name and icon in the
> stores change with a new release; the in-app name changes from the panel
> without a release.

## 10. Per-app control

Every client is a separate **app** in the system, and each one is controlled
independently from the super admin panel. Initial apps:

| App | Platform | Distribution |
|---|---|---|
| `android-bazaar` | Android | Cafe Bazaar |
| `ios-store` | iOS | App Store / Iranian iOS stores |
| `web-pwa` | Web | Website / PWA |

New apps (e.g. another Android store) are added from the panel, not in code.
Each request identifies its app and version, and the per-app settings are held
in server memory (see rule 3.1), so these controls cost no database queries.

### 10.1 Login and sign-up methods
- Sign-in by **email**, by **phone number**, or both, chosen **per app**.
- Verification method per app (one-time code by SMS or email, password).
- SMS and email providers are swappable from the panel.

### 10.2 Access switches
Each switch is independent per app, takes effect immediately, and shows a
translatable message to the user:

| Switch | Effect |
|---|---|
| **Maintenance mode** | The app shows a maintenance screen; admins and test accounts can still enter. Optional planned start/end time. |
| **Sign-up closed** | New accounts cannot be created; existing users are unaffected. |
| **Login closed** | Nobody can sign in; already signed-in users can be kept in or signed out. |
| **Minimum version** | Older versions are asked, or forced, to update. |
| **Feature flags** | Any feature can be enabled or disabled per app. |
| **Payment methods** | The payment options shown in each app. |

### 10.3 Notifications and announcements
- **Broadcast push notifications** to all users of one app, several apps or
  all apps.
- Targeting by app, language, platform and user segment
  (e.g. premium, inactive for 30 days).
- Scheduled sending, in Persian and English, with delivery reports.
- In-app announcements (banner or popup) per app.
- Sending runs as a background job in batches, never as one large burst (rule
  3.3).

## 11. File storage

### 11.1 One permanent address per file
- Every file (e.g. a profile photo) has **one permanent public path** on our
  own domain, for example `/media/<file-id>`. This path **never changes**,
  whatever provider stores the file.
- The database stores only the file id, never a provider URL (see section 9).
- A file is never modified in place: a new photo gets a new id. This lets
  apps, browsers and CDNs cache files forever without ever showing stale
  images.
- Image sizes (thumbnail, card, full) are generated once at upload and served
  from the same permanent path with a size parameter.

### 11.2 Storage providers are swappable
- Storage sits behind a single internal interface; any S3-compatible service
  (MinIO, Arvan, AWS, etc.) or local disk can be a provider.
- The **media gateway** (our server) resolves the permanent path to whichever
  provider currently holds the file. Apps and the website never know or depend
  on the provider.
- Changing the primary provider is a setting change in the panel, with **no
  downtime and no broken images**.

### 11.3 Automatic sync across providers
- A file uploaded to the primary provider is **copied to every other active
  provider automatically** in the background.
- Each copy is verified by checksum, failed copies are retried, and the panel
  shows the sync status of every provider.
- If a provider is unreachable, the gateway serves the file from another
  provider that has it.
- Deletions (e.g. account deletion) are propagated to all providers.
- Adding a new provider starts a backfill of all existing files; it can be
  promoted to primary once fully synced.

### 11.4 Private files
- Sensitive files (e.g. identity verification documents) are never public;
  they are served only to authorized admins through short-lived signed links.

## 12. Deployment

### 12.1 Platform-independent by design
- The project ships only standard `Dockerfile`s and a standard
  `docker-compose.yml`. **No platform-specific configuration lives in the
  code.**
- The same files run unchanged on Coolify, Dokploy, Kamal, plain Docker Compose,
  or a developer's laptop. Changing the deployment platform requires no code
  changes.
- All configuration comes from environment variables (see section 9).

### 12.2 Starting platform: Coolify
- **Coolify** is used to start: automatic SSL, deploy on git push, scheduled
  Postgres backups to S3-compatible storage, and basic monitoring.
- It costs roughly 1–2 GB of RAM, acceptable on servers with 4 GB or more.
- On small servers (2 GB or less), plain Docker Compose with Caddy, or Kamal,
  is used instead so all resources go to the app.

### 12.3 Server layout
- **The database runs separately from the app servers** (a dedicated server or
  Supabase as plain Postgres), so app updates or failures never affect the
  data and each part scales independently.
- **Automatic daily backups** of the database, stored off-server, with
  restores tested regularly.
- **Zero-downtime deploys:** a new version starts and passes its health check
  before the old one stops.
- **Staging environment:** every change is deployed to staging before
  production.

## 13. Security

- **Secrets never live in code:** keys, passwords and tokens come only from
  environment variables or a secret store; `.env` files are never committed.
- **Passwords** are hashed with Argon2id; one-time codes expire quickly and are
  rate-limited.
- **Short-lived access tokens with refresh tokens,** stored per device; users
  can see their active devices and sign out of all of them.
- **Admin panel:** separate login, mandatory 2FA, optional IP allowlist, every
  action in the audit log.
- **All traffic over HTTPS;** sensitive fields (phone number, identity
  documents) encrypted at rest.
- **Strict input validation** on every endpoint; parameterized queries only;
  uploaded files checked by real content type and size, with metadata (EXIF,
  including location) stripped from photos.
- **Authorization checked on every request:** a user can only read or change
  what belongs to them or what they are allowed to see.
- Dependencies are kept up to date and scanned for known vulnerabilities.

## 14. User privacy

- **Exact location is never shown** to other users; only an approximate
  distance (e.g. "5 km away"). Stored location is rounded (about 100 m).
- Users control their visibility: **hide profile, incognito mode** (seen only
  by people they liked), hide age or distance.
- Photos and messages are visible only to the people allowed to see them.
- Users can **download their data** and **delete their account** from inside the
  app; deletion actually removes the data (rule 3.4).
- Personal data is never sold or shared with third parties; analytics use no
  identifying data.

## 15. Anti-spam and fake accounts

- **Selfie verification** (verified badge) by matching a live selfie against
  profile photos; reviewed by moderators or automatically.
- **Bot and duplicate detection:** limits per device, phone number and IP;
  suspicious sign-up patterns flagged to moderators.
- **Messaging only after a match;** daily limits on likes and new chats.
- **Automatic content filtering** of photos and messages (nudity, insults,
  phone numbers or links in early messages), with flagged items queued for
  moderators.
- Every limit and filter is adjustable from the super admin panel per app.

## 16. Monetization

- **Free tier:** the full core experience (profile, discover, match, chat) with
  daily limits.
- **Premium subscription:** e.g. unlimited likes, see who liked you, advanced
  filters, incognito, rewind.
- **Consumables:** e.g. Super Like, Boost.
- Plans, prices, limits and what each tier includes are **defined in the admin
  panel**, never in code, and can differ per app.
- **One purchase, valid everywhere:** a subscription bought on Cafe Bazaar, the
  App Store or the website is honored on every platform.
- Every purchase is verified server-side with its store or gateway receipt; a
  receipt can be used only once.

## 17. Code organization

- **Feature-based structure:** each feature (auth, profiles, discover, matches,
  chat, payments, media, admin, ...) has its own folder with its own files for
  routes, logic, data access and tests. Shared code lives in one clearly named
  shared module.
- **One file, one responsibility;** files stay small and focused.
- **No duplicated code:** logic needed in two places is moved into one shared
  function, component or module and reused.
- **No unnecessary code:** nothing is written "just in case"; unused code is
  deleted.
- **Comments explain why, not what:** every file starts with a short header
  describing its purpose; non-obvious decisions, rules and limits are
  commented. Obvious code is not commented, so comments stay accurate.
- **Consistent naming** across server, apps and database, so any feature can be
  found by searching its name.
- Formatting and lint rules are enforced automatically.

## 18. Technologies

| Area | Choice |
|---|---|
| Android app (later iOS) | Flutter (Dart 3) |
| Website / PWA | Lightweight; framework decided before phase two |
| Super admin panel | Web; framework decided before it starts |
| Android payments | Cafe Bazaar billing (Poolakey) |
| iOS / website payments | Payment gateway via the website |
| Server | Node.js + TypeScript |
| Database | PostgreSQL |
| Realtime chat | WebSocket |
| File storage | S3-compatible providers behind our media gateway (section 11) |
| Deployment | Docker Compose; Coolify to start (section 12) |

Details of each area (server framework, database access layer, Flutter state
management, etc.) are agreed and recorded here before that area starts.
