# Certification API Reference

The Wordmade Certification API is an AI verification service -- an **inverse CAPTCHA** that proves agents are AI, not human. It provides two API layers:

- **Inverse CAPTCHA endpoints** (public) -- challenge-response protocol for AI agent verification
- **Customer Portal API** (authenticated) -- multi-tenant management for site owners integrating the widget

This document covers all endpoints, request/response formats, and integration patterns. A developer should be able to integrate with this document alone.

## Base URL

```
https://certification.wordmade.world
```

---

## Authentication

### Inverse CAPTCHA Endpoints (`/v1/*`)

No authentication required. These endpoints are public so the widget can be embedded on arbitrary third-party websites.

When using multi-tenant siteverify, include your site's secret key in the request body (not a header).

### Customer Portal API (`/api/v1/*`)

Session-based authentication via Bearer token.

```
Authorization: Bearer <session_token>
```

The session token is obtained from the `/api/v1/customers/login` endpoint and is valid for 24 hours.

### Stripe Webhook (`/api/v1/webhooks/stripe`)

No Bearer token. Verified via Stripe's HMAC signature in the `Stripe-Signature` header.

---

## Rate Limiting

All API endpoints are rate limited per IP address using a sliding window algorithm. CORS preflight (`OPTIONS`) requests bypass rate limiting.

### Rate Limit Headers

Every response includes these headers:

| Header | Description |
|--------|-------------|
| `X-RateLimit-Limit` | Maximum requests allowed per window |
| `X-RateLimit-Remaining` | Remaining requests in current window |
| `Retry-After` | Seconds until the window resets (only on 429) |

### Default Limits

| Endpoint | Limit | Window |
|----------|-------|--------|
| `POST /v1/challenge/request` | 30 | 1 minute |
| `POST /v1/challenge/respond` | 60 | 1 minute |
| `GET /v1/verify/{cert}` | 120 | 1 minute |
| `POST /v1/siteverify` | 300 | 1 minute |
| `POST /api/v1/customers/register` | 5 | 1 minute |
| `POST /api/v1/customers/login` | 10 | 1 minute |
| `POST /api/v1/2fa/verify` | 5/customer + 20/IP | 15 minutes |
| All other portal endpoints | 120 | 1 minute |

### 429 Response

When rate limited, the API returns HTTP 429 with a JSON body:

```json
{
  "error": "rate limit exceeded",
  "error-codes": ["rate-limit-exceeded"]
}
```

---

## Inverse CAPTCHA Endpoints

These endpoints implement the challenge-response protocol for AI agent verification.

### POST /v1/challenge/request

Request a new challenge for the agent to solve.

**Rate limit:** 30 requests/minute per IP

#### Request

```http
POST /v1/challenge/request
Content-Type: application/json
```

```json
{
  "level": 2,
  "site_key": "sk_pub_a1b2c3d4e5f6",
  "action": "register",
  "user_hint": "user-12345"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `level` | integer | No | Challenge difficulty (1-5). Default: 1. Clamped to `[1, max_level]`. |
| `site_key` | string | No | Third-party site public key (max 256 chars). Used for multi-tenant tracking and per-site minimum level enforcement. |
| `action` | string | No | Action context, e.g. `"register"`, `"login"` (max 256 chars). Bound into the certificate signature. |
| `user_hint` | string | No | Third-party user identifier (max 512 chars). Bound into the certificate for context verification. |

**Level clamping behavior:**

- Level 0 or below is clamped UP to 1.
- Levels above the server's configured `max_level` are clamped DOWN.
- If the site has a `min_level` configured, the level is clamped UP to that minimum.
- The actual level served is returned in the response.

#### Response

**HTTP 200**

```json
{
  "challenge_id": "ch_a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4",
  "instruction": "eyJncmFwaCI6IjQ4NjU3ODY...",
  "nonce": "dGhpcyBpcyBhIHJhbmRvbSBub25jZQ",
  "level": 2,
  "expires_at": "2026-02-07T12:01:00Z",
  "signature": "a1b2c3d4e5f6...64hex",
  "site_key": "sk_pub_a1b2c3d4e5f6",
  "action": "register",
  "user_hint": "user-12345"
}
```

| Field | Type | Description |
|-------|------|-------------|
| `challenge_id` | string | Unique challenge identifier (format: `ch_` + 32 hex chars). |
| `instruction` | string | The challenge to solve. Often base64-encoded with multi-step decode requirements. |
| `nonce` | string | Cryptographic nonce (base64url). Must be echoed back in the response. |
| `level` | integer | Actual difficulty level served (may differ from requested due to clamping). |
| `expires_at` | string | ISO 8601 timestamp. Challenge is valid for 60 seconds. |
| `signature` | string | HMAC-SHA256 signature covering all fields including hints. Also sent in `X-Challenge-Signature` header. |
| `site_key` | string | Echoed from request (if provided). |
| `action` | string | Echoed from request (if provided). |
| `user_hint` | string | Echoed from request (if provided). |

#### Errors

| Status | Condition |
|--------|-----------|
| 400 | Invalid request body, field exceeds maximum length |
| 405 | Method not POST |
| 429 | Rate limit exceeded |
| 500 | Internal error generating challenge |

```json
{
  "error": "invalid request body"
}
```

---

### POST /v1/challenge/respond

Submit a response to a challenge. If the response is correct and fast enough, a certificate is issued.

**Rate limit:** 60 requests/minute per IP

#### Request

```http
POST /v1/challenge/respond
Content-Type: application/json
```

```json
{
  "challenge_id": "ch_a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4",
  "nonce": "dGhpcyBpcyBhIHJhbmRvbSBub25jZQ",
  "level": 2,
  "expires_at": "2026-02-07T12:01:00Z",
  "signature": "a1b2c3d4e5f6...64hex",
  "response": "42",
  "site_key": "sk_pub_a1b2c3d4e5f6",
  "action": "register",
  "user_hint": "user-12345"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `challenge_id` | string | Yes | From the challenge response. |
| `nonce` | string | Yes | From the challenge response. |
| `level` | integer | No | From the challenge response. |
| `expires_at` | string | No | From the challenge response (RFC 3339). |
| `signature` | string | Yes | From the challenge response. |
| `response` | string | Yes | The agent's answer to the challenge. |
| `site_key` | string | No | Must match the value from the challenge (verified via signature). |
| `action` | string | No | Must match the value from the challenge (verified via signature). |
| `user_hint` | string | No | Must match the value from the challenge (verified via signature). |

All fields from the challenge response should be echoed back. The signature binds all fields (including hints) together, so tampering with any field will cause verification failure.

#### Response

**HTTP 200**

```json
{
  "verified": true,
  "score": 0.85,
  "certificate": "eyJpZCI6IndtY18uLi4iLC...signature",
  "reason": ""
}
```

| Field | Type | Description |
|-------|------|-------------|
| `verified` | boolean | `true` if the agent passed verification (score >= 0.8). |
| `score` | float | Confidence score from 0.0 to 1.0. |
| `certificate` | string | Signed certificate token (only present when `verified` is `true`). Format: `base64url(payload).base64url(signature)`. |
| `reason` | string | Human-readable reason for rejection (only present when `verified` is `false`). |

**Scoring breakdown:**

| Component | Range | Description |
|-----------|-------|-------------|
| Base | 0.50 | Starting score |
| Time factor | -0.20 to +0.30 | Response speed (< 5s = +0.30, > 60s = -0.30) |
| Quality | 0.00 to +0.30 | Answer correctness |
| Relevance | 0.00 to +0.20 | Keyword matching with instruction |
| Format | 0.00 to +0.20 | Structure compliance |
| **Total** | **0.00 to 1.00** | |

**Score thresholds:**

| Score | Result |
|-------|--------|
| >= 0.80 | Certified -- certificate issued |
| 0.60 - 0.79 | Likely AI, but borderline -- no certificate |
| < 0.60 | Rejected |

#### Errors

| Status | Condition |
|--------|-----------|
| 400 | Invalid request body, missing required fields |
| 405 | Method not POST |
| 429 | Rate limit exceeded |
| 500 | Verification failed (internal error) |

---

### GET /v1/verify/{certificate}

Verify a certificate token. This is the agent-side verification endpoint -- the agent can check whether a certificate it holds is still valid.

**Rate limit:** 120 requests/minute per IP

For server-side verification by third parties, use `/v1/siteverify` instead.

#### Request

```http
GET /v1/verify/eyJpZCI6IndtY18uLi4iLC...signature
```

The certificate token is the last path segment.

#### Response -- Valid Certificate

**HTTP 200**

```json
{
  "valid": true,
  "issued_at": "2026-02-07T12:00:00Z",
  "expires_at": "2026-02-07T13:00:00Z",
  "score": 0.85,
  "level": 2
}
```

#### Response -- Invalid Certificate

**HTTP 200**

```json
{
  "valid": false,
  "error": "certificate expired"
}
```

| Field | Type | Description |
|-------|------|-------------|
| `valid` | boolean | Whether the certificate is valid and not expired. |
| `issued_at` | string | ISO 8601 timestamp of when the certificate was issued (only when valid). |
| `expires_at` | string | ISO 8601 timestamp of when the certificate expires (only when valid). Certificates are valid for 60 minutes. |
| `score` | float | Verification score (only when valid). |
| `level` | integer | Challenge difficulty level that was passed (only when valid). |
| `error` | string | Error description (only when invalid). |

#### Errors

| Status | Condition |
|--------|-----------|
| 400 | Missing certificate in path |
| 405 | Method not GET |
| 429 | Rate limit exceeded |

---

### POST /v1/siteverify

Server-side verification for third parties receiving certificates from agents. This is the recommended verification method. The response format mirrors reCAPTCHA's `siteverify` for familiarity.

**Rate limit:** 300 requests/minute per IP

This endpoint operates in two modes:

1. **Multi-tenant mode** (recommended): When `site_key` is provided, validates the secret against the site's stored secrets, tracks usage per site, checks customer quota, and enforces per-site minimum level.
2. **Legacy mode**: When no `site_key` is provided, validates against a globally configured secret.

#### Request

```http
POST /v1/siteverify
Content-Type: application/json
```

**Multi-tenant mode (recommended):**

```json
{
  "certificate": "eyJpZCI6IndtY18uLi4iLC...signature",
  "secret": "sk_sec_a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5f6",
  "site_key": "sk_pub_a1b2c3d4e5f6",
  "remoteip": "203.0.113.42"
}
```

**Legacy mode:**

```json
{
  "certificate": "eyJpZCI6IndtY18uLi4iLC...signature",
  "secret": "your-global-siteverify-secret"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `certificate` | string | Yes | The certificate token received from the agent. |
| `secret` | string | Yes* | Site secret key (`sk_sec_...`) in multi-tenant mode, or global secret in legacy mode. *Required when `site_key` is provided. |
| `site_key` | string | No | Site public key (`sk_pub_...`). When provided, enables multi-tenant mode with per-site secret validation and usage tracking. |
| `remoteip` | string | No | Client IP address for audit logging. |

#### Response -- Success

**HTTP 200**

```json
{
  "success": true,
  "challenge_ts": "2026-02-07T12:00:00Z",
  "score": 0.85,
  "level": 2,
  "action": "register",
  "user_hint": "user-12345",
  "hostname": "sk_pub_a1b2c3d4e5f6",
  "error-codes": []
}
```

| Field | Type | Description |
|-------|------|-------------|
| `success` | boolean | `true` if the certificate is valid. |
| `challenge_ts` | string | ISO 8601 timestamp of when the original challenge was issued. |
| `score` | float | Verification confidence score (0.0-1.0). |
| `level` | integer | Challenge difficulty level that was passed. |
| `action` | string | Echoed action hint from the certificate. Verify this matches your expected action. |
| `user_hint` | string | Echoed user hint from the certificate. Verify this matches your user context. |
| `hostname` | string | Site key / hostname from the certificate. |
| `error-codes` | array | Empty array on success. |

**Important:** Third parties MUST verify that the echoed `action`, `user_hint`, and `hostname` fields match their expected values. This prevents replay attacks where a valid certificate from one context is used in another.

#### Response -- Failure

**HTTP 200** (always 200, errors are in the body -- matches reCAPTCHA convention)

```json
{
  "success": false,
  "error-codes": ["invalid-input-response"]
}
```

#### Error Codes

| Code | Description |
|------|-------------|
| `invalid-input-response` | Certificate is malformed, has an invalid signature, or cannot be parsed. |
| `invalid-input-secret` | The secret does not match the globally configured secret (legacy mode). |
| `missing-input-secret` | No `secret` field provided when one is required. |
| `missing-input-response` | No `certificate` field provided. |
| `timeout-or-duplicate` | Certificate has expired (certificates are valid for 60 minutes). |
| `invalid-site-key` | The `site_key` does not match any registered site (multi-tenant mode). |
| `site-disabled` | The site is not active (deleted or suspended). |
| `invalid-secret` | The secret does not match any active or retired secret for the site (multi-tenant mode). |
| `quota-exceeded` | The customer's monthly verification quota has been exhausted. |
| `level-too-low` | The certificate's level is below the site's configured `min_level` (defense in depth). |

---

## Customer Portal API

These endpoints provide a self-service portal for customers integrating the certification widget into their applications. All portal endpoints (except register/login) require a Bearer token.

### Registration and Authentication

#### POST /api/v1/customers/register

Create a new customer account. All new accounts start on the Free tier.

**Rate limit:** 5 requests/minute per IP

##### Request

```http
POST /api/v1/customers/register
Content-Type: application/json
```

```json
{
  "email": "dev@example.com",
  "password": "Str0ng!Pass#2026",
  "company_name": "Acme Corp"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `email` | string | Yes | Valid email address (ASCII only, max 255 chars). Normalized to lowercase. |
| `password` | string | Yes | 10-72 characters. Must contain at least one uppercase letter, one lowercase letter, one digit, and one special character. |
| `company_name` | string | No | Company or organization name (max 255 chars). |

##### Response

**HTTP 201**

```json
{
  "customer_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "email": "dev@example.com",
  "tier": "free"
}
```

##### Errors

| Status | Condition |
|--------|-----------|
| 400 | Validation error (invalid email, weak password, etc.) |
| 409 | Email already registered |
| 429 | Rate limit exceeded |
| 500 | Internal error |

```json
{
  "error": "password must contain at least one uppercase letter, one lowercase letter, one digit, and one special character"
}
```

---

#### POST /api/v1/customers/login

Authenticate and obtain a session token.

**Rate limit:** 10 requests/minute per IP

**Account lockout:** After 5 consecutive failed login attempts, the account is locked for 15 minutes.

##### Request

```http
POST /api/v1/customers/login
Content-Type: application/json
```

```json
{
  "email": "dev@example.com",
  "password": "Str0ng!Pass#2026"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `email` | string | Yes | Account email address (max 255 chars, ASCII only). |
| `password` | string | Yes | Account password. |

##### Response -- Standard Login

**HTTP 200**

```json
{
  "session_token": "a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4",
  "expires_at": "2026-02-08T12:00:00Z",
  "customer": {
    "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "email": "dev@example.com",
    "company_name": "Acme Corp",
    "tier": "pro",
    "monthly_quota": 500000,
    "status": "active",
    "created_at": "2026-01-15T08:30:00Z",
    "updated_at": "2026-02-01T10:00:00Z",
    "totp_enabled": false
  }
}
```

##### Response -- 2FA Challenge

When the customer has TOTP two-factor authentication enabled, the login endpoint does not issue a session token. Instead, it returns a short-lived pending token that must be exchanged via `POST /api/v1/2fa/verify`.

**HTTP 200**

```json
{
  "requires_2fa": true,
  "pending_token": "e5f6a1b2c3d4e5f6a1b2c3d4e5f6a1b2..."
}
```

| Field | Type | Description |
|-------|------|-------------|
| `session_token` | string | Session token (absent when 2FA is required). |
| `expires_at` | string | Session expiry (absent when 2FA is required). |
| `customer` | object | Customer profile (absent when 2FA is required). |
| `requires_2fa` | boolean | `true` when the customer has 2FA enabled. |
| `pending_token` | string | Short-lived token (5 minutes). Use with `POST /api/v1/2fa/verify`. |

**Client flow when `requires_2fa` is `true`:**

1. Store the `pending_token` in memory (do not persist).
2. Show a 2FA code input form.
3. Submit the code + pending token to `POST /api/v1/2fa/verify`.
4. On success, the verify endpoint returns the full session (token + customer).

##### Errors

| Status | Condition |
|--------|-----------|
| 400 | Missing email or password, email too long |
| 401 | Invalid credentials (generic -- does not reveal whether email exists) |
| 403 | Account suspended |
| 429 | Account locked due to too many failed login attempts, or rate limit exceeded |
| 500 | Internal error |

---

#### POST /api/v1/customers/logout

End the current session. Requires Bearer token.

**Rate limit:** 120 requests/minute per IP

##### Request

```http
POST /api/v1/customers/logout
Authorization: Bearer <session_token>
```

No request body required.

##### Response

**HTTP 200**

```json
{
  "message": "logged out"
}
```

---

#### GET /api/v1/customers/me

Get the current authenticated customer's profile and quota information.

**Rate limit:** 120 requests/minute per IP

##### Request

```http
GET /api/v1/customers/me
Authorization: Bearer <session_token>
```

##### Response

**HTTP 200**

```json
{
  "customer": {
    "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "email": "dev@example.com",
    "company_name": "Acme Corp",
    "tier": "pro",
    "monthly_quota": 500000,
    "status": "active",
    "totp_enabled": false,
    "created_at": "2026-01-15T08:30:00Z",
    "updated_at": "2026-02-01T10:00:00Z"
  },
  "quota_remaining": 87500,
  "quota_exceeded": false
}
```

| Field | Type | Description |
|-------|------|-------------|
| `customer` | object | Customer profile. `monthly_quota` is `null` for unlimited (Enterprise tier). |
| `quota_remaining` | integer | Remaining verifications this month. `-1` if unlimited. |
| `quota_exceeded` | boolean | `true` if the monthly quota has been exhausted. |

##### Errors

| Status | Condition |
|--------|-----------|
| 401 | Missing or invalid session token |
| 405 | Method not GET |

---

### Password Reset

#### POST /api/v1/password/reset-request

Request a password reset email. Always returns success to avoid leaking whether an email is registered.

**Rate limit:** 5 requests/minute per IP

##### Request

```http
POST /api/v1/password/reset-request
Content-Type: application/json
```

```json
{
  "email": "dev@example.com"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `email` | string | Yes | Account email address. |

##### Response

**HTTP 200**

```json
{
  "success": true,
  "message": "If an account exists with that email, a password reset link has been sent."
}
```

##### Errors

| Status | Condition |
|--------|-----------|
| 400 | Missing or invalid email |
| 429 | Rate limit exceeded |

---

#### POST /api/v1/password/reset-confirm

Reset password using a token received via email.

**Rate limit:** 5 requests/minute per IP

##### Request

```http
POST /api/v1/password/reset-confirm
Content-Type: application/json
```

```json
{
  "token": "a1b2c3d4e5f6...",
  "new_password": "NewStr0ng!Pass#2026"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `token` | string | Yes | Reset token from the email link. |
| `new_password` | string | Yes | New password (same requirements as registration). |

##### Response

**HTTP 200**

```json
{
  "success": true,
  "message": "Password has been reset. You can now log in with your new password."
}
```

##### Errors

| Status | Condition |
|--------|-----------|
| 400 | Missing fields, invalid/expired token, weak password |
| 429 | Rate limit exceeded |
| 500 | Internal error |

---

### Email Verification

#### POST /api/v1/verify-email/confirm

Confirm email verification using a token. Supports two modes: initial registration verification and email change verification.

**Rate limit:** 120 requests/minute per IP

##### Request

```http
POST /api/v1/verify-email/confirm
Content-Type: application/json
```

```json
{
  "token": "a1b2c3d4e5f6...",
  "type": ""
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `token` | string | Yes | Verification token from the email link. |
| `type` | string | No | `"change"` for email change verification. Empty/omitted for registration verification. |

##### Response

**HTTP 200**

```json
{
  "success": true,
  "message": "Email verified successfully."
}
```

##### Errors

| Status | Condition |
|--------|-----------|
| 400 | Missing token, invalid/expired token |
| 500 | Internal error |

---

#### POST /api/v1/verify-email/resend

Resend the email verification link. Requires Bearer token.

**Rate limit:** 120 requests/minute per IP

##### Request

```http
POST /api/v1/verify-email/resend
Authorization: Bearer <session_token>
```

No request body required.

##### Response

**HTTP 200**

```json
{
  "success": true,
  "message": "Verification email sent."
}
```

##### Errors

| Status | Condition |
|--------|-----------|
| 400 | Email already verified |
| 401 | Not authenticated |
| 429 | Rate limit exceeded |
| 500 | Internal error |

---

### Account Settings

#### POST /api/v1/settings/email

Request an email address change. Requires current password confirmation. Sends a verification link to the new email address. The change is not applied until the new email is verified via `POST /api/v1/verify-email/confirm` with `type: "change"`.

**Rate limit:** 120 requests/minute per IP

##### Request

```http
POST /api/v1/settings/email
Authorization: Bearer <session_token>
Content-Type: application/json
```

```json
{
  "new_email": "new-email@example.com",
  "password": "Str0ng!Pass#2026"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `new_email` | string | Yes | New email address (ASCII only, max 255 chars). |
| `password` | string | Yes | Current password for confirmation. |

##### Response

**HTTP 200**

```json
{
  "success": true,
  "message": "Verification email sent to your new address. Please check your inbox."
}
```

**Note:** Returns success even if the new email is already taken (to avoid leaking registered emails). The change will silently fail at the verification step.

##### Errors

| Status | Condition |
|--------|-----------|
| 400 | Missing fields, invalid email, same as current email |
| 401 | Not authenticated |
| 403 | Incorrect password |
| 429 | Rate limit exceeded |
| 500 | Internal error |

---

#### POST /api/v1/settings/password

Change the account password. Requires the current password. Invalidates all sessions -- the user must log in again with the new password.

**Rate limit:** 120 requests/minute per IP

##### Request

```http
POST /api/v1/settings/password
Authorization: Bearer <session_token>
Content-Type: application/json
```

```json
{
  "current_password": "Str0ng!Pass#2026",
  "new_password": "Ev3nStr0nger!Pass#2026"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `current_password` | string | Yes | Current password for verification. |
| `new_password` | string | Yes | New password (same requirements as registration: 10-72 chars, uppercase, lowercase, digit, special). |

##### Response

**HTTP 200**

```json
{
  "success": true,
  "message": "Password updated. Please log in again with your new password."
}
```

##### Errors

| Status | Condition |
|--------|-----------|
| 400 | Missing fields, weak password |
| 401 | Not authenticated |
| 403 | Incorrect current password |
| 429 | Rate limit exceeded |
| 500 | Internal error |

---

### Two-Factor Authentication (TOTP)

The certification portal supports TOTP-based two-factor authentication (RFC 6238). When enabled, login requires both a password and a 6-digit code from an authenticator app (or a one-time backup code).

**Key concepts:**
- **Setup** generates a secret (displayed as QR code + manual entry).
- **Enable** verifies the user can generate valid codes before activating 2FA.
- **Backup codes** (10, format `xxxx-xxxx`) are shown once at enable time. Each can be used once.
- **Login with 2FA** returns a pending token instead of a session; the pending token is exchanged for a full session via the verify endpoint.

#### POST /api/v1/settings/2fa/setup

Start TOTP setup by generating a new secret. The secret is encrypted and stored in the database but 2FA is not yet enabled -- the user must verify with a code via the enable endpoint.

**Auth:** Bearer token required

**Rate limit:** 120 requests/minute per IP

##### Request

```http
POST /api/v1/settings/2fa/setup
Authorization: Bearer <session_token>
```

No request body required.

##### Response

**HTTP 200**

```json
{
  "secret": "JBSWY3DPEHPK3PXP",
  "otpauth_uri": "otpauth://totp/Wordmade%20Certification:dev@example.com?secret=JBSWY3DPEHPK3PXP&issuer=Wordmade+Certification"
}
```

| Field | Type | Description |
|-------|------|-------------|
| `secret` | string | Base32-encoded TOTP secret for manual entry into authenticator apps. |
| `otpauth_uri` | string | Standard `otpauth://` URI for QR code generation. |

##### Errors

| Status | Condition |
|--------|-----------|
| 401 | Not authenticated |
| 405 | Method not POST |
| 409 | 2FA is already enabled |
| 503 | TOTP encryption key not configured on server |

---

#### POST /api/v1/settings/2fa/enable

Verify the TOTP setup and enable 2FA. The user must provide a valid 6-digit code from their authenticator app. On success, returns 10 backup codes that must be saved -- they will not be shown again. All other sessions are invalidated.

**Auth:** Bearer token required

**Rate limit:** 120 requests/minute per IP

##### Request

```http
POST /api/v1/settings/2fa/enable
Authorization: Bearer <session_token>
Content-Type: application/json
```

```json
{
  "code": "123456"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `code` | string | Yes | 6-digit TOTP code from the authenticator app. |

##### Response

**HTTP 200**

```json
{
  "success": true,
  "backup_codes": [
    "a1b2-c3d4",
    "e5f6-a7b8",
    "c9d0-e1f2",
    "a3b4-c5d6",
    "e7f8-a9b0",
    "c1d2-e3f4",
    "a5b6-c7d8",
    "e9f0-a1b2",
    "c3d4-e5f6",
    "a7b8-c9d0"
  ],
  "message": "2FA is now enabled. Save your backup codes — they will not be shown again."
}
```

| Field | Type | Description |
|-------|------|-------------|
| `success` | boolean | `true` on successful enable. |
| `backup_codes` | array | 10 one-time backup codes (format: `xxxx-xxxx`). **Shown only once.** |
| `message` | string | Confirmation message. |

##### Errors

| Status | Condition |
|--------|-----------|
| 400 | Missing code, code not 6 digits, setup not started |
| 401 | Invalid TOTP code, or not authenticated |
| 409 | 2FA is already enabled |
| 429 | Too many attempts |

---

#### POST /api/v1/settings/2fa/disable

Disable 2FA. Requires both the current password and a valid TOTP code (or backup code). Clears the TOTP secret and all backup codes. Invalidates all other sessions.

**Auth:** Bearer token required

**Rate limit:** 120 requests/minute per IP

##### Request

```http
POST /api/v1/settings/2fa/disable
Authorization: Bearer <session_token>
Content-Type: application/json
```

```json
{
  "password": "Str0ng!Pass#2026",
  "code": "123456"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `password` | string | Yes | Current account password. |
| `code` | string | Yes | 6-digit TOTP code or a backup code (format `xxxx-xxxx`). |

##### Response

**HTTP 200**

```json
{
  "success": true,
  "message": "2FA has been disabled"
}
```

##### Errors

| Status | Condition |
|--------|-----------|
| 400 | Missing fields, 2FA not enabled |
| 401 | Incorrect password, invalid code, or not authenticated |
| 429 | Too many attempts |

---

#### GET /api/v1/settings/2fa/status

Get the current 2FA status for the authenticated customer.

**Auth:** Bearer token required

**Rate limit:** 120 requests/minute per IP

##### Request

```http
GET /api/v1/settings/2fa/status
Authorization: Bearer <session_token>
```

##### Response

**HTTP 200**

```json
{
  "enabled": true,
  "enabled_at": "2026-02-07T12:00:00Z",
  "backup_codes_left": 8
}
```

| Field | Type | Description |
|-------|------|-------------|
| `enabled` | boolean | Whether 2FA is currently enabled. |
| `enabled_at` | string/null | ISO 8601 timestamp when 2FA was enabled. `null` if disabled. |
| `backup_codes_left` | integer | Number of unused backup codes remaining. `0` if 2FA is disabled. |

##### Errors

| Status | Condition |
|--------|-----------|
| 401 | Not authenticated |
| 405 | Method not GET |

---

#### POST /api/v1/settings/2fa/backup-codes

Regenerate backup codes. Requires password and a valid TOTP code (not a backup code -- to avoid a chicken-and-egg problem). Old backup codes become invalid immediately.

**Auth:** Bearer token required

**Rate limit:** 120 requests/minute per IP

##### Request

```http
POST /api/v1/settings/2fa/backup-codes
Authorization: Bearer <session_token>
Content-Type: application/json
```

```json
{
  "password": "Str0ng!Pass#2026",
  "code": "123456"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `password` | string | Yes | Current account password. |
| `code` | string | Yes | 6-digit TOTP code (backup codes are not accepted here). |

##### Response

**HTTP 200**

```json
{
  "success": true,
  "backup_codes": [
    "a1b2-c3d4",
    "e5f6-a7b8",
    "..."
  ],
  "message": "New backup codes generated. Save them — the old codes are now invalid."
}
```

##### Errors

| Status | Condition |
|--------|-----------|
| 400 | Missing fields, code not 6 digits, 2FA not enabled |
| 401 | Incorrect password, invalid code, or not authenticated |
| 429 | Too many attempts |

---

#### POST /api/v1/2fa/verify

Verify a 2FA code during the login flow. This is a **public endpoint** (no session required) -- it uses the pending token received from the login endpoint.

The pending token is **single-use** and expires after 5 minutes. A failed verification attempt consumes the token, requiring the user to log in again.

**Rate limit:** Per customer (5 attempts/15 min) and per IP (20 attempts/15 min)

##### Request

```http
POST /api/v1/2fa/verify
Content-Type: application/json
```

```json
{
  "pending_token": "e5f6a1b2c3d4e5f6a1b2c3d4e5f6a1b2...",
  "code": "123456"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `pending_token` | string | Yes | The pending token from the login response. |
| `code` | string | Yes | 6-digit TOTP code or a backup code (format `xxxx-xxxx`). |

##### Response

**HTTP 200** -- returns the same format as a standard login response:

```json
{
  "session_token": "a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4",
  "expires_at": "2026-02-08T12:00:00Z",
  "customer": {
    "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "email": "dev@example.com",
    "company_name": "Acme Corp",
    "tier": "pro",
    "monthly_quota": 500000,
    "status": "active",
    "created_at": "2026-01-15T08:30:00Z",
    "updated_at": "2026-02-01T10:00:00Z",
    "totp_enabled": true
  }
}
```

##### Errors

| Status | Condition |
|--------|-----------|
| 400 | Missing fields |
| 401 | Invalid/expired pending token, invalid 2FA code |
| 429 | Rate limit exceeded (per customer or per IP) |
| 500 | Internal error |

---

### Site Management

Sites represent applications or domains that use the certification widget. Each site has a public key (`sk_pub_...`) for the widget and one or more secret keys (`sk_sec_...`) for server-side verification.

#### POST /api/v1/sites

Create a new site. The secret key is returned only once in the response -- store it securely.

**Rate limit:** 120 requests/minute per IP

##### Request

```http
POST /api/v1/sites
Authorization: Bearer <session_token>
Content-Type: application/json
```

```json
{
  "name": "My Web App",
  "domain": "app.example.com"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Site name (max 255 chars). |
| `domain` | string | No | Associated domain. Must be a valid domain format (lowercase, alphanumeric, hyphens, dots). |

##### Response

**HTTP 201**

```json
{
  "site_id": "b2c3d4e5-f6a1-7890-abcd-ef1234567890",
  "name": "My Web App",
  "site_key": "sk_pub_a1b2c3d4e5f6",
  "secret_key": "sk_sec_a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5f6"
}
```

| Field | Type | Description |
|-------|------|-------------|
| `site_id` | string | UUID of the created site. |
| `name` | string | Site name. |
| `site_key` | string | Public key (prefix: `sk_pub_`). Use this in widget embed and challenge requests. |
| `secret_key` | string | **Secret key (prefix: `sk_sec_`). Shown only once.** Use this in server-side siteverify calls. |

##### Errors

| Status | Condition |
|--------|-----------|
| 400 | Missing name, invalid domain format, name too long |
| 401 | Not authenticated |
| 403 | Site limit reached for current tier |
| 429 | Rate limit exceeded |
| 500 | Internal error |

---

#### GET /api/v1/sites

List all sites for the authenticated customer.

**Rate limit:** 120 requests/minute per IP

##### Request

```http
GET /api/v1/sites
Authorization: Bearer <session_token>
```

##### Response

**HTTP 200**

```json
{
  "sites": [
    {
      "id": "b2c3d4e5-f6a1-7890-abcd-ef1234567890",
      "customer_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
      "name": "My Web App",
      "domain": "app.example.com",
      "site_key": "sk_pub_a1b2c3d4e5f6",
      "settings": {
        "min_level": 2
      },
      "status": "active",
      "created_at": "2026-01-20T14:00:00Z",
      "updated_at": "2026-02-01T10:00:00Z",
      "last_used_at": "2026-02-07T11:30:00Z",
      "active_secrets": 1
    }
  ],
  "count": 1
}
```

---

#### GET /api/v1/sites/{id}

Get details for a specific site, including its secrets (without the actual secret values -- only prefixes and metadata).

**Rate limit:** 120 requests/minute per IP

##### Request

```http
GET /api/v1/sites/b2c3d4e5-f6a1-7890-abcd-ef1234567890
Authorization: Bearer <session_token>
```

##### Response

**HTTP 200**

```json
{
  "site": {
    "id": "b2c3d4e5-f6a1-7890-abcd-ef1234567890",
    "customer_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "name": "My Web App",
    "domain": "app.example.com",
    "site_key": "sk_pub_a1b2c3d4e5f6",
    "settings": {
      "min_level": 2
    },
    "status": "active",
    "created_at": "2026-01-20T14:00:00Z",
    "updated_at": "2026-02-01T10:00:00Z",
    "last_used_at": "2026-02-07T11:30:00Z"
  },
  "secrets": [
    {
      "id": "c3d4e5f6-a1b2-7890-abcd-ef1234567890",
      "prefix": "sk_sec_a1",
      "status": "active",
      "created_at": "2026-01-20T14:00:00Z",
      "expires_at": null,
      "last_used_at": "2026-02-07T11:30:00Z"
    }
  ]
}
```

##### Errors

| Status | Condition |
|--------|-----------|
| 400 | Invalid site ID format |
| 401 | Not authenticated |
| 403 | Site belongs to a different customer |
| 404 | Site not found |

---

#### PATCH /api/v1/sites/{id}

Update a site's name, domain, or settings. All fields are optional -- only provided fields are updated.

**Rate limit:** 120 requests/minute per IP

##### Request

```http
PATCH /api/v1/sites/b2c3d4e5-f6a1-7890-abcd-ef1234567890
Authorization: Bearer <session_token>
Content-Type: application/json
```

```json
{
  "name": "My Updated App",
  "domain": "new.example.com",
  "settings": {
    "min_level": 3
  }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | No | New site name (max 255 chars, cannot be empty). |
| `domain` | string | No | New domain (valid format or empty to clear). |
| `settings` | object | No | Site settings. Currently supports `min_level` (integer 1-5). |

**Per-site minimum level (`min_level`):**

When set, two enforcement mechanisms activate:
1. Challenge requests with this `site_key` are clamped UP to `min_level`.
2. Siteverify rejects certificates with a level below `min_level` (defense in depth).

##### Response

**HTTP 200** -- returns the updated site object.

```json
{
  "id": "b2c3d4e5-f6a1-7890-abcd-ef1234567890",
  "customer_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "name": "My Updated App",
  "domain": "new.example.com",
  "site_key": "sk_pub_a1b2c3d4e5f6",
  "settings": {
    "min_level": 3
  },
  "status": "active",
  "created_at": "2026-01-20T14:00:00Z",
  "updated_at": "2026-02-07T12:00:00Z"
}
```

##### Errors

| Status | Condition |
|--------|-----------|
| 400 | Invalid site ID, empty name, invalid domain, `min_level` out of range |
| 401 | Not authenticated |
| 403 | Access denied (site belongs to different customer) |
| 404 | Site not found |

---

#### DELETE /api/v1/sites/{id}

Soft-delete a site. All certificates issued for this site are automatically revoked. This cannot be undone.

**Rate limit:** 120 requests/minute per IP

##### Request

```http
DELETE /api/v1/sites/b2c3d4e5-f6a1-7890-abcd-ef1234567890
Authorization: Bearer <session_token>
```

##### Response

**HTTP 200**

```json
{
  "message": "site deleted"
}
```

##### Errors

| Status | Condition |
|--------|-----------|
| 400 | Invalid site ID format |
| 401 | Not authenticated |
| 403 | Access denied |
| 404 | Site not found |
| 500 | Internal error |

---

### Secret Management

Secrets are used for server-side verification via the `siteverify` endpoint. Secret rotation provides a grace period during which both old and new secrets are accepted.

#### POST /api/v1/sites/{id}/secrets/rotate

Rotate the secret key for a site. Creates a new active secret and retires existing secrets. Retired secrets continue to work during the grace period (default: 24 hours).

**Rate limit:** 120 requests/minute per IP

##### Request

```http
POST /api/v1/sites/b2c3d4e5-f6a1-7890-abcd-ef1234567890/secrets/rotate
Authorization: Bearer <session_token>
```

No request body required.

##### Response

**HTTP 200**

```json
{
  "secret_key": "sk_sec_new1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d",
  "old_secret_id": "c3d4e5f6-a1b2-7890-abcd-ef1234567890",
  "grace_period": "24h0m0s",
  "retired_until": "2026-02-08T12:00:00Z"
}
```

| Field | Type | Description |
|-------|------|-------------|
| `secret_key` | string | **New secret key. Shown only once.** Store it securely and update your server configuration. |
| `old_secret_id` | string | ID(s) of the retired secret(s), comma-separated if multiple. |
| `grace_period` | string | Duration string for how long retired secrets remain valid. |
| `retired_until` | string | ISO 8601 timestamp when the old secret(s) will stop working. |

##### Errors

| Status | Condition |
|--------|-----------|
| 400 | Invalid site ID |
| 401 | Not authenticated |
| 403 | Access denied |
| 404 | Site not found |
| 500 | Internal error |

---

#### POST /api/v1/sites/{siteId}/secrets/{secretId}/revoke

Immediately revoke a specific secret. Unlike rotation, revocation is instant with no grace period. Use this for security incidents.

**Rate limit:** 120 requests/minute per IP

##### Request

```http
POST /api/v1/sites/b2c3d4e5-f6a1-7890-abcd-ef1234567890/secrets/c3d4e5f6-a1b2-7890-abcd-ef1234567890/revoke
Authorization: Bearer <session_token>
```

No request body required.

##### Response

**HTTP 200**

```json
{
  "message": "secret revoked",
  "secret_id": "c3d4e5f6-a1b2-7890-abcd-ef1234567890"
}
```

##### Errors

| Status | Condition |
|--------|-----------|
| 400 | Invalid site ID or secret ID |
| 401 | Not authenticated |
| 403 | Access denied or secret does not belong to this site |
| 404 | Site or secret not found |
| 500 | Internal error |

---

### Certificate Revocation

#### POST /api/v1/certificates/revoke

Revoke a certificate or all certificates for a site. Revoked certificates will fail siteverify checks immediately.

**Rate limit:** 120 requests/minute per IP

##### Request

```http
POST /api/v1/certificates/revoke
Authorization: Bearer <session_token>
Content-Type: application/json
```

**Revoke a specific certificate:**

```json
{
  "certificate_id": "wmc_a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4",
  "site_key": "sk_pub_a1b2c3d4e5f6",
  "reason": "manual"
}
```

**Bulk revoke all certificates for a site:**

```json
{
  "site_key": "sk_pub_a1b2c3d4e5f6",
  "reason": "security_incident"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `certificate_id` | string | No | Specific certificate to revoke (format: `wmc_` + 32 hex chars). When provided, `site_key` is also required for ownership verification. |
| `site_key` | string | Yes* | Public key of the site. Required when `certificate_id` is provided (for ownership check). When provided alone, revokes all certificates for the site. |
| `reason` | string | Yes | Reason for revocation: `manual`, `security_incident`, or `bulk`. |

*At least one of `certificate_id` or `site_key` must be provided.

##### Response

**HTTP 200**

```json
{
  "message": "revocation recorded"
}
```

##### Errors

| Status | Condition |
|--------|-----------|
| 400 | Missing both `certificate_id` and `site_key`, invalid format, invalid reason |
| 401 | Not authenticated |
| 403 | Access denied (site belongs to different customer) |
| 404 | Site not found |
| 500 | Internal error |

---

### Usage and Billing

#### GET /api/v1/usage

Get aggregated usage statistics for the authenticated customer across all sites.

**Rate limit:** 120 requests/minute per IP

##### Request

```http
GET /api/v1/usage?days=30
Authorization: Bearer <session_token>
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `days` | integer | 30 | Number of days to look back (1-365). |

##### Response

**HTTP 200**

```json
{
  "period": {
    "start": "2026-01-08",
    "end": "2026-02-07",
    "days": "30"
  },
  "stats": {
    "challenges": 15420,
    "responses": 14800,
    "verifications": 12500,
    "success_rate": 0.845,
    "responses_success": 12500,
    "responses_failed": 2300
  },
  "quota": {
    "tier": "pro",
    "monthly_limit": 500000,
    "current_month_used": 12500,
    "remaining": 487500,
    "exceeded": false,
    "unlimited": false
  }
}
```

---

#### GET /api/v1/sites/{id}/usage

Get usage statistics for a specific site with daily breakdown.

**Rate limit:** 120 requests/minute per IP

##### Request

```http
GET /api/v1/sites/b2c3d4e5-f6a1-7890-abcd-ef1234567890/usage?days=7
Authorization: Bearer <session_token>
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `days` | integer | 30 | Number of days to look back (1-365). |

##### Response

**HTTP 200**

```json
{
  "site": {
    "id": "b2c3d4e5-f6a1-7890-abcd-ef1234567890",
    "name": "My Web App"
  },
  "period": {
    "start": "2026-01-31",
    "end": "2026-02-07",
    "days": "7"
  },
  "stats": {
    "challenges": 2100,
    "responses": 2050,
    "verifications": 1800,
    "success_rate": 0.878,
    "responses_success": 1800,
    "responses_failed": 250
  },
  "daily": [
    {
      "date": "2026-02-01",
      "challenges": 300,
      "responses": 290,
      "verifications": 260,
      "success": 260,
      "failed": 30
    },
    {
      "date": "2026-02-02",
      "challenges": 310,
      "responses": 305,
      "verifications": 270,
      "success": 270,
      "failed": 35
    }
  ]
}
```

##### Errors

| Status | Condition |
|--------|-----------|
| 400 | Invalid site ID |
| 401 | Not authenticated |
| 403 | Access denied |
| 404 | Site not found |

---

#### GET /api/v1/quota

Get current quota status for the authenticated customer.

**Rate limit:** 120 requests/minute per IP

##### Request

```http
GET /api/v1/quota
Authorization: Bearer <session_token>
```

##### Response

**HTTP 200**

```json
{
  "tier": "pro",
  "monthly_limit": 500000,
  "current_month_used": 12500,
  "remaining": 487500,
  "exceeded": false,
  "unlimited": false,
  "period": "2026-02"
}
```

| Field | Type | Description |
|-------|------|-------------|
| `tier` | string | Current subscription tier. |
| `monthly_limit` | integer/null | Monthly quota. `null` for unlimited (Enterprise). |
| `current_month_used` | integer | Verifications used in the current calendar month. |
| `remaining` | integer | Remaining verifications. `-1` if unlimited. |
| `exceeded` | boolean | `true` if quota is exhausted. |
| `unlimited` | boolean | `true` if on Enterprise tier (no quota limit). |
| `period` | string | Current billing period (YYYY-MM). |

---

#### GET /api/v1/billing/history

Get monthly billing/usage history.

**Rate limit:** 120 requests/minute per IP

##### Request

```http
GET /api/v1/billing/history?months=12
Authorization: Bearer <session_token>
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `months` | integer | 12 | Number of months of history (1-24). |

##### Response

**HTTP 200**

```json
{
  "customer_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "tier": "pro",
  "history": [
    {
      "year_month": "2026-02",
      "verifications": 12500,
      "billed": false
    },
    {
      "year_month": "2026-01",
      "verifications": 45200,
      "billable_amount": 19.00,
      "billed": true,
      "billed_at": "2026-02-01T00:00:00Z"
    }
  ]
}
```

---

#### GET /api/v1/audit

Get recent audit log entries for the customer. Audit logs record security-relevant events.

**Rate limit:** 120 requests/minute per IP

##### Request

```http
GET /api/v1/audit?limit=50
Authorization: Bearer <session_token>
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `limit` | integer | 50 | Maximum entries to return (1-200). |

##### Response

**HTTP 200**

```json
{
  "logs": [
    {
      "id": 1234,
      "event_type": "secret_rotated",
      "event_data": {
        "new_prefix": "sk_sec_ne",
        "retired_count": 1,
        "grace_period": "24h0m0s"
      },
      "site_id": "b2c3d4e5-f6a1-7890-abcd-ef1234567890",
      "created_at": "2026-02-07T12:00:00Z"
    },
    {
      "id": 1233,
      "event_type": "customer_login",
      "event_data": {
        "session_id": "sess_a1b2c3"
      },
      "created_at": "2026-02-07T11:55:00Z"
    }
  ],
  "count": 2
}
```

**Audit event types:**

| Event Type | Description |
|------------|-------------|
| `customer_registered` | New customer account created |
| `customer_login` | Successful login |
| `customer_login_failed` | Failed login attempt |
| `customer.account_locked` | Account locked after too many failed login attempts |
| `site_created` | New site registered |
| `site_updated` | Site name, domain, or settings changed |
| `site_deleted` | Site soft-deleted |
| `secret_rotated` | Secret key rotated |
| `secret_revoked` | Secret key immediately revoked |
| `certificate_revoked` | Certificate revoked |
| `quota_exceeded` | Monthly verification quota exhausted |
| `siteverify_success` | Successful siteverify call |
| `siteverify_failed` | Failed siteverify call |
| `password_reset_requested` | Password reset email sent |
| `password_reset_completed` | Password successfully reset via token |
| `email_verified` | Email address verified |
| `email_change_requested` | Email change initiated (verification sent to new address) |
| `email_change_completed` | Email change confirmed via verification token |
| `password_changed` | Password changed via settings |
| `totp_setup_initiated` | 2FA setup started (secret generated) |
| `totp_enabled` | 2FA enabled (code verified, backup codes issued) |
| `totp_disabled` | 2FA disabled |
| `totp_verify_failed` | Failed 2FA code verification during login |
| `totp_backup_code_used` | Backup code used for login or 2FA disable |
| `totp_backup_codes_regenerated` | Backup codes regenerated |
| `subscription_upgrade` | Tier upgraded |
| `subscription_downgrade` | Tier downgraded |
| `subscription_cancel` | Subscription canceled |
| `subscription_expiry` | Subscription expired (auto-downgraded to free) |

---

### Subscription Management

#### GET /api/v1/subscription

Get the current subscription plan details.

**Rate limit:** 120 requests/minute per IP

##### Request

```http
GET /api/v1/subscription
Authorization: Bearer <session_token>
```

##### Response

**HTTP 200**

```json
{
  "tier": "pro",
  "monthly_quota": 500000,
  "started_at": "2026-01-15T08:30:00Z",
  "expires_at": "2026-02-15T08:30:00Z",
  "price_cents": 1900
}
```

| Field | Type | Description |
|-------|------|-------------|
| `tier` | string | Current tier: `free`, `starter`, `pro`, or `enterprise`. |
| `monthly_quota` | integer/null | Monthly verification limit. `null` for unlimited. |
| `started_at` | string | When the subscription started (omitted for free tier). |
| `expires_at` | string | When the subscription expires (omitted for free tier). |
| `price_cents` | integer | Monthly price in cents. |

---

#### POST /api/v1/subscription/checkout

Initiate a subscription upgrade. Returns a checkout URL for payment processing.

**Rate limit:** 120 requests/minute per IP

##### Request

```http
POST /api/v1/subscription/checkout
Authorization: Bearer <session_token>
Content-Type: application/json
```

```json
{
  "target_tier": "pro"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `target_tier` | string | Yes | Target tier. Must be `pro`. Enterprise requires a custom agreement (contact sales). To downgrade to free, use `/subscription/cancel`. |

##### Response

**HTTP 200**

```json
{
  "checkout_url": "https://checkout.stripe.com/c/pay/cs_..."
}
```

The client should redirect to `checkout_url`. After successful payment, the upgrade is applied automatically.

##### Errors

| Status | Condition |
|--------|-----------|
| 400 | Invalid tier, already on target tier, Enterprise requires sales, use cancel for free |
| 401 | Not authenticated |
| 503 | Payment processing not available |
| 500 | Internal error |

---

#### POST /api/v1/subscription/cancel

Cancel the current subscription and downgrade to the Free tier.

**Rate limit:** 120 requests/minute per IP

##### Request

```http
POST /api/v1/subscription/cancel
Authorization: Bearer <session_token>
```

No request body required.

##### Response

**HTTP 200**

```json
{
  "success": true,
  "tier": "free",
  "message": "Subscription canceled"
}
```

If already on the free tier:

```json
{
  "success": true,
  "tier": "free",
  "message": "Already on free tier"
}
```

---

#### GET /api/v1/subscription/history

Get the subscription change history (last 20 changes).

**Rate limit:** 120 requests/minute per IP

##### Request

```http
GET /api/v1/subscription/history
Authorization: Bearer <session_token>
```

##### Response

**HTTP 200**

```json
{
  "history": [
    {
      "id": 3,
      "previous_tier": "free",
      "new_tier": "pro",
      "change_type": "upgrade",
      "prorated_quota": 8500,
      "provider": "stripe",
      "created_at": "2026-02-01T10:00:00Z"
    },
    {
      "id": 2,
      "previous_tier": "pro",
      "new_tier": "free",
      "change_type": "cancel",
      "prorated_quota": 0,
      "provider": "stripe",
      "created_at": "2026-01-15T08:30:00Z"
    }
  ]
}
```

| Field | Type | Description |
|-------|------|-------------|
| `previous_tier` | string | Tier before the change. |
| `new_tier` | string | Tier after the change. |
| `change_type` | string | Type of change: `upgrade`, `downgrade`, `cancel`, `expiry`. |
| `prorated_quota` | integer | Prorated quota adjustment applied during the change. |
| `provider` | string | Payment provider (e.g. `stripe`). |
| `notes` | string | Optional notes about the change. |

---

### Stripe Webhook

#### POST /api/v1/webhooks/stripe

Receives and processes Stripe webhook events. This endpoint is not rate limited but is verified via Stripe's HMAC signature.

No Bearer token -- authenticated by the `Stripe-Signature` header.

##### Handled Event Types

| Event Type | Action |
|------------|--------|
| `checkout.session.completed` | Completes subscription upgrade. Reads `customer_id` and `target_tier` from session metadata. |
| `customer.subscription.deleted` | Downgrades customer to free tier. |
| `invoice.payment_failed` | Logged; Stripe handles retries automatically. |

Other event types are logged and acknowledged with HTTP 200.

##### Response

**HTTP 200**

```json
{
  "status": "ok"
}
```

##### Errors

| Status | Condition |
|--------|-----------|
| 400 | Invalid request body or invalid Stripe signature |
| 405 | Method not POST |
| 500 | Webhook processing failed (Stripe will retry) |
| 503 | Stripe not configured |

---

## Quotas and Limits

Each tier has a monthly verification quota and a maximum number of sites. See the [pricing page](https://certification.wordmade.world/#pricing) for current tiers and pricing.

When a customer's monthly quota is exhausted, siteverify calls return the `quota-exceeded` error code. Challenge and respond endpoints continue to work (they do not count toward quota), but siteverify will reject verification requests until the next billing period.

---

## Documentation Endpoints

The certification service serves built-in documentation at several paths.

### Agent Documentation

Served at multiple paths for discoverability:

- `/agents.md`, `/agent.md`, `/agent.txt`
- `/llms.md`, `/llm.md`, `/llms.txt`, `/llm.txt`

Returns markdown documentation aimed at AI agents, explaining how to request, solve, and submit challenges.

Any unregistered path (`/`) also returns the agent documentation -- agents exploring the API always get useful instructions.

### Integration Guide

Comprehensive documentation for all integration parties (site owners, developers, agents building sites):

- `/guide.md`, `/guide.txt`
- `/integration.md`, `/integration.txt`
- `/parties.md`, `/parties.txt`
- `/docs`, `/help`

### Widget Demo

- `/demo`, `/customize` -- Interactive widget customizer and live preview.

### Embeddable Widget

- `/embed`, `/widget` -- Self-contained HTML widget with dynamic theming.

Query parameters:

| Parameter | Description | Example |
|-----------|-------------|---------|
| `theme` | Theme preset | `sacred`, `dark`, `light`, `sanctum`, `midnight`, `ember`, `forest`, `flame` |
| `colorScheme` | Base color scheme | `light` or `dark` |
| `site_key` | Site key for data attribute | `sk_pub_a1b2c3d4e5f6` |
| `bg` | Background color | `%230a0a0f` |
| `accent` | Accent color | `%23d4a853` |
| `text` | Text color | `%23e8e6e3` |
| `border` | Border color | `%232a2a35` |
| `markOpacity` | Watermark opacity (0.05-0.25) | `0.15` |
| `markBrightness` | Watermark brightness (0.2-2.0) | `1.2` |
| `embedded` | Remove borders for nested embedding | `1` |

---

## Error Format

All API errors follow one of two formats:

### Standard Error

Used by most endpoints:

```json
{
  "error": "descriptive error message"
}
```

### Siteverify Error

Used by the `/v1/siteverify` endpoint (reCAPTCHA-compatible format). Always returns HTTP 200:

```json
{
  "success": false,
  "error-codes": ["invalid-input-response"]
}
```

---

## Certificate Format

Certificates are compact signed tokens in the format: `base64url(payload).base64url(signature)`.

### Decoded Payload

```json
{
  "v": 2,
  "id": "wmc_a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4",
  "kid": "key-1",
  "site": "sk_pub_a1b2c3d4e5f6",
  "action": "register",
  "uh": "user-12345",
  "iat": "2026-02-07T12:00:00Z",
  "exp": "2026-02-07T13:00:00Z",
  "s": 0.85,
  "l": 2
}
```

| Field | JSON Key | Description |
|-------|----------|-------------|
| Version | `v` | Certificate format version (1 = single-key, 2 = multi-key). |
| Certificate ID | `id` | Unique identifier (prefix: `wmc_`). |
| Key ID | `kid` | Signing key identifier (v2 only, for key rotation). |
| Site | `site` | Site key that requested the challenge. |
| Action | `action` | Action context from the challenge request. |
| User Hint | `uh` | User hint from the challenge request. |
| Issued At | `iat` | Timestamp when the certificate was issued. |
| Expires At | `exp` | Timestamp when the certificate expires (issued + 60 minutes). |
| Score | `s` | Verification score (0.0-1.0). |
| Level | `l` | Challenge difficulty level that was passed. |

---

## Integration Flow

### Complete Flow for Third-Party Sites

```
Agent                    Your Server              Certification API
  |                          |                          |
  |  1. Visit your site      |                          |
  |------------------------->|                          |
  |                          |                          |
  |  2. Get challenge         |                          |
  |----------------------------------------------------->|
  |  POST /v1/challenge/request                         |
  |  {"level":2, "site_key":"sk_pub_...",               |
  |   "action":"register", "user_hint":"user-123"}      |
  |                          |                          |
  |  3. Challenge returned    |                          |
  |<-----------------------------------------------------|
  |  {"challenge_id", "instruction", "nonce", ...}      |
  |                          |                          |
  |  4. Solve challenge       |                          |
  |  (decode + compute)       |                          |
  |                          |                          |
  |  5. Submit response       |                          |
  |----------------------------------------------------->|
  |  POST /v1/challenge/respond                         |
  |  {all challenge fields + "response":"42"}           |
  |                          |                          |
  |  6. Certificate issued    |                          |
  |<-----------------------------------------------------|
  |  {"verified":true, "certificate":"eyJ..."}          |
  |                          |                          |
  |  7. Present certificate   |                          |
  |------------------------->|                          |
  |  X-Agent-Certification: eyJ...                      |
  |                          |                          |
  |                          |  8. Server-side verify   |
  |                          |------------------------->|
  |                          |  POST /v1/siteverify     |
  |                          |  {"certificate":"eyJ...",|
  |                          |   "secret":"sk_sec_...", |
  |                          |   "site_key":"sk_pub_..."}|
  |                          |                          |
  |                          |  9. Verification result  |
  |                          |<-------------------------|
  |                          |  {"success":true,        |
  |                          |   "score":0.85,          |
  |                          |   "action":"register",   |
  |                          |   "user_hint":"user-123"}|
  |                          |                          |
  |                          | 10. Verify context match |
  |                          | assert action=="register"|
  |                          | assert user_hint matches |
  |                          |                          |
  | 11. Grant access          |                          |
  |<-------------------------|                          |
```

### Quick Setup Checklist

1. Register at the customer portal (`POST /api/v1/customers/register`)
2. Create a site (`POST /api/v1/sites`)
3. Save the `site_key` (public, embed in widget) and `secret_key` (private, server-side only)
4. Embed the widget on your site with the `site_key`
5. On your server, verify certificates via `POST /v1/siteverify` with your `secret` and `site_key`
6. Check `success`, `score`, `action`, and `user_hint` in the response
7. Monitor usage via `GET /api/v1/usage` and `GET /api/v1/quota`

---

## CORS Policy

- **Inverse CAPTCHA endpoints** (`/v1/*`): Wildcard `Access-Control-Allow-Origin: *` -- the widget is embedded on arbitrary third-party websites.
- **Customer Portal API** (`/api/v1/*`): Restricted to allowed origins in production.
- **Allowed methods**: `GET, POST, PATCH, DELETE, OPTIONS`
- **Allowed headers**: `Content-Type, Authorization, X-Challenge-Signature`
- **Exposed headers**: `X-Challenge-Signature`
- **Max age**: 3600 seconds (1 hour)

---

## Security Headers

All responses include these security headers:

| Header | Value |
|--------|-------|
| `X-Content-Type-Options` | `nosniff` |
| `X-Frame-Options` | `DENY` (except `/embed` which allows framing) |
| `X-XSS-Protection` | `0` (use CSP instead) |
| `Referrer-Policy` | `strict-origin-when-cross-origin` |
| `Content-Security-Policy` | `default-src 'none'; frame-ancestors 'none'` (relaxed for `/embed` and `/demo`) |

Request body size limit: 256 KB for all POST/PUT/PATCH requests.

---

