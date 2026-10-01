# Blogger Serverpod Engine & Unified Schema.org JSON-LD Platform

> **A Next-Generation, Open-Source Alternative to Google Blogger built with Serverpod (`^4.1.0-beta.1`) and Flutter.**
> Powered by a universal **Type-Safe Schema.org JSON-LD Linked Data Engine**, App Store-compliant Multi-Provider Authentication (Apple Sign-In, Google, JWT IDP), Exact Geolocation & Reverse Geocoding, and the **Kaisel Deep-Link Branch Router**.

---

## 🌟 Executive Overview & Architectural Vision

Google Blogger is a legendary publishing platform loved for its simplicity, speed, and flexibility. However, modern application ecosystems impose constraints that traditional Blogger cannot meet:
1. **App Store Rejection Risks**: Publishing iOS mobile apps integrated solely with Google Sign-In violates Apple App Store Guideline 4.8, which mandates Apple Sign-In alongside third-party social logins.
2. **Rigid Content Schemas**: Traditional blogging platforms store unstructured HTML strings. Modern e-commerce, service bookings, and multivendor marketplaces require structured, machine-readable data.
3. **Missing Business Operations**: Out of the box, legacy Blogger lacks user role management, granular notifications, precise GPS geolocation, and unified multi-vendor order execution.

### The Unified Solution
Our platform re-architects Blogger into a **Serverpod-backed backend and Flutter frontend system** where:
- **Clean, Unified Route Hierarchy**: Every entity—whether a blog post, article, physical product, service package, store location, or job posting—is hosted cleanly under standard blog routes (`/blog/:blogId`, `/blog/:blogId/post/:postId`, `/blog/:blogId/page/:pageId`).
- **Schema.org Linked Data `@type` Disambiguation**: The frontend and backend inspect the JSON-LD `@type` field (`BlogPosting`, `Product`, `Service`, `LocalBusiness`, `JobPosting`, etc.) to dynamically adjust rendering, navigation, cart options, and search indexing without needing extra top-level route namespaces.
- **Linked Data `@id` Cross-Referencing & Overrides**: Standard JSON-LD nodes can be referenced across multiple blog posts via `@id` URIs. Developers and authors can manually specify local property overrides over linked nodes to avoid repetitive data entry while retaining complete customization.
- **Custom Backend & Full Control**: Built on Serverpod with PostgreSQL, offering multi-tenant user access control (RBAC), JWT authentication with Apple Sign-In compliance, exact geolocation services, AdSense earnings tracking, and push notifications.

---

## 📚 Complete Specification Index

The complete platform specifications are organized into 7 modular, in-depth documentation suites inside the [`docs/`](./docs) directory:

| Document | Title | Description |
| :--- | :--- | :--- |
| **[`01_VISION_AND_OVERVIEW.md`](./docs/01_VISION_AND_OVERVIEW.md)** | **Vision & Architectural Core** | Platform goals, Apple App Store publishing compliance, single/multivendor commerce integration, and core architectural principles. |
| **[`02_AUTHENTICATION_AND_IDENTITY.md`](./docs/02_AUTHENTICATION_AND_IDENTITY.md)** | **Authentication & Identity Engine** | Serverpod multi-provider auth (Apple Sign-In, Google OAuth, Email JWT), session management, JWT refresh token endpoint, and RBAC roles. |
| **[`03_SCHEMA_ORG_LINKED_DATA_ENGINE.md`](./docs/03_SCHEMA_ORG_LINKED_DATA_ENGINE.md)** | **Schema.org Linked Data Engine** | Storing type-safe `schemaorg-current-https.jsonld` payloads, `@id` URI cross-referencing, property override inheritance, and web DOM head SEO injection. |
| **[`04_MULTIVENDOR_AND_SERVICE_PROVIDERS.md`](./docs/04_MULTIVENDOR_AND_SERVICE_PROVIDERS.md)** | **Multivendor, Services, Geolocation & Earnings** | Multi-vendor commerce, service package catalogs (`OfferCatalog`), GPS reverse geocoding (`LocationManager`), cart management, and monetization. |
| **[`05_SERVERPOD_BACKEND_SPECIFICATION.md`](./docs/05_SERVERPOD_BACKEND_SPECIFICATION.md)** | **Serverpod Backend & Database Specs** | Complete `.spy.yaml` data models, database schema, Blogger REST API v3 compatibility endpoints (`BloggerV3Endpoint`), and custom protocol endpoints. |
| **[`06_FLUTTER_STUDIO_AND_FRONTEND_SPEC.md`](./docs/06_FLUTTER_STUDIO_AND_FRONTEND_SPEC.md)** | **Flutter Studio & Kaisel Router Spec** | 15-tab Flutter Studio UI, universal JSON-LD renderer, and full `KaiselConfigCodec<AppRoute>` deep-link routing spec (including `gl` language parameter). |
| **[`07_JSON_LD_VISUAL_EDITOR_AND_ENGINE_SPEC.md`](./docs/07_JSON_LD_VISUAL_EDITOR_AND_ENGINE_SPEC.md)** | **Visual Editor & Ingestion Pipeline** | No-code visual schema builder (`JSON-LD-Visual-Editor`), feed extraction pipeline (`SchemaExtractor`), Rich Results validation, and URL scraping. |

---

## 🏛️ High-Level System Architecture

```
                                +-------------------------------------------------------+
                                |                    FLUTTER FRONTEND                   |
                                |  (Studio Dashboard, Universal JSON-LD Visual Renderer)|
                                +---------------------------+---------------------------+
                                                            |
                                        Kaisel Router & Deep Link Engine
                            (/blog/:blogId, /blog/:blogId/post/:postId, /search/label/:name)
                                                            |
                                                            v
                                +-------------------------------------------------------+
                                |               SERVERPOD BACKEND SERVER                |
                                |     (Dart 3.13 / Serverpod ^4.1.0-beta.1 Workspace)   |
                                +-----+--------------------+--------------------+-------+
                                      |                    |                    |
            +-------------------------+          +---------+---------+          +-------------------------+
            |                                    |                   |                                    |
            v                                    v                   v                                    v
+-----------------------+              +------------------+ +------------------+              +-----------------------+
|  MULTI-PROVIDER AUTH  |              | SCHEMA.ORG ENGINE| | BLOGGER V3 REST |              |   LOCATION & GEOPAY   |
| Apple, Google, Email  |              | Type-Safe JSON-LD| | Google Compatibility|             | Nominatim Geocoding,  |
| JWT Session Endpoint  |              | @id Linked Graph | | 13+ Endpoint Ports|             | PIN Lookup, Cart/Pay  |
+-----------+-----------+              +--------+---------+ +--------+---------+              +-----------+-----------+
            |                                   |                    |                                    |
            +-----------------------------------+----------+---------+------------------------------------+
                                                           |
                                                           v
                                            +-----------------------------+
                                            |     POSTGRESQL DATABASE     |
                                            |  Relational Metadata &      |
                                            |  Type-Safe JSON-LD Payloads |
                                            +-----------------------------+
```

---

## ⚙️ Quickstart & Development Setup

### Workspace Requirements
- **Dart SDK**: `>= 3.12.2` (Targeting `3.13.4`)
- **Flutter SDK**: `>= 3.19.0` (Targeting `3.47.5`)
- **Serverpod CLI**: `4.1.0-beta.1`
- **PostgreSQL**: Managed embedded database via Serverpod test environment or standalone Postgres.

### Installation & Execution
```bash
# 1. Install Serverpod CLI 4.1.0-beta.1
dart pub global activate serverpod_cli 4.1.0-beta.1

# 2. Get Monorepo Workspace Dependencies
cd blogger
dart pub get

# 3. Start Serverpod Backend Server
cd blogger_server
dart bin/main.dart

# 4. Launch Flutter Studio & Web Reader App
cd ../blogger_flutter
flutter run -d chrome
```

---

## 📄 License & Attribution

Built with reference to:
- [Serverpod Framework](https://serverpod.dev)
- [Schema.org Specification](https://schema.org)
- [JSON-LD 1.1 Specification](https://www.w3.org/TR/json-ld11/)
- Google Blogger API v3 REST Specification
