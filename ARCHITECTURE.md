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

## 3. Everything self-hosted

- The API server, file storage and database all run on our own infrastructure.
- The whole system starts with `docker compose`.

## 4. Professional super admin

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

## 5. Platforms and release order

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

## 6. Languages

- The apps, website and admin panel are in **Persian (RTL) and English (LTR)**.
- Users can switch language at any moment; the change applies instantly,
  without restarting.
- No user-facing text is hardcoded; all strings come from translation files or
  the translation panel.

## 7. UI design

- The UI of every app is built on **the best and most beautiful open-source UIs
  available on GitHub** (well-known, highly starred kits, components and
  examples).
- **The license of every source is checked** before use, to make sure
  commercial use is allowed.
- A single design system (colors, typography, spacing, components) keeps the
  Android app, the PWA and the admin panel consistent.
- Full support for light and dark mode, and for RTL.

## 8. App name and domains are never hardcoded

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

## 9. Technologies

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
| Photo storage | S3-compatible (MinIO) |
| Deployment | Docker Compose |

Details of each area (server framework, database access layer, Flutter state
management, etc.) are agreed and recorded here before that area starts.
