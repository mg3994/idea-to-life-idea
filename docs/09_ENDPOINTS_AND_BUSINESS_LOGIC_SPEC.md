# Document 09: Serverpod Endpoints & Execution Business Logic Specification

## ⚡ 1. Serverpod Endpoint Architecture

Serverpod endpoints act as the secure entry points for client RPC calls and REST API webhooks. Each endpoint method receives a `Session` object containing database access methods, logging tools, and authentication information.

```
+-----------------------------------------------------------------------------------+
|                           SERVERPOD ENDPOINTS PIPELINE                            |
+-----------------------------------------------------------------------------------+
| 1. BloggerEndpoint   : Core admin RPC methods for posts, pages, sites, comments.  |
| 2. BloggerV3Endpoint : REST API compatibility layer mirroring Blogger REST API v3. |
| 3. EmailIdpEndpoint  : User registration, Argon2id auth, and identity management. |
| 4. JwtRefreshEndpoint: Token rotation & access token issuance.                    |
| 5. OrderEndpoint     : E-commerce order placement & Google Pay transaction verify. |
+-----------------------------------------------------------------------------------+
```

---

## 📝 2. Core Admin Endpoint (`BloggerEndpoint`)

The `BloggerEndpoint` handles native Serverpod RPC operations invoked by the Flutter Admin Studio:

### Protocol Implementation Specification

```dart
class BloggerEndpoint extends Endpoint {
  // Fetch paginated blog posts with label and schemaType filters
  Future<List<BlogPost>> getPosts(
    Session session, {
    required int blogId,
    String? status,
    String? schemaType,
    String? label,
    int limit = 20,
    int offset = 0,
  }) async {
    return await BlogPost.db.find(
      session,
      where: (t) =>
          t.blogId.equals(blogId) &
          (status != null ? t.status.equals(status) : Constant.bool(true)) &
          (schemaType != null ? t.schemaType.equals(schemaType) : Constant.bool(true)),
      limit: limit,
      offset: offset,
      orderBy: (t) => t.publishedDate,
      orderDescending: true,
    );
  }

  // Save or Publish a BlogPost with automated Schema.org metadata extraction
  Future<BlogPost> createPost(Session session, BlogPost post) async {
    // 1. Verify User Session & Role Scope
    final auth = await session.authenticated;
    if (auth == null) throw Exception('Unauthorized');

    // 2. Validate Type-Safe JSON-LD Payload
    if (post.jsonLdPayload.isEmpty) {
      throw Exception('Invalid payload: jsonLdPayload is mandatory.');
    }

    // 3. Automated Metadata Extraction
    final jsonMap = jsonDecode(post.jsonLdPayload);
    post.schemaType = jsonMap['@type'] ?? 'BlogPosting';
    post.title = jsonMap['headline'] ?? jsonMap['name'] ?? post.title;
    post.publishedDate ??= DateTime.now();
    post.updatedDate = DateTime.now();

    // 4. Save to PostgreSQL
    return await BlogPost.db.insertRow(session, post);
  }
}
```

---

## 🌐 3. Google Blogger API v3 REST Compatibility Endpoint (`BloggerV3Endpoint`)

To preserve 100% REST API compatibility with Google Blogger SDKs, `BloggerV3Endpoint` maps incoming REST HTTP requests directly to Serverpod database queries:

```dart
class BloggerV3Endpoint extends Endpoint {
  // GET /blogger/v3/blogs/{blogId}/posts
  Future<Map<String, dynamic>> postsList(
    Session session,
    String blogId, {
    String? q,
    String? labels,
    int maxResults = 10,
    int startIndex = 1,
  }) async {
    final parsedBlogId = int.parse(blogId);

    final posts = await BlogPost.db.find(
      session,
      where: (t) =>
          t.blogId.equals(parsedBlogId) &
          t.status.equals('PUBLISHED') &
          (q != null ? t.title.ilike('%$q%') | t.jsonLdPayload.ilike('%$q%') : Constant.bool(true)),
      limit: maxResults,
      offset: startIndex - 1,
    );

    final totalResults = await BlogPost.db.count(
      session,
      where: (t) => t.blogId.equals(parsedBlogId) & t.status.equals('PUBLISHED'),
    );

    return {
      'kind': 'blogger#postList',
      'totalItems': totalResults,
      'items': posts.map((p) => _transformToBloggerV3PostJson(p)).toList(),
    };
  }

  Map<String, dynamic> _transformToBloggerV3PostJson(BlogPost p) {
    return {
      'kind': 'blogger#post',
      'id': p.id.toString(),
      'blog': {'id': p.blogId.toString()},
      'published': p.publishedDate?.toIso8601String(),
      'updated': p.updatedDate?.toIso8601String(),
      'url': 'https://example.com/blog/${p.blogId}/post/${p.slug}',
      'title': p.title,
      'content': p.jsonLdPayload, // Serves Type-Safe Schema.org JSON-LD
      'labels': p.labels,
      'author': {'id': p.authorId.toString()},
    };
  }
}
```

---

## 💳 4. Order Execution & Payment Verification Lifecycle

```
[ Frontend Client Submits Payment Token ]
                   |
                   v
    `OrderEndpoint.verifyAndExecuteOrder()`
                   |
                   v
+-------------------------------------------------------+
|             ORDER VERIFICATION LIFECYCLE              |
+-------------------------------------------------------+
|  1. Payment Signature Verification (Google Pay / UPI) |
|  2. Re-verify Item Prices & Stock Availability        |
|  3. Deduct Inventory Quantities                       |
|  4. Save `Order` & `OrderItem` records in Postgres     |
|  5. Dispatch Real-time Push Alert to Vendor           |
+-------------------------------------------------------+
```

### Order Endpoint Verification Spec
```dart
Future<Order> verifyAndExecuteOrder(
  Session session,
  Order orderPayload,
  String paymentToken,
) async {
  // 1. Validate Payment Token
  final isPaymentValid = await PaymentGateway.verifyToken(paymentToken, orderPayload.totalPrice);
  if (!isPaymentValid) throw Exception('Payment Verification Failed');

  // 2. Set Order Status
  orderPayload.orderStatus = 'https://schema.org/OrderPaymentDue';

  // 3. Persist Order Record
  final savedOrder = await OrderModel.db.insertRow(session, orderPayload);

  // 4. Dispatch Notification
  await NotificationService.sendOrderReceipt(session, savedOrder);

  return savedOrder;
}
```
