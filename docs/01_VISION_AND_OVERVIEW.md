# Document 01: Vision & Architectural Core

## 🎯 1. Executive Summary & Vision

Google Blogger is one of the most resilient and customizable CMS platforms on the web. It provides authors with simple subdomain management, custom domain support, gadget-based layout structures, and straightforward content rendering.

However, as mobile app publishing and modern digital commerce have evolved, Google Blogger presents severe limitations for developers trying to turn a blog platform into a full-featured iOS/Android app or multivendor digital engine:

```
+-----------------------------------------------------------------------------------+
|                            TRADITIONAL BLOGGER LIMITATIONS                        |
+-----------------------------------------------------------------------------------+
| 1. Restricted Sign-In Methods   : Only Google Sign-In supported.                  |
| 2. App Store Rejection Risk     : Violates Apple App Store Guideline 4.8.         |
| 3. Unstructured Content Storage : Stores plain HTML/escaped strings.              |
| 4. Missing User Management      : No native customer/subscriber profiles or roles.|
| 5. Missing Commerce & Services  : Lacks native product/service catalogs & cart.   |
| 6. Missing Geo & Notifications  : No GPS reverse-geocoding or push engine.        |
+-----------------------------------------------------------------------------------+
```

Our platform addresses every single one of these limitations by constructing a **Serverpod-powered backend (`^4.1.0-beta.1`)** and a **Flutter cross-platform frontend** that maintains Blogger's architectural strengths while elevating its data layer to **Type-Safe Schema.org JSON-LD Linked Data**.

---

## 📱 2. Solving Apple App Store Publishing Compliance

### The Problem: Apple App Store Guideline 4.8
Apple's App Store Review Guidelines mandate:
> *Guideline 4.8 - Sign in with Apple*: Apps that use a third-party or social login service (such as Google Sign-In, Facebook Login, or X Login) to set up or authenticate the user's account must also offer Sign in with Apple as an equivalent option.

Because original Blogger API endpoints only support Google OAuth / Google Account identity, any mobile application built directly on top of legacy Blogger is immediately rejected when submitted to the Apple App Store.

### The Solution: Serverpod Multi-Provider Identity Engine
By bringing the backend in-house with Serverpod, we decouple identity from Google's proprietary auth servers while retaining complete compatibility. The server exposes a unified identity endpoint supporting:
1. **Apple Sign-In** (Native iOS AuthorizationCode / IdentityToken exchange).
2. **Google Sign-In** (Google OAuth idToken verification).
3. **Email / Password IDP** (JWT access token and refresh token management).
4. **Custom Social / OAuth2 Providers**.

```
                         +-----------------------------------+
                         |         FLUTTER APP / WEB         |
                         +-----------------+-----------------+
                                           |
                                  Selects Auth Method
                                           |
                +--------------------------+--------------------------+
                |                          |                          |
                v                          v                          v
      +------------------+       +------------------+       +------------------+
      |  Apple Sign-In   |       |  Google Sign-In  |       |  Email / Password|
      |  (Native Creds)  |       |   (OAuth Token)  |       |    (User Credentials)|
      +--------+---------+       +--------+---------+       +--------+---------+
               |                          |                          |
               +--------------------------+--------------------------+
                                          |
                                          v
                         +-----------------------------------+
                         |    SERVERPOD IDENTITY ENDPOINT    |
                         |   (Validates Provider Signature)  |
                         +-----------------+-----------------+
                                           |
                                           v
                         +-----------------------------------+
                         |      UNIFIED USER PROFILE &       |
                         |    JWT SESSION TOKEN ISSUANCE     |
                         +-----------------------------------+
```

---

## 🌐 3. Unified Blog Route Architecture & `@type` Disambiguation

A key design innovation of this platform is **URL simplicity and route unification**.

Instead of creating disparate, confusing URL namespaces (such as `/vendor/:vendorId/catalog`, `/service/:serviceId/booking`, `/shop/item/:id`), **all content entities reside within standard blog routes**:

- **Blog Home**: `/blog/:blogId`
- **Posts (Products, Articles, Services, Events, Reviews)**: `/blog/:blogId/post/:postId`
- **Pages (About, Store Catalog, Service Directory, Terms)**: `/blog/:blogId/page/:pageId`

### Dynamic `@type` Inspection Engine
When a user navigates to `/blog/:blogId/post/:postId`, the Flutter client fetches the type-safe Schema.org JSON-LD payload stored in Serverpod. The frontend renderer inspects `@type`:

```
                                  GET /blog/:blogId/post/:postId
                                                |
                                                v
                                  Fetch Post JSON-LD Payload
                                                |
                                                v
                                Inspect Schema.org "@type" Field
                                                |
            +-----------------------+-----------+-----------+-----------------------+
            |                       |                       |                       |
            v                       v                       v                       v
   @type: "BlogPosting"      @type: "Product"        @type: "Service"      @type: "LocalBusiness"
            |                       |                       |                       |
            v                       v                       v                       v
    Render Article View     Render Product Card     Render Service Catalog  Render Interactive Map
    - Headline & Author     - Price & Availability  - Service Packages      - Phone & Geolocation
    - Reading Time          - Variant Picker        - Offer Catalog         - Directions Button
    - Comments Section      - Add to Cart Button    - Book Appointment      - Operating Hours
```

This single-route architecture guarantees that the platform remains 100% compatible with Google Blogger's URL routing standard while supporting infinite e-commerce and service provider capabilities.

---

## 🔗 4. Linked Data `@id` Cross-Referencing & Manual Overrides

In multi-vendor marketplaces or multi-site publishing, entity data (such as an Organization, Brand, Seller, or Physical Location) is often shared across dozens of blog posts or product listings.

### `@id` Cross-Referencing
Instead of duplicating the full JSON object in every blog post, entities are assigned a canonical URI string via `@id` (e.g. `"@id": "https://example.com/org/vendor-123"`).

### Manual Property Overrides
When referencing an existing node via `@id`, authors can specify local overrides directly inside the post's JSON-LD payload:

```json
{
  "@context": "https://schema.org",
  "@type": "Product",
  "name": "Custom Leather Jacket",
  "brand": {
    "@id": "https://example.com/brands/artisan-crafts",
    "telephone": "+1-800-OVERRIDE-PHONE"
  },
  "offers": {
    "@type": "Offer",
    "price": "199.99",
    "priceCurrency": "USD",
    "availability": "https://schema.org/InStock"
  }
}
```
*The `brand` node dynamically merges the master entity identified by `@id` with the local property override (`telephone`), giving vendors complete flexibility without data redundancy.*

---

## 🚀 5. Core Platform Capabilities Matrix

| Feature | Legacy Blogger | Our Serverpod + Flutter Platform |
| :--- | :--- | :--- |
| **Authentication** | Google Sign-In Only | Apple Sign-In, Google, Email JWT, Custom OAuth |
| **Data Format** | Unstructured HTML String | Type-Safe Schema.org JSON-LD (`schemaorg-current-https.jsonld`) |
| **Route Hierarchy** | `/blog/:blogId/post/:postId` | `/blog/:blogId/post/:postId` with `@type` Disambiguation |
| **User Roles** | Author, Admin | Super Admin, Vendor, Service Provider, Author, Reader |
| **Commerce Support** | None (Third-party widgets) | Native `Product`, `ProductGroup`, `OfferCatalog`, Cart & Checkout |
| **Geolocation** | None | Real-time GPS, Nominatim Reverse Geocoding, PIN code lookup |
| **Notifications** | Email Comments Only | Firebase/Web Push Notifications, Email Subscriber Lists |
| **Analytics & Geo** | Basic Pageviews | Geolocation logs, referrers, unique visitors, AdSense revenue |
