# Document 08: UI Renderers & Presentation Engine Specification

## 🎨 1. Presentation Layer Architecture

The frontend presentation layer handles the visual transformation of raw **Schema.org JSON-LD data** into reactive, interactive user interface components.

```
+-----------------------------------------------------------------------------------+
|                        FRONTEND PRESENTATION RENDERERS                            |
+-----------------------------------------------------------------------------------+
| 1. ProductRenderer     : Variant selector, availability state, specifications table.|
| 2. CartRenderer        : Floating action button (FAB), cart drawer modal, items.  |
| 3. OrderSummaryRenderer: Price breakdown, delivery address, Google Pay button.   |
| 4. GeoVerificationRenderer: Google Maps pin drop, address typeahead, distance.    |
| 5. PhoneVerificationRenderer: Phone OTP authentication modal for order checkout. |
+-----------------------------------------------------------------------------------+
```

---

## 🛍️ 2. Product & Variant Renderer (`ProductRenderer`)

The `ProductRenderer` dynamically parses Schema.org `Product`, `ProductGroup`, and `Service` objects to build interactive e-commerce product views:

### Key Functions
- **`renderProduct(Product product)`**: Sets product headline, description, image gallery, price badge, and seller details (`Organization`).
- **`renderVariants(ProductGroup group)`**: Renders attribute selection buttons (e.g. Size, Color, Material). When a user taps an attribute option:
  1. Computes matching variant using `SchemaExtractor.findMatchingVariant()`.
  2. Updates active price display and stock status (`InStock` vs `OutOfStock`).
  3. Grays out unavailable attribute combinations.
- **`renderSeller(Organization seller)`**: Renders seller brand box with phone (`telephone`), email (`email`), address (`PostalAddress`), and Google Maps directions link (`hasMap` / `GeoCoordinates`).

---

## 🛒 3. Cart Drawer & Floating Button (`CartRenderer`)

The `CartRenderer` maintains the floating cart FAB button and the slide-over cart drawer drawer:

```typescript
export class CartRenderer {
  // Update floating action button badge counter
  updateUI(): void {
    const count = this.cartManager.getTotalQuantity();
    const fab = UIManager.el("cart-fab");
    if (fab) {
      const countEl = fab.querySelector(".cart-count");
      if (countEl) countEl.textContent = String(count);
      fab.style.transform = count > 0 ? "scale(1)" : "scale(0)";
    }
  }

  // Render slide-over cart modal drawer with quantity selectors
  showModal(): void {
    const order = this.cartManager.getOrder();
    // Maps item list to DOM elements with stock validation
  }
}
```

---

## 🗺️ 4. Interactive Geolocation Verification (`GeoVerificationRenderer`)

For physical deliveries and localized services, `GeoVerificationRenderer` provides a pinpoint delivery location map interface powered by Google Maps JS API and OpenStreetMap Nominatim:

```
[ User Clicks 'Verify Delivery Location' ]
                    |
                    v
    Renders `GeoVerificationRenderer` Popup
                    |
    +---------------+---------------+
    |                               |
[ Typeahead Search Input ]   [ Interactive Map Pin Drop ]
    |                               |
    +---------------+---------------+
                    |
                    v
    Calculate Telemetry & Route
    (Target Lat/Lon, Distance, Estimated Duration)
                    |
                    v
    Save Verified Location to Order State
```

### Telemetry & Route Verification Features
- **Draggable Target Marker**: Users can drag the delivery marker on the map or click anywhere to reposition the pinpoint.
- **Address Typeahead**: Debounced search input querying place suggestion endpoints (`getPlaceSuggestions`).
- **Telemetry Display**: Displays exact GPS coordinates, clean postal address, distance from origin, and estimated transit duration.

---

## 💳 5. Google Pay Integration & Order Summary (`OrderSummaryRenderer`)

The `OrderSummaryRenderer` formats the final order review before triggering payment execution:

```typescript
export class OrderSummaryRenderer {
  renderGooglePayButton(order: Order, verifiedLocation: any): void {
    const btn = document.createElement('button');
    btn.className = "gpay-button dark";
    btn.onclick = () => {
      GooglePayService.initPayment(order, verifiedLocation);
    };
    container.appendChild(btn);
  }
}
```
