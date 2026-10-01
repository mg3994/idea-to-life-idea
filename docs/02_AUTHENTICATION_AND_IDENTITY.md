# Document 02: Authentication & Identity Engine

## 🔑 1. Multi-Provider Identity Framework Architecture

To publish successfully across all platforms—specifically Apple App Store (iOS, iPadOS, macOS), Google Play Store (Android), and Web—the platform replaces legacy Blogger's Google-only login with a comprehensive **Serverpod Multi-Provider Identity Engine**.

```
+-----------------------------------------------------------------------------------+
|                        SERVERPOD MULTI-PROVIDER IDENTITY ENGINE                   |
+-----------------------------------------------------------------------------------+
|  1. Apple Sign-In       : Authorization Code / Identity Token validation (iOS)    |
|  2. Google OAuth        : ID Token verification via Google Auth API (Android/Web) |
|  3. Email IDP           : Native Username/Password JWT Endpoint (`EmailIdpEndpoint`)|
|  4. JWT Refresh         : Token refresh & rotation endpoint (`JwtRefreshEndpoint`)|
|  5. Session State       : Bearer Token Auth with Serverpod Session Key Storage    |
+-----------------------------------------------------------------------------------+
```

---

## 🍏 2. Apple Sign-In Integration (App Store Guideline 4.8 Compliance)

To satisfy Apple App Store Guideline 4.8, Apple Sign-In is built directly into Serverpod as a first-class authentication method.

### Handshake & Validation Flow
1. **Frontend Initiation**: The Flutter client calls `SignInWithApple.getAppleIDCredential()` on iOS/macOS.
2. **Credential Capture**: The client receives `identityToken`, `authorizationCode`, `givenName`, and `familyName`.
3. **Serverpod Endpoint Transmission**: The client sends the credential bundle to Serverpod's `AppleAuthEndpoint`.
4. **Token Verification**: Serverpod fetches Apple's public JWKS (`https://appleid.apple.com/auth/keys`) and verifies the `identityToken` signature, `iss` (`https://appleid.apple.com`), and `aud` (App Bundle Identifier).
5. **Account Linking**: Serverpod checks if an existing `UserProfile` matches the Apple Subject ID (`sub`) or email address. If none exists, a new `UserProfile` is provisioned.
6. **Session Key Issuance**: Serverpod returns an authenticated `SessionKey` and JWT pair to the Flutter client.

```
FLUTTER CLIENT                       SERVERPOD BACKEND                  APPLE AUTH SERVER
     |                                      |                                  |
     |--- 1. Request Apple Credentials ---->|                                  |
     |<-- 2. Returns Token & AuthCode ------|                                  |
     |                                      |                                  |
     |--- 3. Call `appleSignIn(identityToken, authCode)` --------------------->|
     |                                      |                                  |
     |                                      |--- 4. Fetch JWKS Public Keys --->|
     |                                      |<-- 5. Return Public Keys --------|
     |                                      |                                  |
     |                                      |-- 6. Verify Claims & Signature   |
     |                                      |-- 7. Upsert UserProfile in Postgres
     |                                      |                                  |
     |<-- 8. Return Serverpod SessionKey ---|                                  |
```

---

## 📧 3. Native JWT Email Provider (`EmailIdpEndpoint`) & Token Refresh (`JwtRefreshEndpoint`)

For users who do not use social logins, Serverpod implements native Email identity provider endpoints: `EmailIdpEndpoint` and `JwtRefreshEndpoint`.

### Protocol Endpoints Specification

#### A. Email Authentication Endpoint (`EmailIdpEndpoint`)
- **`authenticate(Session session, String email, String password)`**
  - Hash checking using Argon2id / Bcrypt password hashes.
  - Returns `AuthenticationResponse` containing `user`, `keyId`, `key`, and `refreshToken`.
- **`register(Session session, String email, String password, String userName)`**
  - Provisions user account, sends verification email/code, and sets initial role to `Reader`.

#### B. JWT Token Refresh Endpoint (`JwtRefreshEndpoint`)
- **`refreshToken(Session session, String refreshToken)`**
  - Validates the active refresh token string against stored hash in PostgreSQL.
  - Generates a new short-lived JWT access token (15-minute expiration) and rotates the refresh token (7-day sliding window).

```json
{
  "status": "success",
  "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refreshToken": "rf_7a9f8b2c4d6e1f0a...",
  "userInfo": {
    "id": 1024,
    "email": "author@example.com",
    "userName": "ArtisanVendor",
    "role": "Vendor"
  }
}
```

---

## 👥 4. Role-Based Access Control (RBAC) System

The platform manages access privileges through a 5-tier Role-Based Access Control (RBAC) hierarchy defined on the `UserProfile` entity:

```
[ Tier 1: Super Admin ]  --> Platform management, server settings, custom redirects
       |
[ Tier 2: Vendor / Store Owner ]  --> Multi-vendor shop, product catalog, earnings
       |
[ Tier 3: Author / Service Provider ]  --> Blog posts, pages, service bookings
       |
[ Tier 4: Private Reader / Member ]  --> Private blog posts, restricted subscriber content
       |
[ Tier 5: Anonymous Reader ]  --> Public blog view, product browsing, public comments
```

### Role Privilege Matrix

| Operation | Anonymous | Reader | Author / Service Provider | Vendor / Store Owner | Super Admin |
| :--- | :---: | :---: | :---: | :---: | :---: |
| **Read Public Blog Posts / Products** | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Post Comments** | ❌ | ✅ | ✅ | ✅ | ✅ |
| **Access Private / Subscriber Blogs** | ❌ | ✅ (If Member) | ✅ | ✅ | ✅ |
| **Create Blog Posts / Pages** | ❌ | ❌ | ✅ | ✅ | ✅ |
| **Manage Multi-Vendor Catalog** | ❌ | ❌ | ❌ | ✅ | ✅ |
| **View Vendor Earnings & Orders** | ❌ | ❌ | ❌ | ✅ | ✅ |
| **Configure System Redirects & Themes** | ❌ | ❌ | ❌ | ❌ | ✅ |
| **Manage Global Users & Roles** | ❌ | ❌ | ❌ | ❌ | ✅ |

---

## 🔒 5. Bearer Token Request Flow in Serverpod

When the Flutter client issues requests to protected Serverpod endpoints (e.g. `BloggerEndpoint.createPost` or `OrderEndpoint.createOrder`), it includes the session authentication header:

```http
POST /blogger/v3/blogs/102/posts HTTP/1.1
Host: api.example.com
Authorization: Bearer <SERVERPOD_KEY_ID>:<SERVERPOD_KEY_SECRET>
Content-Type: application/json
```

Serverpod's `session.authenticated` handler automatically validates the key against PostgreSQL, attaching the `AuthenticationInfo` object (including `userId` and scopes/roles) to the execution context.
