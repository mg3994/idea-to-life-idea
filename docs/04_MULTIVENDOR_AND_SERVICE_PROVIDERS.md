# Document 04: Multivendor, Service Providers, Geolocation & Earnings

## 🛒 1. Unified Multi-Vendor E-Commerce & Service Provider Architecture

By leveraging **Type-Safe Schema.org JSON-LD Linked Data**, multi-vendor e-commerce stores and service provider marketplaces operate seamlessly directly inside standard blog posts (`/blog/:blogId/post/:postId`) and static pages (`/blog/:blogId/page/:pageId`).

```
+-----------------------------------------------------------------------------------+
|               MULTIVENDOR COMMERCE & SERVICES IN UNIFIED BLOG ROUTES              |
+-----------------------------------------------------------------------------------+
|  1. Product Listings   : `@type: Product` with variant selection (`ProductGroup`). |
|  2. Service Providers  : `@type: Service` with package catalog (`OfferCatalog`).  |
|  3. Storefront Pages   : `@type: LocalBusiness` with GPS geo coordinates.          |
|  4. Order Fulfillment  : Schema.org `Order` and `OrderItem` models in Serverpod.  |
|  5. Cart & Checkout    : Client-side `CartManager` with Google Pay integration.   |
+-----------------------------------------------------------------------------------+
```

---

## 📍 2. Real-Time Geolocation & Reverse Geocoding Engine (`LocationManager`)

For local multi-vendor stores and on-demand service providers, knowing the user's exact geographical location enables localized catalog filtering, store directions, and PIN code delivery checks.

### The `LocationManager` Protocol
The frontend implements a dedicated `LocationManager` service using Nominatim OpenStreetMap reverse geocoding:

```typescript
export interface LocationData {
  lat: number | null;
  lon: number | null;
  pin: string | null;
  city: string | null;
}

export class LocationManager {
  private data: LocationData = { lat: null, lon: null, pin: null, city: null };
  private storageKey = "blogger_user_location";

  // Reverse Geocode GPS Lat/Lon to PIN & City
  async reverseGeocode(lat: number, lon: number): Promise<Partial<LocationData>> {
    const res = await fetch(`https://nominatim.openstreetmap.org/reverse?lat=${lat}&lon=${lon}&format=json`);
    const d = await res.json();
    if (d.address) {
      return {
        pin: d.address.postcode,
        city: d.address.city || d.address.town || d.address.village
      };
    }
    return {};
  }

  // Lookup Location by Postal PIN Code
  async lookupPin(pin: string): Promise<Partial<LocationData>> {
    const res = await fetch(`https://nominatim.openstreetmap.org/search?postalcode=${pin}&format=json`);
    const d = await res.json();
    if (d && d.length > 0) {
      return {
        lat: parseFloat(d[0].lat),
        lon: parseFloat(d[0].lon),
        city: d[0].address.city || d[0].address.town
      };
    }
    return {};
  }
}
```

---

## 🛍️ 3. Cart Lifecycle & Variant Selection (`CartManager`)

When viewing a product post containing Schema.org variants (`ProductGroup.hasVariant`), the frontend client initializes `CartManager` to process item selections:

```
[ User Views Post /blog/102/post/501 ]
                  |
                  v
       Extract Schema.org JSON-LD
                  |
                  v
    @type == "Product" / "Service"
                  |
    +-------------+-------------+
    |                           |
[ Product Variant Selection ] [ Service Package Selection ]
    |                           |
    +-------------+-------------+
                  |
                  v
    Add to Client `CartManager`
                  |
                  v
  Construct Schema.org `Order` Payload
                  |
                  v
 Submit Order to Serverpod `OrderEndpoint`
```

### Schema.org `Order` Representation
```json
{
  "@context": "https://schema.org",
  "@type": "Order",
  "orderNumber": "ORD-2026-9901",
  "orderStatus": "https://schema.org/OrderProcessing",
  "priceCurrency": "USD",
  "totalPrice": "149.50",
  "seller": {
    "@type": "Organization",
    "name": "Artisan Vendor Shop",
    "@id": "https://example.com/org/vendor-102"
  },
  "orderedItem": [
    {
      "@type": "OrderItem",
      "orderQuantity": 2,
      "orderedItem": {
        "@type": "Product",
        "name": "Handmade Ceramic Vase",
        "sku": "VASE-CER-01"
      }
    }
  ]
}
```

---

## 🔔 4. Notification Dispatch Engine

The platform includes a real-time Notification Dispatch Engine that alerts users and vendors across three primary channels:

```
+-----------------------------------------------------------------------------------+
|                           NOTIFICATION DISPATCH ENGINE                            |
+-----------------------------------------------------------------------------------+
| 1. Mobile Push (FCM)   : Immediate alert on order confirmation or post publish.  |
| 2. Web Push (VAPID)    : Browser notifications for desktop subscribers.           |
| 3. Email Subscribers   : Automated newsletter updates via `EmailSubscriber` table.|
+-----------------------------------------------------------------------------------+
```

When a blog author publishes a new post or a vendor receives an order, Serverpod triggers the `NotificationEndpoint`:

```dart
Future<void> sendPostNotification(Session session, int blogId, String postTitle) async {
  final subscribers = await EmailSubscriber.db.find(
    session,
    where: (t) => t.blogId.equals(blogId) & t.isVerified.equals(true),
  );

  for (final sub in subscribers) {
    await EmailService.sendNewPostAlert(sub.email, postTitle);
  }
}
```

---

## 💰 5. Monetization & Earnings Management (`BlogEarning`)

For creators, vendors, and multi-blog network operators, tracking financial statistics is vital.

The platform tracks Google AdSense publisher accounts, estimated ad revenue, e-commerce commissions, and traffic impression metrics via the `BlogEarning` and `BlogStat` models:

```
+-----------------------------------------------------------------------------------+
|                        EARNINGS & ANALYTICS MONITORING LAYER                      |
+-----------------------------------------------------------------------------------+
|  1. Google AdSense ID  : Publisher Account binding (`pub-XXXXXXXXXXXXXXXX`).      |
|  2. Estimated Revenue  : Daily, monthly, and yearly earnings metrics.             |
|  3. Ad Impressions     : Total ad impressions and click-through rates (CTR).      |
|  4. Vendor Commissions : Multi-vendor split payouts for products sold.            |
|  5. Geographic Metrics : Revenue breakdown by country/region from location logs.  |
+-----------------------------------------------------------------------------------+
```

### Relational Earnings Schema
- `publisherId`: Associated Google AdSense ID or merchant payout account.
- `estimatedEarnings`: Decimal monetary value.
- `impressions`: Total ad & product impressions.
- `clicks`: Total user engagement actions.
- `earningsPeriod`: Timestamp grouping for financial reporting.
