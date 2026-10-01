# Document 07: Visual Editor & Ingestion Pipeline Specification

## 🛠️ 1. No-Code Visual Schema.org Editor Architecture (`JSON-LD-Visual-Editor`)

To empower authors, store managers, and service providers who do not write raw JSON by hand, the platform integrates an intelligent **No-Code Visual Schema.org Builder** (inspired by `JSON-LD-Visual-Editor`).

```
+-----------------------------------------------------------------------------------+
|                     NO-CODE VISUAL SCHEMA.ORG BUILDER ENGINE                      |
+-----------------------------------------------------------------------------------+
| 1. Three-Pane Layout   : Active Markups list, Workspace Form, Live JSON-LD Output.|
| 2. 800+ Schema Classes : Searchable class library covering full Schema.org spec.  |
| 3. Recursive Form UI   : Modular field components handling infinite nestings.     |
| 4. Bi-directional Flow : Import existing JSON-LD -> Visual Edit -> Re-Export.     |
| 5. Real-Time Compile   : Zero-lag reactive compilation on every field change.    |
+-----------------------------------------------------------------------------------+
```

### A. Responsive Architecture
- **Desktop Layout**: Three-pane command center (Active Markups on left, Form Workspace in middle, Live JSON-LD Output code editor on right).
- **Mobile Layout**: Responsive 3-tab workspace (Markups, Form, Code Output) with thumb-friendly controls.

### B. Recursive Structural Component Tree
To handle complex linked data without layout shifts (e.g., a `Product` containing an `Organization` brand located at a `PostalAddress` with `GeoCoordinates`), form input fields are implemented as isolated, recursive components that maintain clean state isolation.

---

## 📡 2. Feed Extraction & Entity Ingestion Pipeline (`SchemaExtractor`)

For migrating legacy Blogger content or importing external feeds (`/feeds/posts/default`), the system utilizes a high-velocity extraction pipeline (from `My-Engine-time-to--time-backup`):

```
                       [ Blogger Feed / HTML Content ]
                                      |
                                      v
                       BloggerDataService.fetchFeedData()
                                      |
                                      v
                  +---------------------------------------+
                  |         SchemaExtractor.ts            |
                  +---------------------------------------+
                  |  1. Regex Script Match                |
                  |     `<script type="app/ld+json">`     |
                  |  2. HTML Entity Decoding (`&quot;`)   |
                  |  3. Clean Comments & Trailing Whitespace|
                  |  4. JSON.parse() to Typed Structure   |
                  +-------------------+-------------------+
                                      |
                                      v
          +---------------------------+---------------------------+
          |                           |                           |
          v                           v                           v
  Extract Products            Extract Services            Extract Places/Geo
  (`findMatchingVariant`)     (`findMatchingServicePkg`)  (`LocationManager`)
```

### Protocol Implementation Specification

```typescript
export class SchemaExtractor {
  // Decode HTML entities safely from Blogger post feeds
  static decodeEntities(text: string): string {
    if (!text) return "";
    const textarea = document.createElement("textarea");
    textarea.innerHTML = text;
    let decoded = textarea.value;
    if (decoded.includes("&quot;")) {
      textarea.innerHTML = decoded;
      decoded = textarea.value;
    }
    return decoded;
  }

  // Extract JSON-LD payload from raw HTML strings or script tags
  static extractJsonLd<T>(input: string): T | null {
    if (!input) return null;
    try {
      const scriptMatch = input.match(/<script[^>]*type=["']application\/ld\+json["'][^>]*>([\s\S]*?)<\/script>/i);
      let jsonContent = scriptMatch ? scriptMatch[1] : input;
      let decoded = this.decodeEntities(jsonContent);

      const cleaned = decoded
        .replace(/\/\*[\s\S]*?\*\/|([^\\:]|^)\/\/.*$/gm, "$1")
        .trim();

      return JSON.parse(cleaned) as T;
    } catch (e) {
      return null;
    }
  }

  // Find exact product variant match based on selected attributes
  static findMatchingVariant(parent: any, selectedAttributes: Record<string, string>): any {
    if (!parent) return null;
    const variants = parent.hasVariant || [parent];
    return variants.find((v: any) =>
      Object.entries(selectedAttributes).every(([k, val]) => String(v[k]) === String(val))
    ) || variants[0];
  }
}
```

---

## 🧪 3. Rich Results Validation & URL Scraping Pipeline

```
+-----------------------------------------------------------------------------------+
|                  GOOGLE RICH RESULTS & URL SCRAPING PIPELINE                      |
+-----------------------------------------------------------------------------------+
|  1. Live URL Scraping    : User enters website URL -> Scrapes JSON-LD components. |
|  2. Visual Import        : Populates Visual Builder form fields automatically.    |
|  3. Automated Validation : Queries Google Rich Results Test API to detect errors. |
|  4. Error Resolution     : Highlights invalid fields directly in the form UI.     |
|  5. Direct Export        : One-click export of corrected, validated JSON-LD.      |
+-----------------------------------------------------------------------------------+
```

### URL Scraping Flow
1. User enters any existing public web URL (or Blogger post URL).
2. Serverpod scraper fetches the HTML markup and invokes `SchemaExtractor.extractJsonLd()`.
3. The extracted graph is sent to the Visual Builder, populating form fields.
4. Authors modify or fix broken properties visually.
5. The corrected JSON-LD payload is saved to Serverpod with 100% confidence.
