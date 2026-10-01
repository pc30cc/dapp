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

> **Risks to review before starting phase two:**
> - Publishing on the App Store is restricted for Iranian developers due to
>   sanctions.
> - Apple's rules usually require Apple in-app purchase for digital goods;
>   linking to external payment is allowed only in some countries.
> - Therefore the PWA may become the main channel for iOS users. The final
>   decision is made at the start of phase two and recorded here.

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

## 12. Technologies

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
| Deployment | Docker Compose |

Details of each area (server framework, database access layer, Flutter state
management, etc.) are agreed and recorded here before that area starts.
