# Document 03: Schema.org Linked Data Engine

## 🌐 1. Universal Linked Data Architecture

The central pillar of our platform is the **Universal Type-Safe Schema.org JSON-LD Engine**.

Rather than storing unstructured text or proprietary HTML markup in database text columns, **every blog post, page, product listing, and service offering is stored strictly as a valid, type-safe `Schema.org JSON-LD` graph payload** (`jsonLdPayload`).

```
+-----------------------------------------------------------------------------------+
|                   TYPE-SAFE SCHEMA.ORG JSON-LD PAYLOAD ENGINE                     |
+-----------------------------------------------------------------------------------+
|  Target Specification  : https://schema.org/version/latest/schemaorg-current-https.jsonld
|  Supported Types       : 100% of Schema.org Vocabularies (800+ Classes)           |
|  Core Storage Column   : `BlogPost.jsonldPayload` (JSONB / Raw String in Postgres)|
|  Automated Extraction  : Extract `@type` -> `schemaType`, `headline` -> `title`   |
|  Graph Linked Data     : `@id` URI Cross-Referencing & Local Property Overrides  |
|  Search Engine SEO     : Web DOM Injection of `<script type="application/ld+json">`
+-----------------------------------------------------------------------------------+
```

---

## 📄 2. Supporting 100% Schema.org Vocabularies

Because Serverpod stores the JSON-LD payload as a flexible, type-validated JSON structure, the system natively supports all standard Schema.org types without schema migrations:

### Supported Schema Types & Features Matrix

| Schema.org `@type` | Primary Use Case | Key Rendered UI Elements |
| :--- | :--- | :--- |
| **`BlogPosting` / `Article`** | Editorial blog posts & news | Article header, author byline, reading time, markdown body, comments. |
| **`Product` / `ProductGroup`** | E-commerce items & variants | Price badge, currency, availability, image gallery, variant selector, Add-to-Cart. |
| **`Service` / `OfferCatalog`** | Service providers & packages | Package comparison table, service area (`areaServed`), booking call-to-action. |
| **`LocalBusiness` / `Store`** | Vendor storefronts & places | Interactive map (`GeoCoordinates`), address, phone number, opening hours. |
| **`JobPosting`** | Career portals & recruiting | Job title, hiring organization, base salary, employment type, Apply button. |
| **`Review` / `Rating`** | Product/service customer feedback | Star rating display, reviewer name, itemReviewed link, pros/cons list. |
| **`Event`** | Workshops & webinars | Event date/time picker, ticket link, location, performer list. |

---

## 🔗 3. Graph Linking via `@id` URIs & Manual Property Overrides

### A. Graph Disambiguation with `@id` URIs
In complex digital platforms, entities like Organizations, Stores, Brands, or Authors are shared across multiple posts.

Instead of re-declaring the organization details inside every single post, posts assign a unique canonical `@id` URI string (e.g. `"@id": "https://example.com/org/artisan-store"`).

### B. Example 1: Pure Node Linking
A product post references a vendor store by `@id`:

```json
{
  "@context": "https://schema.org",
  "@type": "Product",
  "@id": "https://example.com/blog/102/post/product-888",
  "name": "Handmade Espresso Cup",
  "brand": {
    "@id": "https://example.com/org/artisan-store"
  },
  "offers": {
    "@type": "Offer",
    "price": "24.50",
    "priceCurrency": "USD",
    "availability": "https://schema.org/InStock"
  }
}
```

### C. Example 2: Node Linking with Manual Property Overrides
If an author wants to override or extend a specific property on the linked node (for instance, providing a localized phone number or custom title for this specific post), they supply the target `@id` along with the override properties:

```json
{
  "@context": "https://schema.org",
  "@type": "Service",
  "@id": "https://example.com/blog/102/post/service-777",
  "name": "Home Plumbing Repair",
  "provider": {
    "@id": "https://example.com/org/plumbing-co",
    "telephone": "+1-555-LOCAL-LINE",
    "areaServed": {
      "@type": "AdministrativeArea",
      "name": "Downtown Metro District"
    }
  },
  "hasOfferCatalog": {
    "@type": "OfferCatalog",
    "name": "Plumbing Maintenance Packages",
    "itemListElement": [
      {
        "@type": "Offer",
        "name": "Standard Inspection",
        "price": "49.00",
        "priceCurrency": "USD"
      }
    ]
  }
}
```
*When the client parser resolves the entity, it fetches the root node for `@id` (`https://example.com/org/plumbing-co`) and merges the local property overrides (`telephone` and `areaServed`) on top.*

---

## 🤖 4. Automated Extraction & Serverpod Ingestion Webhook

When external tools or authors POST raw JSON-LD to Serverpod endpoints, Serverpod's `SchemaIngestionService` automatically extracts key metadata fields into indexed relational table columns:

```
[ Incoming JSON-LD Payload ]
            |
            v
+-------------------------------------------------------+
|            SERVERPOD SCHEMA INGESTION SERVICE          |
+-------------------------------------------------------+
|  1. Parse `@type`       --> Extract to `schemaType`   |
|  2. Parse `headline`    --> Extract to `title`        |
|  3. Parse `keywords`    --> Extract to `labels` list  |
|  4. Parse `dateCreated` --> Extract to `publishedDate`|
+-------------------------------------------------------+
            |
            v
[ Saved to Postgres `BlogPost` Row with Fast Indexed Columns ]
```

---

## 🔍 5. Web DOM SEO Head Injection for Search Engines

For Web clients running on Flutter Web or Serverpod Web Routes, search engines like Googlebot need direct access to structured data.

During Web rendering, the platform injects the post's JSON-LD directly into the HTML document `<head>`:

```html
<!DOCTYPE html>
<html>
<head>
  <title>Handmade Espresso Cup - Artisan Store</title>

  <!-- Universal Schema.org JSON-LD Injection -->
  <script type="application/ld+json">
  {
    "@context": "https://schema.org",
    "@type": "Product",
    "name": "Handmade Espresso Cup",
    "image": "https://example.com/media/cup.jpg",
    "offers": {
      "@type": "Offer",
      "price": "24.50",
      "priceCurrency": "USD",
      "availability": "https://schema.org/InStock"
    }
  }
  </script>
</head>
<body>
  <!-- Flutter App Web Canvas -->
</body>
</html>
```
This guarantees 100% eligibility for **Google Rich Results**, Google Shopping indexing, and rich snippet displays in search engines.
