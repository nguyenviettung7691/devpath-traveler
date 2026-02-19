---
title: "Designing a Passwordless-First PWA Architecture"
seoTitle: "Designing a Passwordless-First PWA Architecture with WebAuthn and OIDC"
seoDescription: "Learn how to design a passwordless-first Progressive Web App architecture. Explore WebAuthn vs fallback decisions, server responsibilities, session manage."
datePublished: Sat Feb 14 2026 10:35:02 GMT+0000 (Coordinated Universal Time)
cuid: cmlm6jxr8000v02lb4ek1dadm
slug: designing-a-passwordless-first-pwa-architecture
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1770993591401/d8499dbf-f1ed-4b1e-8977-57a1e9221e5f.png
tags: progressive-web-apps, web-security, oidc, openid-connect, pkce, application-architecture, webauthn, session-management, passwordless-architecture, authentication-design

---

By this point in the series, we’ve established three things:

* Passwords are structurally fragile.
    
* WebAuthn provides phishing-resistant, device-bound authentication.
    
* OpenID Connect provides portable, federated identity.
    

Now comes the harder question:

How do you design a real Progressive Web App where all of this coexists cleanly?

Because authentication in a PWA is not just about verifying a user once.  
It’s about handling devices, sessions, fallbacks, offline behavior, and long-lived installs — without quietly reintroducing the weaknesses you just eliminated.

A passwordless-first architecture is not “always use WebAuthn.”  
It’s about deciding when to use it, when to fall back, and how to make those decisions explicit.

---

## Decision points: when to attempt WebAuthn vs fallback

A mature system does not guess. It decides.

There are several common entry scenarios:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1771062058030/1beeb680-fead-46a7-b929-d6ad8e669919.png align="center")

### 1\. Known returning user with registered credentials

If the server knows:

* this identity has WebAuthn credentials registered,
    
* and the browser supports WebAuthn,
    

then the system should attempt WebAuthn immediately.

This is the fast path:

* request challenge,
    
* call `navigator.credentials.get()`,
    
* verify assertion,
    
* issue session.
    

This path should feel frictionless.

### 2\. User has no registered credential

If the server sees:

* no WebAuthn credential on file,
    

it must not attempt WebAuthn.

Instead:

* redirect to federated login (OIDC),
    
* or use whatever bootstrap identity method exists.
    

After successful identity verification:

* offer credential registration.
    

Passwordless-first does not mean passwordless-only.

### 3\. WebAuthn attempt fails

Failure can mean:

* user cancels,
    
* authenticator unavailable,
    
* browser does not support feature,
    
* device lost,
    
* counter mismatch,
    
* challenge expired.
    

Your architecture must define what failure means.

Some failures allow retry.  
Some require fallback to OIDC.

The critical point:  
Fallback is not an afterthought. It is a planned branch.

If your system has no defined fallback path, it is not production-ready.

---

## Server responsibilities: the part nobody can skip

Passwordless pushes complexity into correctness.

Your server is responsible for:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1771062454217/a7b1bd4c-ba78-43c9-8ad5-e04fa34718ea.png align="center")

### Credential storage

For each credential, you must store:

* credential ID,
    
* public key,
    
* signature counter,
    
* user association,
    
* metadata (optional).
    

This storage must be:

* integrity-protected,
    
* scoped per user,
    
* revocable.
    

Storing only the public key is not enough.  
You must track counters to detect cloned authenticators.

### Challenge management

Every authentication attempt must include:

* a cryptographically random challenge,
    
* short expiration,
    
* binding to user and session state.
    

Challenges must:

* be unpredictable,
    
* not reusable,
    
* not long-lived.
    

If you reuse challenges, you reintroduce replay risk.

### Assertion verification

Server must:

* validate signature against stored public key,
    
* confirm challenge matches,
    
* check origin and RP ID,
    
* verify counter monotonicity.
    

This is where many implementations quietly break.

WebAuthn security is only as strong as the verification logic.

### Credential lifecycle management

Real systems need:

* credential revocation,
    
* device labeling,
    
* multi-device support,
    
* audit logs.
    

Without lifecycle management, passwordless becomes brittle.

---

## Session management in PWAs

Here is where things become interesting.

PWAs blur the line between:

* website,
    
* installed app,
    
* long-running client.
    

Session management must balance:

* security,
    
* persistence,
    
* user convenience.
    

After WebAuthn or OIDC authentication:

* you still need a session mechanism.
    

Common options:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1771062685370/7edbcffe-ebf2-4a0c-8eb1-f46afe5be007.png align="center")

### Server-side session (recommended)

* Store session ID in HTTP-only cookie.
    
* Session data lives on server.
    
* Token not accessible to JavaScript.
    

This minimizes XSS risk.

### Stateless JWT stored in cookie

* Signed JWT issued after authentication.
    
* Stored in secure, HTTP-only cookie.
    
* Verified per request.
    

Useful for distributed systems, but must:

* be short-lived,
    
* rotated carefully.
    

### Local storage (generally unsafe)

Storing tokens in localStorage:

* exposes them to XSS,
    
* encourages long-lived tokens,
    
* complicates revocation.
    

For PWAs especially, local storage can feel convenient.  
It is usually the wrong tradeoff.

---

## Secure token handling: cookies vs storage

The rule is simple:

<mark>If JavaScript can read it, malicious JavaScript can steal it.</mark>

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1771062893642/33011612-cdcb-48f6-83bb-cdd810a24bb3.png align="center")

HTTP-only cookies:

* are not accessible via JS,
    
* are automatically sent with requests,
    
* support SameSite protections,
    
* reduce XSS impact.
    

When properly configured:

* Secure,
    
* HTTP-only,
    
* SameSite=Lax or Strict,
    

<mark>cookies are safer for session tokens than browser storage.</mark>

The complexity lies in:

* CSRF protection,
    
* cross-origin flows,
    
* OIDC redirect handling.
    

These must be addressed deliberately — not avoided.

---

## Why offline PWAs complicate authentication

PWAs can:

* cache assets,
    
* run offline,
    
* queue background sync,
    
* appear installed and persistent.
    

Authentication systems were not originally designed for this.

Here’s the tension:

WebAuthn requires:

* server-issued challenge,
    
* live verification,
    
* session establishment.
    

Offline mode has:

* no server access,
    
* no challenge generation,
    
* no verification endpoint.
    

Therefore:

<mark>You cannot perform real authentication offline.</mark>

What you can do is:

* cache a previously authenticated session,
    
* gate access behind local checks,
    
* defer sensitive actions until online.
    

This creates design decisions:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1771064892450/27a5d5a2-4830-4fb7-ad8a-1def458520c7.png align="center")

### How long can an offline session persist?

Too short:

* poor UX.
    

Too long:

* increased risk on stolen devices.
    

### What actions are allowed offline?

Read-only?  
Cached data only?  
Queued writes?

Offline capability forces you to define trust boundaries explicitly.

---

## Designing the authentication state machine

A passwordless-first PWA architecture behaves like a state machine:

1. Unknown user → bootstrap via OIDC.
    
2. Known user with credential → attempt WebAuthn.
    
3. WebAuthn success → issue session.
    
4. WebAuthn failure → retry or fallback.
    
5. Credential lost → recover via OIDC.
    
6. Offline mode → limited local session access.
    

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1771065250525/df57fb58-bbfb-46ad-b1a1-45f45f463c24.png align="center")

Every branch must be:

* intentional,
    
* tested,
    
* observable.
    

If your authentication system is not drawn as a flowchart, you probably haven’t finished designing it.

---

## Passwordless-first means opinionated design

It means:

* WebAuthn is default.
    
* Federation is structured fallback.
    
* Sessions are server-controlled.
    
* Tokens are protected from JavaScript.
    
* Recovery is first-class.
    
* Offline mode is constrained deliberately.
    

It does not mean:

* removing identity providers,
    
* removing server state,
    
* trusting devices blindly,
    
* or assuming biometrics solve lifecycle problems.
    

Architecture is about deciding where trust lives.

Passwordless-first architectures shift trust:

* away from shared secrets,
    
* toward device-bound credentials,
    
* while preserving federated identity for continuity.
    

In the next article, we’ll explore how UX decisions — error messages, prompts, retries — shape security outcomes more than cryptography alone.

Because even the strongest architecture must survive human behavior.

---

## ☰ Series Navigation

### Core Series

* [Introduction](https://devpath-traveler.nguyenviettung.id.vn/introduction-to-passwordless-modern-authentication-patterns-for-pwas)
    
* [**Article 1 — Authentication is Not Login**](https://devpath-traveler.nguyenviettung.id.vn/authentication-is-not-login)
    
* [**Article 2 — What “Passwordless” Actually Means**](https://devpath-traveler.nguyenviettung.id.vn/what-passwordless-actually-means)
    
* [**Article 3 — WebAuthn & FIDO2, Explained Without the Spec**](https://devpath-traveler.nguyenviettung.id.vn/webauthn-and-fido2-explained-without-the-spec)
    
* [**Article 4 — OpenID Connect as the Glue**](https://devpath-traveler.nguyenviettung.id.vn/openid-connect-as-the-glue)
    
* → **Article 5 — Designing a Passwordless-First PWA Architecture**
    
* [**Article 6 — UX and Failure Are Part of the Security Model**](https://devpath-traveler.nguyenviettung.id.vn/ux-and-failure-are-part-of-the-security-model)
    
* [**Article 7 — A Real Passwordless PWA Flow (Architecture Walkthrough)**](https://devpath-traveler.nguyenviettung.id.vn/passwordless-pwa-flow-architecture-walkthrough)
    
* [**Article 8 — Implementing WebAuthn in Practice**](https://devpath-traveler.nguyenviettung.id.vn/implementing-webauthn-in-practice)
    
* [**Article 9 — Integrating OIDC (Feide) as Fallback and Recovery**](https://devpath-traveler.nguyenviettung.id.vn/integrating-oidc-feide-as-fallback-and-recovery)
    
* [**Article 10 — What Worked, What Didn’t, What I’d Change**](https://devpath-traveler.nguyenviettung.id.vn/passwordless-what-worked-what-didnt-what-id-change)
    

### Optional Extras

* [**Why Passwordless Alone Is Not an Identity Strategy**](https://devpath-traveler.nguyenviettung.id.vn/why-passwordless-alone-is-not-an-identity-strategy)
    
* [**How Browser UX Shapes Security More Than Cryptography**](https://devpath-traveler.nguyenviettung.id.vn/how-browser-ux-shapes-security-more-than-cryptography)