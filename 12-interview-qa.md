[← Back to README](README.md)

# 12 — Interview Q&A

> 65 curated questions with concise, interview-ready answers. Use as a final review before an interview.

---

## Table of Contents
- [Authentication vs Authorization](#authentication-vs-authorization)
- [Session Management](#session-management)
- [JWT](#jwt)
- [OAuth 2.0](#oauth-20)
- [OpenID Connect (OIDC)](#openid-connect-oidc)
- [SAML](#saml)
- [Single Sign-On (SSO)](#single-sign-on-sso)
- [MFA / 2FA](#mfa--2fa)
- [Identity Concepts](#identity-concepts)
- [Architecture and Design](#architecture-and-design)
- [Security Attacks and Mitigations](#security-attacks-and-mitigations)

---

## Authentication vs Authorization

**Q1: What is the difference between authentication and authorization?**

Authentication verifies *who you are* (identity). Authorization determines *what you can do* (permissions). Authentication always happens first. HTTP 401 means authentication failed or missing; HTTP 403 means authenticated but not authorized.

---

**Q2: What is the difference between 401 Unauthorized and 403 Forbidden?**

`401 Unauthorized` means the request lacks valid authentication credentials — the server does not know who you are (or credentials are invalid). `403 Forbidden` means the server knows who you are (authenticated) but you don't have permission to access the resource (not authorized). The name "Unauthorized" in 401 is a historical misnomer — it really means "unauthenticated."

---

**Q3: Explain RBAC. What is role explosion and how do you prevent it?**

**RBAC** (Role-Based Access Control): Users are assigned roles, roles have permissions, users inherit permissions through roles.

**Role explosion** occurs when fine-grained access requirements lead to creating hundreds of roles (e.g., `editor-region-a`, `editor-region-b`, `viewer-region-a`...). Prevention strategies:
- Use hierarchical RBAC (roles inherit from parent roles)
- Add parameterization (one `editor` role with a region attribute → ABAC)
- Regularly audit and consolidate roles
- For very complex requirements, move to ABAC

---

**Q4: What is ABAC? When would you choose it over RBAC?**

**ABAC** (Attribute-Based Access Control): Access decisions are based on attributes of the subject (user), resource, action, and environment — evaluated at request time.

Choose ABAC over RBAC when:
- Access depends on context (time of day, location)
- Access depends on data attributes ("only users in the same department as the document")
- Role explosion is occurring
- Policies need to be dynamic and composable

Example: "Allow access if user.department == resource.ownerDepartment AND currentTime.isBusinessHours"

---

**Q5: What is BOLA (Broken Object Level Authorization)?**

BOLA (OWASP API Security Top 1) is when an API lets authenticated users access other users' objects by simply changing an ID in the request.

```
GET /api/invoices/12345  ← user's own invoice
GET /api/invoices/12346  ← another user's invoice — server doesn't check ownership!
```

Prevention: Always verify the authenticated user has a relationship (ownership, permission) to the specific object being accessed, not just that they're authenticated.

---

## Session Management

**Q6: What is a session cookie vs a persistent cookie?**

A **session cookie** has no `Expires` or `Max-Age` attribute — it is deleted when the browser closes. A **persistent cookie** has an expiry time set and survives browser restarts. Session cookies are safer for auth (less exposure time) but less convenient (user must re-login on each browser session).

---

**Q7: Explain the CSRF attack and how to mitigate it.**

**CSRF (Cross-Site Request Forgery)**: A malicious website tricks the user's browser into making a request to a target site where the user is authenticated. Since the browser automatically sends cookies, the target site sees a valid authenticated request.

Example: Malicious page has `<form action="https://bank.com/transfer" method="POST">` which auto-submits.

**Mitigations**:
1. `SameSite=Strict` or `SameSite=Lax` cookie attribute — browser won't send cookie on cross-site requests
2. CSRF token — server-generated secret embedded in forms, verified on submission (attacker can't read it due to same-origin policy)
3. Custom request headers (e.g., `X-Requested-With`) — only possible same-origin without CORS preflight
4. Origin/Referer header validation

CSRF does not apply to `Authorization: Bearer` token-based APIs because browsers don't auto-send Authorization headers cross-site.

---

**Q8: Where should you store JWT tokens in a browser application and why?**

| Storage | Verdict |
|---------|---------|
| `localStorage` | **Avoid for sensitive tokens** — accessible to JavaScript, vulnerable to XSS |
| `sessionStorage` | Same risks as localStorage |
| HttpOnly cookie | **Preferred** — inaccessible to JavaScript, needs CSRF protection |
| In-memory (JS variable) | Good security, lost on page refresh — needs silent refresh |

**Recommended**: HttpOnly cookie (especially with BFF pattern). For pure SPAs calling APIs: in-memory + short-lived tokens with silent refresh.

---

**Q9: What are the trade-offs between stateful and stateless authentication?**

| | Stateful (sessions) | Stateless (JWT) |
|--|--------------------|----|
| Revocation | Instant (delete session) | Delayed (wait for expiry or use blocklist) |
| Horizontal scaling | Needs shared session store (Redis) | No shared state needed |
| Request cost | Session store lookup per request | Signature verification (CPU) only |
| Logout | Reliable | "Best effort" unless blocklist |
| Best for | Traditional web apps | APIs, SPAs, microservices |

---

**Q10: How do you implement logout for JWT-based auth?**

1. **Client-side only** (weak): Delete the token from storage. Token technically still valid until `exp`. Risk: stolen token still works.
2. **Token blocklist**: Maintain a Redis set of revoked `jti` values. Check on every request. Re-introduces statefulness but supports true revocation.
3. **Short-lived access tokens + refresh token revocation** (recommended): Keep access tokens at 5-15 minutes. On logout, revoke the refresh token (mark as revoked in DB). Attacker with stolen access token can only abuse it for ≤15 minutes.
4. **Federated logout / SLO**: Send logout request to IdP to also clear the IdP session.

---

## JWT

**Q11: What are the three parts of a JWT?**

1. **Header**: Base64URL-encoded JSON with `alg` (signing algorithm) and `typ` ("JWT"), optionally `kid` (key ID for key rotation)
2. **Payload**: Base64URL-encoded JSON with claims (sub, iss, aud, exp, iat, custom claims)
3. **Signature**: Cryptographic signature of `base64url(header) + "." + base64url(payload)` using the algorithm in the header

Separated by dots: `header.payload.signature`

---

**Q12: Is JWT payload encrypted? What are the security implications?**

**No**, the JWT payload is only Base64URL-encoded (reversible), not encrypted. Anyone who obtains the token can decode and read all claims.

**Implication**: Never put sensitive information (passwords, PII, secrets) in a JWT payload unless using **JWE** (JSON Web Encryption).

**JWE** uses encryption to make the payload confidential — only the intended recipient with the private key can read it. Standard JWTs (JWS) only provide integrity (tamper detection), not confidentiality.

---

**Q13: What is the difference between HS256 and RS256? Which would you use in production?**

| | HS256 | RS256 |
|--|-------|-------|
| Type | Symmetric (HMAC) | Asymmetric (RSA) |
| Signing key | Shared secret | Private key |
| Verification key | Same shared secret | Public key (JWKS) |
| Key exposure risk | Any verifier knows the signing key | Only issuer knows private key |
| Distribution | Manual secret sharing | Publish public key (JWKS endpoint) |

**Production**: RS256 or ES256. Multiple services can verify tokens without knowing the signing key. If any service is compromised, the private key is still safe.

**Use HS256 only when**: Issuer and verifier are the same system (e.g., single monolith using JWT for session).

---

**Q14: What is the `alg: none` attack?**

An attacker modifies the JWT header to `"alg": "none"` and removes the signature. Vulnerable JWT libraries that accept `none` as a valid algorithm treat the token as unsigned — and if they don't check the signature requirement, accept any payload the attacker crafts.

**Prevention**: Explicitly whitelist algorithms on the server. Never trust the `alg` field in the token header — verify only against your configured expected algorithm.

---

**Q15: What JWT claims must a service validate?**

1. **Signature** — verify against the signing key
2. **`alg`** — must match your whitelist (reject `none`, unknown algorithms)
3. **`exp`** — must be in the future (with small clock skew tolerance)
4. **`nbf`** — must be in the past (if present)
5. **`iss`** — must match your expected issuer
6. **`aud`** — must include your service's identifier
7. **`jti`** — check not in blocklist (for high-security systems needing replay prevention)

For OIDC ID tokens, also validate: `nonce`, `at_hash`, `auth_time` if max_age was requested.

---

**Q16: How does JWT key rotation work?**

1. Generate new key pair, assign new `kid`
2. Publish new key in JWKS endpoint alongside old key (two keys active)
3. Start signing new tokens with new `kid`
4. Wait for existing tokens signed with old `kid` to expire (typically 15-60 min)
5. Remove old key from JWKS

Clients should: look up `kid` from token header → find matching key in JWKS → verify signature. On `kid` not found → refresh JWKS (handles key rotation). Cache JWKS with short TTL.

---

## OAuth 2.0

**Q17: What is OAuth 2.0 and what problem does it solve?**

OAuth 2.0 is a **delegated authorization framework** that lets an application (client) access resources on behalf of a user, without the user sharing their credentials with that application.

Example: A photo editing app can access your Google Photos without you giving it your Google password. You authorize at Google, Google issues a scoped access token to the app.

**It does NOT solve authentication** — it doesn't tell you who the user is. That's what OIDC adds.

---

**Q18: Explain the Authorization Code + PKCE flow step by step.**

1. App generates `code_verifier` (random string, 43-128 chars, URL-safe)
2. App computes `code_challenge = BASE64URL(SHA256(code_verifier))`
3. App redirects user to: `GET /authorize?response_type=code&client_id=...&code_challenge=X&code_challenge_method=S256&state=RANDOM`
4. User authenticates at Authorization Server
5. AS redirects back: `GET /callback?code=AUTH_CODE&state=RANDOM`
6. App verifies `state` matches (prevents CSRF)
7. App sends: `POST /token {grant_type=authorization_code, code=AUTH_CODE, code_verifier=ORIGINAL_VERIFIER}`
8. AS verifies: `SHA256(code_verifier) == code_challenge` stored from step 2
9. AS issues: `access_token + refresh_token + id_token (if OIDC)`

**PKCE prevents**: Code interception — attacker intercepts `AUTH_CODE` but can't exchange it without the `code_verifier`.

---

**Q19: What is the `state` parameter and what does it prevent?**

`state` is a random, unguessable value generated by the client and included in the authorization request. The AS returns it unchanged in the callback. The client verifies it matches before proceeding.

**Prevents**: CSRF on the OAuth callback. Without state, an attacker could craft a callback URL with their own auth code, tricking the user's app into using the attacker's session.

---

**Q20: When would you use Client Credentials vs Authorization Code grant?**

- **Client Credentials**: Machine-to-machine (no user). Background jobs, microservices calling other services, CI/CD pipelines. The client authenticates as itself.
- **Authorization Code + PKCE**: Any flow where a human user authorizes access. Web apps, SPAs, mobile apps.

---

**Q21: Why is the Implicit grant deprecated?**

The Implicit grant returned the access token directly in the URL fragment after the redirect. Problems:
1. Token visible in browser URL, history, and server logs
2. No way to verify the token was intended for this client (no client authentication)
3. Replaced by Authorization Code + PKCE, which is more secure and works for all client types

OAuth 2.1 removes the Implicit grant entirely.

---

**Q22: What is refresh token rotation and why does it improve security?**

Each time a refresh token is used, it is invalidated and a new one is issued. If a refresh token is stolen, the attacker uses it once. When the legitimate client later tries to use the (now-invalidated) refresh token, the AS detects reuse — a signal of compromise — and revokes the entire token family. The attacker loses access.

Without rotation: stolen refresh token is valid indefinitely until manually revoked.

---

**Q23: What is token introspection and when would you use it over local JWT validation?**

**Token introspection** (RFC 7662): Resource server calls the Authorization Server to verify a token's validity and get its metadata.

Use introspection when:
- Tokens are **opaque** (not JWTs) — resource server can't validate locally
- **Immediate revocation** is required — local JWT validation can't detect early revocation
- You need fresh metadata not in the token

Use local JWT validation when:
- Tokens are JWTs with a short TTL — validation is stateless and fast
- Performance at scale matters — no external call per request
- Revocation can be accepted to work with short token TTL

---

## OpenID Connect (OIDC)

**Q24: What is the difference between OAuth 2.0 and OIDC?**

OAuth 2.0 is an authorization framework — it grants limited access to resources but does not define user identity. OIDC is an identity layer built on top of OAuth 2.0 that adds:
- **ID Token** (JWT) containing verified user identity claims
- **UserInfo endpoint** for additional user profile data
- **Discovery document** for auto-configuration
- Standard scopes (`openid`, `profile`, `email`)
- Standardized user identifier (`sub` claim)

Simply: OAuth 2.0 says "you can access this." OIDC says "this is who the user is."

---

**Q25: What is the ID Token and how is it different from an access token?**

| | ID Token | Access Token |
|--|----------|-------------|
| Purpose | Identify the user — consumed by the RP | Access APIs — consumed by the RS |
| Format | Always JWT | JWT or opaque |
| Audience | Your client_id | The Resource Server |
| Who validates | Your application (RP) | The API (RS) |
| Should call APIs with it? | **No** | **Yes** |

The ID token proves the user is who they say they are. The access token proves you have permission to call the API.

---

**Q26: Why should you use `sub` and not `email` as a user identifier in your database?**

- `sub` is a stable, permanent, unique identifier per user per IdP — it never changes
- `email` can change (user changes their email address)
- `email` might be reused (previous user deleted, new user with same email)
- Using `sub` as the primary key prevents account hijacking via email reassignment

Store `{iss, sub}` tuple as the unique identifier (the same `sub` value might appear across different IdPs).

---

**Q27: What is the OIDC discovery document?**

A JSON document published at `https://{issuer}/.well-known/openid-configuration` that describes the OpenID Provider's capabilities and endpoints:
- `authorization_endpoint`, `token_endpoint`, `userinfo_endpoint`
- `jwks_uri` (where to get public keys)
- Supported scopes, response types, grant types
- Supported claims

OIDC clients can **auto-configure** by fetching this document — they only need to know the issuer URL. This enables zero-configuration IdP integration.

---

**Q28: What is the nonce in OIDC?**

A random, unguessable value generated by the RP, included in the authorization request, and embedded in the ID token by the OP. The RP verifies that the nonce in the returned ID token matches the one it sent.

**Prevents**: ID token replay attacks. An attacker who captures an ID token cannot use it for a different authorization session because the nonce won't match.

After the nonce is used, mark it as consumed — reject future tokens with the same nonce.

---

## SAML

**Q29: What is a SAML Assertion and what are its three parts?**

A SAML Assertion is an XML document issued by the IdP asserting facts about a user:

1. **Authentication Statement**: How and when the user authenticated (`AuthnContextClassRef`)
2. **Attribute Statement**: User attributes (email, name, groups, roles)
3. **Authorization Decision Statement**: Access decision for a specific resource (rarely used in practice)

---

**Q30: What is the difference between SP-initiated and IdP-initiated SSO?**

**SP-initiated**: User goes to the SP (e.g., Salesforce) → SP redirects to IdP with `AuthnRequest` → IdP authenticates → IdP sends `SAMLResponse` back to SP. SP can validate the `InResponseTo` value against the original request.

**IdP-initiated**: User starts from the IdP's portal → IdP sends `SAMLResponse` to SP without a prior `AuthnRequest`. There is no `InResponseTo` to check.

**Security difference**: IdP-initiated is less secure because there's no original request to bind to. Attackers can potentially forge or replay assertions. Some SPs refuse IdP-initiated SSO.

---

**Q31: What is the ACS URL?**

**Assertion Consumer Service (ACS) URL** is the SP endpoint that receives SAML Responses via HTTP POST. It must be:
- Registered in the SP's own configuration (where to listen)
- Registered in the IdP's SP configuration (where to send assertions)
- Used verbatim in the `Recipient` field of the assertion

A mismatch between the `Recipient` in the assertion and the actual receiving URL indicates a potential attack and the SP must reject the assertion.

---

**Q32: What is an XML Signature Wrapping (XSW) attack?**

An attacker exploits the difference between the element that was signed and the element used by the application:

```xml
<outer_assertion ID="evil" user="attacker">
  <inner_assertion ID="legit" user="alice">
    <Signature>validates ID=legit</Signature>
  </inner_assertion>
</outer_assertion>
```

A vulnerable SAML library validates the signature (which is valid for `inner_assertion`) but then uses the `outer_assertion` (unsigned). The attacker gets access as "attacker" while the signature validates.

**Prevention**: After verifying signature on element with ID X, use *only* that element. Use robust SAML libraries. Avoid custom XML parsing.

---

## Single Sign-On (SSO)

**Q33: How does SSO work mechanically?**

The IdP maintains its own authenticated session (an HttpOnly cookie on the IdP's domain). When App 1 redirects to the IdP for authentication and the user logs in, the IdP sets its session cookie. When App 2 later redirects to the same IdP, the browser sends the IdP session cookie. The IdP sees the active session, issues an identity assertion for App 2 without prompting the user, and redirects back to App 2.

The key: IdP session is the shared state. Apps have their own shorter-lived local sessions that are created/refreshed via the IdP.

---

**Q34: What is the difference between IdP session and application session in SSO?**

| | IdP Session | Application Session |
|--|------------|---------------------|
| Domain | IdP's domain (e.g., okta.com) | Application's domain |
| Lifetime | Longer (hours–days; configurable) | Shorter (minutes–hours) |
| Managed by | IdP | Each application |
| Content | User's authenticated state | Local user context |

When the IdP session expires, the next time any app needs authentication, the user must re-authenticate at the IdP. This is the "global" session timeout. App sessions can expire independently (idle timeout) and be silently refreshed via `prompt=none`.

---

**Q35: What are the security risks of SSO and how do you mitigate the "single point of failure"?**

| Risk | Mitigation |
|------|-----------|
| IdP downtime = no one logs in | IdP HA, multi-region, fallback auth (break-glass) |
| Compromised IdP credential = all apps compromised | Strong MFA on IdP, privileged access workstations, conditional access |
| Zombie sessions after IdP logout | Back-channel logout; short app session TTLs |
| SSO bypass via fallback login | Disable local passwords for SSO-enabled accounts |

---

**Q36: What is JIT provisioning in SSO?**

**Just-in-Time (JIT) provisioning** automatically creates a user account in the SP the first time that user authenticates via SSO, using attributes from the identity assertion.

```
First SSO login:
  Assertion: {email: alice@acme.com, name: "Alice Smith", groups: ["dev"]}
  SP: No user with sub=xxx found
  SP: Create user {email, name, groups} from assertion
  SP: Create session
```

Alternative: **SCIM** (System for Cross-domain Identity Management) — the IdP proactively pushes user create/update/delete events to the SP via REST API. More reliable than JIT for large organizations.

---

## MFA / 2FA

**Q37: What is the difference between TOTP and HOTP?**

Both are OTP standards (RFC 6238 and RFC 4226 respectively), both use `HMAC(secret, counter)`:

| | TOTP | HOTP |
|--|------|------|
| Counter | Time-based (30-second windows) | Incrementing integer |
| Trigger | Automatic (every 30s) | Button press |
| Validity window | 30 seconds (+ clock skew) | Until next code pressed |
| Synchronization | Requires time sync | Counter must stay in sync |
| Used in | Software authenticator apps | Hardware tokens (YubiKey OTP) |

---

**Q38: Why is SMS OTP considered weak for MFA?**

1. **SIM swapping**: Attacker convinces carrier to port victim's number to a new SIM. They receive all SMS.
2. **SS7 vulnerabilities**: Telecom protocol weaknesses allow intercepting SMS in transit.
3. **Real-time phishing**: Attacker's site relays the OTP to the real site within seconds of the victim receiving it.
4. **Social engineering**: Attacker calls victim claiming to be support, asks them to share the code.

NIST SP 800-63B no longer recommends SMS OTP as a sole authentication factor for high-assurance systems.

---

**Q39: How does FIDO2 prevent phishing? Explain the mechanism.**

FIDO2 binds the credential cryptographically to the **Relying Party ID** (the origin/domain). When the authenticator signs a challenge, it includes the `rpId` (domain) in the signed data.

If a user is on `https://fakebank.com` (phishing), their browser sends `rpId = fakebank.com` to the authenticator. The authenticator only has a key registered for `bank.com` — no matching key exists for `fakebank.com`, so it refuses to sign. The phishing attack fails even if the user doesn't notice the fake site.

This is fundamentally different from TOTP where the code is domain-agnostic and can be relayed to any site.

---

**Q40: What is MFA fatigue and how does number matching mitigate it?**

**MFA fatigue**: Attacker obtains the victim's password and repeatedly sends push notification MFA requests. The user, annoyed by constant notifications, eventually taps "Approve" — granting access.

**Number matching**: The push notification shows a number (e.g., "42"). The user must type that same number on their phone to approve. The number is shown on the login screen the attacker controls. Even if the user receives a push, they see no number context on their phone — they must look at the login screen to approve, making accidental approval much harder.

Other mitigations: Geographic context in push ("Login from Bucharest, Romania — is this you?"); rate limiting push requests.

---

**Q41: What is step-up authentication?**

Step-up authentication requires re-authentication at a higher assurance level for sensitive operations, even if the user is already logged in.

Example: User is logged in with password (AAL1). They try to initiate a wire transfer. The app requires TOTP or biometric verification (AAL2) for this specific action.

In OIDC, request step-up using `acr_values` or `max_age` parameters. The `amr` claim in the returned ID token indicates what methods were used.

---

## Identity Concepts

**Q42: What is an Identity Provider (IdP)?**

An IdP is a system that:
1. Manages user identities (credentials, attributes, lifecycle)
2. Authenticates users when they request access
3. Issues trusted assertions (SAML assertion, OIDC ID token) about authenticated users to Service Providers

Examples: Okta, Azure AD/Entra ID, Google Identity, Auth0, Keycloak, ADFS.

---

**Q43: What is the difference between LDAP and Active Directory?**

**LDAP** (Lightweight Directory Access Protocol) is a protocol for reading/writing to a directory service. It defines the message format and operations (search, bind, add, modify, delete).

**Active Directory (AD)** is Microsoft's directory service product that implements LDAP as one of its protocols. AD adds Kerberos (for authentication), DNS, Group Policy, replication between Domain Controllers, and many enterprise features. AD is a product; LDAP is a protocol.

"Using LDAP" usually means querying a directory (AD or OpenLDAP) using the LDAP protocol.

---

**Q44: What is the difference between Azure AD (Entra ID) and on-premises Active Directory?**

| | On-Prem AD | Azure AD / Entra ID |
|--|-----------|---------------------|
| Auth protocol | Kerberos, NTLM, LDAP | OIDC, OAuth 2.0, SAML |
| Access type | Domain-joined computers on network | Cloud SaaS via browser/HTTPS |
| Token type | Kerberos tickets | JWTs |
| MFA | RADIUS + NPS (complex) | Native (built-in) |
| Sync | Source of truth | Can sync from on-prem AD via Azure AD Connect |
| Conditional Access | Limited (via GPO) | Rich ABAC-style policies |

They are separate products. Most enterprises run both, synchronized via Azure AD Connect.

---

**Q45: What is identity federation?**

Federation is a trust relationship where one organization accepts authentication assertions from another organization's IdP. Users from Organization A can access Organization B's applications without creating separate accounts in Org B. Trust is established via metadata exchange (SAML) or OIDC client registration.

Benefit: Single identity across organizational boundaries; centralized identity management; no credential sprawl.

---

## Architecture and Design

**Q46: How would you design auth for a SaaS product supporting 100 enterprise customers, each with their own IdP?**

1. **Multi-IdP support**: Support both SAML 2.0 and OIDC per tenant (enterprises may use either)
2. **Tenant configuration store**: Map each customer's email domains to their IdP config (SSO URL, certificate, client_id, etc.)
3. **Email domain-based routing**: Detect IdP from email domain at login; route to correct IdP
4. **JIT provisioning** or **SCIM**: Create users on first SSO login or via SCIM push
5. **Attribute mapping**: Allow tenants to configure attribute mapping (their group names → your role names)
6. **SSO enforcement**: Option for tenants to require SSO and disable email/password login
7. **Admin fallback**: Break-glass admin access for tenant admins if IdP is unavailable
8. **Audit logs**: Log all SSO events per tenant

---

**Q47: How do microservices authenticate each other?**

Three main approaches:
1. **JWT from Authorization Server**: Each service gets a short-lived JWT (Client Credentials grant). Downstream services validate JWT signature against JWKS. Stateless, scales well.
2. **mTLS (Mutual TLS)**: Each service has an X.509 certificate. Services verify each other's identity via TLS handshake. Used in service meshes (Istio, Linkerd). Strong but operationally complex.
3. **API Gateway trust**: Gateway validates external tokens, injects trusted headers. Internal services trust headers without re-validating. Simpler but requires all traffic to go through gateway; internal network must be protected.

Best practice: Combine mTLS for transport security + JWT for identity propagation.

---

**Q48: What is the BFF (Backend-for-Frontend) pattern for auth?**

The BFF is a server-side component (per client type) that handles OAuth flows on behalf of browser clients:
1. Browser talks to BFF via session cookie
2. BFF performs OAuth Authorization Code flow (with client secret it can protect)
3. BFF stores tokens server-side or in HttpOnly cookie
4. BFF uses access token to call downstream APIs
5. Tokens never reach the browser's JavaScript

**Why it matters**: XSS in the SPA cannot steal tokens. BFF can rotate tokens, enforce authorization, aggregate APIs.

---

**Q49: In a multi-tenant app, how do you prevent tenant A from accessing tenant B's data?**

1. **JWT claim**: Include `tenant_id` in access token. Every API call must verify `tenant_id` in token matches the resource's tenant.
2. **Row-level security**: Database enforces tenant isolation at the query level (PostgreSQL RLS, Hibernate filters).
3. **Separate databases/schemas**: Strongest isolation, highest cost.
4. **API gateway routing**: Route tenant A traffic to tenant A's isolated backend.

Always enforce isolation in the backend — never trust client-provided tenant IDs.

---

**Q50: When would you use SAML vs OIDC in 2026?**

Use **SAML** when:
- The SP (target application) only supports SAML (legacy enterprise apps: Salesforce classic, older ServiceNow)
- Enterprise customer explicitly requires SAML
- Government/compliance mandates SAML
- Existing SAML infrastructure is in place

Use **OIDC** when:
- Building new integrations
- Mobile or SPA clients
- API-based services
- Modern IdPs (all support OIDC)
- Developer-friendly setup is a priority

In practice: **support both** in B2B SaaS. Enterprises vary widely — some require SAML, most accept OIDC.

---

**Q51: What is token exchange (RFC 8693) and when would you use it?**

Token exchange allows a service to request a new token from the Authorization Server, with a different audience or subject, based on an existing token.

Use cases:
1. **Service A calls Service B on user's behalf**: Exchange user's token for a token scoped to Service B
2. **Impersonation**: Privileged service acts as a specific user (with AS consent)
3. **Delegation**: Service A acts on behalf of Service B which acts on behalf of user (full chain)
4. **Cross-domain federation**: Exchange a partner's token for your system's token

This preserves the identity chain across service calls, enabling audit trails of who originated a request.

---

## Security Attacks and Mitigations

**Q52: Describe the open redirect vulnerability in OAuth and how to prevent it.**

**Attack**: Attacker registers a malicious `redirect_uri` (or exploits a wildcard/prefix match) and tricks the AS into redirecting the authorization code to a site they control.

```
/authorize?client_id=legit&redirect_uri=https://attacker.com&response_type=code
```

AS redirects with `?code=AUTH_CODE` to attacker's site → attacker exchanges code for tokens.

**Prevention**:
- AS must enforce **exact match** of `redirect_uri` against pre-registered values (no wildcards, no prefix matching)
- OAuth 2.1 mandates exact match
- Validate registered redirect URIs are on known secure domains

---

**Q53: An attacker performs an alg:none attack on your JWT validation. How does it work and how do you prevent it?**

Attacker modifies JWT header to `{"alg":"none","typ":"JWT"}`, changes payload claims to whatever they want (e.g., admin roles), and removes the signature. Some vulnerable libraries, when they see `alg=none`, skip signature validation entirely and accept the token.

**Prevention**: Hardcode the expected algorithm on the server. Never read `alg` from the incoming token header to choose how to validate. Use well-maintained JWT libraries. Enable explicit algorithm whitelisting.

---

**Q54: What is a CSRF attack and how does SameSite=Lax protect against it without a CSRF token?**

In CSRF, a malicious site triggers a state-changing request to a vulnerable site. The browser automatically sends the victim's cookies.

`SameSite=Lax` instructs the browser to only send the cookie when the request originates from:
- Top-level navigation GET requests (clicking a link)
- Same-site requests

The cookie is **not sent** on cross-site POST requests, XHR, or fetch calls — which are the typical CSRF attack vectors. Since the auth cookie isn't sent, the server sees an unauthenticated request.

Limitation: `SameSite=Lax` doesn't protect GET requests. Any state-changing operations must use POST/PUT/DELETE (which they should anyway per REST principles).

---

**Q55: What is credential stuffing and how do you detect and prevent it?**

**Credential stuffing**: Using lists of leaked username:password pairs (from other breaches) to log into other services, exploiting password reuse.

**Detection**:
- Unusual spike in login attempts
- Many failed logins followed by a successful one
- Logins from multiple IPs in rapid succession
- ASN/country anomalies

**Prevention**:
- Rate limiting + exponential backoff on login endpoint
- CAPTCHA after failed attempts
- Check passwords against breach databases (HIBP API)
- MFA — even correct credential can't log in without second factor
- Anomaly detection / behavioral analytics
- Bot detection (fingerprinting, browser behavior analysis)
- Leaked credential monitoring

---

**Q56: What is the difference between authentication injection and SAML XML Signature Wrapping?**

**Authentication injection**: Broadly, any attack where an attacker injects forged authentication data. Includes injecting fake auth headers, forging session cookies, etc.

**SAML XSW** is a specific, XML-parsing-based attack unique to SAML: the attacker exploits a disconnect between which XML element is verified (by the signature) and which element is consumed (by the application) by cleverly wrapping elements. The signature is technically valid, but the data used is not what was signed.

→ See [07 — SAML](07-saml.md#saml-security-considerations) for the detailed attack structure.

---

**Q57: How does PKCE prevent an authorization code interception attack?**

Without PKCE: An attacker intercepts the authorization code (via URL logging, redirect manipulation). They can exchange it for tokens because the token endpoint only requires the code + client_id (public client has no secret).

With PKCE:
1. Client generates `code_verifier` (secret) + sends `code_challenge = SHA256(code_verifier)` to AS
2. AS stores the `code_challenge` bound to this authorization request
3. Attacker intercepts `code` — but doesn't know `code_verifier`
4. Attacker tries to exchange code: `POST /token {code=stolen, code_verifier=???}` — they can't provide the right `code_verifier`
5. Client exchanges: `POST /token {code=code, code_verifier=CORRECT}` → AS computes `SHA256(CORRECT) == stored_challenge` ✓

---

**Q58: Why do short-lived access tokens improve security?**

JWTs are stateless — once issued, they're valid until expiry. There's no built-in revocation (unlike sessions). If a token is stolen:
- 24-hour token → attacker has 24 hours of access
- 5-minute token → attacker has at most 5 minutes

Combined with refresh token rotation:
- Attacker needs both the access token AND the refresh token to maintain access
- Stolen access token expires in minutes
- Stolen refresh token is detected on first reuse (via rotation)

---

**Q59: What is SCIM and how does it relate to SSO?**

**SCIM (System for Cross-domain Identity Management)** is a REST API standard for automating user provisioning and deprovisioning between IdP and SP. While SSO handles authentication (login), SCIM handles the user lifecycle:
- **Create**: New employee added to IdP → SCIM pushes user to all connected SPs
- **Update**: User's role changes → SCIM updates attributes in SPs
- **Delete/Deactivate**: Employee leaves → SCIM deprovisions across all SPs

Without SCIM: You rely on JIT provisioning (create on first login) but have no way to delete users automatically → security risk (ex-employees retain access until they try to log in).

---

**Q60: What is Kerberos and how does it achieve SSO in an enterprise?**

Kerberos is a ticket-based authentication protocol for network services. The Key Distribution Center (KDC) is the trusted third party:

1. User logs in to Windows → AS (part of KDC) issues a **TGT** (Ticket Granting Ticket) — valid 8-10 hours
2. User accesses a service → client presents TGT to TGS (Ticket Granting Service) → gets a **service ticket** for that service
3. Service validates the service ticket → grants access without seeing the password

**SSO mechanism**: The TGT is cached on login. Steps 2-3 happen transparently. User logs in once → all Kerberos-aware services authenticate automatically via service tickets.

**SPNs** (Service Principal Names) identify services in Kerberos. Domain-joined web apps register SPNs → browsers present Kerberos tickets → Windows Integrated Authentication (transparent SSO).

---

**Q61: What is the principle of least privilege and how do you apply it in microservices?**

**Principle of Least Privilege**: Every entity (user, service, process) should have only the minimum permissions needed to perform its function, nothing more.

In microservices:
- Each service gets its own **service account** with only the scopes/permissions it needs
- A read-only reporting service should not have write scopes
- Services communicate with specific, scoped JWTs — not admin tokens
- Database: Each service connects with a DB user that has only `SELECT` on its tables (or `INSERT/UPDATE` on its own tables only)
- **Token scopes**: OAuth scopes enforce least privilege — `read:orders` not `admin:all`
- **RBAC/ABAC**: Define minimum roles needed for each service function

---

**Q62: You find an API endpoint `/api/v1/users/{id}/documents` that returns documents. What authorization checks should be in place?**

Minimum checks:
1. **Authentication**: Is the caller authenticated? Valid token?
2. **User exists**: Does `{id}` correspond to a real user?
3. **Object-level authorization (BOLA)**: Is the authenticated caller the user with `{id}`, OR do they have explicit permission to view this user's documents? (Don't just check they're authenticated — check they're authorized for this specific resource)
4. **Tenant isolation (if multi-tenant)**: The `{id}` user must belong to the same tenant as the caller
5. **Scope check**: Does the token have `read:documents` scope?
6. **Resource filtering**: Even with access, filter out documents the user shouldn't see (e.g., classified documents for non-cleared users)

---

**Q63: What is the difference between front-channel and back-channel in OIDC/SAML?**

**Front-channel**: Communication via the user's browser (redirects, form POSTs). Browser is in the middle — messages are visible to the user and stored in browser history.

**Back-channel**: Direct server-to-server communication, bypassing the browser. More reliable, not visible to the user, can deliver richer payloads.

| | Front-channel | Back-channel |
|--|-------------|-------------|
| Via | Browser redirect | HTTPS server-to-server |
| Reliability | Depends on browser | More reliable |
| For logout | May fail (pop-up blocked, user closed tab) | More reliable |
| Examples | SAML POST binding, OIDC front-channel logout | SAML artifact binding, OIDC back-channel logout |

---

**Q64: What is the "confused deputy" problem in OAuth?**

A **confused deputy** is a security problem where a program (the deputy) with permissions is manipulated by a less-privileged caller to use those permissions on the caller's behalf in unintended ways.

In OAuth context: A service (Service A) has an access token from a user. A malicious attacker tricks Service A into making API calls on their behalf using that token — the server is the "confused deputy."

**Prevention**: 
- Service A should only call APIs explicitly authorized in the user's original flow
- Token exchange (RFC 8693) creates new, scoped tokens for specific delegated purposes
- Validate `audience` and `scope` at every service

---

**Q65: Walk me through all the steps needed to validate an incoming OIDC ID token.**

1. **Parse** the three Base64URL-encoded parts
2. **Decode header** → get `alg` and `kid`
3. **Check alg** is in your whitelist (reject `none` and unknown algorithms)
4. **Fetch JWKS** from the OP's JWKS endpoint (or use cached copy); find key with matching `kid`
5. **Verify signature** using the public key and declared algorithm
6. **Check `iss`** matches the expected OpenID Provider issuer
7. **Check `aud`** includes your `client_id`
8. **Check `exp`** — current time must be before expiry (allow ±60 seconds clock skew)
9. **Check `nbf`** (if present) — current time must be after "not before"
10. **Check `iat`** — issued-at time should not be unreasonably far in the past
11. **Verify `nonce`** matches the one you sent in the authorization request (prevents replay)
12. **Verify `at_hash`** (if present and using hybrid flow) — validates binding to access token
13. **Check `auth_time`** — if `max_age` was requested, verify authentication was recent enough
14. **Extract claims** and use `sub` + `iss` as the stable user identifier

---

## Quick Reference: Key Numbers to Know

| Parameter | Typical Value | Notes |
|-----------|-------------|-------|
| Access token TTL | 5–60 minutes | Short is better for security |
| ID token TTL | 1 hour | Typically equal to session length |
| Refresh token TTL | 7–90 days | Application-specific |
| TOTP time step | 30 seconds | Standard (RFC 6238) |
| PKCE code_verifier length | 43–128 characters | URL-safe random |
| PKCE challenge method | S256 (SHA-256) | `plain` is deprecated |
| Session cookie inactivity timeout | 15–30 minutes | Enterprise security baseline |
| Session absolute maximum | 8–24 hours | Enterprise baseline |
| JWT minimum HS256 key | 256 bits (32 bytes) | NIST recommendation |
| RSA key minimum | 2048 bits | Use 4096 for long-lived certs |
| Clock skew tolerance | 60–300 seconds | Typically 60s for JWT validation |

---

[← Back to README](README.md)
