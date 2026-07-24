# Authentication and Authorization

Language: English | [中文](../后端知识库/16-认证授权专题.md)

---

## Table of Contents

1. [Authentication vs Authorization](#1-authentication-vs-authorization)
2. [Session-Cookie Authentication](#2-session-cookie-authentication)
3. [JWT Authentication](#3-jwt-authentication)
4. [Password Security](#4-password-security)
5. [OAuth 2.0](#5-oauth-20)
6. [SSO and OIDC](#6-sso-and-oidc)
7. [MFA](#7-mfa)
8. [RBAC and ABAC](#8-rbac-and-abac)
9. [API Security](#9-api-security)
10. [Practical System Design](#10-practical-system-design)
11. [Interview Self-Check](#11-interview-self-check)

---

## 1. Authentication vs Authorization

Authentication answers: who are you?

Authorization answers: what are you allowed to do?

```text
identity -> authentication -> principal
principal + policy -> authorization decision
```

Common authentication methods:

- password.
- session cookie.
- token.
- OAuth/OIDC login.
- MFA.
- certificate.

---

## 2. Session-Cookie Authentication

### 2.1 Flow

```text
login -> server creates session -> Set-Cookie session_id -> browser sends cookie -> server looks up session
```

Pros:

- easy revocation.
- server-side control.
- mature browser support.

Cons:

- server-side session storage.
- cross-region/session sharing complexity.
- CSRF risk if not protected.

### 2.2 Distributed Session

Options:

- sticky session.
- centralized Redis session.
- database session.
- stateless token.

Redis is common, but session expiration, eviction, and HA must be designed carefully.

---

## 3. JWT Authentication ⭐⭐⭐

### 3.1 JWT Structure

JWT consists of:

```text
header.payload.signature
```

Payload contains claims such as `sub`, `iss`, `aud`, `exp`, and custom claims.

### 3.2 Signature

The signature protects integrity, not confidentiality. Anyone can decode base64 payload unless encrypted separately.

Common algorithms:

- HS256: shared secret.
- RS256/ES256: asymmetric key.

### 3.3 Access Token + Refresh Token

Access token:

- short-lived.
- used for API requests.

Refresh token:

- longer-lived.
- used to obtain new access tokens.
- should be stored and rotated securely.

### 3.4 JWT Security

Checklist:

- Validate signature.
- Validate `exp`, `iss`, and `aud`.
- Avoid putting sensitive data in payload.
- Use short expiration.
- Rotate keys.
- Support revocation for high-risk scenarios.
- Do not accept `alg=none`.

### 3.5 Token Storage

Browser storage choices:

- HttpOnly Secure SameSite cookie: reduces XSS token theft but needs CSRF protection.
- localStorage: easy but vulnerable to XSS.
- memory: safer but lost on refresh.

---

## 4. Password Security

Never store plaintext passwords.

Use password hashing algorithms:

- bcrypt.
- scrypt.
- Argon2.

Password hash should include salt and suitable cost factor.

Additional controls:

- rate limiting.
- MFA.
- breach password detection.
- account lock or risk-based challenge.

---

## 5. OAuth 2.0

OAuth 2.0 is an authorization framework for delegated access.

Roles:

- Resource Owner.
- Client.
- Authorization Server.
- Resource Server.

Common grant types:

- Authorization Code.
- Authorization Code with PKCE.
- Client Credentials.
- Device Code.
- Refresh Token.

For browser/mobile apps, Authorization Code with PKCE is preferred.

---

## 6. SSO and OIDC

### 6.1 SSO

Single Sign-On lets users log in once and access multiple applications.

### 6.2 CAS, SAML, OIDC

| Protocol | Common Use |
|----------|------------|
| CAS | enterprise web SSO |
| SAML | enterprise identity federation |
| OIDC | modern identity layer on OAuth 2.0 |

OIDC adds identity authentication on top of OAuth 2.0 through ID Token and user info endpoint.

---

## 7. MFA

MFA combines multiple factors:

- something you know: password.
- something you have: device/security key.
- something you are: biometric.

TOTP generates time-based one-time passwords using shared secret and time window.

Recovery codes are important for account recovery.

---

## 8. RBAC and ABAC

### 8.1 RBAC

RBAC models permissions through users, roles, and permissions.

```text
user -> role -> permission
```

Good for stable organization roles.

### 8.2 ABAC

ABAC uses attributes of subject, object, action, and environment.

Example:

```text
allow if user.department == resource.department and action == "read"
```

ABAC is flexible but policy governance is more complex.

### 8.3 Casbin

Casbin is a policy engine supporting RBAC, ABAC, ACL, and custom models.

---

## 9. API Security

### 9.1 HMAC Signature

Common signed request fields:

- app key.
- timestamp.
- nonce.
- body hash.
- signature.

Protects request integrity and helps prevent replay when combined with timestamp and nonce.

### 9.2 Rate Limiting

Apply rate limits by:

- IP.
- user.
- app key.
- tenant.
- endpoint.

### 9.3 CORS

CORS controls which origins can call APIs from browsers. It is not an authentication mechanism.

### 9.4 Security Checklist

- TLS everywhere.
- least privilege.
- strong password hashing.
- token expiration and revocation.
- audit logs.
- secret rotation.
- input validation.
- sensitive data masking.

---

## 10. Practical System Design

Design a complete auth system:

- identity provider.
- login and registration.
- session/token management.
- refresh token rotation.
- password reset.
- MFA.
- RBAC/ABAC authorization.
- audit logs.
- risk control.
- admin console.

Production concerns:

- account takeover protection.
- brute-force defense.
- secret/key management.
- compliance and audit.
- multi-tenant isolation.

---

## 11. Interview Self-Check

### Q1: Authentication vs authorization?

**Answer:** Authentication verifies identity. Authorization checks permissions for an authenticated principal.

### Q2: Session vs JWT?

**Answer:** Session stores state on server and is easy to revoke. JWT is stateless and scalable but revocation and token leakage handling are harder.

### Q3: Why should JWT access tokens be short-lived?

**Answer:** Short lifetime limits damage if leaked and reduces revocation pressure.

### Q4: What is refresh token rotation?

**Answer:** Each refresh replaces the old refresh token with a new one. Reuse of an old token can indicate theft.

### Q5: OAuth2 vs OIDC?

**Answer:** OAuth2 delegates authorization. OIDC adds authentication and identity information on top of OAuth2.

### Q6: Why use PKCE?

**Answer:** PKCE prevents authorization code interception attacks, especially for public clients like mobile apps and SPAs.

### Q7: What is RBAC?

**Answer:** Role-Based Access Control assigns permissions to roles and roles to users.

### Q8: What is ABAC?

**Answer:** Attribute-Based Access Control makes decisions based on subject, object, action, and environment attributes.

### Q9: How does HMAC API signing prevent tampering?

**Answer:** The server recomputes signature from request fields and shared secret. If the request is modified, signature verification fails.

### Q10: What are common auth system risks?

**Answer:** XSS token theft, CSRF, weak password hashing, missing token validation, replay attacks, over-permissive roles, missing audit logs, and poor key rotation.

### Senior Interview Follow-Ups

### Q11: How do you respond to suspected refresh token theft?

**Answer:** Revoke the affected token family, invalidate active sessions for the user or risk scope, rotate signing keys only if key compromise is suspected, and preserve audit evidence. Check reuse detection, impossible travel, device fingerprint changes, IP reputation, and recent password/MFA changes. Recovery should include user notification, step-up authentication, forced password reset when appropriate, and metrics for similar patterns across tenants.

### Q12: How would you design permission checks for a multi-tenant SaaS?

**Answer:** Model tenant boundary as a first-class authorization dimension. Every permission decision should include subject, tenant, resource, action, and context. Use RBAC for coarse roles and ABAC/policy rules for resource ownership, environment, or risk-based conditions. Enforce checks server-side near business operations, not only at the gateway, and add audit logs, policy tests, and least-privilege admin tooling.

### Q13: What are the trade-offs between stateless JWT and centralized session/token introspection?

**Answer:** Stateless JWT scales well and reduces central lookup latency, but revocation, claim freshness, and leakage response are harder. Centralized sessions or token introspection give stronger control and immediate revocation, but add dependency latency and availability risk. Production systems often combine short-lived access tokens with refresh token rotation, key rotation, risk-based revocation, and cacheable introspection for high-risk APIs.
